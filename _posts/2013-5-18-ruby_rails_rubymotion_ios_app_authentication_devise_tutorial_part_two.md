---
layout: post
title: Ruby on Rails and RubyMotion Authentication Part Two
tagline: A Complete iOS App with a Rails API backend
description: The second part of the RubyMotion tutorial will guide you to complete the ToDo example app with all the features like tasks display, task creation and completion.
category: tutorial
tags: [Ruby on Rails, Rubymotion, iPhone, iOS, Devise, authentication, API]
---
{% include JB/setup %}

## Displaying User's Tasks

To do so we create a simple Ruby model/class called <code>Task</code> and we use some Ruby meta-programming to create some accessors method (getter and setter); it seems complicated but it's not: for each of the the Task's properties (id, title and completed flag) we create a getter and a setter through <code>attr_accessor</code> and we override the <code>initialize</code> method to pass an Hash of values to the <code>Task.new</code> call to set them to the correct property.

{% highlight ruby %}
# file app/models/Task.rb
class Task
  API_TASKS_ENDPOINT = "http://localhost:3000/api/v1/tasks"

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
    BW::HTTP.get("#{API_TASKS_ENDPOINT}.json", { headers: Task.headers }) do |response|
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

  def self.headers
    {
      'Content-Type' => 'application/json',
      'Authorization' => "Token token=\"#{App::Persistence['authToken']}\""
    }
  end
end
{% endhighlight %}

<code>tableView(tableView, numberOfRowsInSection:section)</code>
<code>tableView(tableView, cellForRowAtIndexPath:indexPath)</code>
<code>refresh</code>

{% highlight ruby %}
# file app/controllers/TasksController.rb
class TasksListController < UIViewController
  attr_accessor :tasks

  def self.controller
    @controller ||= TasksListController.alloc.initWithNibName(nil, bundle:nil)
  end

  def viewDidLoad
    super

    self.tasks = []

    self.title = "Tasks"
    self.view.backgroundColor = UIColor.whiteColor

    logoutButton = UIBarButtonItem.alloc.initWithTitle("Logout",
                                                       style:UIBarButtonItemStylePlain,
                                                       target:self,
                                                       action:'logout')
    self.navigationItem.leftBarButtonItem = logoutButton

    refreshButton = UIBarButtonItem.alloc.initWithBarButtonSystemItem(UIBarButtonSystemItemRefresh,
                                                                      target:self,
                                                                      action:'refresh')
    self.navigationItem.rightBarButtonItems = [refreshButton]

    @tasksTableView = UITableView.alloc.initWithFrame([[0, 0], [self.view.bounds.size.width, self.view.bounds.size.height]], style:UITableViewStylePlain)
    @tasksTableView.dataSource = self
    @tasksTableView.delegate = self
    @tasksTableView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight

    self.view.addSubview(@tasksTableView)

    refresh if App::Persistence['authToken']
  end

  # UITableView delegate methods
  def tableView(tableView, numberOfRowsInSection:section)
    self.tasks.count
  end

  def tableView(tableView, cellForRowAtIndexPath:indexPath)
    @reuseIdentifier ||= "CELL_IDENTIFIER"

    cell = tableView.dequeueReusableCellWithIdentifier(@reuseIdentifier) || begin
      UITableViewCell.alloc.initWithStyle(UITableViewCellStyleDefault, reuseIdentifier:@reuseIdentifier)
    end

    task = self.tasks[indexPath.row]

    cell.textLabel.text = task.title

    cell
  end

  # Controller methods
  def refresh
    SVProgressHUD.showWithStatus("Loading", maskType:SVProgressHUDMaskTypeGradient)
    Task.all do |jsonTasks|
      self.tasks.clear
      self.tasks = jsonTasks
      @tasksTableView.reloadData
      SVProgressHUD.dismiss
    end
  end

  def logout
    UIApplication.sharedApplication.delegate.logout
  end
end
{% endhighlight %}

![TasksController](/assets/uploads/images/TasksController.png)

*The TasksController populated with the tasks read from the backend*

## Create a new Task

<code>addNewTask</code>

{% highlight ruby %}
# file app/controllers/TasksListController.rb
class TasksListController < UIViewController

  def viewDidLoad
    # Other code

    newTaskButton = UIBarButtonItem.alloc.initWithBarButtonSystemItem(UIBarButtonSystemItemAdd,
                                                                      target:self,
                                                                      action:'addNewTask')
    self.navigationItem.rightBarButtonItems = [refreshButton, newTaskButton]

    # Other code
  end

  # Other code

  def addNewTask
    @newTaskController = NewTaskController.alloc.init
    @newTaskNavigationController = UINavigationController.alloc.init
    @newTaskNavigationController.pushViewController(@newTaskController, animated:false)

    self.presentModalViewController(@newTaskNavigationController, animated:true)
  end
