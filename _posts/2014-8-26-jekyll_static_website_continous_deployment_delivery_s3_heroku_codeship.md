---
layout: post
title: Jekyll Static Website with Continuous Deployment/Delivery to S3/Heroku
tagline: Develop a static website with Jekyll and deploy it automatically to S3 and/or Heroku
description: Jekyll - the Static Website generator gem - can be extendend to use a Rails-like assets pipeline to develop even complex Single Page Applications and can be configured to be automatically deployed to a public S3 bucket and/or to Heroku, using a Continuous Deployment/Delivery service like Codeship.io
category: tutorial
tags: [Ruby, Jekyll, S3, Heroku, Codeship.io]
---

## Requirements
* [jekyll](http://jekyllrb.com/)
* [jekyll-assets](https://github.com/ixti/jekyll-assets) (optional)
* [bootstrap-sass](https://github.com/twbs/bootstrap-sass) (optional)
* [s3_website](https://github.com/laurilehmijoki/s3_website)
* [Amazon S3](https://aws.amazon.com/s3)
* [Heroku](http://heroku.com)
* [CodeShip.io](https://github.com/laurilehmijoki/s3_website) (optional)

## Intro
As a Ruby (on Rails) developer, I sometimes wonder if a web application is always the right solution for all my problems. Most of the time it is not, a simple HTML static website would be enough. But then it would be nice to have some of the nice tools that you can use developing a web application, like templates and partials, an assets pipeline, easy deployment and some sort of intermediate markup or scripting language to dynamically generate pages and assets (like markdown, ERB, SCSS/Less and Coffescript).

To find a viable solution to these needs, I would recommend [jekyll](http://jekyllrb.com/): an awesome and really popular gem to build static websites with powerful tools, like markdown syntax, [liquid](https://github.com/Shopify/liquid) templates, assets pipeline and many more. Github uses it to provide simple to mantain [pages](https://pages.github.com/) to be hosted for free on their servers.

But what if you don't want to use Github to host your static websites or you want other options instead of hosting them on your own servers?

This tutorial will explain own to use Jekyll and some of its plugins to develop a static website with a Rails-like assets pipeline and deploy it to [Amazon S3](https://aws.amazon.com/s3) and/or [Heroku](http://heroku.com) with a [Continuous Deployment/Delivery system](https://github.com/laurilehmijoki/s3_website).

Let's start by installing and configuring the basics.

## Basic setup
Install the [Jekyll](http://jekyllrb.com) gem

{% highlight bash %}
gem install jekyll
{% endhighlight %}

Create a new jekyll project

{% highlight bash %}
jekyll new example_website
cd example_website
{% endhighlight %}

Run the jekyll server

{% highlight bash %}
jekyll serve --watch
{% endhighlight %}

and visit `http://localhost:4000`: you will see a nice looking page with some examples of the potentialities of Jekyll.

## Configure the project
For the pourpose of this tutorial, I want to start from a clean slate. I will remove the boilerplate files created by Jekyll and create a slightly different directory structure. You are free to skip this passage, if you wish to keep the example files.

{% highlight bash %}
rm -fr css _posts _layouts/post.html _layouts/page.html about.md feed.xml
mkdir -p _assets/javascripts/ _assets/stylesheets/ _plugins/ _vendors/javascripts/ _vendors/stylesheets/ assets/
touch _assets/javascripts/application.js.coffee _assets/stylesheets/application.css.scss _plugins/ext.rb Gemfile
{% endhighlight %}

The project directory structure:

{% highlight bash %}
example_website
├── _assets
│   ├── javascripts
│   │   └── application.js.coffee
│   └── stylesheets
│       └── application.css.scss
├── _includes
│   ├── footer.html
│   ├── head.html
│   └── header.html
├── _layouts
│   └── default.html
├── _plugins
│   └── ext.rb
├── _vendors
│   ├── javascripts
│   └── stylesheets
├── assets
├── _config.yml
├── Gemfile
├── Gemfile.lock
└── index.html
{% endhighlight %}

Create a `Gemfile` in the root directory and add these gems.

{% highlight ruby %}
source 'https://rubygems.org'

gem 'jekyll-assets'
gem 'bootstrap-sass'
gem 'uglifier'
{% endhighlight %}

Install the gems

{% highlight bash %}
bundle install
{% endhighlight %}

## Assets Pipeline
Add the `jekyll-assets` plugin to the `_plugins/ext.rb` file. This plugin allows to use a Rails-like assets pipeline in a Jekyll project, including using CoffeScript, Sass, Less and ERB as intermediate languages to write your assets and pages, automatic minification of code, cache busting and many other cool features that you can discover on the [Github repository](https://github.com/ixti/jekyll-assets) of the plugin.

{% highlight ruby %}
require 'jekyll-assets'
require 'jekyll-assets/bootstrap'
{% endhighlight %}

We also added the bootstrap plugin to automatically use the framework in the project. Feel free to skip this if you want to start from scratch or with an other framework/library.

More information about Jekyll plugins can be found in the [Jekyll documentation](http://jekyllrb.com/docs/plugins/).

To use the assets pipeline, we need to add the `assets` configuration to the `_config.yml` file.

{% highlight yaml %}
# Site settings
title: Your awesome title
email: your-email@domain.com
description: "Write an awesome description for your new site here. You can edit this line in _config.yml. It will appear in your document head meta (for Google search results) and in your feed.xml site description."
baseurl: ""
url: "http://yourdomain.com"

# Build settings
exclude: ['README.md', 'Gemfile', 'Gemfile.lock']
markdown: kramdown
permalink: pretty

assets:
  debug: false
  js_compressor:  uglifier
  css_compressor: sass
  sources:
    - _assets/javascripts
    - _assets/stylesheets
    - _vendors/stylesheets
    - _vendors/javascripts
{% endhighlight %}

If you set the `debug` flag to true, you skip the minifications of the assets. It can be useful during development to debug the Javascript.

## Add some content
JQuery is required to use the bootstrap JS components, so you can download the latest version (1.11.1 at the time of writing) of the library and put the file in `_vendors/javascripts/`. As a convention, it's better to put external libraries, stylesheets and files in the `_vendors` directory and keep your own files separated from them.

{% highlight bash %}
curl -o ./_vendors/javascripts/jquery.min.js http://code.jquery.com/jquery.min.js
{% endhighlight %}

Require both the jquery minified file and bootstrap in the `_assets/javascripts/application.js.coffee` file. You can add more files and require them in this one.

{% highlight javascript %}
#= require jquery.min
#= require bootstrap
{% endhighlight %}

Similar to the javascripts, we import the main bootstrap stylesheet in the `_assets/stylesheets/applications.css.scss` file. We added just a little style to account the fact that we use a fixed navigation bar and we need to push the main content a bit down.

{% highlight css %}
@import "bootstrap";

body {
  padding-top: 70px;
  padding-bottom: 30px;
}
{% endhighlight %}

Finally let's setup the default layout `_layouts/default.html` template. As you can see we are using some helpers to include other html files that you can find in the `_includes` directory. We us also the `javascript` helper provided by `jekyll-assets` to include our application javascript file.

{% highlight html %}{% raw %}
<!DOCTYPE html>
<html>

  {% include head.html %}

  <body>

    {% include header.html %}

    <div class="container" role="main">
      {{ content }}
    </div>

    {% include footer.html %}

    {% javascript application %}
  </body>
</html>
{% endraw %}{% endhighlight %}

Nothing special here apart the usage of the `stylesheet` helper, always provided by `jekyll-assets` in order to include the application stylesheet in the `_includes/head.html` file.

{% highlight html %}{% raw %}
<head>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <title>{% if page.title %}{{ page.title }}{% else %}{{ site.title }}{% endif %}</title>
  <meta name="viewport" content="width=device-width">
  <meta name="description" content="{{ site.description }}">
  <meta name="robots" content="noindex, noarchive">
  <link rel="canonical" href="{{ page.url | replace:'index.html','' | prepend: site.baseurl | prepend: site.url }}">

  <!-- Custom CSS -->
  {% stylesheet application %}
</head>
{% endraw %}{% endhighlight %}

Our `_includes/header.html` file with just a basic nav-bar provided by bootstrap.

{% highlight html %}{% raw %}
<nav class="navbar navbar-inverse navbar-fixed-top" role="navigation">
  <div class="container">
    <div class="navbar-header">
      <a class="navbar-brand" href="/">Example Website</a>
    </div>
  </div>
</nav>
{% endraw %}{% endhighlight %}

And a simple `_includes/footer.html` file.

{% highlight html %}{% raw %}
<footer>
  <div class="container">
    <hr>
    <small>
      2014 Copyright Example Ltd
    </small>
  </div>
</footer>
{% endraw %}{% endhighlight %}

Finally add some content to the `index.html` file. It will be the default page diplayed when accessing your website.

{% highlight html %}{% raw %}
---
layout: default
---

<h1 class="page-header">Example Website</h1>
<p class="lead">
  An example of using Jekyll with an assets pipeline, automated build and deployment to S3 or Heroku.
</p>
{% endraw %}{% endhighlight %}

And we are done for this part. Run the server (if you haven't already).

{% highlight bash %}
jekyll serve --watch
{% endhighlight %}

Visit `localhost:4000` and check the results.

## Deploy to S3
Amazon S3 is mostly used as an external storage for assets or files, but can also be configured to serve an entire static website in a scalable, cheap and performant way. To deploy our static website to it, we can use the `s3_website`. To start using it, add the gem to the `Gemfile`.

{% highlight ruby %}
gem 's3_website'
{% endhighlight %}

and bundle it.

{% highlight bash %}
bundle install
{% endhighlight %}

We are going to use a YAML file to hold the information that the s3_website gem will use during the deployment process. Create a new file called `s3_website.yml` in the root dir with this content.

{% highlight yaml %}
s3_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
s3_secret: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
s3_bucket: <%= ENV['S3_BUCKET_NAME'] %>
{% endhighlight %}

As you can see we are not going to store any sensitive information - like your AWS keys - in the repositories file. Create an `.env` file in the root dir of the project and add there the credentials that you want to use. Remember to add the `.env` file to the `.gitignore`. In this way you can have other environment variables on your servers. More information about storing your variables in the enviroment can be found on the [12factor website](http://12factor.net/config).

{% highlight bash %}
AWS_ACCESS_KEY_ID=AKFKSKSDK8DSDSMFJA
AWS_SECRET_ACCESS_KEY=0fsd0fdsf0sd9f0fs0dBKmG6BfOVPYoHs
S3_BUCKET_NAME=example-website
{% endhighlight %}

More information about setting up a IAM user on AWS for `s3_website` can be found [on the github repository](https://github.com/laurilehmijoki/s3_website/blob/master/additional-docs/setting-up-aws-credentials.md)

Create a new bucket in your S3 account. Open the `Static Website Hosting` menu in the properties for the new bucket and check `Enable website hosting` and adding `index.html` as the Index Document.

In order to upload correctly the compiled files with `s3_website` we need to give the permission to do so to the IAM user we just created. Go to the `Permissions` tab and click on `Add more permissions`. Select `Authenticated Users` and grant at least `List` and `Upload/Delete`.

We need also to setup the bucket policy to allow every page to be seen as public. To so click on `Add bucket policy` in the `Permissions` tab again and paste this code in the textarea. Remember to substitute `BUCKET_NAME` with your correct bucket name.

{% highlight json %}
{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "Allow Public Access to All Objects",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::BUCKET_NAME/*"
    }
  ]
}
{% endhighlight %}

You can try if your AWS setup and configuration is ok by running

{% highlight bash %}
s3_website push --dry-run
{% endhighlight %}

It will execute all the commands to build the files from Jekyll and test the permissions to upload them on S3, without actually doing it and cleaning up afterwise.

If everything went ok, you can push your website for real, by removing the `--dry-run` option.

{% highlight bash %}
s3_website push
{% endhighlight %}

You can now check on your AWS management console if the files were uploaded and visit the S3 url to see your new shiny pages. The url will be composed by your `BUCKET_NAME` and the `REGION` were you created your bucket. More information on the [S3 documentation](http://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteEndpoints.html) about website endpoints.

{% highlight bash %}
http://BUCKET_NAME.s3-website-REGION.amazonaws.com
{% endhighlight %}

## Deploy the static website to Heroku with Rack
If for whatever reason you don't want to deploy your static website to S3 (or not only), you have another option by deploying it to Heroku as a simple `Rack` application.

*This tutorial is not about setting up an Heroku app, more information on how to get started with Heroku can be found in the [official documentation](https://devcenter.heroku.com/articles/getting-started-with-ruby).*

Start by adding these gems to the `Gemfile`:

{% highlight ruby %}
gem 'rack'
gem 'rack-contrib'
gem 'thin'
{% endhighlight %}

And run the `bundle install` command.

We are going to render a 404 page for all the requests that don't exist. Create a `404.html` file in the root directory.

{% highlight html %}
<!DOCTYPE html>
<html>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <title>Your awesome title</title>
  <!-- Custom CSS -->
  <link rel="stylesheet" href="/assets/application-713ae02b40c09a6a380683ba251607be.css">
  <body>
    <nav class="navbar navbar-inverse navbar-fixed-top" role="navigation">
      <div class="container">
        <div class="navbar-header">
          <a class="navbar-brand" href="/">Example Website</a>
        </div>
      </div>
    </nav>
    <div class="container" role="main">
      <h1>404 - Content Not Found</h1>
      <a href="/">Go back home</a>
    </div>
  </body>
</html>
{% endhighlight %}

To run our static website as a Rack application, we need a `config.ru` file to specify the configuration. In this case we are going to use `Rack::TryStatic` from the `rack-contrib` gem. It similar to the standard `Rack::Static` middleware, but it allows us to specify which files should be tried to be loaded for a request: for example a url like `/about` will try to load an `about.html`, `about/index.html` file or render the 404 page.

Credits to [Matthew Manning](http://mwmanning.com/2011/12/04/Jekyll-on-Heroku-Part-2.html) for the tutorial on how to deploy a static website to Heroku with `Rack::TryStatic`.

{% highlight ruby %}
require 'rack/contrib/try_static'

use Rack::TryStatic,
  urls: %w[/],
  root: '_site',
  try: ['.html', 'index.html', '/index.html'],
  header_rules: [
    [['html'],  { 'Content-Type'  => 'text/html; charset=utf-8' }],
    [['css'],   { 'Content-Type'  => 'text/css' }],
    [['js'],    { 'Content-Type'  => 'text/javascript' }],
    [['png'],   { 'Content-Type'  => 'image/png' }],
    ['/assets', { 'Cache-Control' => 'public, max-age=31536000' }],
  ]

run lambda { |env|
  [404, { 'Content-Type' => 'text/html' }, File.open('_site/404.html', File::RDONLY)]
}
{% endhighlight %}

Add some new files to the exclude key in your `_config.yml` file.

{% highlight yaml %}
# ...
exclude: ['README.md', 'Gemfile', 'Gemfile.lock', 's3_website.yml', 'Rakefile', 'config.ru', 'Procfile', 'vendor']
# ...
{% endhighlight %}

To try if our configuration is working you can either use the `rackup` command or a server like `thin` (that we bundled in the Gemfile). With `rackup` just visit `localhost:9292`, while `thin` runs the server by default on `localhost:3000`, but first remember to build the website with `jekyll build`.

{% highlight bash %}
jekyll build
rackup
# visit localhost:9292
thin start
# visit localhost:3000
{% endhighlight %}

For the purpouse of this guide, let's assume that you have already an Heroku account and everything ready to create a new app. First thing we need to create a git repository inside the project directory and create the first commit to be deployed.

First create a simple `.gitignore` file in which we specify not to commit the `_site` and `.sass-cache` directories and the `.env` file. We will build the files during the deployment with the clever trick of using the `assets:precompile` rake task to actually build the website with Jekyll.

{% highlight bash %}
.sass-cache
_site
.env
{% endhighlight %}

Then initialize the git repository, add and commit the files.

{% highlight bash %}
git init .
git add .
git commit -m 'First commit'
{% endhighlight %}

In order to compile our files and not needing to commit them to repository, we are going to use a clever trick. Create a `Rakefile` in the root directory of the project and add these two tasks:

{% highlight ruby %}
task default: 'assets:precompile'

namespace :assets do
  desc 'Precompile assets'
  task :precompile do
    Rake::Task['clean'].invoke
    sh 'bundle exec jekyll build'
  end
end

desc 'Remove compiled files'
task :clean do
  sh "rm -rf #{File.dirname(__FILE__)}/_site/*"
end
{% endhighlight %}

As you can see, we are overriding the usual `assets:precompile` task that is run to compile the assets during deployment to actually build the whole website with Jeyll. In this way Heroku will build the website for us!

Credits for this solution have to be given to [Jesse B. Hannah](http://jbhannah.net/blog/2013/01/16/jekyll-on-heroku-without-rack-jekyll-or-custom-buildpacks.html).

To tell Heroku how to run our Rack application, we need to create a `Procfile` file in the root directory. We are going to use thin and let the Heroku environment picks up the right port.

{% highlight bash %}
web: bundle exec thin start -p $PORT
{% endhighlight %}

Finally we came to the point where we can create the Heroku app and push our website to it! We can pass the name that we want to use for the app, `example-static-jekyll` in my case. Remember to change it with your own. The app will be then available at the url `YOUR_APP_NAME.herokuapp.com`. In my case I also create the app on the European region, if you are somewhere else, just skip it.

{% highlight bash %}
heroku apps:create example-static-jekyll --region eu
git push heroku master
{% endhighlight %}

If everything went right, you should be able to reach your website at `YOUR_APP_NAME.herokuapp.com`. For more information on how use your own domain and other things that you can do with your Heroku app, please refer to the [official documentation](https://devcenter.heroku.com/).

## Continuous Deployment/Delivery with Codeship.io
Whether you choose to deploy your website to S3 or to Heroku or both, it would be nice if you can have a [Continuous Deployment/Delivery](http://en.wikipedia.org/wiki/Continuous_delivery) process in place: you do some changes to your source code, commit and push and automatically some external system deploys/delivers your compiled website somewhere.

I use [Codeship](https://codeship.io/) for my own company and personal projects and I will recommend you to try it. You can signup for a free account with 100 included deploys a month (more than enough for a side project) and the setup is really easy if you host your repository on Github or Bitbucket and deploy to Herou or similar system. With a little bit of configuration you can also put complex continous integraion and delivery processes for big applications.

If you want to try out this way of deploying your website, just create an account and setup your project by giving the possibility to Codeship to pull your repository, depending if you are using Github or Bitbucket.

You need then to specify your build command to test and deploy your application/website. First select `Ruby` from the technology dropdown in the `Test` section and add these commands to the `Setup Commands` area:

{% highlight bash %}
bundle install
jekyll build
{% endhighlight %}

In order to "test" our application we will use the `s3_website push --dry-run` command in the case of S3 deployment. You can leave the test commands section empty if you plan to deploy to Heroku.

If you want to deploy to S3 with `s3_website` we need to specify our credentials in the `Environment` section in the sidebar. Just copy the variables that you set in the `.env` file.

{% highlight bash %}
AWS_ACCESS_KEY_ID=AKFKSKSDK8DSDSMFJA
AWS_SECRET_ACCESS_KEY=0fsd0fdsf0sd9f0fs0dBKmG6BfOVPYoHs
S3_BUCKET_NAME=example-website
{% endhighlight %}

Finally we need to configure the actual deployment process: go to the `Deployment` section and select `Custom Script` for S3 or `Heroku` for that. In the custom script (for S3 deployment) just add `s3_website push` and you are done.

For the Heroku setup you need to specifiy the name of the Heroku app that you created some steps before and give the Heroku api key that you can find following the instructions. You can also specify the url to check to confirm the proper deployment of the app.

This is it! To test the final process that you've just configured, just edit something in the project repository and push to Github or Bitbucket. Codeship will start the test and deployment process as soon it finds a new commit to the repo.

## Bonus: Add Basic HTTP Authentication
One of the reasons to move from a simple S3 hosting to an Heroku app, is because we can leverage on the amount of Rack middlewares, for example to add a basic HTTP authentication to our static website with three lines of code.

To do so, add the `Rack::Auth::Basic` middleware to the `config.ru` file, before the `Rack::TryStatic` configuration.

{% highlight ruby %}
# ...
use Rack::Auth::Basic, 'Restricted Area' do |username, password|
  [username, password] == ['my_user', 'secret_password']
end
# ...
{% endhighlight %}

Remember to change the username and password with something less trivial. Commit and push your app to Heroku and enjoy an added layer of security to your website.

## Conclusion
Whoa! It was a long ride, wasn't it?

You can find the code generated for this tutorial on [Github](https://github.com/lucatironi/example-static-jekyll/tree/master).

I hope I gave you some good tips on how to leverage the potentialities of the awesome Jekyll gem, S3, Heroku and Codeship. If you have any question feel free to comment on this page or send me and email to [luca.tironi@gmail.com](mailto:luca.tironi@gmail.com).

Bye,

Luca

