+++
title = "Standardizing data migrations in Elixir with Monarch"
draft = false
date = 2026-02-02
+++

<!--more-->

<link href="/styles/common.css" rel="stylesheet">

<figure style="text-align: center;">
    <img src="https://cdn-images-1.medium.com/max/1600/1*NgT5kO6Xg_qeqFn9xosJng.jpeg" style="max-width: 100%; height: auto;" />
    <figcaption style="margin-top: 0.5em;">Real image of who's waiting for you to finish verifying your data migration</figcaption>
</figure>

If I had to guess, you're probably using at least three different approaches to do data migrations in production today:

* You put data changes directly into an Ecto migration
* You write one-off scripts, iex into production, and run them manually
* You write Oban workers and enqueue them after a deploy

At least, that's been the case for me in my Elixir positions. And while this works, there's never been a standard process for engineers to follow.

The other issue I've run into is that these don't scale particularly well. Ecto migrations block deploys, while one-off scripts and Oban jobs are entirely manual—easy to forget, hard to track, and tricky to rerun safely.

These are the issues that led my coworker, [Chris Bailey](https://www.linkedin.com/in/yiiniiniin/), and I to create [Monarch](https://github.com/Vetspire-VSP/monarch). Monarch is an Oban powered process that automatically runs data migrations when your application starts.

### How Monarch Works

Monarch provides a behaviour that defines the standard interface for your data migration jobs. We call any module that implements this behaviour a Monarch job.

Once you have your own Monarch jobs, Monarch uses a single entry point to discover and run them. That entry point is `Monarch.run/2`. When you call `Monarch.run/2`, Monarch:

1. Scans your application for modules that implement the Monarch behaviour
2. Checks whether each job has already been run or is currently running
3. Automatically enqueues any jobs that still need to be processed using Oban

No manual steps. No remembering. Your jobs run automatically 🎉

### What Is a Monarch Job?

A Monarch job is simply a module that implements the Monarch behaviour. That behaviour is defined by two core functions:

#### `query/0`

Returns a list of records that still need to be updated. This function must be idempotent and should always return records that still need to be processed.

#### `update/1`

Takes the records returned by `query/0` and performs the necessary updates.

Monarch calls these functions recursively. It runs `query/0`, passes the results to `update/1`, and then reruns `query/0` to check if there are more records remaining. This continues until there are no more records left to be updated.

### Starting Monarch

Since Monarch is built on top of Oban, your application must already have Oban installed and be configured with at least one queue.

Once that's done, start Monarch by calling:

```elixir
Monarch.run(Oban, <my_queue>)
```

* `Oban` is the Oban instance you want to use
* `<my_queue>` is the default queue Monarch will use for data migrations

You can call this run function from a release task or directly in `application.ex`—anywhere that ensures it runs automatically after a deploy.

### Writing Your First Monarch Job

Once you're using the `Monarch.run/2` function, your application is ready to start looking for Monarch jobs! You just need to write one.

Monarch includes a mix task to make getting started easy:

```bash
mix monarch --monarch-path myapp/lib/myapp/workers/monarch my_monarch_module
```

This generates a boilerplate module with the required callbacks:

```elixir
defmodule MyApp.Workers.Monarch.MyFirstMonarchJob do
  @behaviour Monarch

  @impl Monarch
  def skip, do: false

  @impl Monarch
  def scheduled_at, do: DateTime.utc_now()

  @impl Monarch
  def query do
    # Return a chunk of records that still need to be updated
    []
  end

  @impl Monarch
  def update(_records) do
    # Process the returned records
  end
end
```

From now on, you only need to focus on your data migration logic, which is all you wanted to do anyway.

### Simple Example: Backfilling Usernames

Here's a real-world example of using Monarch to backfill a newly added username field on an existing users table.

Assume we've just introduced this username column and want to make it a required field. Before we can enforce that constraint, we need to backfill existing users to have a default value.

Our business requirements look like this:

* Every user must have a username
* Usernames must be unique
* The default username should be derived from the user's first and last name
* If multiple users share the same name, we append a number to ensure they are unique

Below is what our Monarch job could look like:

```elixir
defmodule MyApp.Workers.Monarch.Backfill do
  @behaviour Monarch
  import Ecto.Query
  alias MyApp.Repo
  alias MyApp.Users.User

  @impl Monarch
  def skip, do: false

  @impl Monarch
  def scheduled_at, do: DateTime.utc_now()

  @impl Monarch
  def query do
    from(u in User,
      where: is_nil(u.username),
      limit: 100
    )
    |> Repo.all()
  end

  @impl Monarch
  def update(users) when is_list(users) do
    Enum.each(users, fn user ->
      base_username =
        [user.first_name, user.last_name]
        |> Enum.join("")
        |> String.downcase()
      username = unique_username(base_username)
      user
      |> User.changeset(%{username: username})
      |> Repo.update!()
    end)
  end

  defp unique_username(base_username, attempt \\ 0) do
    username = if attempt == 0, do: base_username, else: "#{base_username}#{attempt}"
    case Repo.get_by(User, username: username) do
      nil -> username
      _ -> unique_username(base_username, attempt + 1)
    end
  end
end
```

Because `query/0` only returns users without usernames, this job is safe to retry. Once all users are updated, Monarch marks the job as completed.

Please note, I don't want to focus on the logic of this migration example itself but rather give you ideas on the kind of things you could do with Monarch.

### So why Monarch?

We've been using Monarch in production for over a year now and I've found that it helps me stay focused on actually writing my data migration and not writing the thing that writes my migration.

* **Standardization**
Every data migration could follow the same pattern across the codebase.

* **Automatic execution**
Migrations run automatically after deploys in every single environment and in every instance of production. I write my update one time.

* **Secretly Oban**
Retries, monitoring, and job tracking come for free using tooling you already have.

* **Idempotent**
Jobs are always safe to rerun, with no risk of duplicate updates.

However, Monarch is currently at a 0.3.0 release and has a lot more improvements to come.
### Things to Consider

There are a few important considerations worth calling out before adopting Monarch in your own application.

**Race Conditions**

By default, Monarch enqueues all of its jobs into a single default Oban queue you define. If multiple Monarch jobs run concurrently and update the same table, this can cause race conditions.

For example, imagine two Monarch jobs running at the same time, both operating on the users table we updated from before but updating on different fields. If those jobs overlap, they may attempt to update the same records concurrently and could cause one job to overwrite the other. This can cause a Monarch job to believe it has been completed successfully even though its changes were later overwritten and undone.

To address this, Monarch allows individual jobs to define a custom queue via an optional callback in the behaviour. For tables that are frequently updated, I'd suggest you create a dedicated Oban queue with a `global_limit: 1` and assign it to those relevant Monarch jobs. Doing this ensures that jobs that could conflict are run one at a time and that any unrelated Monarch jobs can still continue running concurrently on the default queue.

**Jobs that never finish**

Monarch detects jobs that repeatedly return the same data without making progress and discards them. However, the job itself is still not considered completed. This will cause a job to be reinserted on every deployment until it's fixed or removed from the codebase.

For now, the simplest solution is to either delete the Monarch job or fix the problem so that it can be completed. Handling these "stuck" jobs is something we plan to address in the future.

**Rolling Back**

Currently, rolling back a Monarch job is manual. Again, I hope to add support for this soon but hopefully this is not an issue you are running into often. However, it is important to note that either the job cannot be rolled back or you would need to be prepared to roll it back manually.

*****

Thanks for reading! This was meant to be a short, fun article highlighting a library I helped build and how it could be useful for you. If you're looking for more information about Monarch, I'd highly suggest you go look at our library. It is open source! 🙂