end
{% endhighlight %}

<code>NewTaskController</code>

{% highlight ruby %}
# file app/controllers/NewTaskController.rb
class NewTaskController < Formotion::FormController
  def init
    form = Formotion::Form.new({
      sections: [{
        rows: [{
          title: "Title",
          key: :title,
          placeholder: "Task title",
          type: :string,
          auto_correction: :yes,
          auto_capitalization: :none
        }],
      }, {
        rows: [{
          title: "Save",
          type: :submit,
        }]
      }]
    })
    form.on_submit do
      self.createTask
    end
    super.initWithForm(form)
  end

  def viewDidLoad
    super

    self.title = "New Task"

    cancelButton = UIBarButtonItem.alloc.initWithTitle("Cancel",
                                                       style:UIBarButtonItemStylePlain,
                                                       target:self,
                                                       action:'cancel')
    self.navigationItem.rightBarButtonItem = cancelButton
  end

  def cancel
    self.navigationController.dismissModalViewControllerAnimated(true)
  end
end
{% endhighlight %}


<code>Task.create</code>

{% highlight ruby %}
# file app/models/Task.rb
class Task
  # Other code

  def self.create(params = {}, &block)
    data = BW::JSON.generate(params)

    BW::HTTP.post("#{API_TASKS_ENDPOINT}.json", { headers: Task.headers, payload: data } ) do |response|
      if response.status_description.nil?
        App.alert(response.error_message)
      else
        if response.ok?
          json = BW::JSON.parse(response.body.to_str)
          block.call(json)
        elsif response.status_code.to_s =~ /40\d/
          App.alert("Task creation failed")
        else
          App.alert(response.to_str)
        end
      end
    end
  end
end
{% endhighlight %}

<code>createTask</code>

{% highlight ruby %}
# file app/controllers/NewTaskController.rb
class NewTaskController < Formotion::FormController
  # Other code

  def createTask
    title = form.render[:title]
    if title.strip == ""
      App.alert("Please enter a title for the task.")
    else
      taskParams = { task: { title: title } }

      SVProgressHUD.showWithStatus("Loading", maskType:SVProgressHUDMaskTypeGradient)
      Task.create(taskParams) do |json|
        App.alert(json['info'])
        self.navigationController.dismissModalViewControllerAnimated(true)
        TasksListController.controller.refresh
        SVProgressHUD.dismiss
      end
    end
  end
end
{% endhighlight %}

## Mark the task complete

<code>tableView(tableView, cellForRowAtIndexPath:indexPath)</code>
<code>tableView(tableView, didSelectRowAtIndexPath:indexPath)</code>

{% highlight ruby %}
# file app/controllers/TasksListController.rb
class TasksListController < UIViewController
  # Other code

  def tableView(tableView, cellForRowAtIndexPath:indexPath)
    @reuseIdentifier ||= "CELL_IDENTIFIER"

    cell = tableView.dequeueReusableCellWithIdentifier(@reuseIdentifier) || begin
      UITableViewCell.alloc.initWithStyle(UITableViewCellStyleDefault, reuseIdentifier:@reuseIdentifier)
    end

    task = self.tasks[indexPath.row]

    cell.textLabel.text = task.title

    if task.completed
      cell.textLabel.color = '#aaaaaa'.to_color
      cell.accessoryType = UITableViewCellAccessoryCheckmark
    else
      cell.textLabel.color = '#222222'.to_color
      cell.accessoryType = UITableViewCellAccessoryNone
    end

    cell
  end

  def tableView(tableView, didSelectRowAtIndexPath:indexPath)
    tableView.deselectRowAtIndexPath(indexPath, animated:true)
    task = self.tasks[indexPath.row]

    task.toggle_completed do
      refresh
    end
  end
end
{% endhighlight %}

<code>toggle_completed</code>

{% highlight ruby %}
# file app/models/Task.rb
class Task
  # Other code

  def toggle_completed(&block)
    BW::HTTP.put("#{API_TASKS_ENDPOINT}/#{self.id}/#{self.completed ? 'open' : 'complete'}.json", { headers: Task.headers }) do |response|
      if response.status_description.nil?
        App.alert(response.error_message)
      else
        if response.ok?
          json = BW::JSON.parse(response.body.to_str)
          taskData = json[:data][:task]
          task = Task.new(taskData)
          block.call(task)
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
