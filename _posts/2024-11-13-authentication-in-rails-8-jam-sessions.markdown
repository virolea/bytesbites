---
layout: post
title:  "Authentication in Rails 8: Jam sessions"
excerpt: "A new authentication is available with Rails 8. One area of interest is its DB-backed session system. In this article, we explore three real-life features enabled by this new system."
image: "assets/images/social-cards/authentication-in-rails-8.png"
author: Vincent Rolea
tags: rails authentication
---

# Authentication in Rails 8: Jam sessions

[Rails 8 is out](https://rubyonrails.org/2024/11/7/rails-8-no-paas-required)! Along with the plethora of goodies (Kamal 2, the Solid libraries and production-ready SQLite), a complete authentication system shipped with this brand new major version.

This new feature offer everything needed to run a “session-based, password-resettable, metadata-tracking” authentication system. This release timed perfectly as I started to work on a new product last month to save all the cool resources, articles and videos I stumble upon online, called [Laterbox](https://laterbox.io). I decided to use this new authentication system instead of relying on good ol’ [Devise](https://github.com/heartcombo/devise) as I was interested in the possibilities offered by using a separate DB-backed session object in the Rails 8 authentication system.

In this article we’ll explore three real-life features from [Laterbox](https://laterbox.io) built on top of sessions. But first, let’s look into more details at how the session records are managed in the system.

## Authentication with DB-backed sessions objects

Each time someone logs into the application, a new session record is created:

```ruby
# app/controllers/concerns/authentication.rb

def start_new_session_for(user)
  user.sessions.create!(user_agent: request.user_agent, ip_address: request.remote_ip).tap do |session|
    Current.session = session
    cookies.signed.permanent[:session_id] = { value: session.id, httponly: true, same_site: :lax }
  end
end
```

On subsequent requests, we lookup the session_id in the cookies - should it exist - and lookup and existing corresponding  session in our database:

```ruby
# app/controllers/concerns/authentication.rb

def resume_session
  Current.session = find_session_by_cookie
end

def find_session_by_cookie
  Session.find_by(id: cookies.signed[:session_id])
end
```

Why would a session be missing from the DB? This is how signing-out works, we destroy the session objects:

```ruby
# app/controllers/concerns/authentication.rb
def terminate_session
  Current.session.destroy
  cookies.delete(:session_id)
end

# app/controllers/sessions_controller.rb
def destroy
  terminate_session
  redirect_to new_session_url
end
```

Comparatively, Devise stores the `user_id` in the cookies. The user_id will never be invalid. As a result, a DB-backed session object offers more security, as destroying a session invalidates  it, it cannot be reused. Moreover, a new session object is created for every device I log into my app with, which offers more visibility and granularity.

Let’s see how we can take advantage of this.

## Feature 1: Session management

Since a new session object is created every time I log into the application, I can build a system to help users manage it.

The system would help:
- Gather information on my current sessions: their number, the user-agent and the login IP address
- Invalidate a given session, to log out from a specific device
- See my current session

We saw previously that sessions are created in the database on each sign in, and destroyed after signing out. All we need here is one controller and two actions: index and destroy.

```ruby
class Users::SessionsController < ApplicationController
  def index
    @sessions = Current.user.sessions
  end

  def destroy
    session = Current.user.sessions.find(params[:id])
    session.destroy
    redirect_back
  end
end
```

Let’s create a UI for this. This is the code is extracted from the session management feature from [Laterbox](https://laterbox.io):

```erb
<div class="space-y-2 mt-4">
  <% @sessions.each do |session| %>
    <div class="bg-white p-4 rounded border">
      <div>
        <h4 class="font-medium text-gray-900"><%= session.user_agent %></h4>
        <p class="text-gray-700 text-sm mt-2">Created: <%= session.created_at.strftime("%b %d, %Y") %></p>

        <div class="flex space-x-2">
          <% if session == Current.session %>
            <p class="text-green-500">
              current session
            </p>
          <% end %>

          <%= button_to "Revoke session",
            user_session_path(session),
            method: :delete,
            class: "text-red-600 hover:underline" %>
        </div>
      </div>
    </div>
  <% end %>
</div>
```

And here’s the result:

![Screenshot of laterbox sessions](/assets/images/authentication-in-rails-8-article/laterbox-sessions.png)


This offers a clear view of the user’s sessions, where and when they logged in, and helps them revoke a given session if needed.

## Send a notification email when a new sign-in is detected

Have you ever used a service that sends you an email every time a new login happens with your credentials? Me too. This feature is not exclusive to using DB-backed sessions, however it’s a breeze to implement in this particular setup.

From earlier:

> Each time someone logs into the application, a new session record is created

Thus, I can notify users of a new sign-in every time a new Session object is created. There are plenty of ways for implementing it, however for simplicity’s sake, let’s do it using a callback:

```ruby
class Session < ApplicationRecord
  belongs_to :user

  after_create_commit :notify

  private

  def notify
    SessionsMailer.new_session(self).deliver_later
  end
end
```

And voilà.

![Screenshot of laterbox sessions](/assets/images/authentication-in-rails-8-article/laterbox-new-signin-email.png)

Now, what if you’re not the one who logged in? What if your account was compromised? Our third feature will offer an answer.

## Revoking all sessions after a password change

In case your account is compromised, two things need to be done:
- Reset your password
- Invalidate all user sessions.

If you are using Devise and storing sessions in the `ActionDispatch::CookieStore`, you simply can’t achieve the latter.

With the authentication system from Rails 8, it’s pretty easy. Let’s have a look at the feature we’ll build:

![Screenshot of laterbox sessions](/assets/images/authentication-in-rails-8-article/laterbox-password-form.png)

Notice the “Reset existing sessions” checkbox at the bottom of the Password form? By toggling it, all session but the current will be revoked after updating the password. Hence an attacker that managed to log in with your old password will be automatically logged out.

Here’s the form:

```erb
<%= form_with model: @user, url: user_password_path do |f| %>
  <div class="space-y-4">
    <div>
      <%= f.label :password_challenge, "Current password", class: "label" %>
      <%= f.password_field :password_challenge,
        class: "input",
        required: true %>
    </div>

    <div>
      <%= f.label :password, "New password", class: "label" %>
      <%= f.password_field :password,
        class: "input",
        pattern: ".{8,}",
        title: "8 characters minimum",
        required: true %>
      <p class="text-xs text-gray-600">Minum length: 8</p>
    </div>

    <div>
      <%= f.label :reset_sessions, "Reset existing sessions", class: "label" %>
      <%= f.check_box :reset_sessions %>
    </div>

    <div class="text-right">
      <%= f.submit "Update password", class: "btn btn-primary" %>
    </div>
  </div>
<% end %>
```

The `password_challenge` comes from the [`has_secure_password`](https://api.rubyonrails.org/classes/ActiveModel/SecurePassword/ClassMethods.html#method-i-has_secure_password) class method, and its value is expected to be the current account password (hence labelled as “Current password”).

Form is submitted to a dedicated controller, that pass the form data to the `User#update_password` method shown below.

```ruby
# app/controllers/users/passwords_controller.rb
class Users::PasswordsController < ApplicationController
  def update
    Current.user.update_password(
      password: password_params[:password],
      password_challenge: password_params[:password_challenge],
      reset_sessions: password_params[:reset_sessions]
    )

    redirect_to edit_user_settings_path
  end

  private

  def password_params
    params.require(:user).permit(:password, :password_challenge, :reset_sessions)
  end
end
```

```ruby
# app/models/user.rb
def update_password(password:, password_challenge:, reset_sessions: false)
  raise ArgumentError, "password_challenge is required" if password_challenge.blank?

  if update(password: password, password_challenge: password_challenge)
    sessions.where.not(id: Current.session).destroy_all if reset_sessions == "1"
  end
end
```

Notice the raised `ArgumentError` below in case the `password_challenge` is missing. We need it as the challenge verification will not be triggered in case the value is not present, as described in the [docs](https://api.rubyonrails.org/classes/ActiveModel/SecurePassword/ClassMethods.html#method-i-has_secure_password):

> Additionally, a XXX_challenge attribute is created. When set to a value other than nil, it will validate against the currently persisted password

By toggling the checkbox, I’ll ensure all other sessions for my account are revoked upon a successful password reset.

## Final words

The new Rails 8 Authentication system offers everything to get you running quickly. But more importantly, its DB-backed session records enable the development of user-friendly and secure new features, in the likes of session management, sign-in notifications and session revocation upon password reset.

I am pretty sure there’s a lot more to build on top of it, and the fact the code is available right into your app makes it even more easy.
