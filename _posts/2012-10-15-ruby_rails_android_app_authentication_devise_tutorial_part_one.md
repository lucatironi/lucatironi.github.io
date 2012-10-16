---
layout: post
title: Ruby on Rails and Android Authentication Part One
tagline: The Rails Web Application
category: tutorial
tags: [ruby on rails, android, devise, authentication]
comments: true
---
{% include JB/setup %}

## Intro
In this two-part tutorial you'll learn first how to build an authentication API that can allow external users to register, login and logout through JSON requests. After having successfully logged in, a user will receive an authentication token that could be used in following API requests to authorize the user, securing the access to your application's resources.

In the second part, you will build an Android application that will be able to consume this API, allowing the user to register and login directly from the app.

With these tutorials I want to explain, with a step-by-step approach, how to build a complete solution that can be used as a base for a more complex scenario.

Supplemental bonuses will be given to enhance both the backend and the Android app with additional nice-to-have features.

Let's start!

### Ingredients
Here's the list of what we are going to use in this tutorial:

- Rails 3.2 - [rubyonrails.org](https://rubyonrails.org)
- Devise - [github.com/plataformatec/devise](https://github.com/plataformatec/devise)
- ActiveAdmin - [activeadmin.info](http://activeadmin.info)
- Heroku - [heroku.com](http://heroku.com)
- Android 4.1 (API 16) - [developer.android.com](http://developer.android.com)
- UrlJsonAsyncTask - [github.com/tonylukasavage/com.savagelook.android](https://github.com/tonylukasavage/com.savagelook.android)
- ActionBarSherlock - [actionbarsherlock.com](http://actionbarsherlock.com)

## The Rails Backend
Start by creating a new Rails app. At the moment I'm writing the latest version of rails is 3.2.8, check yours by using <code>rails -v</code> in the command line.

{% highlight bash %}
$ rails new authexample_webapp
$ cd authexample_webapp
{% endhighlight %}

### Devise Setup
Add the Devise gem to your application:

{% highlight ruby %}
# file: Gemfile
gem 'devise'
{% endhighlight %}

Install the gems and create the default user model with the Devise generators.

{% highlight bash %}
$ bundle install
$ rails generate devise:install
$ rails generate devise user
{% endhighlight %}

Uncomment the following lines in the migration that are relative to the <code>token_authenticatable</code> module:

{% highlight ruby %}
# file: db/migrate/<timestamp>_devise_create_users.rb
## Token authenticatable
t.string :authentication_token
add_index :users, :authentication_token, :unique => true
{% endhighlight %}

Add the <code>:token_authenticatable</code> to the devise modules in the user model:

{% highlight ruby %}
# file: app/models/user.rb
devise :database_authenticatable, :registerable,
       :recoverable, :rememberable, :trackable, :validatable,
       :token_authenticatable
{% endhighlight %}

Add the before filter <code>:ensure_authentication_token</code> to the user model:

{% highlight ruby %}
# file: app/models/user.rb
before_save :ensure_authentication_token
{% endhighlight %}

And finally uncomment the following line in the Devise initializer to enable the auth_token:

{% highlight ruby %}
# file: config/initializers/devise.rb
# ==> Configuration for :token_authenticatable
# Defines name of the authentication token params key
config.token_authentication_key = :auth_token
{% endhighlight %}

#### Bonus: add username to the user model
In our Android app we want to give possibility to the user to specify a username in the registration form. Let's add this column to the table 'users' and the attribute to the <code>attr_accesible</code> list in the user model.

Add the following line to the <code>change</code> method in the migration.

{% highlight ruby %}
# file: db/migrate/<timestamp>_devise_create_users.rb
t.string :name, :null => false, :default => ""
{% endhighlight %}

{% highlight ruby %}
# file: app/models/user.rb
attr_accessible :name, :email, :password, :password_confirmation, :remember_me
{% endhighlight %}

#### Bonus: email confirmation from web, skip it from the Android app
The users of our Android app don't want to wait to use it, so we skip the confirmation email check provided by Devise with the <code>confirmable</code> module.
If you still want to use the module for the users that register from the webapp, just add this lines to the code.

Uncomment the following lines in the migration that are relative to the <code>confirmable</code> module:

{% highlight ruby %}
# file: db/migrate/<timestamp>_devise_create_users.rb
## Confirmable
t.string   :confirmation_token
t.datetime :confirmed_at
t.datetime :confirmation_sent_at
add_index :users, :confirmation_token,   :unique => true
{% endhighlight %}

Add the <code>:confirmable</code> to the devise available modules in the user model:

{% highlight ruby %}
# file: app/models/user.rb
devise :database_authenticatable, :registerable,
       :recoverable, :rememberable, :trackable, :validatable,
       :confirmable, :token_authenticatable
{% endhighlight %}

We need a way to bypass the confirmation step after the creation of a new user: setting a date to the <code>confirmed_at</code> will do this. We add a new method to the user model that will be used in our Api controller.

{% highlight ruby %}
# file: app/models/user.rb
def skip_confirmation!
  self.confirmed_at = Time.now
end
{% endhighlight %}

We are done with the setup of our user model, we can now launch the rake tasks to create and migrate the database:

{% highlight bash %}
$ rake db:create db:migrate
{% endhighlight %}

### API Sessions Controller (login and logout)
Let's start coding the sessions controller that will be used by our Android app to authenticate the users.
I want to use a namespace and a version for our API, so its controllers will be under the <code>app/controller/api/v1/</code> folder.

The sessions controller has two actions: create for login and destroy for logout. The first accepts a <code>user</code> JSON object as POST data with an <code>email</code> and a <code>password</code> parameters and returns an <code>auth_token</code> if the user exists in the database and the password is correct. The logout action expects an <code>auth_token</code> parameter in the url.

{% highlight ruby %}
# file: app/controller/api/v1/sessions_controller.rb
class Api::V1::SessionsController < Devise::SessionsController
  skip_before_filter :verify_authenticity_token,
                     :if => Proc.new { |c| c.request.format == 'application/json' }

  respond_to :json

  def create
    warden.authenticate!(:scope => resource_name, :recall => "#{controller_path}#failure")
    render :status => 200,
           :json => { :success => true,
                      :info => "Logged in",
                      :data => { :auth_token => current_user.authentication_token } }
  end

  def destroy
    warden.authenticate!(:scope => resource_name, :recall => "#{controller_path}#failure")
    current_user.update_column(:authentication_token, nil)
    render :status => 200,
           :json => { :success => true,
                      :info => "Logged out",
                      :data => {} }
  end

  def failure
    render :status => 401,
           :json => { :success => false,
                      :info => "Login Failed",
                      :data => {} }
  end
end
{% endhighlight %}

In the routes definition we add our namespace and the two login and logout routes.

{% highlight ruby %}
# file: config/routes.rb
namespace :api do
  namespace :v1 do
    devise_scope :user do
      post 'sessions' => 'sessions#create', :as => 'login'
      delete 'sessions' => 'sessions#destroy', :as => 'logout'
    end
  end
end
{% endhighlight %}

#### Test the login
Let's create a user to test the login with, open the rails console with <code>rails console</code> in the command line and write:

{% highlight ruby %}
user = User.new(:name => 'testuser', :email => 'user@example.com', :password => 'secret', :password_confirmation => 'secret')
user.skip_confirmation!
user.save
{% endhighlight %}

Close the console and fire up the <code>rails server</code> and in another command line use curl to invoke our new login API:

{% highlight bash %}
curl -v -H 'Content-Type: application/json' -H 'Accept: application/json' -X POST http://localhost:3000/api/v1/sessions -d "{\"user\":{\"email\":\"user@example.com\",\"password\":\"secret\"}}"
{% endhighlight %}

If everything went fine, you should see the last line saying this (the auth_token will be different):

{% highlight bash %}
{"success":true,"info":"Logged in","data":{"auth_token":"JRYodzXgrLsk157ioYHf"}}
{% endhighlight %}

#### Test the logout
Using the auth_token that the API gave us back when we logged in and specifying the <code>DELETE</code> verb we can reset the authentication token of the user, logging him out.

{% highlight bash %}
curl -v -H 'Content-Type: application/json' -H 'Accept: application/json' -X DELETE http://localhost:3000/api/v1/sessions/\?auth_token\=JRYodzXgrLsk157ioYHf
{% endhighlight %}

The result will be a nice message informing us that we are logged out.

{% highlight bash %}
{"success":true,"info":"Logged out","data":{}}
{% endhighlight %}

### API Registrations Controller (register a new user)
The registrations controller extends the Devise one and has only one action, create.
As you can see we skip the confirmation step with the method we added previously to the user model, we then save the new user and automatically log it in, returning the auth_token associated. Our user can then start using the API already logged in after the registration.

{% highlight ruby %}
# file: app/controllers/api/v1/registrations_controller.rb
class Api::V1::RegistrationsController < Devise::RegistrationsController
  skip_before_filter :verify_authenticity_token,
                     :if => Proc.new { |c| c.request.format == 'application/json' }

  respond_to :json

  def create
    build_resource
    resource.skip_confirmation!
    if resource.save
      sign_in resource
      render :status => 200,
           :json => { :success => true,
                      :info => "Registered",
                      :data => { :user => resource,
                                 :auth_token => current_user.authentication_token } }
    else
      render :status => :unprocessable_entity,
             :json => { :success => false,
                        :info => resource.errors,
                        :data => {} }
    end
  end
end
{% endhighlight %}

Add the register route to the API namespace:

{% highlight ruby %}
# file: config/routes.rb
namespace :api do
  namespace :v1 do
    devise_scope :user do
      post 'registrations' => 'registrations#create', :as => 'register'
      post 'sessions' => 'sessions#create', :as => 'login'
      delete 'sessions' => 'sessions#destroy', :as => 'logout'
    end
  end
end
{% endhighlight %}

#### Test the registration
Using the code we just added, we can now register new users from the JSON API. Try it out opening a command line and pasting this code.

{% highlight bash %}
curl -v -H 'Content-Type: application/json' -H 'Accept: application/json' -X POST http://localhost:3000/api/v1/registrations -d "{\"user\":{\"email\":\"user1@example.com\",\"name\":\"anotheruser\",\"password\":\"secret\",\"password_confirmation\":\"secret\"}}"
{% endhighlight %}

If everything went fine, you should see the last line saying this (the id, dates and auth_token will be different):

{% highlight bash %}
{"success":true,"info":"Registered","data":{"user":{"created_at":"2012-10-08T19:57:20Z","email":"user1@example.com","id":3,"name":"anotheruser","updated_at":"2012-10-08T19:57:20Z"},"auth_token":"N8N5MPqFNdDz3G1jRsC9"}}
{% endhighlight %}

### ActiveAdmin administrative interface
Just to ease the following steps, I wanted to introduce a simple administrative interface to our web application: let's add the ActiveAdmin gem (with dependencies) and with few lines of code we'll have a full featured admin area.

{% highlight ruby %}
# file: Gemfile
gem 'activeadmin'
gem 'meta_search', '>= 1.1.0.pre' # activeadmin needs this if Rails >= 3.1
{% endhighlight %}

Don't forget to launch the <code>bundle install</code> command.
ActiveAdmin will then generate the configuration file and some migrations. You can find more information in the official documentation: www.activeadmin.info

{% highlight bash %}
$ bundle install
$ rails generate active_admin:install
$ rake db:migrate
{% endhighlight %}

Add a new file to the <code>app/admin</code> folder to configure the users admin interface:

{% highlight ruby %}
# file: app/admin/user.rb
ActiveAdmin.register User do
  index do
    column :name
    column :email
    column :current_sign_in_at
    column :last_sign_in_at
    column :sign_in_count
    default_actions
  end

  filter :name
  filter :email

  form do |f|
    f.inputs "User Details" do
      f.input :name
      f.input :email
      f.input :password
      f.input :password_confirmation
    end
    f.buttons
  end

  show do
    attributes_table do
      row :name
      row :email
      row :authentication_token
      row :confirmed_at
      row :current_sign_in_at
      row :last_sign_in_at
      row :sign_in_count
    end
    active_admin_comments
  end
end
{% endhighlight %}

Launch the <code>rails server</code> and go to [http://localhost:3000/admin](http://localhost:3000/admin) and login with the default ActiveAdmin credentials:
- User: admin@example.com
- Password: password

#### Bonus: Deploy to Heroku
In the next steps we will be building an Android app from scratch that will consume our API. To better test it on our smartphones, we will need the web app reachable from the web, what better place than Heroku?

This tutorial won't delve to much on the details of deploying a Rails app on the service, but you can read more on the official documentation: [devcenter.heroku.com](http://devcenter.heroku.com).

Download the [Heroku toolbelt](https://toolbelt.heroku.com) and create an account if you don't have one.

Let's start creating a Git repository and pushing it to Heroku. Take note of the app's name that Heroku will create for you (something.herokuapp.com).

{% highlight bash %}
$ rm public/index.html
$ git init
$ git add .
$ git commit -m "Initial commit"
$ heroku apps:create
$ git push heroku master
$ heroku run rake db:migrate
{% endhighlight %}

Now go to the address that Heroku created for your app and see if everything worked or not. More details on issues can be spotted with the <code>heroku logs</code> command.

## Part Two: The Android App
Proceed now to the [second part of the tutorial](/tutorial/2012/10/16/ruby_rails_android_app_authentication_devise_tutorial_part_two), to learn how to build a fully featured Android App that can interact with the Ruby on Rails backend we just coded.