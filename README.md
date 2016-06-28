# FrontEndAuth
React/Flux Implementation of Front End Auth for Single Page Loading Apps

## Phase 9: Front-End User Authentication

In this phase, we are going to implement front-end user sign-up and login.
Goodbye Rails views; hello, single-page app! **Read through the instructions for
the entire phase before building anything.** This will give you the context to
understand each individual step.

**Our authentication pattern must:**
  * sign up new users
  * know who's logged in
  * log users in
  * log them out
  * restrict access to certain routes based on whether someone is logged in

### 1. Build Your Backend.

Read the instructions below, and then create an API with the following
endpoints:

  * [POST] api/users: "users#create" (signup),
  * [POST] api/session: "session#create" (login),
  * [DELETE] api/session: "session#destroy" (logout)

**Create a `User` model, `UsersController`, and `SessionsController`.** Follow
the basic pattern you used during the [Rails curriculum][rails], with some key
differences:

* **Namespace**: Your controllers should live under an `Api` namespace.
* **Response Format**: render JSON formatted responses by default.
* **Views**: You'll want an **`api/users/show.json.jbuilder`**, which you can use for multiple controller actions. This should delegate to partial **`api/users/_user.json.jbuilder`**, which we'll use later.
* **`Sessions#destroy`**: render an empty `{}` upon successful logout.
  * Render a `404` message if there is no `current_user` to logout.
