---
layout: post
title: (Updated) Use docker-compose to develop a Ruby on Rails application
tagline: Using docker-compose to develop a Rails application
description: I updated the tutorial and the code from last year on how to use Docker and docker-compose to bootstrap and develop a Rails application without even have ruby installed on your development machine.
category: tutorial
tags: [Ruby on Rails, Docker, docker-compose]
---

You can find the code for this tutorial on my [Github repository](https://github.com/lucatironi/rails-docker).

In order to kickstart the application we need to install the rails gem and run the `rails new <appname>` command. We will do this inside the same Docker container that will be used to run the application itself. To do so we need to create a `Dockerfile` to create the Docker image with the necessary dependencies, a `docker-compose.yml` file to provision the other services needed (a postgres database and a volume store) and a `Gemfile` (with an empty `Gemfile.lock`) to install the `rails` gem and bundle install its dependencies.

Create a new directory and some empty files:

{% highlight bash %}
mkdir rails-docker-compose
cd rails-docker-compose
touch Dockerfile docker-compose.yml Gemfile Gemfile.lock
{% endhighlight %}

Copy and paste the following content in the respective files:

File `Dockerfile`

{% highlight bash %}
FROM ruby:2.4.2

ENV APP_ROOT /app
ENV BUNDLE_PATH /usr/local/bundle

RUN apt-get update -qq && \
    apt-get install -y build-essential libpq-dev nodejs

WORKDIR $APP_ROOT
ADD . $APP_ROOT
{% endhighlight %}

File `docker-compose.yml`

{% highlight yaml %}
version: '3'

volumes:
  store:
    driver: local
  bundle:
    driver: local

services:
  web:
    build: .
    ports:
      - 3000:3000
    volumes:
      - .:/app
      - bundle:/usr/local/bundle
    links:
      - db
    # Keep the stdin open, so we can attach to our app container's process and do things such as
    # byebug, etc:
    stdin_open: true
    # Enable sending signals (CTRL+C, CTRL+P + CTRL+Q) into the container:
    tty: true
    command: ./bin/start.sh
    environment: &app_env
      PORT: 3000
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: postgres
      DB_PSWD: postgres
      DB_NAME: development_database
  db:
    image: postgres:latest
    ports:
      - 5432:5432
    volumes:
      - store:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: development_database
{% endhighlight %}

File `Gemfile`

{% highlight ruby %}
source 'https://rubygems.org'

gem 'rails', '~> 5'
{% endhighlight %}

## Build the project

First we need to bundle the rails 5 dependencies:

{% highlight bash %}
docker-compose run --rm web bundle --jobs=10 --retry=5
{% endhighlight %}

And then use the `rails new` command to create the new application:

{% highlight bash %}
docker-compose run --rm web bundle exec rails new . --force --database=postgresql --skip-bundle
{% endhighlight %}

## Configure the database

We need to change slightly the database configuration to use the environment
variables set in the docker-compose file:

File: `config/database.yml`

{% highlight yaml %}
default: &default
  adapter: postgresql
  encoding: unicode
  username: <%= ENV['DB_USER'] %>
  password: <%= ENV['DB_PSWD'] %>
  host: <%= ENV['DB_HOST'] %>
  # For details on connection pooling, see rails configuration guide
  # http://guides.rubyonrails.org/configuring.html#database-pooling
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: <%= ENV['DB_NAME'] %>

test:
  <<: *default
  database: test_database
{% endhighlight %}

## Setup the app

{% highlight bash %}
docker-compose run --rm web bin/setup
{% endhighlight %}

Create a `start.sh` file in the `bin` dir:

File: `bin/start.sh`

{% highlight bash %}
#!/bin/bash

bundle check || bundle install

if [ -f tmp/pids/server.pid ]; then
  rm -f tmp/pids/server.pid
fi

bundle exec rails s -p $PORT -b 0.0.0.0
{% endhighlight %}

It automatically removes the `server.pid` that will create problems when you stop
and restart the app.
Make the file executable with the `chmod` command:

{% highlight bash %}
chmod +x bin/start.sh
{% endhighlight %}

Finally start your newly created Rails application and visit [localhost:3000](http://localhost:3000):

{% highlight bash %}
docker-compose up
{% endhighlight %}
