---
layout: post
title: Source code for the authentication tutorials
tagline: The Ruby on Rails application and the Android app code repositories
description: Following several requests, I've decided to publish the revised code that I used to write the previous tutorials.
category: code
tags: [Ruby on Rails, Devise, authentication, API]
---
{% include JB/setup %}

Following several requests, I've decided to publish the revised code that I used to write the previous tutorials. I revised the code since the first post and pushed them to Github.

I hope they can be useful for your projects, feel free to fork and contribute to them if you wish to.

### Ruby on Rails Backend/API
Repository: [github.com/lucatironi/authexample-ror-tutorial](https://github.com/lucatironi/authexample-ror-tutorial)

You can find the guide to create this Android application in the [part one](http://lucatironi.github.io/tutorial/2012/10/15/ruby_rails_android_app_authentication_devise_tutorial_part_one) and [part three](http://lucatironi.github.io/tutorial/2012/12/07/ruby_rails_android_app_authentication_devise_tutorial_part_three) of the authentication tutorials.

#### Installation

{% highlight bash %}
git clone https://github.com/lucatironi/authexample-ror-tutorial.git
cd authexample-ror-tutorial
bundle install
rake db:create db:migrate
rails s
{% endhighlight %}

### Android App
Repository: [github.com/lucatironi/authexample-android-tutorial](https://github.com/lucatironi/authexample-android-tutorial)

You can find the guide to create this Android application in the [part two](http://lucatironi.github.io/tutorial/2012/10/16/ruby_rails_android_app_authentication_devise_tutorial_part_two) and [part three](http://lucatironi.github.io/tutorial/2012/12/07/ruby_rails_android_app_authentication_devise_tutorial_part_three) of the authenticaion tutorials.

#### Dependencies

[ActionBarSherlock](http://actionbarsherlock.com): see [usage](http://actionbarsherlock.com/usage.html) on how to install it as a library for this project.

#### Installation

- Clone or [download](https://github.com/lucatironi/authexample-android-tutorial/archive/master.zip) the project
- Import it in your Eclipse's workspace: **"File > Import > Existing Android Code Into Workspace"**
- Browse for the directory and check **"Copy projects into workspace"**
- [Download](http://actionbarsherlock.com/download.html) ActionBarSherlock
- Extract the zip file and import it as a library: **"File > New > Other ... > Android Project from Existing Code"**
- Navigate the ActionBarSherlock directory and select the "actionbarsherlock" dir.
- Ignore the warning/error that may arise and right-click on the new project and select **"Refactor > Rename"** and rename the project "ABS" (or whatever you like).
- Right-click on the AuthExample project and select **"Properties"**: click on the **"Android"** section and add the just added ABS library in the **"Library"** sub-section.
