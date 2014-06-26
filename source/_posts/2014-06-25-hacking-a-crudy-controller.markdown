---
layout: post
title: "Hacking a CRUDy Controller"
date: 2014-06-25 22:41:49 -0700
comments: true
categories: [rails, controllers]
---

> An experiment on finding a way to create a base controller to inherit from when dealing mostly with CRUD operations.

Have you noticed that if you are working on an app, and need to create an admin interface for it, most of the work to be performed is your typical create, update, destroy and list actions?

In my "day-to-day work" in Rails-based applications, this is typically the case. I bet the same case presents to you as well, and though I am aware of sound solutions like the [ActiveAdmin](https://github.com/gregbell/active_admin) and [RailsAdmin](https://github.com/sferik/rails_admin) gems, in some edge cases the queries returned by their default implementations always leave something to be desired.

Now, let's not confuse this with an attachment to the NIH (Not Invented Here) syndrome, but sometimes, having an in-house and flexible implementation is more than sufficient to achieve "Just the Right Feature<sup>TM</sup>".

Getting back to day-to-day cases in controllers, specially in a Rails application, you can easily find a pattern:

* you need to retrieve a record to display it
* you need to retrieve a record to edit/update it
* you need to delete a record
* you need to create a new record
* you need to list all existing records

Though one of Rails tenets is DRY (Don't Repeat Yourself), you find yourself adding similar actions across multiple controller files, only changing model and routes specific details.

### Let's see an example

Suppose you're working in a Rails app that deals with a Books and Movies catalog (among other things). You will obviously need an interface to perform CRUD operations on these resources, so you start working your way through the first controller.

```ruby
# app/controllers/admin/books_controller.rb
class Admin::BooksController < ApplicationController
  def index
    # code to list all books
    # could include logic for ordering, pagination, etc.
  end
  
  def show
    # code to retrieve a single book
  end
  
  def new
    # code to instantiate a new book
  end
  
  def create
    # code to instantiate and persist a new book
  end
  
  def edit
    # code to retrieve a single book
  end
  
  def update
    # code to retrieve a single book
    # and persist updated info
  end
  
  def destroy
    # code to delete a single book
  end
  
  private
  
  def books_params
    # rules for accepted params to pass to a book instance
  end
end
```

Then, comes the time to work on the next controller and you come up with this:

```ruby
# app/controllers/admin/movies_controller.rb
class Admin::MoviesController < ApplicationController
  def index
    # code to list all movies
    # could include logic for ordering, pagination, etc.
  end
  
  def show
    # code to retrieve a single movie
  end
  
  def new
    # code to instantiate a new movie
  end
  
  def create
    # code to instantiate and persist a new movie
  end
  
  def edit
    # code to retrieve a single movie
  end
  
  def update
    # code to retrieve a single movie
    # and persist updated info
  end
  
  def destroy
    # code to delete a single movie
  end
  
  private
  
  def movies_params
    # rules for accepted params to pass to a movie instance
  end
end
```

Then comes the time to work on another controller, and another, and another, and yet another controller... and if you pause for a moment and look at the big picture... you basically have only one controller repeated N times your models count.

How could we refactor these controllers into a flexible base use case? <br />Well, I recently had some time to read [Growing Rails Applications in Practice](https://leanpub.com/growing-rails) and pretty much liked the suggestion on the __"Beautiful controllers"__ chapter and decided to take it as a base to build upon... after practicing in an app I was developing at that moment, I came up with:

### A small step in refactoring

Going back to our example... Let's declare a base controller and a mixable methods module:

```ruby
# app/controllers/admin/crud_methods.rb
module Admin
  module CRUDMethods
    def list_resources
      @resources = model.load
    end
    
    def build_resource(attrs = {})
      @resource = model.new(attrs)
    end
    
    def load_resource(id)
      @resource = model.find(id)
    end
    
    def save_resource
      if @resource.save
        success_action('Successfully created resource.')
      else
        error_action(@resource, :new)
      end
    end
    
    def update_resource(attrs)
      if @resource.update_attributes(attrs)
        success_action('Successfully updated resource.')
      else
        error_action(@resource, :edit)
      end
    end
    
    def delete_resource(id)
      load_resource(id)
      @resource.destroy
      destroy_success_action
    end
  end
end

# app/controllers/admin/base.rb
class Admin::BaseController
  include Admin::CRUDMethods
  
  class NotImplementedError < StandardError; end
  
  def index
    list_resources
  end
  
  def new
    build_resource
  end
  
  def create
    build_resource(accepted_params)
    save_resource
  end
  
  def show
    load_resource(id_key)
  end
  
  def edit
    load_resource(id_key)
  end
  
  def update
    load_resource(id_key)
    update_resource(accepted_params)
  end
  
  def destroy
    delete_resource(id_key)
  end
  
  private
  
  def model
    fail(
      NotImplementedError,
      'Configure main ActiveRecord class to manage in this controller.'
    )
  end
  
  def id_key
    fail(
      NotImplementedError,
      'Configure params[:key] needed to retrieve a record for this controller.'
    )
  end
  
  def accepted_params
    fail(
      NotImplementedError,
      'Configure the required key and accepted params for a record in this controller'
    )
  end
  
  def success_action
    fail(
      NotImplementedError,
      'Configure what to do when saving/updating a record succeeds.'
    )
  end
  
  def error_action(resource, view)
    @resource = resource
    flash[:error] = 'Something went wrong.'
    render view
  end
  
  def destroy_success_action
    fail(
      NotImplementedError,
      'Configure what to do when deleting a record succeeds.'
    )
  end
end
```

I know, I know... this looks like an awful lot of boilerplate... but bear with me for a second while I show you how the Books and Movies controllers will look like now that we have this base class in place.

```ruby
# app/controller/admin/books_controller.rb
class Admin::BooksController < Admin::BaseController
  private
  
  def model
    Book
  end
  
  def id_key
    params[:id]
  end
  
  def accepted_params
    params.require(:book).permit(
      :title, :author, :isbn, :page_count, :price, :category, :etc
    )
  end
  
  def success_action(message)
    redirect_to admin_book_path(@resource), notice: message
  end
  
  def destroy_success_action(message)
    redirect_to admin_books_path, notice: message
  end
end

# app/controllers/admin/movies_controller.rb
class Admin::MoviesController < Admin::BaseController
  private
  
  def model
    Movie
  end
  
  def id_key
    params[:id]
  end
  
  def accepted_params
    params.require(:movie).permit(
      :title, :director, :upc_code, :duration, :price, :category, :etc
    )
  end

  def success_action(message)
    redirect_to admin_movie_path(@resource), notice: message
  end
  
  def destroy_success_action(message)
    redirect_to admin_movies_path, notice: message
  end
end
```
If you take a look at the resulting controllers, you'll notice we have separated the things that change (params, model, routes, etc. on each derivative controller), from those that don't (the base CRUD methods). _I bet there's even a pattern name for this ; )_

Now, suppose you need to work on building CRUD actions for a MusicAlbum controller... guess what that would look like? And the next controller you need to implement the same actions all over again?

Exactly... now that we have ourselves a base platform, we can re-use and override at will; do you need a more complicated logic in any of the actions? do you need to ensure your queries are database optimized when retrieving a collection or a single record and its associations? <br />Not a problem, simply override the method you need, be it the main CRUD-based action in the controller (`index, show, update, etc.`), or the abstract methods provided by the base controller or the mixable module (`load_resource, update_resource, etc.`) and voilá! you can have just the flexibility you need while keeping your eyes in the big picture.

While this is not a silver bullet design, I pretty much like what it provides so far. Let's not forget that we don't need all our controllers to inherit from this base controller; we need to evaluate the corresponding use case, but if CRUD operations is what you need repeated ad-infinitum in a uniform approach, why not take this approach for a spin?

Do you like what we've done here? Do you have a different opinion or approach? I'd love to know your comments on this topic. You can [tweet](http://twitter.com/antillas21) or [email](mailto:antillas21@gmail.com) me and start a conversation.