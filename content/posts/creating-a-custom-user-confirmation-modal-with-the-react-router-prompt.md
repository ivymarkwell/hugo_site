+++
title = 'Creating a Custom User Confirmation Modal With the React Router Prompt'
draft = false
date =  2019-09-16
+++

<!--more-->

<link href="/styles/common.css" rel="stylesheet">

<figure style="text-align: center;">
    <img src='https://cdn-images-1.medium.com/max/2400/1*lYSWM1q3d5jGuZxGDK6iwQ.jpeg' style="max-width: 100%; height: auto;" />
    <figcaption style="margin-top: 0.5em;">A person up in arms about wanting custom modals in their Prompt component</figcaption>
</figure>

This is a short walkthrough on how to use React-Router’s v5 `Prompt` component
to replace the built in user confirmation window object with your own custom
`UserConfirmation` modal.

There is another really [good
article](https://medium.com/@michaelchan_13570/using-react-router-v4-prompt-with-custom-modal-component-ca839f5faf39)
about blocking the navigation to prevent the default modal so that we can render
our own custom component instead. They briefly mention an alternative method
that involves utilizing `BrowserRouter` and the `getUserConfirmation` prop,
which is exactly the solution this post covers!

### Briefly Behind the Scenes

Without going into too much detail, essentially you can trace the `Prompt`
component all the way through `RouterContext`, `Router` and `BrowserRouter`.

`BrowserRouter` is where the magic begins. Specifically where the history prop
is created on [line
11](https://github.com/ReactTraining/react-router/blob/v5.0.0/packages/react-router-dom/modules/BrowserRouter.js).
`react-router` uses `createBrowserHistory` to create our rendered history prop
as an extension of the `window.history` that’s available in any browser. If you
look in `createBrowserHistory`, you’ll also see the final layer of our
confirmation window which comes from the initialization of `getUserConfirmation`
from `getConfirmation` defined in `DomUtils`. This is where the default modal
gets defined as a window object.

```javascript
export function getConfirmation(message, callback) {
   callback(window.confirm(message));
}
```

### Creating our Custom UserConfirmation modal

Since we are opting out of the window object and replacing it with our own
custom component, we not only need to render a nicer modal but we also need to
include all of the functionality for canceling and confirming the callback.

Remember that `getUserConfirmation` is initialized in `BrowserRouter` and is
what determines what our confirmation modal displays. So wherever you’re
initializing your application with `BrowserRouter` is where we’re going to pass
our custom component into the `getUserConfirmation` prop.


And you’re ready to go! You’re now overwriting the default window component and
rendering your own custom `UserConfirmation` (even if it might not exist yet).

So let’s make it happen!

```javascript
import React from "react";
import ReactDOM from "react-dom";
import { Dialog } from "evergreen-ui";

const UserConfirmation = (message, callback) => {
  const container = document.createElement("div");
  container.setAttribute("custom-confirmation-navigation", "");
  document.body.appendChild(container);

  const closeModal = (callbackState) => {
    ReactDOM.unmountComponentAtNode(container);
    callback(callbackState);
  };

  ReactDOM.render(
    <Dialog
      cancelLabel="Cancel"
      confirmLabel="Confirm"
      isShown={true}
      onCancel={() => closeModal(false)}
      onConfirm={() => closeModal(true)}
      title="Warning"
    >
      {message}
    </Dialog>,
    container
  );
};

export default UserConfirmation;
```

There are a few key things that we needed to happen in our custom component
besides the UI. Since `BrowserRouter` and our custom component is rendered a
layer above where our normal application is rendered, we start by initializing a
container that we use to directly render the modal. Then we actually define what
the container is rendering.

This is where we create the functionality of closing the modal through either a
cancel or confirm action. In our `closeModal` function we unmount our manually
rendered component and either push the user to the next location or stay at the
location whether the `Prompt` was triggered. Fortunately, the callback function
that is passed from `Prompt` makes sending the user to the next location or
staying on the page really simple for us. All we need to pass `callback` is a
boolean that tells us whether the user wants to proceed with going to their next
location or doesn’t. So, when the user clicks “Cancel” we can call
`callback(false)` and likewise when they click “OK” we call `callback(true)`.

And that’s it! You’re done.

### But I want more than a message and a callback

This is the part where most people would choose another solution because they
want additional customization of the modal that go outside the two props that
are given to us.

Sadly, that is a legitimate concern. We can attempt to alleviate this a little
by manipulating our `message` prop. Our `Prompt` component will only accept a
string into `message`. So what we can do here is pass in a stringified JSON
object with key-value pairs of things we’d like to have additional control over
and then parse them out later.

```javascript
<Prompt
   when={when}
   message={
      JSON.stringify(
         `{
           "confirmText": "Continue",
           "messageText": "It looks like you might have some unsaved
                       changes! Are you sure you want to continue?",
           "cancelText": "Do not continue"
          }`
      )
   }
/>
```

Then in our `UserConfirmation` modal instead of using `message` directly to
display text we need to create a `textObj` that we use to reference our
customized text keys.

```javascript
const textObj = JSON.parse(message);

<Dialog
   cancelLabel={textObj.cancelText}
   confirmLabel={textObj.confirmText}
   isShown={true}
   onCancel={() => closeModal(false)}
   onConfirm={() => closeModal(true)}
   title="Warning"
>
   {textObj.messageText}
</Dialog>
```

Now we’re at least going beyond a one-dimensional modal that only allows us to
customize the prompt message displayed to the user. Similarly, you could use the
`message` prop to conditionally determine what to render. If we hadn’t passed for
example a `cancelText`, then in our Evergreen `Dialog` we could choose to set
`hasCancel` to false so we’re only rendering a confirm button and not a close
icon.

*****

If you have any questions or feedback please leave a comment below! You can
follow me on [Medium](https://medium.com/@ivymarkwell) and
[Github](https://github.com/ivymarkwell) to see more fun stuff I’m working on.
Special thanks to [Joerg Baier](https://medium.com/u/f27d2add6165) for helping
me work through and understand this solution and many of my other coworkers at
[Rabbet](https://rabbet.com/) for giving me helpful feedback on this article.