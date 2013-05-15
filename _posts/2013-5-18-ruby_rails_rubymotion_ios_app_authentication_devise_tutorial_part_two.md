---
layout: post
title: Ruby on Rails and RubyMotion Authentication Part Two
tagline: A Complete iOS App with a Rails API backend
description: The second part of the RubyMotion tutorial will guide you to complete the ToDo example app with all the features like tasks display, task creation and completion.
category: tutorial
tags: [Ruby on Rails, Rubymotion, iPhone, iOS, Devise, authentication, API]
---
{% include JB/setup %}

To do so we create a simple Ruby model/class called <code>Task</code> and we use some Ruby meta-programming to create some accessors method (getter and setter); it seems complicated but it's not: for each of the the Task's properties (id, title and completed flag) we create a getter and a setter through <code>attr_accessor</code> and we override the <code>initialize</code> method to pass an Hash of values to the <code>Task.new</code> call to set them to the correct property.

{% highlight ruby %}
# file app/models/Task.rb
class Task
  PROPERTIES = [:id, :title, :completed]

  PROPERTIES.each do |prop|
    attr_accessor prop
  end

  def initialize(hash = {})
    hash.each do |key, value|
      if PROPERTIES.member? key.to_sym
        self.send((key.to_s + "=").to_s, value)
      end
    end
  end

  def self.all(&block)
    headers = {
      'Content-Type' => 'application/json',
      'Authorization' => "Token token=\"#{App::Persistence['authToken']}\""
    }

    BW::HTTP.get("http://localhost:3000/api/v1/tasks.json", { headers: headers }) do |response|
      if response.status_description.nil?
        App.alert(response.error_message)
      else
        if response.ok?
          json = BW::JSON.parse(response.body.to_str)
          tasksData = json[:data][:tasks] || []
          tasks = tasksData.map { |task| Task.new(task) }
          block.call(tasks)
        elsif response.status_code.to_s =~ /40\d/
          App.alert("Not authorized")
        else
          App.alert("Something went wrong")
        end
      end
    end
  end
end
{% endhighlight %}

![TasksController](/assets/uploads/images/TasksController.png)

*The TasksController populated with the tasks read from the backend*