---
layout: post
title: Using Rails 5, Docker and docker-compose to build an authenticated JSON API with warden
tagline: I migrated my last tutorial from last year for an authenticated JSON API with Ruby on Rails 5 to using Docker and docker-compose
description: In the last few months I experimented a lot with Docker and docker-compose. I wanted to spin up a development enviroment with it and I used my previous tutorial as a base app to run. To do so I updated it to the latest Rails - 5.0.0.1 - and refactored a bit the spec.
category: tutorial
tags: [Ruby on Rails, Warden, Ruby, authentication, API, Docker]
---

You can find the complete application on my [github profile](https://github.com/lucatironi/rails-5-api-docker). It's basically the same code described in [my tutorial](/tutorial/2015/08/23/rails_api_authentication_warden/) from last year but starting from a newly created Rails 5 application with the <code>--api</code> flag coming from the integration of the previously separated <code>rails-api</code> gem, now part of Rails.

I used the methods to bootstrap the application described in my [rails-docker](https://github.com/lucatironi/rails-docker) project. With this setup you don't even need to have a ruby environment running on your local development machine, given you can run Docker.

I run the test and other scripts/rake commands with a simple <code>docker-compose run web bin/rspec</code> or I can even enter the docker container and run the command inside by using <code>docker-compose run web bash</code>.

If you are curious about this setup, please send me an email to [luca.tironi@gmail.com](mailto:luca.tironi@gmail.com) or comment in the comment section.

Have fun!
