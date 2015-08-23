---
layout: post
title: Using rails-api to build an authenticated JSON API with warden
tagline: An updated version of my previous tutorials on building an authenticated JSON API with Ruby on Rails
description: Two years ago I published a series of tutorials to explain how to build a JSON API with Ruby on Rails and setting up an authentication with Devise. This new tutorial uses a test driven approach (RSpec) and rails-api with warden, so we can now build the same backend with even less code.
category: tutorial
tags: [Ruby on Rails, Warden, Ruby, authentication, API]
---

In this tutorial I will build a small web application that provides a JSON API to manage <code>customers</code> through a REST interface. The requests to the endpoints will be authenticated through a token based authentication strategy, passing custom headers (<code>X-User-Email</code> and <code>X-Auth-Token</code>) containing the user's credentials.

A <code>sessions</code> endpoint is available to issue a new authentication token on login and disposing it on logout.

The goal of the tutorial is building the base of an up-to-date, well tested, minimal and functional backend API that can be used for client applications such as Angular, Ember web apps or even Mobile apps. Have a look to the previous tutorials to have an idea of the differences with the last examples.

## Requirements

* [Rails::API](https://github.com/rails-api/rails-api) is a subset of a normal Rails application, created for applications that don't require all functionality that a complete Rails application provides.
* [Warden](https://github.com/hassox/warden) provides a mechanism for authentication in Rack based Ruby applications. It is used by many other libraries and gem, like [Devise](https://github.com/plataformatec/devise).
* [RSpec](https://github.com/rspec/rspec-rails) is a gem to do Behaviour Driven Development in Ruby (on Rails). We will use the Rails helpers for our models, controllers, routes and integration tests.

The complete code for this tutorial can be found on my [Github account](https://github.com/lucatironi/example_rails_api).

## Setup

Let's start by installing the [rails-api gem](https://github.com/rails-api/rails-api) and creating the app using the <code>rails-api</code> command in the terminal.

{% highlight bash %}
gem install rails-api

rails-api new example_api --skip-turbolinks --skip-sprockets --skip-test-unit --skip-javascript
{% endhighlight %}

I issued the command with some options to skip unused functionalities like turbolinks and sprockets, since we will not have a "frontend". I also disabled the default <code>test-unit</code> framework since we will use RSpec for our tests.

## Testing

Speaking of RSpec, let's start the development of our new app by installing and configuring it. First add the gem and some companion gems like [shoulda-matchers](https://github.com/thoughtbot/shoulda-matchers) and the [spring-commands-rspec](https://github.com/jonleighton/spring-commands-rspec) to enable the usage of RSpec with Spring.

{% highlight ruby %}
# file: Gemfile
source 'https://rubygems.org'

gem 'rails', '4.2.3'
gem 'rails-api'
gem 'sqlite3'

group :development do
  gem 'spring'
  gem 'spring-commands-rspec'
end

group :test do
  gem 'shoulda-matchers', require: false
end

group :development, :test do
  gem 'rspec-rails', '~> 3.0'
end
{% endhighlight %}

Install all the gems, run the installation of RSpec to generate the <code>spec_helper.rb</code> and <code>rails_helper.rb</code> files and finally create the <code>bin/rspec</code> binstub.

{% highlight bash %}
bundle install

bin/rails generate rspec:install

bundle exec spring binstub rspec
{% endhighlight %}

For more information about Spring, refer to the [official documentation](https://github.com/rails/spring).

## Customers

Once we have our testing framework up and running, we can start by creating our first model. I will use a <code>Customer</code> resource as an example to build the first (an only) endpoint. If you are building a real application, please create your model(s) accordingly.

### Model

Use the rails generator to create the model and the migration. Our model will have three attributes: <code>full_name</code>, <code>email</code> and <code>phone</code>. Remember to run the migration after the files are autmatically generated.

{% highlight bash %}
bin/rails generate model customer full_name:string email:string phone:string

bin/rake db:migrate
{% endhighlight %}

I will use [shoulda-matchers](https://github.com/thoughtbot/shoulda-matchers) to test our models, specifying which database columns (and attributes) it should have. I add also the matcher for the validation that the model is expected to have.

{% highlight ruby %}
# file: spec/models/customer_spec.rb
require 'rails_helper'

RSpec.describe Customer, type: :model do

  describe "db structure" do
    it { is_expected.to have_db_column(:full_name).of_type(:string) }
    it { is_expected.to have_db_column(:email).of_type(:string) }
    it { is_expected.to have_db_column(:phone).of_type(:string) }
    it { is_expected.to have_db_column(:created_at).of_type(:datetime) }
    it { is_expected.to have_db_column(:updated_at).of_type(:datetime) }
  end

  describe "validations" do
    it { is_expected.to validate_presence_of(:full_name) }
  end

end
{% endhighlight %}

Run the specs with <code>bin/rspec</code> and see them fail once. Add the <code>Customer</code> model and relaunch the specs.

{% highlight ruby %}
# file: app/models/customer.rb
class Customer < ActiveRecord::Base
  validates_presence_of :full_name
end
{% endhighlight %}

{% highlight bash %}
bin/rspec
{% endhighlight %}

The output of the specs should be similar to this one:

{% highlight bash %}
$ bin/rspec

......

Finished in 0.02619 seconds (files took 0.49037 seconds to load)
6 examples, 0 failures
{% endhighlight %}

If everything looks fine, let's start adding the first endpoint to our API.

### Routing

Start by using again the rails generator to create the <code>CustomersController</code> and related routing specs.

{% highlight bash %}
bin/rails generate controller customers index show create update destroy --no-view-specs --skip-routes
{% endhighlight %}

I will start by adding the routing specs to be sure that our controller is routable as expected.

{% highlight ruby %}
# file: specs/routing/customers_routing_spec.rb
require 'rails_helper'

RSpec.describe CustomersController, type: :routing do
  it { expect(get:    "/customers").to   route_to("customers#index") }
  it { expect(get:    "/customers/1").to route_to("customers#show", id: "1") }
  it { expect(post:   "/customers").to   route_to("customers#create") }
  it { expect(put:    "/customers/1").to route_to("customers#update", id: "1") }
  it { expect(delete: "/customers/1").to route_to("customers#destroy", id: "1") }
end
{% endhighlight %}

To make these specs pass we need to add the customers resource to the <code>routes.rb</code> file.

{% highlight ruby %}
# file: config/routes.rb
Rails.application.routes.draw do
  resources :customers, only: [:index, :show, :create, :update, :destroy]
end
{% endhighlight %}

Launch again the <code>bin/rspec</code> command and see the specs pass.

### Controller

It's time to add real value to our API. Let's start by defining how our controller is expected to behave when is issued the canonical REST actions: <code>index</code>, <code>show</code>, <code>create</code>, <code>update</code> and <code>destroy</code>.

{% highlight ruby %}
# file: specs/controllers/customers_controller_spec.rb
require 'rails_helper'

RSpec.describe CustomersController, type: :controller do

  # This should return the minimal set of attributes required to create a valid
  # Customer. As you add validations to Customer, be sure to
  # adjust the attributes here as well.
  let(:valid_attributes) {
    { full_name: "John Doe", email: "john.doe@example.com", phone: "123456789" }
  }

  let(:invalid_attributes) {
    { full_name: nil, email: "john.doe@example.com", phone: "123456789" }
  }

  let!(:customer) { Customer.create(valid_attributes) }

  describe "GET #index" do
    it "assigns all customers as @customers" do
      get :index, { format: :json }
      expect(assigns(:customers)).to eq([customer])
    end
  end

  describe "GET #show" do
    it "assigns the requested customer as @customer" do
      get :show, { id: customer.id, format: :json }
      expect(assigns(:customer)).to eq(customer)
    end
  end

  describe "POST #create" do
    context "with valid params" do
      it "creates a new Customer" do
        expect {
          post :create, { customer: valid_attributes, format: :json  }
        }.to change(Customer, :count).by(1)
      end

      it "assigns a newly created customer as @customer" do
        post :create, { customer: valid_attributes, format: :json  }
        expect(assigns(:customer)).to be_a(Customer)
        expect(assigns(:customer)).to be_persisted
      end
    end

    context "with invalid params" do
      it "assigns a newly created but unsaved customer as @customer" do
        post :create, { customer: invalid_attributes, format: :json  }
        expect(assigns(:customer)).to be_a_new(Customer)
      end

      it "returns unprocessable_entity status" do
        put :create, { customer: invalid_attributes }
        expect(response.status).to eq(422)
      end
    end
  end

  describe "PUT #update" do
    context "with valid params" do
      let(:new_attributes) {
        { full_name: "John F. Doe", phone: "234567890" }
      }

      it "updates the requested customer" do
        put :update, { id: customer.id, customer: new_attributes, format: :json  }
        customer.reload
        expect(customer.full_name).to eq("John F. Doe")
        expect(customer.phone).to eq("234567890")
      end

      it "assigns the requested customer as @customer" do
        put :update, { id: customer.id, customer: valid_attributes, format: :json  }
        expect(assigns(:customer)).to eq(customer)
      end
    end

    context "with invalid params" do
      it "assigns the customer as @customer" do
        put :update, { id: customer.id, customer: invalid_attributes, format: :json  }
        expect(assigns(:customer)).to eq(customer)
      end

      it "returns unprocessable_entity status" do
        put :update, { id: customer.id, customer: invalid_attributes, format: :json }
        expect(response.status).to eq(422)
      end
    end
  end

  describe "DELETE #destroy" do
    it "destroys the requested customer" do
      expect {
        delete :destroy, { id: customer.id, format: :json  }
      }.to change(Customer, :count).by(-1)
    end

    it "redirects to the customers list" do
      delete :destroy, { id: customer.id, format: :json  }
      expect(response.status).to eq(204)
    end
  end

end
{% endhighlight %}

The controller specs are long and detailed, but they are covering all the possible expectations about the controller. We are testing that our index actions return the right collection of <code>Customer</code> objects, the show action retrieves the right object, create is persisting a new object only if it's valid and update is modifying accordingly to the given new attributes. Finally we check that destroy actually deletes the given record from the database.

{% highlight ruby %}
# file: app/controllers/customers_controller.rb
class CustomersController < ApplicationController
  before_action :set_customer, only: [:show, :update, :destroy]

  def index
    @customers = Customer.all

    render json: @customers
  end

  def show
    render json: @customer
  end

  def create
    @customer = Customer.new(customer_params)

    if @customer.save
      render json: @customer, status: :created, location: @customer
    else
      render json: @customer.errors, status: :unprocessable_entity
    end
  end

  def update
    @customer = Customer.find(params[:id])

    if @customer.update(customer_params)
      head :no_content
    else
      render json: @customer.errors, status: :unprocessable_entity
    end
  end

  def destroy
    @customer.destroy

    head :no_content
  end

  private

    def set_customer
      @customer = Customer.find(params[:id])
    end

    def customer_params
      params.require(:customer).permit(:full_name, :email, :phone)
    end

end
{% endhighlight %}

The actual code for the controller is as simple as this: not so different from the one you can get from the rails scaffold command. We are returning the objects as JSON and rendering also the errors as JSON in case of failed validations or exceptions.

#### Adding some real data

We tested our controller with the specs, but it's now time to see the actual result in action. Let's add some fake data with the seeds.

{% highlight ruby %}
# file: db/seeds.rb
[
  { full_name: "John Doe",   email: "john.doe@example.com",   phone: "033 1234 5678"},
  { full_name: "Mark Smith", email: "mark.smith@example.com", phone: "034 6789 1234"},
  { full_name: "Tom Clark",  email: "tom.clark@example.com",  phone: "033 4321 9876"},
  { full_name: "Sue Palmer", email: "sue.palmer@example.com", phone: "034 9876 1234"},
  { full_name: "Kate Lee",   email: "kate.lee@example.com",   phone: "033 6789 4321"}
].each do |customer_attributes|
  Customer.create(customer_attributes)
end
{% endhighlight %}

Run the rake command to insert the seeds in your database and start the rails application server.

{% highlight bash %}
bin/rake db:seed

bin/rails s
{% endhighlight %}

Visit [localhost:3000/customers.json](http://localhost:3000/customers.json) to see the result:

{% highlight json %}
[
  {
    id: 1,
    full_name: "John Doe",
    email: "john.doe@example.com",
    phone: "033 1234 5678",
    created_at: "2015-08-22T18:11:46.572Z",
    updated_at: "2015-08-22T18:11:46.572Z"
  }, {
    id: 2,
    full_name: "Mark Smith",
    email: "mark.smith@example.com",
    phone: "034 6789 1234",
    created_at: "2015-08-22T18:11:46.584Z",
    updated_at: "2015-08-22T18:11:46.584Z"
  }, {
    id: 3,
    full_name: "Tom Clark",
    email: "tom.clark@example.com",
    phone: "033 4321 9876",
    created_at: "2015-08-22T18:11:46.587Z",
    updated_at: "2015-08-22T18:11:46.587Z"
  }, {
    id: 4,
    full_name: "Sue Palmer",
    email: "sue.palmer@example.com",
    phone: "034 9876 1234",
    created_at: "2015-08-22T18:11:46.591Z",
    updated_at: "2015-08-22T18:11:46.591Z"
  }, {
    id: 5,
    full_name: "Kate Lee",
    email: "kate.lee@example.com",
    phone: "033 6789 4321",
    created_at: "2015-08-22T18:11:46.595Z",
    updated_at: "2015-08-22T18:11:46.595Z"
  }
]
{% endhighlight %}

You should see the JSON payload above as expected.

#### Using ActiveModel::Serializer to build the JSON response

Looking at the payload we just created, you can see that we don't have control on which data we would like to expose. To do so, we will use the [ActiveModel::Serializer gem](https://github.com/rails-api/active_model_serializers), created as a companion of the rails-api project.

{% highlight ruby %}
# file: Gemfile
gem 'active_model_serializers', github: 'rails-api/active_model_serializers'
{% endhighlight %}

Add the gem to Gemfile and bundle it. Use the rails generator provided to add the <code>CustomerSerializer</code> model.

{% highlight bash %}
bundle install

bin/rails generate serializer customer
{% endhighlight %}

Add the attributes that we want to return as JSON:

{% highlight ruby %}
# file: app/serializers/customer_serializer.rb
class CustomerSerializer < ActiveModel::Serializer
  attributes :id, :full_name, :email, :phone
end
{% endhighlight %}

The payload should be now like the following one, without the timestamps.

{% highlight json %}
[
  {
    id: 1,
    full_name: "John Doe",
    email: "john.doe@example.com",
    phone: "033 1234 5678"
  }, {
    id: 2,
    full_name: "Mark Smith",
    email: "mark.smith@example.com",
    phone: "034 6789 1234"
  }, {
    id: 3,
    full_name: "Tom Clark",
    email: "tom.clark@example.com",
    phone: "033 4321 9876"
  }, {
    id: 4,
    full_name: "Sue Palmer",
    email: "sue.palmer@example.com",
    phone: "034 9876 1234"
  }, {
    id: 5,
    full_name: "Kate Lee",
    email: "kate.lee@example.com",
    phone: "033 6789 4321"
  }
]
{% endhighlight %}

#### Adding better errors on exceptions

Our endpoint is getting better and better, but we will encounter some issues when some of the common exceptions are raised, like 404 on record not found or strong_parameters is returning <code>ActionController::ParameterMissing</code>: the API will return normal html without a clear way to understand the error from the client perspective. To fix this we will add some specs to define that all our controllers will behave accordingly in those cases, rescuing the exceptions and return well formatted JSON.

Let's add first a shared example to our specs. Start by enabling the usage of the <code>spec/support</code> directory where wil create our shared example file.

{% highlight ruby %}
# file: specs/rails_helper.rb
Dir[Rails.root.join('spec/support/**/*.rb')].each { |f| require f }
{% endhighlight %}

The spec will be used in controllers and defines that in case of <code>ActiveRecord::RecordNotFound</code> or <code>ActionController::ParameterMissing</code>, the payload will be JSON with the correct status code and error message.

{% highlight ruby %}
# file: specs/support/api_controller.rb
require 'rails_helper'

RSpec.shared_examples "api_controller" do

  describe "rescues from ActiveRecord::RecordNotFound" do

    context "on GET #show" do
      before { get :show, { id: 'not-existing', format: :json } }

      it { expect(response.status).to eq(404) }
      it { expect(response.body).to be_blank }
    end

    context "on PUT #update" do
      before { put :update, { id: 'not-existing', format: :json } }

      it { expect(response.status).to eq(404) }
      it { expect(response.body).to be_blank }
    end

    context "on DELETE #destroy" do
      before { delete :destroy, { id: 'not-existing', format: :json } }

      it { expect(response.status).to eq(404) }
      it { expect(response.body).to be_blank }
    end

  end

  describe "rescues from ActionController::ParameterMissing" do

    context "on POST #create" do
      before { post :create, { wrong_params: { foo: :bar }, format: :json } }

      it { expect(response.status).to eq(422) }
      it { expect(response.body).to match(/error/) }
    end

  end

end
{% endhighlight %}

Add to the customer controller specs the shared example with the <code>it_behaves_like</code> method call.

{% highlight ruby %}
# file: specs/controllers/customer_controller_spec.rb
require 'rails_helper'

RSpec.describe CustomersController, type: :controller do

  it_behaves_like "api_controller"

  # ...
end
{% endhighlight %}

If you launche specs now, they will fail because we haven't define yet the code to rescue the two exceptions. Add the following code to the <code>application_controller.rb</code>:

{% highlight ruby %}
# file: app/controllers/application_controller.rb
class ApplicationController < ActionController::API

  rescue_from ActiveRecord::RecordNotFound,       with: :not_found
  rescue_from ActionController::ParameterMissing, with: :missing_param_error

  def not_found
    render status: :not_found, json: ""
  end

  def missing_param_error(exception)
    render status: :unprocessable_entity, json: { error: exception.message }
  end

end
{% endhighlight %}

By running again the <code>bin/rspec</code> command, the specs should pass as expected and our customers endpoint will return proper JSON on exceptions.

## Authentication

The second part of the tutorial will add an authentication layer to the API. Let's start by building a service that can issue secure tokens to an existing user in order to use them to make authenticated requests to the protected endpoints.

Usually Ruby on Rails applications rely on complex setups to provide authentication (and user account management), tipically using Devise as a go-to solution. In this tutorial I would like to implment something from scratch, building the simplest working and secure authentication system possible, starting from the tools that Rails provide by default like [has_secure_password](http://api.rubyonrails.org/classes/ActiveModel/SecurePassword/ClassMethods.html#method-i-has_secure_password) and [has_secure_token](https://github.com/robertomiranda/has_secure_token) (a backport of a functionality from Rails 5) and packing everything with a custom built strategy with [Warden](https://github.com/hassox/warden).

### Setup

Let's start by adding the gems needed for the first iteration.

{% highlight ruby %}
# file: Gemfile
gem 'bcrypt', '~> 3.1.7'
gem 'has_secure_token'
{% endhighlight %}

bcrypt is needed to use the <code>has_secure_password</code> feature from Rails and <code>has_secure_token</code> will enable the automatic generation of secure and unique token in our service object.

{% highlight bash %}
bundle install
{% endhighlight %}

Install the gems with the bundle command.

### Token Issuer

We are going to create a service object to issue new authentication tokens for our logged in user, in order to have multiple authenticated sessions for a given user and being able to log out each one of them without affecting the others. Usually the authentication token is saved in the user record, enabling only one sessions at the time, since logging out and resetting the token will de facto log out every other sessions.

Let's start by enabling the autoload of files in the <code>app/services</code> directory.

{% highlight ruby %}
# file: config/application.rb

# ...

module ExampleApi
  class Application < Rails::Application
    # Settings in config/environments/* take precedence over those specified here.
    # Application configuration should go into files in config/initializers
    # -- all .rb files in that directory are automatically loaded.
    config.autoload_paths += %W(#{config.root}/app/services/**/)

    # ...

  end
end
{% endhighlight %}

Let's then create the specs for our <code>TokenIssuer</code> class. Its responsabilities are to create and return tokens (the model will come later) and purge the expired tokens of a user.

{% highlight ruby %}
# file: specs/services/token_issuer_spec.rb
require 'rails_helper'

RSpec.describe TokenIssuer, type: :model do

  let(:resource) {
    double(:resource, id: 1,
      authentication_tokens: authentication_tokens) }
  let(:authentication_tokens) {
    double(:authentication_tokens, create!: authentication_token) }
  let(:authentication_token) {
    double(:authentication_token, body: "token") }
  let(:request) {
    double(:request, remote_ip: "100.10.10.23", user_agent: "Test Browser") }

  describe ".create_and_return_token" do

    it "creates a new token for the user" do
      expect(resource.authentication_tokens).to receive(:create!)
        .with(last_used_at: DateTime.current,
          ip_address: request.remote_ip,
          user_agent: request.user_agent)
        .and_return(authentication_token)
      described_class.create_and_return_token(resource, request)
    end

    it "returns the token body" do
      allow(resource.authentication_tokens).to receive(:create!)
        .and_return(authentication_token)
      expect(described_class.create_and_return_token(resource, request)).to eq("token")
    end

  end

  describe ".purge_old_tokens" do

    it "deletes all the user's tokens" do
      expect(resource.authentication_tokens).to receive_message_chain(:order, :offset, :destroy_all)
      described_class.purge_old_tokens(resource)
    end

  end

end
{% endhighlight %}

The implementation of the class is as follow. The basic functionality is to create and return a new <code>AuthenticationToken</code> for a user, to find a token between the user's tokens and finally destroy an expired token.

A constant <code>MAXIMUM_TOKENS_PER_USER</code> overridable on initialization of the service sets how many tokens a user can keep active whenever the purge is called. This method can be used in a cron job in order to keep the unused tokens at bay.

{% highlight ruby %}
# file: app/services/token_issuer.rb
class TokenIssuer
  MAXIMUM_TOKENS_PER_USER = 20

  def self.build
    new(MAXIMUM_TOKENS_PER_USER)
  end

  def self.create_and_return_token(resource, request)
    build.create_and_return_token(resource, request)
  end

  def self.expire_token(resource, request)
    build.expire_token(resource, request)
  end

  def self.purge_old_tokens(resource)
    build.purge_old_tokens(resource)
  end

  def initialize(maximum_tokens_per_user)
    self.maximum_tokens_per_user = maximum_tokens_per_user
  end

  def create_and_return_token(resource, request)
    token = resource.authentication_tokens.create!(
      last_used_at: DateTime.current,
      ip_address:   request.remote_ip,
      user_agent:   request.user_agent)

    token.body
  end

  def expire_token(resource, request)
    find_token(resource, request.headers["X-Auth-Token"]).try(:destroy)
  end

  def find_token(resource, token_from_headers)
    resource.authentication_tokens.detect do |token|
      token.body == token_from_headers
    end
  end

  def purge_old_tokens(resource)
    resource.authentication_tokens
      .order(last_used_at: :desc)
      .offset(maximum_tokens_per_user)
      .destroy_all
  end

  private

    attr_accessor :maximum_tokens_per_user

end
{% endhighlight %}

### User and AuthenticationToken models

Let's create now the <code>User</code> and <code>AuthenticationToken</code> models.

{% highlight bash %}
bin/rails generate model authentication_token body:string user:references last_used_at:datetime ip_address:string user_agent:string

bin/rails generate model user email:string password_digest:string
{% endhighlight %}

{% highlight ruby %}
# file: db/migrate/XXX_create_authentication_tokens.rb
class CreateAuthenticationTokens < ActiveRecord::Migration
  def change
    create_table :authentication_tokens do |t|
      t.string :body
      t.references :user, index: true, foreign_key: true
      t.datetime :last_used_at
      t.string :ip_address
      t.string :user_agent

      t.timestamps null: false
    end
  end
end
{% endhighlight %}

I added an index on the <code>email</code> attribute since we will look for users through the email while doing the authentication.

{% highlight ruby %}
# file: db/migrate/XXX_create_users.rb
class CreateUsers < ActiveRecord::Migration
  def change
    create_table :users do |t|
      t.string :email, index: :email
      t.string :password_digest

      t.timestamps null: false
    end
  end
end
{% endhighlight %}

{% highlight ruby %}
# file: specs/models/authentication_token_spec.rb
require 'rails_helper'

RSpec.describe AuthenticationToken, type: :model do

  describe "db structure" do
    it { is_expected.to have_db_column(:user_id).of_type(:integer) }
    it { is_expected.to have_db_column(:body).of_type(:string) }
    it { is_expected.to have_db_column(:ip_address).of_type(:string) }
    it { is_expected.to have_db_column(:user_agent).of_type(:string) }
    it { is_expected.to have_db_column(:last_used_at).of_type(:datetime) }
    it { is_expected.to have_db_column(:created_at).of_type(:datetime) }
    it { is_expected.to have_db_column(:updated_at).of_type(:datetime) }

    it { is_expected.to have_db_index(:user_id) }
  end

  describe "associations" do
    it { is_expected.to belong_to(:user) }
  end

end
{% endhighlight %}

{% highlight ruby %}
# file: app/models/authentication_token.rb
class AuthenticationToken < ActiveRecord::Base
  belongs_to :user
  has_secure_token :body
end
{% endhighlight %}

{% highlight ruby %}
# file: specs/models/user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do

  describe "db structure" do
    it { is_expected.to have_db_column(:email).of_type(:string) }
    it { is_expected.to have_db_column(:password_digest).of_type(:string) }
    it { is_expected.to have_db_column(:created_at).of_type(:datetime) }
    it { is_expected.to have_db_column(:updated_at).of_type(:datetime) }

    it { is_expected.to have_db_index(:email) }
  end

  describe "associations" do
    it { is_expected.to have_many(:authentication_tokens) }
  end

  describe "secure password" do
    it { is_expected.to have_secure_password }
    it { is_expected.to validate_length_of(:password) }

    it { expect(User.new({ email: "user@email.com", password: nil }).save).to be_falsey }
    it { expect(User.new({ email: "user@email.com", password: "foo" }).save).to be_falsey }
    it { expect(User.new({ email: "user@email.com", password: "af3714ff0ffae" }).save).to be_truthy }
  end

end
{% endhighlight %}

{% highlight ruby %}
# file: app/models/user.rb
class User < ActiveRecord::Base
  has_many :authentication_tokens
  has_secure_password
  validates :password, length: { minimum: 8 }
end
{% endhighlight %}

The specs and the models are simple and we are just testing the attributes, validations an the <code>has_secure_password</code> and <code>has_secure_token</code> features.

### Adding Warden to handle authentication

We are coming to the core of our authentication layer. I was inspired by [this tutorial](http://blog.maestrano.com/rails-api-authentication-with-warden-without-devise/) by Oliver Brisse, so all the credits are due to him.

Start by adding the Warden gem and install it.

{% highlight ruby %}
# file: Gemfile
gem 'warden'
{% endhighlight %}

{% highlight bash %}
bundle install
{% endhighlight %}

Let's add an initializer to require and load our new strategy and setup the middleware.

{% highlight ruby %}
# file: initializers/warden.rb
require 'authentication_token_strategy'

Warden::Strategies.add(:authentication_token, AuthenticationTokenStrategy)

Rails.application.config.middleware.insert_after ActionDispatch::ParamsParser, Warden::Manager do |manager|
  manager.default_strategies :authentication_token
  manager.failure_app = UnauthenticatedController
end
{% endhighlight %}

### Authentication Token Strategy

We now just need to create the authentication token strategy specs and class.

{% highlight ruby %}
# file: spec/lib/authentication_token_strategy_spec.rb
require 'rails_helper'

RSpec.describe AuthenticationTokenStrategy, type: :model do
  let!(:user) {
    User.create(email: "user@example.com", password: "password") }
  let!(:authentication_token) {
    AuthenticationToken.create(user_id: user.id, body: "token", last_used_at: DateTime.current) }

  let(:env) {
    { "HTTP_X_USER_EMAIL" => user.email,
      "HTTP_X_AUTH_TOKEN" => authentication_token.body } }

  let(:subject) { described_class.new(nil) }

  describe "#valid?" do

    context "with valid credentials" do
      before { allow(subject).to receive(:env).and_return(env) }

      it { is_expected.to be_valid }
    end

    context "with invalid credentials" do
      before { allow(subject).to receive(:env).and_return({}) }

      it { is_expected.not_to be_valid }
    end

  end

  describe "#authenticate!" do

    context "with valid credentials" do
      before { allow(subject).to receive(:env).and_return(env) }

      it "returns success" do
        expect(User).to receive(:find_by)
          .with(email: user.email)
          .and_return(user)
        expect(TokenIssuer).to receive_message_chain(:build, :find_token)
          .with(user, authentication_token.body)
          .and_return(authentication_token)
        expect(subject).to receive(:success!).with(user)
        subject.authenticate!
      end

      it "touches the token" do
        expect(subject).to receive(:touch_token)
          .with(authentication_token)
        subject.authenticate!
      end
    end

    context "with invalid user" do
      before { allow(subject).to receive(:env)
        .and_return({ "HTTP_X_USER_EMAIL" => "invalid@email",
                      "HTTP_X_AUTH_TOKEN" => "invalid-token" }) }

      it "fails" do
        expect(User).to receive(:find_by)
          .with(email: "invalid@email")
          .and_return(nil)
        expect(TokenIssuer).not_to receive(:build)
        expect(subject).not_to receive(:success!)
        expect(subject).to receive(:fail!)
        subject.authenticate!
      end
    end

    context "with invalid token" do
      before { allow(subject).to receive(:env)
        .and_return({ "HTTP_X_USER_EMAIL" => user.email,
                      "HTTP_X_AUTH_TOKEN" => "invalid-token" }) }

      it "fails" do
        expect(User).to receive(:find_by)
          .with(email: user.email)
          .and_return(user)
        expect(TokenIssuer).to receive_message_chain(:build, :find_token)
          .with(user, "invalid-token")
          .and_return(nil)
        expect(subject).not_to receive(:success!)
        expect(subject).to receive(:fail!)
        subject.authenticate!
      end
    end
  end

end
{% endhighlight %}

As you can see, the strategy uses the <code>valid?</code> and <code>authenticate!</code> methods to check if the parameters (our two custom headers) are present and if the user exists and has a valid token.

{% highlight ruby %}
# file: lib/authentication_token_strategy.rb
class AuthenticationTokenStrategy < ::Warden::Strategies::Base

  def valid?
    user_email_from_headers.present? && auth_token_from_headers.present?
  end

  def authenticate!
    failure_message = "Authentication failed for user/token"

    user = User.find_by(email: user_email_from_headers)
    return fail!(failure_message) unless user

    token = TokenIssuer.build.find_token(user, auth_token_from_headers)
    if token
      touch_token(token)
      return success!(user)
    end

    fail!(failure_message)
  end

  def store?
    false
  end

  private

    def user_email_from_headers
      env["HTTP_X_USER_EMAIL"]
    end

    def auth_token_from_headers
      env["HTTP_X_AUTH_TOKEN"]
    end

    def touch_token(token)
      token.update_attribute(:last_used_at, DateTime.current) if token.last_used_at < 1.hour.ago
    end

end
{% endhighlight %}

One last touch is updating the token every time the user uses it to authenticate itself in order to keep it between the non-expirable tokens.

### Controllers

With the strategy up and running, we need now to add some helpers to the application controller in order to require the authentication to all the actions we want to restrict access to.

I will create a controller concern with some helper methods and a before_action prepended to all the actions that will try to authenticate the user with the provided credentials (if any) present in the request.

{% highlight ruby %}
# file: app/controllers/concerns/warden_helper.rb
module WardenHelper
  extend ActiveSupport::Concern

  included do
    helper_method :warden, :current_user

    prepend_before_action :authenticate!
  end

  def current_user
    warden.user
  end

  def warden
    request.env['warden']
  end

  def authenticate!
    warden.authenticate!
  end
end
{% endhighlight %}

By including the concern in the <code>ApplicationController</code> we assure that all the controllers that inherit from it will require authentication.

{% highlight ruby %}
# file: app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include WardenHelper

  # ...
end
{% endhighlight %}

As you could see in the initializer, we delegate to a special <code>UnauthenticatedController</code> controller the handling of failed authentications.
It will respond with a 401 status code and an error message to all the requests that don't satisfy the authentication.

{% highlight ruby %}
# file: app/controllers/unauthenticated_controller.rb
class UnauthenticatedController < ActionController::Metal

  def self.call(env)
    @respond ||= action(:respond)
    @respond.call(env)
  end

  def respond
    self.status        = :unauthorized
    self.content_type  = "application/json"
    self.response_body = { errors: ["Unauthorized Request"] }.to_json
  end

end
{% endhighlight %}

In order to test the authentication layer, we need to add some helpers to config RSpec with Warden. I had some issues and found a solution in a StackOverflow post:

{% highlight ruby %}
# file: specs/support/warden.rb
# Based on http://stackoverflow.com/questions/13420923/configuring-warden-for-use-in-rspec-controller-specs
module Warden
  # Warden::Test::ControllerHelpers provides a facility to test controllers in isolation
  # Most of the code was extracted from Devise's Devise::TestHelpers.
  module Test
    module ControllerHelpers
      def self.included(base)
        base.class_eval do
          setup :setup_controller_for_warden, :warden if respond_to?(:setup)
        end
      end

      # Override process to consider warden.
      def process(*)
        # Make sure we always return @response, a la ActionController::TestCase::Behavior#process, even if warden interrupts
        _catch_warden {super} || @response
      end

      # We need to setup the environment variables and the response in the controller
      def setup_controller_for_warden
        @request.env['action_controller.instance'] = @controller
      end

      # Quick access to Warden::Proxy.
      def warden
        @warden ||= begin
          manager = Warden::Manager.new(nil, &Rails.application.config.middleware.detect{|m| m.name == 'Warden::Manager'}.block)
          @request.env['warden'] = Warden::Proxy.new(@request.env, manager)
        end
      end

      protected

      # Catch warden continuations and handle like the middleware would.
      # Returns nil when interrupted, otherwise the normal result of the block.
      def _catch_warden(&block)
        result = catch(:warden, &block)

        if result.is_a?(Hash) && !warden.custom_failure? && !@controller.send(:performed?)
          result[:action] ||= :unauthenticated

          env = @controller.request.env
          env['PATH_INFO'] = "/#{result[:action]}"
          env['warden.options'] = result
          Warden::Manager._run_callbacks(:before_failure, env, result)

          status, headers, body = warden.config[:failure_app].call(env).to_a
          @controller.send :render, status: status, text: body,
            content_type: headers['Content-Type'], location: headers['Location']

          nil
        else
          result
        end
      end
    end
  end
end

RSpec.configure do |config|
  config.include Warden::Test::ControllerHelpers, type: :controller
end
{% endhighlight %}

Just add this code in a <code>specs/support/warden.rb</code> file and you will be fine.

Let's then add another shared example to gather the common specs for an authenticated controller.

{% highlight ruby %}
# file: specs/support/authenticated_api_controller.rb
require 'rails_helper'

RSpec.shared_examples "authenticated_api_controller" do

  describe "authentiation" do

    it "returns unauthorized request without email and token" do
      request.env["HTTP_X_USER_EMAIL"] = nil
      request.env["HTTP_X_AUTH_TOKEN"] = nil
      get :index, { format: :json }

      expect(response.status).to eq(401)
    end

    it "returns unauthorized request without token" do
      user = User.create(email: "user@example.com", password: "password")
      request.env["HTTP_X_USER_EMAIL"] = user.email
      request.env["HTTP_X_AUTH_TOKEN"] = nil
      get :index, { format: :json }

      expect(response.status).to eq(401)
    end

  end

end
{% endhighlight %}

By adding a before block where we create a user and its token and set them in the headers, we can now test that our customers controller specs are still passsing.

Add the shared example as well to be sure that the controller respect the authentication strategy as well in cases where it's not valid.

{% highlight ruby %}
# file: specs/controllers/customers_controller_spec.rb
require 'rails_helper'

RSpec.describe CustomersController, type: :controller do

  before do
    user = User.create(email: "user@example.com", password: "password")
    authentication_token = AuthenticationToken.create(user_id: user.id,
      body: "token", last_used_at: DateTime.current)
    request.env["HTTP_X_USER_EMAIL"] = user.email
    request.env["HTTP_X_AUTH_TOKEN"] = authentication_token.body
  end

  it_behaves_like "api_controller"
  it_behaves_like "authenticated_api_controller"

  # ...

end
{% endhighlight %}

### Adding a sessions controller to login users

Our last step in order to see some real results for our API is adding a way for the user to log in and receive a valid token to authenticate subsequent requests to the endpoints.

Let's add some routing with specs for the sessions controller:

{% highlight ruby %}
# file: specs/routing/sessions_routing_spec.rb
require 'rails_helper'

RSpec.describe SessionsController, type: :routing do
  it { expect(post:   "/sessions").to route_to("sessions#create") }
  it { expect(delete: "/sessions").to route_to("sessions#destroy") }
end
{% endhighlight %}

{% highlight ruby %}
# file: config/routes.rb
Rails.application.routes.draw do
  resource  :sessions,  only: [:create, :destroy]
  resources :customers, only: [:index, :show, :create, :update, :destroy]
end
{% endhighlight %}

And write the controller specs for the create (login) and destroy (logout) actions:

{% highlight ruby %}
# file: specs/controllers/sessions_controller_spec.rb
require 'rails_helper'

RSpec.describe SessionsController, type: :controller do

  let!(:user) { User.create(email: "user@example.com", password: "password") }
  let!(:authentication_token) { AuthenticationToken.create(user_id: user.id,
      body: "token", last_used_at: DateTime.current) }

  let(:valid_attributes) {
    { user: { email: user.email, password: "password" } }
  }

  let(:invalid_attributes) {
    { user: { email: user.email, password: "not-the-right-password" } }
  }

  let(:parsed_response) { JSON.parse(response.body) }

  def set_auth_headers
    request.env["HTTP_X_USER_EMAIL"] = user.email
    request.env["HTTP_X_AUTH_TOKEN"] = authentication_token.body
  end

  before do
    allow(TokenIssuer).to receive(:create_and_return_token).and_return(authentication_token.body)
  end

  describe "POST #create" do
    context "with valid credentials" do
      before { post :create, valid_attributes, format: :json  }

      it { expect(response).to be_success }
      it { expect(parsed_response).to eq({ "user_email" => user.email, "auth_token" => authentication_token.body }) }
    end

    context "with invalid credentials" do
      before { post :create, invalid_attributes, format: :json }

      it { expect(response.status).to eq(401) }
    end

    context "with missing/invalid params" do
      before { post :create, { foo: { bar: "baz" }, format: :json } }

      it { expect(response.status).to eq(422) }
    end
  end

  describe "DELETE #destroy" do
    context "with valid credentials" do
      before do
        set_auth_headers
        delete :destroy, { format: :json }
      end

      it { expect(response).to be_success }
    end

    context "with invalid credentials" do
      before { delete :destroy, { format: :json } }

      it { expect(response.status).to eq(401) }
    end
  end

end
{% endhighlight %}

The implementation uses the <code>TookenIssuer</code> to create and return a new token for the user with valid credentials while logging in and again uses the service to get rid of the expired token on logout.

As you might notice, I skip the <code>authenticate!</code> before_action on the create action since we want to enable the user to make this request while not been authenticated for obvious reasons.

The destroy (logout) action instead would still require the user to provide the authentication credentials to be executed.

{% highlight ruby %}
# file: app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  skip_before_action :authenticate!, only: [:create]

  def create
    user = User.find_by(email: session_params[:email])
    if user && user.authenticate(session_params[:password])
      token = TokenIssuer.create_and_return_token(user, request)
      render status: :ok, json: { user_email: user.email, auth_token: token }
    else
      render status: :unauthorized, json: ""
    end
  end

  def destroy
    TokenIssuer.expire_token(current_user, request) if current_user
    render status: :ok, json: ""
  end

  private

    def session_params
      params.require(:user).permit(:email, :password)
    end

end
{% endhighlight %}

Let's add an example user to our seeds and then test with curl the working login and an authenticated request to the customers endpoint.

{% highlight ruby %}
# file: db/seeds.rb
# ...
User.create(email: "admin@example.com", password: "password")
{% endhighlight %}

{% highlight bash %}
curl -i -X POST -H "Content-Type:application/json" -d '{ "user": { "email": "admin@example.com", "password": "password" } }' http://localhost:3000/sessions.json
{% endhighlight %}

{% highlight bash %}
{"user_email":"admin@example.com","auth_token":"m5d2eADqgZ5pX7aE4daSkevg"}
{% endhighlight %}

If you received the valid token after logging in, you can now request the list of customers providing the user email and token through the custom headers:

{% highlight bash %}
curl -i -X GET -H "Content-Type:application/json" -H "X-User-Email:admin@example.com" -H "X-Auth-Token:m5d2eADqgZ5pX7aE4daSkevg" http://localhost:3000/customers.json
{% endhighlight %}

If everything is working properly, you should receive the payload we created some steps ago. Congratulations!

## Cross-origin resource sharing (CORS)

The previous step marks the last pieace of the standard functionalities that I would require from a simple API application. I would still add one bonus step in order to make sure that our API can be used in client web applications through AJAX calls. In order to do this, the browser expects the API to provide Cross-origin resource sharing (CORS) headers. You can find more information about them [on Wikipedia](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing).

The most important thing is that all the responses of our application will contain these additional headers and a special route that respond to <code>OPTIONS</code> http requests is present.

Let's start by adding the <code>config.action_dispatch.default_headers</code> to the <code>config/application.rb</code> file. I user constants to set the headers so we can reuse them in other places centralizing the setup.

{% highlight ruby %}
# file: config/application.rb
# ...

CORS_ALLOW_ORIGIN  = "*"
CORS_ALLOW_METHODS = %w{GET POST PUT OPTIONS DELETE}.join(',')
CORS_ALLOW_HEADERS = %w{Content-Type Accept X-User-Email X-Auth-Token}.join(',')

module ExampleApi
  class Application < Rails::Application
    # ...

    config.action_dispatch.default_headers = {
      "Access-Control-Allow-Origin"  => CORS_ALLOW_ORIGIN,
      "Access-Control-Allow-Methods" => CORS_ALLOW_METHODS,
      "Access-Control-Allow-Headers" => CORS_ALLOW_HEADERS
    }
  end
end
{% endhighlight %}

I also added to the <code>api_controller</code> shared example in the support directory, some specs that check the presence of those headers in all the responses.

{% highlight ruby %}
# file: specs/support/api_controller.rb
require 'rails_helper'

RSpec.shared_examples "api_controller" do

  # ...

  describe "responds to OPTIONS requests to return CORS headers" do

    before { process :index, 'OPTIONS' }

    context "CORS requests" do
      it "returns the Access-Control-Allow-Origin header to allow CORS from anywhere" do
        expect(response.headers['Access-Control-Allow-Origin']).to eq('*')
      end

      it "returns general HTTP methods through CORS (GET/POST/PUT/DELETE)" do
        %w{GET POST PUT DELETE}.each do |method|
          expect(response.headers['Access-Control-Allow-Methods']).to include(method)
        end
      end

      it "returns the allowed headers" do
        %w{Content-Type Accept X-User-Email X-Auth-Token}.each do |header|
          expect(response.headers['Access-Control-Allow-Headers']).to include(header)
        end
      end
    end

  end

end
{% endhighlight %}

Since the Warden fail_app is not using the default_headers set by our Rails application, we need to manually set again the CORS header in the <code>UnauthenticatedController</code> respond method.

{% highlight ruby %}
# file: app/controllers/unauthenticated_controller.rb
class UnauthenticatedController < ActionController::Metal

  # ...

  def respond
    self.status        = :unauthorized
    self.content_type  = "application/json"
    self.response_body = { errors: ["Unauthorized Request"] }.to_json
    self.headers["Access-Control-Allow-Origin"]  = CORS_ALLOW_ORIGIN
    self.headers["Access-Control-Allow-Methods"] = CORS_ALLOW_METHODS
    self.headers["Access-Control-Allow-Headers"] = CORS_ALLOW_HEADERS
  end

end
{% endhighlight %}

Finally we create a "catch all" route (be sure it's at bottom of your list) that will respond to every <code>OPTIONS</code> HTTP request with only the CORS headers.

In this way the browser pre-flight request to enable the usage of our API will be fullfilled and we will allow the requests as we like. Please customize the CORS header constants accordingly.

{% highlight ruby %}
# file: config/routes.rb
Rails.application.routes.draw do
  # ...

  match "/*path",
    to: proc {
      [
        204,
        {
          "Content-Type"                 => "text/plain",
          "Access-Control-Allow-Origin"  => CORS_ALLOW_ORIGIN,
          "Access-Control-Allow-Methods" => CORS_ALLOW_METHODS,
          "Access-Control-Allow-Headers" => CORS_ALLOW_HEADERS
        },
        []
      ]
    }, via: [:options, :head]

end
{% endhighlight %}

## Conclusion

It was a long tutorial and we just skimmed the surface of the topic. I will probably extend in the future the application with some more features and write other tutorials about. For example I would like to use this API on an Angular.js web app to show how you can buld a Single Page Application using these technologies.

You can find the complete repository for the tutorial on [Github](https://github.com/lucatironi/example_rails_api).

For any question or request, you can use the comments below or send an email to [luca.tironi@gmail.com](mailto:luca.tironi@gmail.com)

Stay tuned and thanks for your time, I hope you will find this useful.

L
