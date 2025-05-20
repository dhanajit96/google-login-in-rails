# Google OAuth Authentication Setup with Devise in Rails

---

## 1) Create Root Controller

```bash
rails generate controller Home index
```

This will generate the `HomeController` with an `index` action and corresponding view.

---

## 2) Add Root Route

Update `config/routes.rb`:

```ruby
get "home/index"
root "home#index"
```

---

## 3) Add Gems to `Gemfile`

```ruby
# Authentication and OAuth gems
gem "devise"
gem "dotenv" # To manage environment variables

gem "omniauth"
gem "omniauth-google-oauth2"
gem "omniauth-rails_csrf_protection", "~> 1.0"
```

---

## 4) Install Bundled Gems

```bash
bundle install
```

---

## 5) Install Devise

```bash
rails generate devise:install
```

---

## 6) Configure Mailer for Devise

In `config/environments/development.rb`, add:

```ruby
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
```

---

## 7) Add Notice and Alert Flash Messages

In `app/views/layouts/application.html.erb`, inside `<body>`, add:

```erb
<p class="notice"><%= notice %></p>
<p class="alert"><%= alert %></p>
```

---

## 8) Generate Devise Views

```bash
rails g devise:views
```

---

## 9) Create Devise User Model

```bash
rails generate devise User
```

---

## 10) Add Custom Fields to Users Table

Edit the generated migration file:

```ruby
# custom fields
t.string :full_name
t.string :uid
t.string :avatar_url
t.string :provider
```

Then run:

```bash
rails db:migrate
```

---

## 11) Generate Devise Users Controllers for Customization

```bash
rails generate devise:controllers users
```

---

## 12) Update Devise Routes for Custom Controllers

Update `config/routes.rb`:

```ruby
devise_for :users, controllers: {
  omniauth_callbacks: 'users/omniauth_callbacks',
  sessions: 'users/sessions',
  registrations: 'users/registrations'
}
```

---

## 13) Add `.env` to `.gitignore` and `.dockerignore`

Update both `.gitignore` and `.dockerignore`:

```
.env
```

---

## 14) Add Google OAuth Keys to `.env` File

Create or update `.env`:

```env
GOOGLE_OAUTH_CLIENT_ID=example
GOOGLE_OAUTH_CLIENT_SECRET=pass
```

---

## 15) Update User Model for OmniAuth

In `app/models/user.rb`:

```ruby
class User < ApplicationRecord
  devise :omniauthable, :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable, omniauth_providers: [:google_oauth2]

  def self.from_omniauth(auth)
    where(provider: auth.provider, uid: auth.uid).first_or_create do |user|
      user.email = auth.info.email
      user.password = Devise.friendly_token[0, 20]
      user.full_name = auth.info.name
      user.avatar_url = auth.info.image

      if user.save
        Rails.logger.info "User saved successfully: #{user.inspect}"
      else
        Rails.logger.error "User save failed: #{user.errors.full_messages}"
      end
    end
  end
end
```

---

## 16) Add Google OAuth Config to Devise

In `config/initializers/devise.rb`, add:

```ruby
config.omniauth :google_oauth2,
                ENV["GOOGLE_OAUTH_CLIENT_ID"],
                ENV["GOOGLE_OAUTH_CLIENT_SECRET"],
                scope: "email,profile"
```

---

## 17) Add `google_oauth2` Method to OmniAuthCallbacks Controller

In `app/controllers/users/omniauth_callbacks_controller.rb`:

```ruby
def google_oauth2
  user = User.from_omniauth(auth)

  if user.present?
    sign_out_all_scopes
    flash[:success] = t "devise.omniauth_callbacks.success", kind: "Google"
    sign_in_and_redirect user, event: :authentication
  else
    flash[:alert] = t "devise.omniauth_callbacks.failure", kind: "Google", reason: "#{auth.info.email} is not authorized."
    redirect_to new_user_session_path
  end
end

private

def auth
  @auth ||= request.env["omniauth.auth"]
end
```

---

## 18) Customize User Registration Update Logic

In `app/controllers/users/registrations_controller.rb`:

```ruby
def update_resource(resource, params)
  if resource.provider == 'google_oauth2'
    params.delete('current_password')
    resource.password = params['password']
    resource.update_without_password(params)
  else
    resource.update_with_password(params)
  end
end
```

---

## 19) Customize Sessions Controller Redirects

In `app/controllers/users/sessions_controller.rb`:

```ruby
def after_sign_out_path_for(_resource_or_scope)
  new_user_session_path
end

def after_sign_in_path_for(resource_or_scope)
  stored_location_for(resource_or_scope) || root_path
end
```

---

## 20) Update OmniAuth Links in Devise Shared View

In `app/views/devise/shared/_links.html.erb`:

```erb
<%- if devise_mapping.omniauthable? %>
  <%- resource_class.omniauth_providers.each do |provider| %>
    <%= form_for 'login', url: omniauth_authorize_path(resource_name, provider), method: :post, data: { turbo: false } do |f| %>
      <%= f.submit 'Login', type: "image", src: image_path("images/web_light_sq_SU@4x.png"), width: 150, height: 40 %>
    <% end %>
  <% end %>
<% end %>
```

---

## 21) Display Authenticated User Info on Home Page

In `app/views/home/index.html.erb`:

```erb
<h1>Welcome to My Auth App</h1>

<% if user_signed_in? %>
  <p>Hello, <%= current_user.full_name %></p>
  <%- if current_user.avatar_url %>
    <%= image_tag(current_user.avatar_url) %>
  <% end %>
  <%= form_with url: destroy_user_session_path, method: :delete, data: { turbo: false } do %>
    <%= submit_tag 'Logout' %>
  <% end %>
<% else %>
  <%= link_to 'Login', new_user_session_path %> |
  <%= link_to 'Sign Up', new_user_registration_path %>
<% end %>
```
