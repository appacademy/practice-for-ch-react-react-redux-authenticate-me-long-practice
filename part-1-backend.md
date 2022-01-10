# Authenticate Me - Backend

In this multi-part project, you will learn how to put together an entire Rails +
React application with authentication.

## Phase 0: Backend Setup

In this project, you will completely separate the backend Rails code from the
frontend React code.

Create a folder called `authenticate-me`. Inside that folder, create two more
folders called `backend` and `frontend`.

Your file structure should look like this:

```plaintext
authenticate-me
├── backend
└── frontend
```

### Create a New Rails Application

Next, `cd` into your `backend` directory and initialize a new Rails application:

```sh
rails new . -d=postgresql --api --minimal --skip-test
```

> _Note:_ the `.` in `rails new .` creates your Rails app within the current
> directory.

Create an initial git commit: run `git add .` and `git commit -m "Initial
commit"`. Then, rename your primary git branch (if it is not already named
`main`): `git branch -M main`.

During development, you'll want your server to listen to requests on port 5000,
reserving port 3000 for your frontend. Go to __config/puma.rb__ and look for the
following line, around line 18:

```rb
# config/puma.rb

port ENV.fetch("PORT") { 3000 }
```

Change `3000` to `5000`:

```rb
# config/puma.rb

port ENV.fetch("PORT") { 5000 }
```

Finally, initialize your database:

```sh
rails db:create
```

### Add Dependencies to Gemfile

Next, head to your Gemfile. At the top level, you should add `bcrypt`,
`jbuilder`, `pry-rails`, and `faker`. 

Then, within the `:development, :test` group, add some convenience gems:
`better_errors`, `binding_of_caller`, and `annotate`. 

Within this same group, you can optionally replace the default `debug` gem with
`byebug` if you want to use byebug for debugging. If you do so, you'll want to
add the line `.byebug_history` to your __.gitignore__.

```rb
# Gemfile

# Add the following 4 gems at the top-level
gem "bcrypt"
gem "jbuilder"
gem "pry-rails"
gem "faker"

# Add the following gems to the `:development, :test` group
# Optionally replace `"debug", ">= 1.0.0"` with `"byebug"`
group :development, :test do
  gem "byebug", platforms: %i[ mri mingw x64_mingw ]
  gem 'better_errors'
  gem 'binding_of_caller'
  gem 'annotate'
end
```

After adding all these gems, run `bundle install`.

### Add Session Support

By default, an API-only Rails application does not include cookie support.
Session-based authentication requires cookies, however, so you'll need to
explicitly opt-in to using them.

To do so, you'll need to add middleware for cookie and session management. Head
to __config/application.rb__ and add the following lines inside the
`Application` class:

```rb
# config/application.rb

module Backend
  class Application < Rails::Application
    # ...
    config.middleware.use ActionDispatch::Cookies
    config.middleware.use ActionDispatch::Session::CookieStore,
      key: '_auth_me_session'
  end
end
```

The `key` option here simply sets the name of the session cookie.

### Transform Request and Response Keys

Lastly, you'll want to set up automatic transformations of keys in requests and
responses so that your frontend code can consistently use camelCase and your
backend code can consistently use snake_case.

Requests are created on the frontend and processed on the backend. So their
keys should be converted from camelCase to snake_case. You'll do this by using a
`before_action` [controller filter][filters] to transform the keys in your
`params` before you hit any controller actions. Head to
__app/controllers/application_controller.rb__ and add the following:

```rb
# app/controllers/application_controller.rb

before_action :snake_case_params

private

def snake_case_params
  params.deep_transform_keys!(&:underscore)
end
```

Responses, on the other hand, are created on the backend but handled on the
frontend, so their keys should be converted from snake_case to camelCase. You'll
be using Jbuilder to construct your responses; thankfully, you can instruct
Jbuilder to transform the keys in your responses before you send them out. Head
to __config/environment.rb__ and add the following lines at the end:

```rb
# config/environment.rb

# ...
Jbuilder.key_format camelize: :lower
Jbuilder.deep_format_keys true
```

## Phase 1: User Table and Model

Now that setup is complete, it's time to build out the basics of user
authentication. You'll start with the data layer--the 'M' within MVC--by
creating a `users` table and `User` Active Record model.

### Generate a Users Table and Model

What data will you need to store in the `users` table?

To start, each user will have an `email` and a `username`. Both should be
strings and should be unique.

