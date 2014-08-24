---
layout: post
title: Jekyll Static Website with Continous Deployment to S3/Heroku
tagline: Develop a static website with Jekyll and deploy automatically to S3 and/or Heroku
description: Jekyll can be extendend to use a Rails-like assets pipeline to develop even complex Single Page Applications and can be configured to be automatically deployed to a public S3 bucket and/or to Heroku, using a Continous Deployment service like Codeshio.io
category: tutorial
tags: [Ruby, Jekyll, S3, Heroku, Codeship.io]
---

## Requirements

* [jekyll](http://jekyllrb.com/)
* [jekyll-assets](https://github.com/ixti/jekyll-assets) (optional)
* [bootstrap-sass](https://github.com/twbs/bootstrap-sass) (optional)
* [s3_website](https://github.com/laurilehmijoki/s3_website)
* [Amazon S3](https://aws.amazon.com/s3)
* [CodeShip.io](https://github.com/laurilehmijoki/s3_website) (optional)
* [Heroku](http://heroku.com) (optional)

## Basic setup

Install the [Jekyll](http://jekyllrb.com) gem

{% highlight bash %}
$ gem install jekyll
{% endhighlight %}

Create a new jekyll project

{% highlight bash %}
$ jekyll new example_website
$ cd  example_website
{% endhighlight %}

Run the jekyll server

{% highlight bash %}
$ jekyll serve --watch
{% endhighlight %}

and visit `http://localhost:4000`: you will see a nice looking page with some examples of the potentialities of Jekyll.

## Configure the project

For the pourpose of this tutorial, I want to start from a clean slate. I will remove the boilerplate files created by Jekyll and create a slightly different directory structure. You are free to skip this passage, if you wish to keep the example files.

{% highlight bash %}
$ rm -fr css _posts _layouts/post.html _layouts/page.html about.md feed.xml
$ mkdir -p _assets/javascripts/ _assets/stylesheets/ _plugins/ _vendors/javascripts/ _vendors/stylesheets/ assest/
$ touch _assets/javascripts/application.js.coffee _assets/stylesheets/application.css.scss _plugins/ext.rb Gemfile
{% endhighlight %}

Project directory structure

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

Create a `Gemfile` in the root directory and add this gems.

{% highlight ruby %}
source 'https://rubygems.org'

gem 'jekyll-assets'
gem 'bootstrap-sass'
gem 'uglifier'
{% endhighlight %}

Install the gems

{% highlight bash %}
$ bundle install
{% endhighlight %}

## Assets Pipeline

Add a the `jekyll-assets` plugin to the `_plugins/ext.rb` file. This plugin allows to use a Rails-like assets pipeline in a Jekyll project, including using CoffeScript, Sass, Less and ERB as intermediate languages to write your assets and pages, automatic minification of code, cache busting and many other cool features that you can discover on the [Github repository](https://github.com/ixti/jekyll-assets) of the plugin.

{% highlight ruby %}
require 'jekyll-assets'
require 'jekyll-assets/bootstrap'
{% endhighlight %}

We also added the bootstrap plugin to automatically use the framework in the project. Feel free to skip this if you want to start from scratch or with other frameworks/libraries.

More information about Jekyll plugins can be found in the [Jekyll documentation](http://jekyllrb.com/docs/plugins/).

In order to use the assets pipeline, we need to add the `assets` configuration to the `_config.yml` file.

{% highlight yaml %}
# Site settings
title: Your awesome title
email: your-email@domain.com
description: "Write an awesome description for your new site here. You can edit this line in _config.yml. It will appear in your document head meta (for Google search results) and in your feed.xml site description."
baseurl: ""
url: "http://yourdomain.com"

# Build settings
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
$ curl -o ./_vendors/javascripts/jquery.min.js http://code.jquery.com/jquery.min.js
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
$ jekyll serve --watch
{% endhighlight %}

Visit `localhost:4000` and check the results.

## Deploy to S3

Add the `s3_website` gem to the `Gemfile`

{% highlight ruby %}
gem 's3_website'
{% endhighlight %}

and bundle

{% highlight bash %}
$ bundle install
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
$ s3_website push --dry-run
{% endhighlight %}

It will execute all the commands to build the files from Jekyll and test the permissions to upload them on S3, without actually doing it and cleaning up afterwise.

If everything went ok, you can push your website for real, by removing the `--dry-run` option.

{% highlight bash %}
$ s3_website push
{% endhighlight %}

You can now check on your AWS management console if the files were uploaded and visit the S3 url to see your new shiny pages. The url will be composed by your `BUCKET_NAME` and the `REGION` were you created your bucket. More information on the [S3 documentation](http://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteEndpoints.html) about website endpoints.

{% highlight bash %}
http://BUCKET_NAME.s3-website-REGION.amazonaws.com
{% endhighlight %}

## Continous Deployment with Codeship.io

