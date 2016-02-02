---
layout: post
title: "Using Procs to DRY up Ruby Code"
date: 2016-01-30 22:58:51 -0800
comments: true
categories: ruby
---
Ruby code is full of blocks. Blocks are essentially chunks of code that allow you to defer
the precise way a method will work to the time of the method invocation. For example,
if you encounter a scenario where you're calling a method from multiple places,
with one little tweak in each case, it may be a good idea implement the
method in a generic way by yielding to a block. By defining one method instead of many similar
methods your code stays DRY. The problem is Ruby only lets you
yield to one block per method call and that limits your ability to customize methods. In this post
I'll explain how to send procs to methods so you can customize your generic methods in
more than one way at a time.  Along the way I explain
how some of Ruby's syntactic sugar works, and I close with a real world example of how I implemented
all this in an app. <!--more-->

In the context of a method call, putting an ampersand in front of the
<a href="#LastArgument">*last argument</a>
tells Ruby to convert whatever is immediately to the right of the ampersand
to a proc if it isn't a proc already and then use that proc as the method’s block.
For example, these 5 examples all return the same thing:

1) The traditional way:
```ruby
[1].map { |n| n.to_s } # => ["1"]
```

2) Below the ampersand calls ``Symbol#to_proc`` on ``:to_s`` and then makes sure that proc is used as the method's block.
Click <a href="#SymbolToProc">here</a> for an explanation of how ``Symbol#to_proc`` works.
<a name="LastArgument"></a>*Technically speaking, Ruby does not consider the ampersand followed by a proc (or in this case a symbol being
converted to a proc) to be an argument for the method. If it did, then this would cause a wrong number of arguments
(1 for 0) error since ``Array#map`` doesn't take any arguments.

```ruby
[1].map(&:to_s) # => ["1"]
```

3) Since ``a_proc`` is already a proc, the ampersand doesn't need to call ``#to_proc`` on it. Instead it will just make
sure the proc is used as the method's block:

```ruby
a_proc = :to_s.to_proc
[1].map(&a_proc) # => ["1"]
```

4) You can create a proc and assign it to a local variable that describes what the proc does:

```ruby
call_to_s_on_the_object = Proc.new { |object| object.to_s }
[1].map(&call_to_s_on_the_object) # => ["1"]
```

5) Instead of ``Proc.new`` you can just write this:
```ruby
a_shorter_way_to_create_a_proc = proc { |object| object.to_s }
[1].map(&a_shorter_way_to_create_a_proc) # => ["1"]
```

Procs do more than just give us syntactic sugar.
<a href="#RealWorldExample">They helped me combine two methods into one.</a>
If you have two methods that are the same, except for a block inside the methods and both methods are
already yielding to a block like below, you can't yield to two different blocks, but you can
pass in a proc to handle the difference. Below are three methods. You can't add numbers to strings and
you can't add strings to numbers hence the need for the first two methods. However, the first two methods
can be replaced by the third.
<a href="https://gist.github.com/durrellchamorro/220045206c525bd72f78">Here is code</a> you
can run in your terminal to see how most of the examples in this post work firsthand.

```ruby
def my_first_method(array_of_strings)
  array_of_strings.each { |string| p string + ' whatever' }
  array_of_strings.each { |object| p yield object }
end

def my_second_method(array_of_numbers)
  array_of_numbers.each { |number| p number + 5 }
  array_of_numbers.each { |object| p yield object }
end

def you_only_need_one_method(array_of_anything, some_proc)
  array_of_anything.each(&some_proc)
  array_of_anything.each { |object| p yield object }
end
```

<a name="SymbolToProc"></a>
###The Symbol class defines ``#to_proc`` like so:

```ruby
class Symbol
  def to_proc
    Proc.new { |obj, *args| obj.send(self, *args) }
  end
end
```

Therefore, calling ``#to_proc`` on a symbol returns a proc. The ``*args`` means
the proc can take any number of arguments. In other words, just like blocks, procs don't enforce argument
count. The formal word for this is <a href="https://en.wikipedia.org/wiki/Arity">Arity</a>.
The proc created when ``#to_proc`` is called on ``:to_s`` looks like this:

```ruby
Proc.new { |obj, *args| obj.send(:to_s, *args) }
```

Even though this proc can take more than one argument, you wouldn't want to yield more than one object to the proc
because the proc would send that object to ``#to_s`` and ``#to_s`` doesn't take any arguments so you
would get an ``ArgumentError``. The examples above with ``#map`` and ``:to_s.to_proc`` work because
``1.send(:to_s)`` is the same as ``1.to_s``. Likewise, ``[1, 2].delete_at(0)`` is the same as
``[1, 2].send(:delete_at, 0)``.
With this in mind, consider the following cases where yielding more than one object will not produce an ``ArgumentError``:

```ruby
def foo!(array, index)
  yield array, index
end

my_array = [1,2]
foo!(my_array, 0) { |obj, index|  obj.delete_at(index) }
my_array # => [2]

my_array = [1, 2]
foo!(my_array, 0) { |obj, index| obj.send(:delete_at, index) }
my_array # => [2]

my_array = [1, 2]
delete_at_proc = Proc.new { |obj, index| obj.delete_at(index) }
foo!(my_array, 0, &delete_at_proc)
my_array # => [2]

my_array = [1, 2]
delete_at_proc2 = :delete_at.to_proc
foo!(my_array, 0, &delete_at_proc2)
my_array # => [2]

my_array = [1, 2]
foo!(my_array, 0, &:delete_at)
my_array # => [2]
```

### <a name="RealWorldExample"></a> A Real World Example of all this in Action
The ``#sort`` helper method below is written in a generic way so two things can be customized
at method invocation time.

####1. The collection can be any object that can be sent ``#partition`` and ``#index``.
####2. The block being yielded to can be customized.

``#sort`` uses ``#partition`` to split a collection into two arrays. By using parallel assignment, the first
array is assigned to the ``complete_collection`` local variable and the second array is assigned to the
``incomplete_collection`` local variable.
``#partition`` iterates through each element in the collection and decides
which array each element of the collection will go to depending
on whether ``#partition``'s block evaluates to true. Next, each item in the newly created arrays
is yielded with its original index to the block in a view. In order for ``#partition``'s block to
evaluate to true or false for different kinds collections its block needs to be compatible with
each kind of collection. The problem is there is no one size fits all block because the data structures of
the collections are quite different.
Instead of defining a sorting method for each kind of collection that would have a unique block for ``#partition``
I created a unique proc for each kind of collection to use as ``#partition``'s block with that collection.
In this way I wrote one method that sorts different kinds of collections and yields the sorted collection's
items to a custom block.

```ruby
def sort(collection, check_item_status)
  complete_collection, incomplete_collection = collection.partition(&check_item_status)

  incomplete_collection.each { |item| yield item, collection.index(item) }
  complete_collection.each { |item| yield item, collection.index(item) }
end
```

In this view ``#sort`` yields to a custom block of erb code that displays the contents of
one kind of collection (todo items) in an unordered list:

```erb
<ul>
  <% check_todo_status = proc { |todo| todo[:status] == 'complete' } %>
  <% sort(@list[:todos], check_todo_status) do |todo, index| %>
    <li class="<%= todo[:status] %>">
      <form action="/lists/<%= @list_id %>/todos/<%= index %>" method="post" class="check">
        <input type="hidden" name="status" value="<%= opposite_status_of(todo[:status]) %>">
        <button type="submit">Complete</button>
      </form>
      <h3><%= todo[:name] %></h3>
      <form action="/lists/<%= @list_id %>/todos/<%= index %>/destroy"
        method="post" class="delete">
        <button type="submit">Delete</button>
      </form>
    </li>
  <% end %>
</ul>
```

In the view below ``#sort`` yields to a custom block of erb code that displays the contents of a
different kind of collection (a list of todo lists) in an unordered list.

```erb
<ul id="lists">
  <% check_list_status = proc { |list| all_complete?(list) == 'complete' } %>
  <% sort(@lists, check_list_status) do |list, index| %>
    <li class="<%=all_complete?(list)%>">
      <h2><a href="lists/<%= index %>"><%= list[:name] %></a></h2>
      <%= display_progress(list) %>
    </li>
  <% end %>
</ul>
```