Each user will also have a password. However, you won't store that directly
(that would be a huge security risk). Rather, you'll want to store a hashed
version of their password called a `password_digest`. You'll never be able to
reverse-engineer a user's password from its digest; however, you'll be able to
check if a provided password matches a user's digest when they attempt to
login.

Finally, recall that after a user logs in or signs up, they're given a unique
`session_token`, which acts like temporary credentials for as long as they are
logged in (more on that later). This will be a unique, random string.

So you will need four columns, besides the customary `timestamps`: `email`,
`username`, `password_digest`, and `session_token`. Run the following generator
command to create your `User` migration and model:

```sh
rails g model User email:string:uniq username:string:uniq password:digest session_token:string:uniq
```

Note that`password:digest` is a special attribute syntax that differs from the
typical `name:data_type:options`. It generates a `password_digest` column with a
type of string, and it also automatically includes [`has_secure_password`] in
your `User` model--refer back to the reading on `has_secure_password` for a
reminder on what this macro does.

First, you should complete your migration to create a `users` table. Head to the
newly created migration file in __db/migrate/__ and add `null: false`
constraints to every column:

```rb
# db/migrations/<UTC timestamp>_create_users.rb

# ...
t.string :email, null: false
t.string :username, null: false
t.string :password_digest, null: false
t.string :session_token, null: false
# ...
```

Next, run this migration to create the `users` table:

```sh
rails db:migrate
```

If you head to __db/schema.rb__, you should see your `users` table defined
there. Nice!

### Add Validations and Callbacks

Next, let's add some validations:

1. `username`, `email`, and `session_token` must all be present and unique (as
   per your database constraints).
2. `username` should be between 3 and 30 characters long
3. `email` should be between 3 and 255 characters long
4. `email` should have the format of a valid email address. You can test this
   using a _regular expression_ (regex) with the [`format`] validation helper.
   Find an email regex online or use the `URI::MailTo::EMAIL_REGEXP` built into
   Ruby.
5. `username` should _not_ have the format of a valid email address. Follow a
   similar strategy to the one you used for `email`. Include a custom message
   for this validation: `can't be an email`.
6. `password`, if present, should be between 6 and 255 characters. Remember, a
   `password` will typically only be present upon creating a new `User`. Any
   existing `User` retrieved from the database will have a `nil` value for
   `password`, unless you explicitly provide the instance with a new
   `password`. For this reason, you should [`allow_nil`].

If you are forgetting some of the syntax for writing validations, be sure to
visit the [Rails guide][validations-guide] on validations.

Including a validation for `session_token` is a safeguard; however, this isn't
the user's responsibility to provide. You, the developer, should ensure
`session_token` is never `nil` by generating one for each new user
automatically. Let's do that before testing your validations.

Add the following lines to your `User` model class, below your validations:

```rb
# app/models/user.rb

after_initialize :ensure_session_token

def self.generate_session_token
  # in a loop:
    # use SecureRandom.base64 to generate a random token
    # use `User.exists?` to check if this `session_token` is already in use
    # if already in use, continue the loop, generating a new token
    # if not in use, return the token
end

private

def ensure_session_token
  # if `self.session_token` is already present, leave it be
  # if `self.session_token` is nil, set it to `User.generate_session_token`
end
```

This code defines an `ensure_session_token` [callback] method that runs
automatically after a new instance is initialized. To get a random string,
`ensure_session_token` should call the `generate_session_token` class method,
which itself should use [`SecureRandom.base64`][base64]. Replace the
pseudocode provided above with your own code.

Let's test that your code works. In the Rails console (`rails c`), check that
`User.generate_session_token` actually generates and returns a random string.

Then, test that `ensure_session_token` successfully assigns a token to users by
running `User.new.session_token`; a random string should be returned.

Next, it's time to test your validations. First, try creating a user without any
attributes:

```rb
[1] > User.new.tap(&:valid?).errors.messages
```

> _Note:_ This uses [`Kernel#tap`][tap], a method that passes its receiver to
> the provided block, then returns its receiver. In the example above, `tap` is
> called on a `User` instance, the provided block calls `.valid?` on that user,
> then `tap` returns that same user so we can call `.errors.messages` on it
> afterward. `tap` allows you to chain together multiple method calls on the
> same object without ever having to save it to an intermediate variable. This
> is generally a bad idea in real code, but it's perfect for `pry`.

You should get something like the following output in your console:

```rb
=> {:password=>["can't be blank"],
 :username=>["is too short (minimum is 3 characters)"],
 :email=>["is too short (minimum is 3 characters)", "is invalid"]}
```

