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
    the_slug = to_slug(self.send(self.class.slug_column))
    object = self.class.find_by(slug: the_slug)
    count = 2
    while object && object != self
      the_slug = append_suffix(the_slug, count)
      object = self.class.find_by(slug: the_slug)
      count += 1
    end
    self.slug = the_slug
  end

  def append_suffix(the_slug, count)
    if the_slug.split('-').last.to_i != 0
      return the_slug.split('-').slice(0...-1).join('-') + "-" + count.to_s
    else
      return the_slug + "-" + count.to_s
    end
  end

  def to_slug(name)
    name.parameterize.gsub("_", "").downcase
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
in the ``Sluggable`` module become instance methods of the class that includes the ``Sluggable`` module. It
also looks for a module named ``ClassMethods`` and includes the methods inside the ``ClassMethods`` module as
class methods of the class that includes the Sluggable module. Click <a href="http://www.fakingfantastic.com/2010/09/20/concerning-yourself-with-active-support-concern/">here</a>
for a more in depth explanation of how ``ActiveSupport::Concern`` works.

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

```ruby
def generate_slug
  the_slug = to_slug(self.send(self.class.slug_column))
...
```
``self.class`` returns the class of ``self`` so in our example ``self.class.slug_column``
is the same as ``Category.slug_column``. The reason we can call ``slug_column`` on the Category
class is because as I explained in the beginning, ActiveSupport::Concern looks for a module named ClassMethods
and includes all the methods inside that module as class methods of the class that includes the Sluggable module. In
our example the Category class/model looks something like this:

```ruby
class Category < ActiveRecord::Base
  include Sluggable
  sluggable_column :name
end
```

As you can see, the ``name`` column from the categories table is being sent to the ``sluggable_column`` method as a
symbol.

```ruby
module ClassMethods
  def sluggable_column(column_name)
    self.slug_column = column_name
  end
end
```

The ``sluggable_column`` method takes ``:name`` and sets the ``slug_column`` attribute equal to ``:name``.
Therefore, ``self.class.slug_column`` is equal to ``:name`` as demonstrated in the rails console:

```bash
category = Category.first
category.class.slug_column
=> :name
```
 Now we can read:

```ruby
the_slug = to_slug(self.send(self.class.slug_column))
```
as:

```ruby
the_slug = to_slug(self.send(:name)
```

``self.send(:name)`` is the same as ``self.name`` so we can just read this as:

```ruby
the slug = to_slug(self.name)
```

Now I can explain what the ``to_slug`` method does.

```ruby
def to_slug(name)
  name.parameterize.gsub("_", "").downcase
end
```

``self.name``returns a string and the ``parameterize`` method is being called on that string. The
``parameterize`` method replaces special characters in a string so that it may be used as part of a ‘pretty’ URL.
The ``parameterize`` method doesn't replace the underscore character so ``gsub`` is called on the string to remove any underscore and then
downcase is called on it to remove all uppercase characters. This way we'll have a string that can be used in a URL.

```ruby
def generate_slug
  the_slug = to_slug(self.send(self.class.slug_column))
  #in our ongoing example, the line above becomes: the_slug = "programming"
  #in our ongoing example, the line below becomes:
  #object = Category.find_by(slug: "programming")
  object = self.class.find_by(slug: the_slug)
...
end
```

In this case ``object`` will equal ``nil`` because we haven't saved the new category to the database yet and we are assuming
I didn't already save a category named "programming". Since ``object``
is nil, the while loop will be skipped. This line:

```ruby
self.slug = the_slug
```

sets the slug column equal to "programming". Since we called
``before_save :generate_slug`` the slug will be generated in the create action when
``@category = Category.new(category_params)`` is called. Then the slug will be saved to the database with the ``@category`` instance
when the create action calls ``@category.save``.

