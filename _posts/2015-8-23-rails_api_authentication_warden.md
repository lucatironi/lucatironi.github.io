---
layout: post
title: Using rails-api to build an authenticated JSON API with warden
tagline: An updated version of my previous tutorials on building an authenticated JSON API with Ruby on Rails
description: Two years ago I published a series of tutorials to explain how to build a JSON API with Ruby on Rails and setting up an authentication with Devise. This new tutorial uses a test driven approach (RSpec) and rails-api with warden, so we can now build the same backend with even less code.
category: tutorial
tags: [Ruby on Rails, Warden, Ruby, authentication, API]
draft: true
---

## Requirements

* [Rails::API](https://github.com/rails-api/rails-api)
* [Warden](https://github.com/hassox/warden)
* [RSpec](https://github.com/rspec/rspec-rails)

The complete code for this tutorial can be found on my [Github account](https://github.com/lucatironi/example_rails_api).

## Setup

{% highlight bash %}
rails-api new example_api --skip-turbolinks --skip-sprockets --skip-test-unit --skip-javascript
{% endhighlight %}

## Testing

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

[spring-commands-rspec](https://github.com/jonleighton/spring-commands-rspec)

{% highlight bash %}
bundle install

bin/rails generate rspec:install

bundle exec spring binstub rspec
{% endhighlight %}

## Customers

### Model

{% highlight bash %}
bin/rails g model customer full_name:string email:string phone:string

bin/rake db:migrate
{% endhighlight %}

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

{% highlight ruby %}
# file: app/models/customer.rb
class Customer < ActiveRecord::Base
  validates_presence_of :full_name
end
{% endhighlight %}

{% highlight bash %}
bin/rspec
{% endhighlight %}

{% highlight bash %}
$ bin/rspec

......

Finished in 0.02619 seconds (files took 0.49037 seconds to load)
6 examples, 0 failures
{% endhighlight %}

### Routing

{% highlight bash %}
bin/rails generate controller customers index show create update destroy --no-view-specs --skip-routes
{% endhighlight %}

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

{% highlight ruby %}
# file: config/routes.rb
Rails.application.routes.draw do
  resources :customers, only: [:index, :show, :create, :update, :destroy]
end
{% endhighlight %}

### Controller

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

#### Adding some real data

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

{% highlight bash %}
bin/rake db:seed

bin/rails s
{% endhighlight %}

Visit [localhost:3000/customers.json](http://localhost:3000/customers.json)

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

#### Using ActiveModel::Serializer to build the JSON response

{% highlight ruby %}
# file: Gemfile
gem 'active_model_serializers', github: 'rails-api/active_model_serializers'
{% endhighlight %}

{% highlight bash %}
bundle install

bin/rails generate serializer customer
{% endhighlight %}

{% highlight ruby %}
# file: app/serializers/customer_serializer.rb
class CustomerSerializer < ActiveModel::Serializer
  attributes :id, :full_name, :email, :phone
end
{% endhighlight %}

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

{% highlight ruby %}
# file: specs/rails_helper.rb
Dir[Rails.root.join('spec/support/**/*.rb')].each { |f| require f }
{% endhighlight %}

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

{% highlight ruby %}
# file: specs/controllers/customer_controller_spec.rb
require 'rails_helper'

RSpec.describe CustomersController, type: :controller do

  it_behaves_like "api_controller"

  # ...
end
{% endhighlight %}

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

## Authentication

### Setup

{% highlight ruby %}
# file: Gemfile
gem 'bcrypt', '~> 3.1.7'
gem 'has_secure_token'
{% endhighlight %}

{% highlight bash %}
bundle install
{% endhighlight %}

### Token Issuer

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

{% highlight bash %}
bin/rails generate model authentication_token body:string user:references last_used_at:datetime ip_address:string user_agent:string

bin/rails generate model user email:string password_digest:string
{% endhighlight %}

### User model

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

### Adding Warden to handle authentication

[Reference](https://github.com/ouranos/rails-api-warden)

{% highlight ruby %}
# file: Gemfile
gem 'warden'
{% endhighlight %}

{% highlight bash %}
bundle install
{% endhighlight %}

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

{% highlight ruby %}
# file: specs/lib/authentication_token_strategy_spec.rb
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

### Controllers

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

{% highlight ruby %}
# file: app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include WardenHelper

  # ...
end
{% endhighlight %}

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

{% highlight bash %}
curl -i -X GET -H "Content-Type:application/json" -H "X-User-Email:admin@example.com" -H "X-Auth-Token:m5d2eADqgZ5pX7aE4daSkevg" http://localhost:3000/customers.json
{% endhighlight %}

## Cross-origin resource sharing (CORS)

[CORS on Wikipedia](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing)

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