It is not surprising that the `length` validations failed for `username` and
`email`. And it's good that the `session_token` didn't fail its `presence`
validation--that just means `ensure_session_token` is working.

But hold on, why is there a failed validation for `password`? Isn't it allowed
to be `nil`? Don't worry, Rails is just being sneaky. In fact, it is the
`presence` validation for `password_digest`--added by
`has_secure_password`--that has failed. Rails simply customized the resulting
error so that it would be associated with the `password` attribute.

Feel free to perform a few more tests on the `presence`, `length`, and `format`
validations. When you're ready, test the `uniqueness` validations:

```rb
[1] > attributes = Faker::Internet.user('username', 'email', 'password')
[2] > User.create(attributes)
[3] > User.create(attributes).tap(&:valid?).errors.messages
```

Here, you are using [`Faker`][faker] to generate some random attributes. The
first time you create a `User` with these attributes, it should save to the
database. The second attempt, though, should fail the `uniqueness` validations
for `username` and `email`, resulting in the following output:

```rb
=> {:username=>["has already been taken"], :email=>["has already been taken"]}
```

### Authentication Methods

Finally, it is time to create the User authentication methods you'll be calling
from your controller actions: `User::find_by_credentials` and
`User#reset_session_token!`.

Anywhere below your validations and above your `private` methods, define a class
method `self.find_by_credentials`. This should take in a `credential` (either a
username or email) and a plaintext `password`. 