Let's say there is already a "programming" category and a user creates a category named "programming" again. We
don't want different pages to have identical URLs. The solution is to append a suffix to
"programming" and make sure that suffix is never the same. This example might seem contrived, but if you are
generating the slug based on, say, the title of a post instead of the name of a category, then it is realistic to think
more than one user might have thought of the same post title. Another way around this is to validate the uniqueness of the
sluggable column in the model, but appending a suffix like this allows for flexibility in design and adds a layer
of protection against duplicate URLs.

Continuing with our example, since there is already an instance of Category with "programming" in the slug column, this time ``object`` will not be
equal to ``nil`` at this point in the method:

```ruby
object = self.class.find_by(slug: the_slug)
```

so the first condition of the while loop will be met. The second condition is that ``object`` not
be equal to ``self``. That condition is necessary in case ``self`` is only being updated. If ``self`` is being updated,
``object`` will equal ``self`` and we don't want to append anything to the URL. In the case where ``self`` is being created ``object``
will instead be equal to the previous instance of Category with "programming" in the slug column
so both conditions of the while loop will be true. In the line below ``the_slug`` is being set to "programming-2".
Let's look at how the ``append_suffix`` method works.

```ruby
the_slug = append_suffix(the_slug, count)
```

The first line of the ``append_suffix`` method is:

```ruby
if the_slug.split('-').last.to_i != 0
```

This is saying, since the ``parameterize`` method used inside the ``to_slug`` method generates
a string that replaces spaces with dashes, ``the_slug`` should either be a one word string,
or it should be a string of any number of words separated by dashes instead of spaces.
Therefore, call split on that string to get an array formatted like this:

```bash
the_slug = "a-url-safe-string"
the_slug.split('-')
=> ["a", "url", "safe", "string"]
```
then it takes the last element in that array and calls ``to_i`` on it. Calling ``to_i`` on
a string of letters returns 0, and calling ``to_i`` on a string that only contains a Fixnum or a Float
returns the appropriate integer.

```ruby
def append_suffix(the_slug, count)
  if the_slug.split('-').last.to_i != 0
    return the_slug.split('-').slice(0...-1).join('-') + "-" + count.to_s
  else
    return the_slug + "-" + count.to_s
  end
end
```

Therefore, this is saying if the last element in the array is a number, then take all the elements
but the last one in that array and put them into a string, separating them with a
dash. Then add a dash to the end of the string and then add the integer ``count`` equals to the end
of the string such that if ``the_slug`` was ``"programming-2"`` it would become ``"programming-3"``
Otherwise, the last element in the array is not a number, so just add a dash to the end of ``the_slug`` and
then add the integer ``count`` equals to the end of ``the_slug`` such that if ``the_slug`` was ``"programming"``
it would become ``"programming-2"``.

Now, back to the next line of the while loop inside the ``generate_slug`` method:

```ruby
object = self.class.find_by(slug: the_slug)
```

This is resetting ``object``. If there is an instance of Category saved in the database
with a slug value equal to ``the_slug`` then ``object`` will be truthy so the loop will continue.

```ruby
def generate_slug
...
  while object && object != self
    the_slug = append_suffix(the_slug, count)
    object = self.class.find_by(slug: the_slug)
    count += 1
  end
  self.slug = the_slug
end
```

The next time around ``count`` will be one number greater which means ``the_slug``
will have a different number appended to it which means ``object`` will be
set to something different. This loop will continue until ``object`` is falsy which means ``the_slug`` is
finally a unique string.

Finally, the last line of the ``generate_slug`` method sets the slug column of ``self``
to the string ``the_slug`` was set to. (``self`` is the object ``generate_slug`` is being called on.
The example I've used throughout this post has been ``@category.generate_slug``)
When ``@category`` is saved in the create action, ``@category.slug`` will equal something like "programming-2".
Now wherever you look up a resource that has been slugified you have to change this format:

```ruby
@category = Category.find(params[:id])
```

to this:

```ruby
@category = Category.find_by(slug: params[:id])
```

because ``find`` will look for a primary key, but ``params[:id]`` returns a slug now
since the ``to_param`` method is overridden to return the slug instead of the primary key.

