* **Auth Errors**: Render auth errors (e.g. 'invalid credentials' or 'username
already exists') in your response with a corresponding error status.
  * Use `@user.errors` when applicable.
  * Render single errors in the format `{base: ['error message here']}`
  * ex. `render json: {base: ['invalid credentials']}, status: 401`
  * **Caution**: Error responses are formatted differently than normal
  responses.

Test your routes using `$.ajax` in the console before moving on.

### 2. Build Your Frontend

Once your API is fully built and tested, set up the following flux architecture
components.

####SessionApiUtil

Create a `SessionApiUtil` with the following methods:
  * **`signup`:** POST 'api/users'
  * **`login`:** POST 'api/session'
  * **`logout`:** DELETE 'api/session'

Each `SessionApiUtil` method should take `success` and `error` callbacks.

Before moving on, put `SessionApiUtil` on the window in your entry file and test
it out. Make sure all your endpoints behave properly for both successful and
erroneous requests. Pay attention to:
* the `Network` tab of Dev Tools
* the Rails server logs

####SessionStore

Create a `SessionStore` that will keep track of the current user.

**Variables:**
  * `_currentUser`: should be `{}` if no one is logged in

**Closures:**
  * `_login`: set `_currentUser` and `__emitChange()`
  * `_logout`: set `_currentUser` to `{}` and `__emitChange()`

**Public Methods:**
  * `currentUser`: a reader
  * `isUserLoggedIn`: should return a boolean of whether a user is logged in
    * hint: you can check `_currentUser.id`
`SessionStore.__onDispatch`
  * Define `SessionConstants` for `LOGIN` and `LOGOUT`
  * Setup `SessionStore.__onDispatch` to call the appropriate methods

####SessionActions

0. Create **`signup`, `login`, and `logout`** methods for client-generated actions:
Each method should:
  * Invoke the appropriate `SessionApiUtil` action.
  * Pass `receiveCurrentUser` as a `success` callback.

0. Write `receiveCurrentUser`
  * dispatch to the `SessionStore`

Attach your `SessionActions` to the window and test them out before moving on.

### 3. Create `LoginForm` and `SignupForm`

Create a `LoginForm` component and a React `Route` for it.

  * `render` a form for users to enter their username and password.
  * `onSubmit`, call your `SessionActions#login` and pass in the credentials.
  * After a successful login, redirect to your root Route (`/`)
    * In `#componentDidMount`, register a listener with the `SessionStore`.
    * When the `SessionStore` emits change, check if a user is logged in.
    * If so, redirect them (`this.context.router.push("/")`).

Once your `LoginForm` works, make a similar `SignupForm` component and route that calls `SessionActions#signup`.

### 4. Errors

Your forms should display errors if user submissions cause validation/authentication errors.

####  Create an `ErrorStore`.

Variables:
* `_errors`: This `{}` should store key-value pairs where:
  * each key is an error field, and
  * each value is an array of error messages for that field.
    ```js
    _errors === {
      username: [
        "has already been taken",
        "is too short (minimum is 4 character)"
      ],
      password: [
        "is too short (minimum is 6 characters)"
      ]
    }
    ```
*`_form`: The store should only ever have the errors for a single form. This
variable should store the name of the form currently being tracked as a string.

Public Methods:
* `formErrors`: takes a `form` argument and returns a copy of `_errors` if and only
if `form === _form`.
* `form`: returns the current `_form`.

Closures:
* `setErrors`: takes a `form` name and an `errors` object and saves them in `_form`
and `_errors`; emits change.
* `clearErrors`: sets `_form` and `_errors` to `""` and `{}` respectively; emits
change.

`__onDispatch`:
Create `ErrorConstants` and handle the following `actionTypes`:
* `ErrorConstants.SET_ERRORS`
* `ErrorConstants.CLEAR_ERRORS`
* Wait until

####  Define `ErrorActions`

Define the following actions:
* `setErrors`
* `clearErrors`

Both methods should take the raw API error response generated by your `$.ajax` call and dispatch the proper error information to your `ErrorStore`. Keep in mind that Rails error responses return XHR objects, not raw response data, which lives in the `.responseJSON` attribute of the XHR object returned.

Once you've written these methods, return to your `SessionActions` and `SessionApiUtil`:
* Modify `SessionApiUtil` so that your methods take an `error` callback to invoke in  the `error` response of your `$.ajax` call.
* Modify `SessionActions` to pass `ErrorActions.setErrors` as the `error` argument to your `SessionApiUtil`.

####  Update your `LoginForm` and `SignupForm`

Your components should listen to your `ErrorStore` on `componentDidMount`. Make
sure they track `this.state.errors`. When the store changes, `setState` to
the new errors on the form. Make sure to check `ErrorStore.form` to ensure that
your `LoginForm` doesn't render `SignupForm` errors and vice versa.

Update both forms to `render` their errors if any are present.

### 4. Create a greeting in the header.

Modify your app to provide a greeting to users when they are signed in.
  * In the `App` component, render a `<header>` with information about the
  `SessionStore#currentUser`.
  * If a user is logged in, display their username and a logout button.
    * `onClick` of logout button, `preventDefault` and `SessionActions#logout`
  * If user IS NOT logged in, give them links to "#/login" and "#/signup"

### 5. Bootstrap the current user before the app mounts.

When our static `root` page loads, our app mounts without being aware of who the
current user is.

One solution to this problem is to create another API hook that returns the
current user and then fetch that information when the app mounts. However, since
the request would be asynchronous, our app would momentarily have no current
user. This would cause it to briefly render in a 'not-logged-in' state and then
re-render when the current user was received, causing a strange, flickering
effect. To circumvent this, we'll bootstrap the current user alongside our html
when the page initially loads.

#### Edit your `root.html.erb`

Add a `<script></script>` element to the top of your `root.html.erb` file. Give
it a `type="text/javascript"`.

Inside your `<script>`, we're going to assign `window.currentUser`. In order to
get the proper value, we'll need to ask our controller for the `current_user`
and then `render` that information inside the script tag using `ERB`
interpolation. The result will be a hard-coded assignment in our rendered html
that looks something like this:

```html
...
<script type="text/javascript">
    window.currentUser = {"id":3,"username":"bobross"}
</script>

<main id="content"></main>
...

``` 

where `{"id":3,"username":"bobross"}` is inserted via `ERB`.

#### Interpolate the current user information

In your script, assign your `window.currentUser` to an erb expression:

```js
  window.currentUser = <%=  %>
```

Make sure to use `<%= %>` so that the result of your ruby code is rendered into the
script ( it will eventually return a JSON object).

Inside your erb expression, `render` your jbuilder `_user` partial, passing it
the `current_user`. To prevent rails from automatically looking for a html
partial, specify the whole path, including `.json.jbuilder`.Mark your `render`
result `html_safe` to avoid escaping certain characters. You should get a JS-
compatible object to assign to `window.currentUser`. Add interpolation around
your  `window.currentUser=` assignment so that it only runs if someone is logged
in. You should have something like this:

```html

<script type="text/javascript">
  <% if logged_in? %>
    window.currentUser = <%= render("api/users/user.json.jbuilder", 
      user: current_user).html_safe %>
  <% end %>
</script>

```

Log in, refresh your page, and check out your `elements` in the Dev Tools.
Verify that the  `script` contains an object literal of the current user and
properly assigns  `window.currentUser`.

Finally, inside the `DOMContentLoaded` callback in your entry file, call
`SessionActions.receiveCurrentUser(window.currentUser)` before
`ReactDom.render`. This will ensure that the `SessionStore` has the
`_currentUser` before any of your components render.

Test that your bootstrapping worked by logging in and refreshing the page. If you 
don't get logged out, it worked!

### 6. Protect your front-end routes.

Let's make sure users can't get to our "/benches/new" or "benches/:id/review" routes on the frontend if they're not logged in.

  * Add an `onEnter` prop to the Routes we want to protect:

    ```html
    <Route path="benches/new" component = { BenchForm } onEnter={ _ensureLoggedIn } />
    ```
  * Define an `_ensureLoggedIn` function in your entry file. It should:
    * have `nextState` and `replace` parameters.
    * check `SessionStore#isUserLoggedIn`.
    * If `true`, do nothing.
    * Otherwise, `replace` the path with "/login". (Remember that `replace` won't
    add a "fake" entry to the browser's history, whereas `push` will.)
    * We don't need an `asyncDoneCallback` because `_ensureLoggedIn` runs synchronously.

Test your work before continuing.