In this method, use the [`find_by`] Active Record method to find a user whose
email or username matches the provided `credential`. To determine which field to
query by--`email` or `username`--check if the provided `credential` matches the
`URI::MailTo::EMAIL_REGEXP` regular expression. (Look through the documentation
for [`String`] and [`Regexp`] if you're not sure how to do that.)

If a matching user is found, check if the provided `password` matches their
`password_digest`. Use the `authenticate` instance method, defined by
`has_secure_password`, to do so--an example is provided in the documentation for
[`has_secure_password`]. 

Return the matching user if the provided password is correct. If no matching
user exists, or if the provided password is incorrect, return a falsey value.

Here's a method stub with some pseudocode to get you started:

```rb
# app/models/user.b

# ...
def self.find_by_credentials(credential, password)
  # determine the field you need to query: 
  #   * `email` if `credential` matches `URI::MailTo::EMAIL_REGEXP`
  #   * `username` if not
  # find the user whose email/username is equal to `credential`

  # if no such user exists, return a falsey value

  # if a matching user exists, use `authenticate` to check the provided password
  # return the user if the password is correct, otherwise return a falsey value
end
# ...
```

Go ahead and test that `find_by_credentials` works:

```rb
[1] > User.destroy_all
[2] > User.create(username: 'usr', email: 'usr@email.io', password: 'starwars')
[3] > User.find_by_credentials('usr@email.com', 'starwars')
[4] > User.find_by_credentials('usr@email.io', 'startrek')
[5] > User.find_by_credentials('usr', 'starwars')
[6] > User.find_by_credentials('usr@email.io', 'starwars')
```

Here, you start by destroying existing users to avoid accidentally failing any
uniqueness validations, then you create a new `User`. The first two calls to
`find_by_credentials` should return falsey values, because of a wrong
`credential` in the first case, and a wrong `password` in the second. The last
two calls should return the `User` instance you created.

After passing these tests, go ahead and write a `reset_session_token!` instance
method. Use `User::generate_session_token` and the [`update!`] Active Record
instance method:

```rb
# app/models/user.b

def reset_session_token!
  # `update!` the user's session token to a new, random token
  # return the new session token, for convenience
end
```

Then, test your method:

```rb
[1] > u = User.first
[2] > old_token = u.session_token
[3] > new_token = u.reset_session_token!
[4] > u.session_token != old_token
[5] > u.session_token == new_token
```

The last two conditions should both return `true`.

Congratulations! Your `User` model is now complete, with working validations and
authentication methods!

### Seeding Your Database

It'll be nice to have some preexisting `User` instances for testing your
upcoming controllers and routes. Paste the following into __db/seeds.rb__:

```rb
# db/seeds.rb

ApplicationRecord.transaction do 
  # Unnecessary if using `rails db:seed:replant`
  User.destroy_all

  # For easy testing, so that after seeding, the first `User` has `id` of 1
  ApplicationRecord.connection.reset_pk_sequence!('users')

  # Create one user with an easy to remember username, email, and password:
  User.create!(
    username: 'Demo-lition', 
    email: 'demo@user.io', 
    password: 'password'
  )

  # More users
  10.times do 
    User.create!({
      username: Faker::Internet.unique.username(specifier: 3),
      email: Faker::Internet.unique.email,
      password: 'password'
    }) 
  end
end
```

Then, run `rails db:seed`.

## Phase 2: User Authentication and API Routes

Now that the data layer is complete, it is time to build out your application's
API and session-based authentication system. Let's quickly review authentication
conceptually before you write any code.

### Review of Session-Based Authentication

To log in, a user must provide a valid credential and password. This establishes
a new login _session_, which is active until they log out. While this session is
active, whenever the user makes a new request, the server should be able to see
that they have an active session and identify who they are.

So how is this done?

1. After a user logs in or signs up, a `session_token` representing the newly
   active session is generated. It is stored in the `session_token` column for
   that user in the database. Until the user logs out, they just need to provide
   this token to establish their identity--it acts like temporary credentials.
2. The server gives this token to the user by putting it in the Rails `session`
   cookie, which is included in the server's response. The `session` cookie is a
   special cookie provided by the session middleware you added; it is encrypted
   and tamper-proof. As with all cookies, the session cookie is stored in the
   user's browser and sent back automatically with every subsequent request.
   Read more about it [here][session].
3. On each new request, the server checks for a token stored in the `session`
   cookie that was sent with the request. The server then looks for a
   corresponding `User` in the database whose `session_token` attribute matches
   the token in the request. If a matching user is found, they are considered
   the `current_user` for the current request-response cycle.
4. Upon logging out, a user's `session_token` should no longer be valid. Thus,
   the `session_token` attribute is reset for this user in the database. For
   good measure, it is removed from the `session` cookie as well.

### Session Authentication Methods in `ApplicationController`

To facilitate this process, you'll be writing some helper methods in
`ApplicationController`:

* `current_user`: returns the `User` whose `session_token` attribute matches the
  token provided in the `session` cookie.
* `login!(user)`: takes a `User` instance and resets their session token. The
  new token is then stored in the `session` cookie.
* `logout!(user)`: resets the session token of the `current_user` and clears
  the session token from the `session` cookie.
* `require_logged_in`: renders a message of 'Unauthorized' with a 401 status if
  there is no `current_user`. You can add this as a `before_action` callback for
  any controller actions that require a logged in user.

Here's a skeleton with pseudocode to get started:

```rb
# app/controllers/application_controller.rb

def current_user
  @current_user ||= # user whose `session_token` == token in `session` cookie
end

def login!(user)
  # reset `user`'s `session_token` and store in `session` cookie
end

def logout!
  # reset the `current_user`'s session cookie, if one exists
  # clear out token from `session` cookie
  @current_user = nil # so that subsequent calls to `current_user` return nil
end

def require_logged_in
  unless current_user
    render json: { message: 'Unauthorized' }, status: :unauthorized 
  end
end
```

Go ahead and fill out these methods. Remember that the `session` getter, defined
for controllers, returns a Hash-like object. Here's an example of how to use it:

```rb
# request from brand new user / browser, Sennacy's MacBook (Chrome)
session[:banana]            # => nil

# your app responds by changing `session[:banana]` to 'hello'
session[:banana] = 'hello'  

# another request from Sennacy's MacBook (Chrome)
session[:banana]            # => 'hello'

# your app responds by changing `session[:banana]` to nil
session[:banana] = nil  

# one more request from Sennacy's MacBook (Chrome)
session[:banana]            # => nil
```

You should choose a consistent key within the session where you'll store the
token, e.g., `session[:session_token]` or `session[:token]`.

### Testing Your `ApplicationController` Authentication Methods

Once you've finished filling out these methods, it's time to test them. But how?
The `session` cookie comes from requests, which you can't create from the Rails
console. To test, you'll create a simple `test` route. Head to
__config/routes.rb__ and define the following route:

```rb
# config/routes.rb

post 'api/test', to: 'application#test'
```

All of your routes will start with `api/` to show that you're only returning
JSON data. You'll see later why you should use the `POST` method here. Run
`rails routes` to see the route this defined.

> _Note:_ You never want any 'real' routes mapped to actions within
> `ApplicationController`, but for a simple test route like this, it's the
> easiest thing to do.

Go back to `ApplicationController` and define the following `test` action:

```rb
# app/controllers/application_controller.rb

# ...
def test
  if params.has_key?(:login)
    login!(User.first)
  elsif params.has_key?(:logout)
    logout!
  end

  if current_user
    render json: current_user.slice('id', 'username', 'session_token')
  else
    render json: ['No current user']
  end
end
# ...
```

Carefully read through this action and make sure you understand what each line
is doing. Read more about `slice` [here][slice].

Next, start your server (`rails s`), go to `localhost:5000` in your browser,
open up the Chrome console, and make the following `fetch` request:

```js
> await fetch('/api/test', { method: 'POST' }).then(res => res.json())
```

You should get `['No current user']` as the JSON response. (If not, debug your
`current_user` method). This makes sense since you haven't logged in yet!

Next, make a request with a `login` query string param:

```js
> await fetch('/api/test?login', { method: 'POST' }).then(res => res.json())
```

You should see the 'Demo-lition' user returned! (If not, debug your `login!`
method.) Take note of Demo-lition's `session_token`.

Next, remove the `login` query string param, and run the `fetch` request a few
more times:

```js
> await fetch('/api/test', { method: 'POST' }).then(res => res.json())
```

Verify that Demo-lition is still logged in and that their `session_token`
doesn't change.

Finally, add a `logout` query string param:

```js
> await fetch('/api/test?logout', { method: 'POST' }).then(res => res.json())
```

You should get back `['No current user']`. Remove `logout` from the query
string, and make a few more requests. You should still get back `['No current
user']`. If not, debug your `logout!` method.

### User and Session Routes

You now have the necessary logic in place to support session authentication.
It's time to build out your real routes and controllers!

As mentioned before, since your backend's role is to be an API serving JSON
data, all your routes will be served at URL paths starting with `api/`.

You will be creating the following routes:

* **Signing up:** `POST api/users`
* **Retrieving the current user:** `GET api/session`
* **Logging in:** `POST api/session`
* **Logging out:** `DELETE api/session`

Go ahead and add the following routes to your __config/routes.rb__:

```rb
# config/routes.rb

namespace :api, defaults: { format: :json } do
  resources :users, only: :create
  resource :session, only: [:show, :create, :destroy]
end
```

Remember that by nesting your routes within `namespace :api`, each route path is
automatically prefixed with `api/`, each controller must be defined within
__app/controllers/api/__, and each controller class name must begin with `Api::`.

Run `rails routes` to verify the routes you have created. Note that the singular
`resource` before `:session` causes the `show` and `destroy` routes to have no
`:id` wildcard. This is because, with your current application architecture,
each client can have at most one session.

### Generating User and Session Controllers

Next, it's time to define user and session controllers to handle requests to
your user and session routes.

Conveniently, Rails provides a generator for controller files. Run the following
command to create an `Api::UsersController` within __controllers/api__:

```sh
rails g controller api/users create --skip-routes
```

Here, `api/users` is the controller name. By prefixing it with `api/`, you
essentially tell Rails to do everything necessary for creating a namespaced
controller.

`create` tells Rails to include a stubbed `create` action.

`--skip-routes` tells Rails not to define new routes for the provided actions
(since you already defined them).

After verifying that the generator was successful, run:

```sh
rails g controller api/sessions show create destroy --skip-routes
```

### Api::SessionsController

Next, it's time to fill out your stubbed actions. Start with the
`Api::SessionsController` in __app/controllers/api/sessions_controller.rb__,
using the following pseudocode as guidance:

* `show`
  * if there is a `current_user`: render `current_user` as JSON
  * if there is not a `current_user`: render an empty Hash as JSON
* `create`
  * pass the credentials from the request body, stored under top level keys of
    `credential` and `password`, to `User::find_by_credentials`; save the result
    to a `user` variable
  * if a user with matching credentials was found (`user` is truthy):
    * login `user`
    * render `user` as JSON
  * if no user was found (`user` is falsey):
    * render `{ message: 'The provided credentials were invalid.' }` as JSON,
      with a status of `:unauthorized`
* `destroy`
  * log out the `current_user`, if one exists
  * render `{ message: 'success' }` as JSON

> _Note:_ You'll be replacing these JSON responses shortly. **You should never
> actually send a user's complete database data--which includes their
> `session_token`, `password_digest`, etc.--in a JSON response.**

Restart your server, then test your code by making the following `fetch`
requests in your Chrome console.

```js
> loginRequestOptions = {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ credential: 'Demo-lition', password: 'password' })
  }

> await fetch('/api/session', loginRequestOptions).then(res => res.json())
// should return the 'Demo-lition' user

> await fetch('/api/session', { method: 'GET' }).then(res => res.json())
// should return the 'Demo-lition' user

> await fetch('/api/session', { method: 'DELETE' }).then(res => res.json())
// should return `{ message: 'success' }`

> await fetch('/api/session', { method: 'GET' }).then(res => res.json())
// should return `{}`
```

### Api::UsersController

Head to `Api::UsersController`.

The `create` action, which is used to sign up a new user, will use mass
assignment with [strong params] to set the new user's attributes.

Add the following code to the bottom of your users controller:

```rb
# app/controllers/api/users_controller.rb

private

def user_params
  params.require(:user).permit(:email, :username, :password)
end
```

To generate the strong params, you first `require` a top level key of `user`,
under which all of the user attributes from the request body will exist. This is
to clearly separate user attribute params from any other params that shouldn't
get passed to `User.new`.

It would be nice, though, if the frontend didn't need to use this nested
structure in every request that involves creating or updating a resource--the
frontend shouldn't have to care about the backend's implementation details.

Thankfully, Rails performs resource-based nesting automatically. For requests
whose body is formatted as JSON (and only those requests), Rails sees if it can
perform automatic nesting: it looks at the name of the controller that will
handle the request--say, `Api::BananasController`--and checks if there is an
Active Record model with a matching name (`Banana`). 

If there is, Rails will look for top-level keys in the request body matching any
of the model's attributes (`Banana.attribute_names`), duplicating matching
parameters and nesting them under a top-level key corresponding to the model's
name (here, `banana`).

By default, then, any `username` or `email` keys in the body of `POST api/users`
requests will get automatically nested under a top-level key of `user`.

Unfortunately, a `password` will also be included in requests to sign up, and
that is technically not a `User` attribute--`password_digest` is. 

You can override what keys you want Rails to automatically nest by making a call
to [`wrap_parameters`] at the top of any controller definition. Add the
following to the top of your `UsersController`:

```rb
# app/controllers/api/users_controller.rb

wrap_parameters include: User.attribute_names + ['password']
```

> _Note:_ Since this automatic nesting occurs before the `params` object ever
> reaches your controllers, if you ever want automatic nesting of attributes
> that are camelCased in the request body, you must `include` them explicitly,
> just as you did above for `password`.

To test that your strong params are working, add `render json: user_params` to
your `create` action:

```rb
# app/controllers/api/users_controller.rb

def create
  render json: user_params
end
```

Then, make the following request from your Chrome console:

```js
> signupRequestOptions = {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ 
      email: 'coolemail@hotmail.net', 
      username: 'cooluser',
      password: 'starwars',
    })
  }

> await fetch('/api/users', signupRequestOptions).then(res => res.json())
// should return an object with all three attributes from the request body
```

After confirming that your `user_params` work, fill out the `create` action:

* instantiate a new `User` instance, passing in `user_params`, and save it to a
  `user` variable
* try to save this `user` to the database with `user.save`
* if `user.save` returns a `true` (the `user` was saved to the database):
  * login `user`
  * render `user` as JSON
* if `user.save` returns `false` (the `user` failed your validations):
  * render `{ errors: user.errors.full_messages }` as JSON, with a status of
    `:unprocessable_entity`

Test that your `create` action is working by using the same `fetch` request you
used to test `user_params`. Test that you successfully logged in this new user
by making a `fetch` request to `GET api/session`; this should return the new
user.

Congratulations! Your API is fully functional: you can sign up, log in, log out,
and retrieve the current user!

## Phase 3: Jbuilder Views, CSRF Protection, Error Handling

Although your app is functional, it's not quite complete: the data being
returned is too extensive, and you still haven't accounted for CSRF attacks.

### Jbuilder Views

As mentioned before, it's not ideal that in several controller actions you are
rendering all of a user's data. Now is a good time to start building Jbuilder
views for more customized JSON responses.

Within __app__, define a __views__ directory with an __api__ subdirectory. All
your Jbuilder views will live in this __app/views/api/__ folder. 

Next, create a subdirectory __users__ within __app/views/api/__ and create a
partial called __\_user.json.jbuilder__. This partial takes in a `user` and
defines the data that should be rendered for that `user` in your JSON responses:

```rb
# app/views/api/users/_user.json.jbuilder

json.extract! user, :id, :email, :username
```

Next, replace every `render json:` of user data in your controller actions with
a `render partial:` of the user partial:

```rb
# app/controllers/api/sessions_controller.rb

def show
  if current_user
    render partial: 'api/users/user', locals: { user: current_user }
  # ...
end

def create
  # ...
  if user
    # ...
    render partial: 'api/users/user', locals: { user: user }
  # ...
end

# app/controllers/api/users_controller.rb

def create
  # ...
  if user.save
    # ...
    render partial: 'user', locals: { user: user }
  # ...
end
```

> _Note:_ Since you are currently only ever rendering a single user's data in
> all of your JSON responses, you are not really using your partial _as_ a
> partial, i.e., you are not embedding it within another view. By making it a
> partial, though, you can easily use it within future JSON responses that
> include additional data without having to refactor your code.

Go ahead and test these new Jbuilder views by making `fetch` requests in your
Chrome console to hit the three actions you changed: `users#create`,
`session#show`, and `session#create`. (You may also want to make a logout
request between testing the login and sign up routes.) Make sure the user data
returned includes only the `id`, `email`, and `username` attributes.

### CSRF Protection

Remember Cross Site Request Forgery (CSRF) attacks? These occur when a user with
an active session visits a malicious website, which sends a non-GET request to
your app without the user's knowledge (triggered, perhaps, by the user clicking
a harmless looking link that secretly submits a form). Since the user's
`session` cookie is automatically included in their request, the server will
authenticate the request. The server doesn't know that the request is
malicious.

