---
layout: post
title: "Rails Magic With Partials"
date: 2015-07-03 06:33:02 -0700
comments: true
categories: rails
---
When you want to send a collection of objects to a partial you can use Rails
conventions to eliminate code if you know what rails automatically assumes.
<!--more-->Normally, for a partial containing the code in Example 2 you would
write something like this in the view:

Example 1
```ruby
<%= render "shared/title", title: "All Posts" %>
```
where ```”shared/title"``` is the relative path to the partial and ```title:```
is the name of the local variable in the partial, and ```”All Posts”``` is the
string that local variable will be set to. It is important to remember the actual
partial needs to have an underscore in front of it so the actual file in this
example is ```_title.html.erb``` but it is just referenced as ```title``` as
shown above.

Example 2
```ruby
  <h3><%= title %></h3>
```
Here is where you can take advantage of Rails conventions and assumptions. Let’s
say you have an object ```@posts``` that is set in your Controller to your Post
model like this: ```@posts = Post.all``` so ```@posts``` contains many post
objects, and each post has its own categories. Instead of writing all this:

Example 3
```ruby
<% @posts.each do |post| %>
  <% post.categories.each do |category| %>
    <%= render '/categories/category', category: category %>
  <% end %>
<% end %>
```

You can simply write:

Example 4
```ruby
<% @posts.each do  |post| %>
  <%= render post.categories %>
<% end %>
```
The partial for Example 4 might look something like this:

Example 5
```ruby
<%= link_to category.name, post_path(category) %>
```
In Example 4 the resources we are rendering are categories. Rails will assume
there is a folder named after those resources that contains a partial named
after the singular of those resources. Given that knowledge, all we have to do
is create a ```categories``` folder relative to the view and put a
```category.html.erb``` file in it that contains the code in Example 5.

