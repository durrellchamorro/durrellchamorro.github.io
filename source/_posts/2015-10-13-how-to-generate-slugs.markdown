---
layout: post
title: "How to Generate Slugs"
date: 2015-10-13 20:06:52 -0700
comments: true
categories: rails
---
Slugs make pretty URLs. Pretty URLs help optimize your app for search engines and they
hide sensitive database information. In this post, I'll explain, line by line, one approach
to generating slugs. <!--more-->

Here is the code I will explain. You can include it in your project as
a gem by putting ``gem 'sluggable_durrell'`` in your Gemfile or you can just copy
the code below into a module.

```ruby
module Sluggable
  extend ActiveSupport::Concern

  included do
    before_save :generate_slug
    class_attribute :slug_column
  end

  def to_param
    slug
  end

  def generate_slug
    the_slug = to_slug(self.send(self.class.slug_column.to_sym))
    object = self.class.find_by(slug: the_slug)
    count = 2
    while object && object != self
      the_slug = append_suffix(the_slug, count)
      object = self.class.find_by(slug: the_slug)
      count += 1
    end
    self.slug = the_slug.downcase
  end

  def append_suffix(str, count)
    if str.split('-').last.to_i != 0
      return str.split('-').slice(0...-1).join('-') + "-" + count.to_s
    else
      return str + "-" + count.to_s
    end
  end

  def to_slug(name)
    str = name.parameterize.gsub("_", "").downcase
  end

  module ClassMethods
    def sluggable_column(column_name)
      self.slug_column = column_name
    end
  end
end
```

```
extend ActiveSupport::Concern
```
To make a long story short, this line lets the methods
in the ``Sluggable`` module become instance methods of the class that includes the ``Sluggable`` module.

```ruby
included do
  before_save :generate_slug
  class_attribute :slug_column
end
```
This fires the code inside the do block when the module is included in the class.
The ``before_save`` method tells the class to call ``generate_slug`` on each instance of the class
before the instance is saved. The ``class_attribute`` method adds an attribute named ``slug_column`` to the
class.

Let's say you have a Categories controller with a ``show`` action like this:

```ruby
def show
  @category = Category.find(params[:id])
end
```

Let's say you have a corresponding view, ``/views/categories/show.html.erb`` and you have the line: ``resources :categories``
in your ``routes.rb`` file which gives you a ``category_path``
helper such that you can link to this show page like so:
```erb
<%= link_to @category.name, category_path(@category) %>
```
or if you prefer the shorter way:
```erb
<%= link_to @category.name, @category %>
```

If you hover over the link with your mouse in Chrome, you will see the URL
that link points to in the bottom left hand corner of your browser. In this case
the development URL might look something like this:
``http://localhost:3000/categories/1``. This is what we would expect since if you go to
``http://localhost:3000/rails/info/routes`` you will see the path for the ``category_path`` helper
is ``/categories/:id(.:format)``.

The 1 at the end of the URL in the example above is is the ``:id`` or in other words the
primary key for that particular category. How does the link know to point to the correct ``:id`` if
all it is given is the ``@category`` object? The trick is behind the scenes rails calls ``to_param`` on
``@category`` and ``to_param`` returns the primary key in a string as shown below in the rails console.

```bash
category = Category.first
category.to_param
 => "1"
```
With that being said I can now explain why the ``to_param`` method is overridden in the Sluggable module:

```ruby
def to_param
  slug
end
```

Instead of returning the primary key of the category, we want to return the slug so the URL
will read like this instead ``http://localhost:3000/categories/programming``. If you put a
``binding.pry`` in the show action like so:

```bash
   19: def show
=> 20:   binding.pry
   21:   @category = Category.find(params[:id])
   22: end

[1] pry(#<CategoriesController>)> params
=> {"action"=>"show", "controller"=>"categories", "id"=>"programming"}
```
you can see that the ``:id`` is now equal to the slug value instead of the primary key which is what we'd
expect since we overrode the ``to_param`` method to return the slug instead of the primary key. In this example,
I already generated the slug for this category which is why it returned "programming". Now I can explain how to
generate the slug.