You can protect against CSRF attacks by requiring that each non-GET request
include an authenticity token, either in the body or in a header, signifying
that it is legitimate. This token must be obtained from the server. Typically,
Rails provides authenticity tokens in its HTML responses automatically via
`<meta>` tags in `application.html.erb`. Since your server will operate solely
as an API--i.e., no HTML responses--you'll instead need to put an authenticity
token inside a response header manually.

> These tokens are session-specific: your authenticity token will only
> authenticate requests coming from your session. A malicious actor therefore
> cannot fake authentication by attaching an authenticity token generated on
> their browser to a form submitted from your browser--the token would not match
> your `session`.

Head to your `ApplicationController`. First, you want to ensure Rails rejects
any non-GET requests that lack a valid authenticity token. To do so, add the
following two lines to the top of your controller:

```rb
# app/controllers/application_controller.rb

include ActionController::RequestForgeryProtection
  
protect_from_forgery with: :exception
```

> **Note:** You have to add the `RequestForgeryProtection` module because you
> used the `--api` flag when generating your initial Rails application. If your
> `ApplicationController` inherited from `ActionController::Base` instead of
> `ActionController::API`, then this module would already be included with the
> protection turned on by default.
Then, you want to make sure you generate a valid authenticity token and store it
in a header for each response. To do so, you'll add a new `before_action`
filter, `attach_authenticity_token`:

```rb
# app/controllers/application_controller.rb

# update this line:
before_action :snake_case_params, :attach_authenticity_token

# add this method to the `private` section of your controller:
def attach_authenticity_token
  headers['X-CSRF-Token'] = masked_authenticity_token(session)
end
```

Time to test that you are protecting against CSRF attacks! Make sure your server
is running and open up your Chrome console.

First, make a GET request--which won't be blocked by CSRF protection--in order
to obtain an authenticity token from the response headers:

```js
> response = await fetch('/api/session')
  token = response.headers.get('X-CSRF-Token')
```

Next, try making two POST requests to your `api/test` route (now you see why
it's a POST route!). One should include the authenticity token in the request
headers (at the key of `X-CSRF-TOKEN`, where Rails expects it), and one should
omit the token. Only the one without the token should be rejected for having an
invalid authenticity token.

```js
> withToken = { method: 'POST', headers: { 'X-CSRF-Token': token } }
  withoutToken = { method: 'POST' }

> await fetch('/api/test', withToken).then(res => res.json())
// should get a 200 response

> await fetch('/api/test', withoutToken)
// should get a 422 response and `InvalidAuthenticityToken` error in server log
```

### Error Handling

Great work setting up CSRF protection! However, it's not ideal that you had to
check your server log in order to confirm your request was rejected because of
an invalid authenticity token. It would be nicer if **all** error responses from
your backend were formatted as JSON.

To do so, you'll need to [`rescue_from`] errors for which you'd like to render a
custom response. You can use this macro method like so:

```rb
rescue_from BrewException, with: :handle_brew_exception
```

In the above example, if an unhandled `BrewException` error is raised before
your controller can render a response, Rails will rescue this error and invoke
`#handle_brew_exception`, passing in the error object as an argument. There, you
can render a custom response:

```rb
def handle_brew_exception
  render json: ["I can't brew; I'm a teapot!"], status: 418
end
```

> **Note:** If an error matches multiple `rescue_from`s, only the method provided
> to the _last_ one will run.

Head to your `ApplicationController` and add the following code:

```rb
# app/controllers/application_controller.rb

# at top of ApplicationController definition:
rescue_from StandardError, with: :unhandled_error
rescue_from ActionController::InvalidAuthenticityToken,
  with: :invalid_authenticity_token

# ...

# at bottom of ApplicationController definition (under `private`):
def invalid_authenticity_token
  render json: { message: 'Invalid authenticity token' }, 
    status: :unprocessable_entity
end

def unhandled_error(error)
  if request.accepts.first.html?
    raise error
  else
    @error = error
    render 'api/errors/internal_server_error', status: :internal_server_error
  end
end
```

Whenever an `InvalidAuthenticityToken` error is raised,
`#invalid_authenticity_token` is run, rendering a nice, custom JSON response.

When any other error is raised, `#unhandled_error` is run. There, you check if
the request is looking for an HTML response (based on its `Accepts` header). If
so, it's probably not a `fetch` request, so you re-raise the error. (This could
happen from navigating to an API route via your URL bar; in that case, it'd be
nice to get the `better_errors` page with all its debugging tools.)

