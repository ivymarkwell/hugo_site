+++
title = 'Implementing Permission Based Authorization in Absinthe With Custom Graphql Directives'
draft = false
date = 2025-04-11
+++

<!--more-->

<figure style="text-align: center;">
    <img src="https://www.outcyders.net/images/quizzes/22/question10.jpg" style="max-width: 100%; height: auto;" />
    <figcaption style="margin-top: 0.5em;">Who's That User?</figcaption>
</figure>

Quick disclaimer! Writing and learning about this topic was a challenge so this article will be a heavier read. However, I expect if you’ve found yourself here that you are either familiar with Absinthe or have an existing application with a similar authorization problem. With that being said, let’s get into it!

### The Problem

Imagine the following query that’s exposed in GraphQL:

```elixir
query getUser($id: ID!) {  
    getUser(id: $id) {  
        id  
        name  
        address  
    }  
}
```

There are many complex actions this can be broken into. The naive question in a REST system would be: does the user making the request have permission to `read` the exposed `user` object?

In GraphQL we can get much more granular than just basic CRUD permissions. Does the user have access to `read` the `user` object? Does the user have access to `read` the exposed `address` field under the `user` object? What about any of the arguments being passed to our query? Does the user have access to pass the argument `id` in order to query a specific user?

I decided to [define my own custom Absinthe directive](https://hexdocs.pm/absinthe/Absinthe.Schema.Notation.html#directive/3-defining-a-directive). A directive would allow me to decorate my GraphQL schemas and grant control over user access as the schema resolves the requested fields.

If you aren’t familiar with GraphQL directives but you are using Absinthe, you likely have already worked with directives and haven’t even realized it! GraphQL offers built in directives that are used in Absinthe. For example:

```elixir
field :old_field, :string do  
    deprecate "Please use :new_field"  
end
```

`deprecate` is a [built-in directive that Absinthe](https://github.com/absinthe-graphql/absinthe/blob/v1.7.8/lib/absinthe/schema/prototype/notation.ex) uses for you to mark that a field should no longer be used directly in the schema.

### Defining our directive

In my case, directives would allow me to get very granular about the permissions a user needed in order to access a field. Let’s go back to our query example from before. Let’s say we wanted to define permissions around reading a user. However, it’s not that simple. Only certain users should be allowed to query a user, read a user, their name and address, and we want specific permissions to reflect that. So, the goal of this post is to define the following permissions: `query_user`, `read_user`, `read_user_name`, and `read_user_address`.

My Absinthe schema looks something like this:

```elixir
defmodule MyApp.Schema do
  use Absinthe.Schema
  
  object :user do
    field(:id, non_null(:id))
    field(:name, :string)
    field(:address, :string)
  end
  
  query do
    field :get_user, :user do
      arg(:id, non_null(:id))
      
      resolve(fn _, %{id: id}, _ ->
        {:ok, UserResolver.get_user(id)}
      end)
    end
  end
end
```

So I defined a directive that would allow me to pass permissions onto the GraphQL fields and objects themselves. We do this by defining our own prototype schema and then exposing it as our [@prototype_schema](https://hexdocs.pm/absinthe/Absinthe.Schema.Prototype.html) in our Absinthe Schema file.

```elixir
defmodule MyApp.SchemaPrototype do  
    @moduledoc """  
    Defines our custom schema directives.  
    """  
    
    use Absinthe.Schema.Prototype  
    
    @doc """  
    Authorization directive that allows us to pass a list of permissions to  
    GraphQL fields and objects.  
    """  
    directive :auth do  
        arg(:permissions, list_of(:string))  
    
        on([:field_definition, :object])  
    
        expand(fn %{permissions: permissions}, node ->  
            %{node | __private__: Keyword.put(node.__private__, :permissions, permissions)}  
        end)  
    end  
end
```

This creates a custom Absinthe directive `:auth` that allows us to pass a list of string permissions as an argument on any GraphQL fields or objects we define. Then, we can expose our `prototype_schema` to our Absinthe schema file like this:

```elixir
defmodule MyApp.Schema do  
    use Absinthe.Schema  
    
    @prototype_schema MyApp.SchemaPrototype  
    
    # query and mutation definitions...  
end
```

### Understanding directives

If you’re wanting to understand more about the Absinthe `directive` itself, you’re going to have to read some Absinthe source code. You can follow along if you look at our `:auth` directive. First, we are using the [use Absinthe.Schema.Prototype](https://github.com/absinthe-graphql/absinthe/blob/main/lib/absinthe/schema/prototype.ex) to access a macro Absinthe has defined for schema prototypes. When we add our directive all we are basically doing is importing the [already defined directives code](https://github.com/absinthe-graphql/absinthe/blob/main/lib/absinthe/schema/prototype/notation.ex) and adding another one.

Then in the directive itself we are defining [the expand](https://hexdocs.pm/elixir/1.12/Macro.html#expand/2) to [alter the node that Absinthe is going to pass to us](https://github.com/absinthe-graphql/absinthe/blob/main/lib/absinthe/phase/document/directives.ex) when it applies our directive in the [GraphQL pipeline](https://github.com/absinthe-graphql/absinthe/blob/main/lib/absinthe/pipeline.ex#L112). Now we can focus on what we have access to alter in the node itself. If you look at the node Absinthe gives us, it has a type [Blueprint.node_t()](https://github.com/absinthe-graphql/absinthe/blob/main/lib/absinthe/blueprint.ex#L55). There you can find the following definitions for our node:

```elixir
@type node_t ::  
        t()  
        | Blueprint.Directive.t()          
        | Blueprint.Document.t()  
        | Blueprint.Schema.t()          
        | Blueprint.Input.t()  
        | Blueprint.TypeReference.t()
```

Depending on what your directive is allowed `on` will determine what type you’re accessing in your `expand/2` function. Since our directive is only allowed on `field_definition` and `object` we are using the [object type definition](https://github.com/absinthe-graphql/absinthe/blob/main/lib/absinthe/blueprint/schema/object_type_definition.ex) and the [field definition](https://github.com/absinthe-graphql/absinthe/blob/main/lib/absinthe/blueprint/schema/field_definition.ex). Either way, our nodes have access to a few common fields we can alter so we can pass information down to our middleware later; `__private__` being one of them. We are guaranteed this is for [internal use only](https://hexdocs.pm/absinthe/search.html?q=__private__) and shouldn’t be altered by Absinthe itself. So we’re able to add a string of `permissions` on these nodes using the `__private__` field that we will handle later in our middleware when our fields are getting resolved.

TLDR; We add a directive that has access to a `node_t()` . `node_t()` has access to a `__private__` field that we can use to pass down information into our middleware later when GraphQL is resolving our fields.

😅Anyways! Now that our detour is over, let’s get into passing down our permissions so we can actually use them and start writing our authorization logic.

### Using the directive via middleware

If you were paying attention earlier, what’s going to happen after Absinthe applies our directive is that the node we altered earlier is going to be passed down to our [middleware](https://hexdocs.pm/absinthe/Absinthe.Middleware.html). This is because as Absinthe attempts to resolve the fields the user is accessing, we are passing around an [`Absinthe.Resolution`](https://hexdocs.pm/absinthe/Absinthe.Resolution.html#t:t/0) struct. If you look closely at the type for `Resolution` it has a `definition` field that is our node we altered earlier 👏.

Now that our directive is defined we can update our GraphQL fields from earlier to use our `:auth` directive with all the permissions we needed from before!

```elixir
defmodule MyApp.Schema do  
    use Absinthe.Schema  
        
    object :user do  
        directive(:auth, permissions: ["read_user"])  
    
        field(:id, non_null(:id))  
        field(:name, :string, directives: [auth: [permissions: ["read_user_name"]]])  
        field(:address, :string, directives: [auth: [permissions: ["read_user_address"]]])  
    end  
        
    query do  
        field :get_user, :user, directives: [auth: [permissions: ["query_user"]]] do  
            arg(:id, non_null(:id))  
        
            resolve(fn _, %{id: id}, _ ->  
                {:ok, UserResolver.get_user(id)}  
            end)  
        end  
    end  
end
```

Then we are going to create our own custom middleware `MyApp.Middleware.Authorization` to access our permissions and restrict user access. We can update our `MyApp.Schema` file again to use that middleware:

```elixir
defmodule MyApp.Schema do  
    use Absinthe.Schema  
    
    @prototype_schema MyApp.SchemaPrototype  
    
    def middleware(middleware, _field, _object) do  
        [MyApp.Middleware.Authorization | middleware]  
    end  
    
    # query and mutation definitions...  
end
```

Finally… our authorization logic. Let’s write our middleware that will be in charge of allowing users to gain access to the GraphQL fields.


```elixir
defmodule MyApp.Middleware.Authorization do
  @moduledoc """
  Authorization middleware that verifies users have the necessary permissions to
  access specific GraphQL fields and objects.
  """

  @behaviour Absinthe.Middleware

  alias Absinthe.Blueprint
  alias Absinthe.Type.Object

  def call(resolution, _config) do
    user = resolution.context.current_user
    user_permissions = get_in(user.permissions)

    # unauthorized_permissions = // Logic to check for missing permissions...

    if unauthorized_permissions == [] do
      resolution
    else
      Absinthe.Resolution.put_result(
        resolution,
        {:error,
         "Unauthorized to perform the following action(s): #{Enum.join(unauthorized_permissions, ", ")}"}
      )
    end
  end
end
```

There’s not a lot going on here yet but you can assume I have my Absinthe project setup to pass around the correct `current_user` in my resolution context with an imaginary permissions field that contains a list of strings that defines the permissions that user has access to.

This middleware will check for any permissions that were passed on `object` or `field_definition` types and return an error to our resolution if we found the user doesn’t have the right permissions. This will happen as Absinthe resolves each of the fields the user is accessing. Let’s go back to the first query I showed to see how this works.

As Absinthe resolves the fields in the `getUser` query, the middleware is triggered up to four times. The middleware can be triggered once for the top level query itself, `:get_user`, and once for each field being queried, i.e. `id`, `name` and `address`. This matters because depending on what’s currently being resolved will determine what we have access to in our middleware at the time.

The first thing that will resolve and our middleware will check is the top level query itself, `:get_user` which has the `query_user` permission defined using our directive.

Our [`resolution.definition`](https://hexdocs.pm/absinthe/Absinthe.Resolution.html#module-contents) that is getting passed into our middleware will look something like this:

```elixir
%Absinthe.Blueprint.Document.Field{  
  name: "getUser",  
  arguments: [  
    %Absinthe.Blueprint.Input.Argument{  
      name: "id",  
      schema_node: %Absinthe.Type.Argument{  
        identifier: :id,  
        name: "id",  
        __private__: [],  
        ...remaining_fields  
      },  
    }  
  ],  
  schema_node: %Absinthe.Type.Field{  
    identifier: :get_user,  
    name: "get_user",  
    __private__: [permissions: ["query_user"]],  
    ...remaining_fields  
  },  
  ...remaining_fields  
}
```

It’s important to note that while we are only looking at a query that does not have any permissions on its arguments, that we could have added a permission to an argument for the field, like `:id`. This matters a lot more when we start considering mutations that could be passing arguments we don’t want to give a user access to. Imagine we had an `update_user` mutation and we wanted a permission allowing only certain users to update the `:address`. We could add a directive on the argument and it would be in our resolution definition like the above example. We will come back to this in a minute.

For now, let’s focus on getting our `query_user` permission and checking that the user has it!

```elixir
defp field_permissions_unauthorized(resolution, user_permissions) do  
  resolution.definition.schema_node.__private__  
  |> Keyword.get(:permissions, [])  |> Enum.reject(fn permission ->  
    has_permission?(permission, user_permissions)  
  end)  
end  
  
def has_permission?(_, nil), do: false  
  
def has_permission?(permission, user_permissions) do  
  Enum.find_value(user_permissions, false, fn user_permission ->  
    user_permission == permission  
  end)  
end
```

Hopefully the above code is really straight forward to reason about. Using the snippet of our `resolution.definition` from above, in order to get the permissions off the top level query (or mutation) we access the `schema_node.__private__` and pluck off any permissions we find! Then, we have a helper function `has_permission?/2` that is going to be reused to determine if our current user has the permission we parsed out from our node. In this example, we would find the permission `query_user` defined and check to verify that our current user has that permission. If they didn’t, we would return that permission back from our function so we can use it in our error message from before.

Now that we have our first permission verified, let’s circle back to our arguments’ permissions discussion.

```elixir
defp argument_permissions_unauthorized(resolution, user_permissions) do  
  resolution.definition.arguments  
  |> Blueprint.find(fn  
    %Absinthe.Blueprint.Input.Field{  
      schema_node: %Absinthe.Type.Field{__private__: [permissions: permissions]}  
    } ->  
      Enum.any?(permissions, fn permission ->  
        not has_permission?(permission, provider_permissions)  
      end)  
  
    _ ->  
      false  
  end)  
  |> case do  
    %Absinthe.Blueprint.Input.Field{  
      schema_node: %Absinthe.Type.Field{__private__: [permissions: permissions]}  
    } ->  
      Enum.reject(permissions, fn permission ->  
        has_permission?(permission, provider_permissions)  
      end)  
  
    _ ->  
      []  
  end  
end
```

You can see this function is a little bit more complex than when we were just accessing the permissions on the query itself. Here we are accessing the `arguments` passed to our query and using Absinthe’s [Blueprint.find/2](https://hexdocs.pm/absinthe/Absinthe.Blueprint.html#find/2) to find any arguments that have permissions defined on them. We do this by pattern matching on the `schema_node` for our `Blueprint.Input.Field` and checking if the field for the argument has any permissions defined on it. Then if we find a field that has permissions defined on it, we verify if the current user does not have any of those permissions. We do this because `Blueprint.find/2` will return the first field it finds that returns true. So we want to return any field that does not have the necessary permissions. After our `Blueprint.find/2` returns a field that is missing permissions, we then iterate through it one more time so that we only return the permissions that were missing from our function.

This does mean that if there were multiple arguments with permissions defined the user didn’t have access to, there would only be one permission returned in our error. However, this is enough to make sure our endpoint is secure. Regardless of whether a user does not have access to pass one argument or many, since the user does not have access to something being resolved on the top level query, the rest of the query would halt. The same would apply to any mutations that had arguments. If the user doesn’t have access to an argument they are trying to mutate, the mutation will fail before an update can happen.

Also, you might be thinking that I didn’t define my directive to be used on `argument_definition` types and you would be correct! Currently my directive is setup so the only arguments that would be passed with permissions would be from an `input_object`. Any fields with permissions under an `input_object` would be caught the same way in my `argument_permissions_unauthorized/2` function. We could iterate on this later so we can pass directives directly on the `arg` itself but this functionality is enough for me for now.

Moving on, the next thing that gets resolved in our query are the fields of the object that are getting accessed. The first field to get resolved is `id`. Let’s take a look at what our `resolution.definition` looks like when we are resolving `id`.

```elixir
%Absinthe.Blueprint.Document.Field{  
  name: "id",  
  schema_node: %Absinthe.Type.Field{  
    identifier: :id,  
    name: "id",  
    __private__: [],  
    ...remaining_fields  
  },  
  parent_type: %Absinthe.Type.Object{  
    identifier: :user  
    name: "User",  
    fields: ...userFields,  
    __private__: [__absinthe_referenced__: true, permissions: ["read_user"]],  
    ...remaining_fields  
  }  
}
```

Even though we don’t have any directives defined on the `id` field itself, there are 2 different things happening here that we could care about. The first is the permission that _could_ have been on `id`. For example, when our `name` and `address` fields get resolved next they will both have permissions here like `read_user_name` and `read_user_address`. Any permissions defined on the fields themselves will get caught and verified in our `field_permissions_unauthorized/2` function so we don’t need to do anything else here!

However, the second is that our field might have a parent object with restrictive permissions on it. Here you can see we have access to the `parent_type` which defines any permissions on the parent object of the field. In this case `id` has the `read_user` permission defined on it’s `user` parent object. So let’s deal with that.

```elixir
defp object_permissions_unauthorized(resolution, provider_permissions) do  
    resolution.definition  
    |> Blueprint.find(fn 
        %Absinthe.Blueprint.Document.Field{
            parent_type: %Object{__private__: private}
        } ->  
            Keyword.has_key?(private, :permissions)  
    
        _ ->  
            false  
    end)  
    |> case do  
        %Absinthe.Blueprint.Document.Field{
            parent_type: %Object{__private__: private}
        } ->  
            permissions = Keyword.get(private, :permissions, [])  
    
            Enum.reject(permissions, fn permission ->  
                has_permission?(permission, provider_permissions)  
            end)  
    
        _ ->  
            []  
    end  
end
```

This is almost exactly the same as when we parsed out the argument’s permissions. The only difference here is that we are now looking at the `resolution.definition` to find any `Blueprint.Document.Field` with a `parent_type` that has permissions. In this scenario just like before when we were finding any arguments the user did not have permission to access, this function would return the `read_user` permission if the user did not have that permission.

…And that’s it! When looking for permissions on queries or mutations there are only a few things we care about:

*   If there are any arguments with permissions defined (none from our query example)
*   Whether the field itself has a permission defined (`query_user`, `read_user_name`, `read_user_address`)
*   If any of the fields being accessed have a parent object with permissions defined (`read_user`)

Now let’s put this all back together in our middleware call:

```elixir
def call(resolution, _config) do  
    user = resolution.context.current_user  
    user_permissions = get_in(user.permissions)  
    
    # Unauthorized permissions found from arguments that were passed as input_object GraphQL fields  
    unauthorized_argument_permissions =  
        argument_permissions_unauthorized(resolution, user_permissions)  
    
    # Unathorized permissions found from defined GraphQL fields  
    unauthorized_field_permissions =  
        field_permissions_unauthorized(resolution, user_permissions)  
    
    # Unathorized permissions found from defined GraphQL objects  
    unauthorized_object_permissions =  
        object_permissions_unauthorized(resolution, user_permissions)  
    
    unauthorized_permissions =  
        unauthorized_argument_permissions ++  
        unauthorized_field_permissions ++ unauthorized_object_permissions  
    
    if unauthorized_permissions == [] do  
        resolution  
    else  
        Absinthe.Resolution.put_result(  
            resolution,  
            {:error, "Unauthorized to perform the following action(s): #{Enum.join(unauthorized_permissions, ", ")}"}  
        )  
    end  
end
```

Our middleware will now verify the user has access to all necessary permissions defined from our directives and return an error message with any of the permissions it found that the user was unauthorized for.

### Conclusion

There are lots of ways you can enforce permissions and this way might not feel like the most efficient for your use case. You could easily handle permissions in your resolver layer where you can check any necessary permissions at one time and return the error you want. There are also loads of libraries that can help you do this.

Another problem I want to call out is that we can’t always return all the errors at once the way this is currently written. Ideally we want to show the user one error message that includes all the permissions they are lacking so they can take action. Going back to our `arguments_permissions_unauthorized/2` function, our error when we are accessing unauthorized arguments will only ever contain the first argument it found that was lacking a permission.

Lastly, depending on where the resolution fails will determine what action the user succeeds in taking. For example, consider a mutation like `update_user` that requires specific update permissions. If the user has the `update_user` permission, the mutation itself will succeed even if the user lacks read access to the returned user object. In this scenario, the mutation executes but the requested fields in the response won’t be resolved, and the user will receive an unauthorized error for those fields. This is because of the order in which GraphQL resolves operations. It first resolves the mutation itself and runs the resolver function. Then after the resolver function has run for the mutation, the queried fields will attempt to resolve. It’s at this layer when the queried fields are resolving that the middleware would error, even though the mutation itself succeeded. This seems reasonable but it’s just something to note if you expect the mutation to fail even if the user has update access but not read access.

However all of those things being said, this directive is very powerful. It allows us to get very granular with how we define permissions without the engineer having to do additional leg work. All our permissions can be enforced through a one line change that we add directly to the GraphQL schema itself. It’s easy to write, easy to review and can give product owners the granularity they want for security access.

I hope this gives you some deeper insight on how you can utilize Absinthe directives in your system and as always, thanks for reading!