Otherwise, you render a JSON response containing information about the error.
You do so using an `internal_server_error` Jbuilder view located in
__app/views/api/errors/__, saving the error object to an instance variable.

Go ahead and create this view:

```rb
# app/views/api/errors/internal_server_error.json.jbuilder

json.title 'Server Error'
json.message @error.message
json.stack @error.full_message unless Rails.env.production?
```

Now it's time to test! You'll start by testing your invalid authenticity token
response. From your Chrome console, make the following token-less POST request:

```js
> await fetch('/api/test', { method: 'POST' }).then(res => res.json())
// should return `{ message: 'Invalid authenticity token' }`
```

Then, test your unhandled error response. Add an undefined variable, `banana`,
at the top of your `sessions#show` controller action. Then, make this request:

```js
> error = await fetch('/api/session').then(res => res.json())
```

You should get a JSON response with a status code of 500. If you
`console.log(error.stack)`, you can even read through the stack trace. Nice! Go
ahead and remove `banana` from `sessions#show`.

## Wrapping up

Awesome work! You just finished setting up the entire backend for this project!
In the next part, you will implement the React frontend.

[filters]: https://guides.rubyonrails.org/v7.0.0/action_controller_overview.html#filters
[callback]: https://guides.rubyonrails.org/v7.0.0/active_record_callbacks.html
[faker]: https://github.com/faker-ruby/faker
[tap]: https://ruby-doc.org/core-3.0.2/Kernel.html#method-i-tap
[`has_secure_password`]: https://api.rubyonrails.org/classes/ActiveModel/SecurePassword/ClassMethods.html 
[`allow_nil`]: https://guides.rubyonrails.org/v7.0.0/active_record_validations.html#allow-nil
[`format`]: https://guides.rubyonrails.org/v7.0.0/active_record_validations.html#format
[validations-guide]: https://guides.rubyonrails.org/v7.0.0/active_record_validations.html
[base64]: https://ruby-doc.org/stdlib-3.0.2/libdoc/securerandom/rdoc/Random/Formatter.html#method-i-base64
[`find_by`]: https://api.rubyonrails.org/v7.0.0/classes/ActiveRecord/FinderMethods.html#method-i-find_by
[`String`]: https://ruby-doc.org/core-3.0.2/String.html
[`Regexp`]: https://ruby-doc.org/core-3.0.2/Regexp.html
[`update!`]: https://api.rubyonrails.org/v7.0.0/classes/ActiveRecord/Persistence.html#method-i-update-21
[session]: https://guides.rubyonrails.org/v7.0.0/action_controller_overview.html#session
[slice]: https://api.rubyonrails.org/v7.0.0/classes/ActiveRecord/Core.html#method-i-slice
[strong params]: https://api.rubyonrails.org/v7.0.0/classes/ActionController/StrongParameters.html
[`wrap-parameters`]: https://api.rubyonrails.org/v7.0.0/classes/ActionController/ParamsWrapper.html
[`rescue_from`]: https://api.rubyonrails.org/v7.0.0/classes/ActiveSupport/Rescuable/ClassMethods.html#method-i-rescue_from