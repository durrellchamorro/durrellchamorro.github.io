---
layout: post
title: "Self Referential Associations in Rails"
date: 2015-10-05 12:25:45 -0700
comments: true
categories: rails
---
In this post I'll compare and contrast different database associations in rails with the goal of
explaining how to set up self referential associations in rails. My conclusion is the conventions
rails provides don't always make code easier to read.

Rails makes it easy to associate different models with one another. Take these
two models as a straight forward example:

``` ruby
class God < ActiveRecord::Base
  belongs_to :human
end
```

```ruby
class Human < ActiveRecord::Base
  self.table_name =“humans”
  has_many :gods
end
```

In this model ```self.table_name = “humans” ``` tells rails to look for a table
named ```humans``` instead of ```humen``` which it will automatically look for
with a model named ```Human```. This can be demonstrated by typing the following
command into the terminal:

```
“human”.tableize => “humen”
```



So long as you have a foreign key called ```human_id``` in the gods table, rails
will figure out the association and let you call things like
```
zeus.human => #<Human id: 1, name: “bob">
``` and

```
bob.gods => #<ActiveRecord::Associations::CollectionProxy [#<God id: 1, name: "Zeus", human_id: 1>, #<God id: 2, name: "Athena", human_id: 1>]>
```

The ```has_many :gods``` line in the Human model is just a rails convention for the following method:

```
  def gods
    God.where(human_id: self.id)
  end
```

As you can see from the above method, rails assumes you are querying the gods table and it assumes you are looking for the foreign key in the gods table that equals the primary key of the object the ```gods``` method is being called on.

Likewise, the ```belongs_to :human ``` line in the God model is a rails convention for the following method:

```
  def human
    Human.find(self.human_id)
  end
```

As you can see from the method above, rails assumes you are querying the humans table for the primary key that equals the foreign key (in this case the foreign key is human_id) of the object (in the example above the object was zeus) the ```human``` method is being called on. It automatically looks for a foreign key called ```human_id``` by convention as well since the relationship is ```belongs to :human```

The rails conventions work beautifully for the above situation, but if you wanted to show a god has many human followers and a human follows many different gods? The ```belongs_to :human``` line above doesn’t allow you to call ```zeus.humans``` to get a list of humans. It only allows  you to call ```zeus.human``` to get one human. To fix this we need to set up a many to many relationship.

First we’ll create a new table called ```relationships```. The model that hooks up to that table will look like this:

```
class Relationship < ActiveRecord::Base
  belongs_to :god
  belongs_to :human
end
```

The relationships table will have foreign keys called ```god_id``` and ```human_id```. The God model will need to be changed to this:

``` ruby
class God < ActiveRecord::Base
  has_many :relationships
  has_many :humans, through: :relationships
end
```

If the relationships table has a row with god_id equal to 1 and human_id equal to 1 and another row with god_id equal
to 1 and human_id equal to 2 and we do this:
```
zeus = God.first
```

then we get this:

 ```zeus.humans => #<ActiveRecord::Associations::CollectionProxy [#<Human id: 2, name: "marcus">, #<Human id: 1, name: "maximus">]> ```

the rails convention ```has_many :humans, through: :relationships``` is the same as writing this method. The difference
is the convention returns an ActiveRecord_Associations_CollectionProxy and the
method below returns an array. This is a minor difference because you just need
to call ```to_a``` on it to turn it into an array.

``` ruby
def humans
  Human.find(Relationship.where(god_id: self.id).pluck(:human_id))
end
```

As you can see from the method, this convention assumes you are querying the
relationships table for all entries where the god_id foreign key is equal to the
primary key of the object the ```humans``` method is being called on.  It then
assumes you want a list of all the humans
whose primary key equals the human_id foreign key in the list of relationships
it got from the previous query.


If the relationships table has a row with god_id equal to 1 and human_id equal
to 1 and another row with god_id equal to 2 and human_id equal to 1, and we do this:

```
max = Human.first
```
then we get this:

```
max.gods  => #<ActiveRecord::Associations::CollectionProxy [#<God id: 1, name: "Zeus", human_id: 1>, #<God id: 2, name: "Athena", human_id: 1>]>
```

The rails convention

``` ruby
has_many: gods, through: :relationships
```

is the same as writing the method below. The difference is the convention returns
an ActiveRecord_Associations_CollectionProxy and the method below returns an
array. This is a minor difference because you just need to call ```to_a``` on
it to turn it into an array.

```
 def gods
    God.find(Relationship.where(human_id: self.id).pluck(:god_id))
  end
```

As you can see from the method, this convention assumes you are querying the relationships table for all entries where the human_id foreign key is equal to the primary key of the object the ```gods``` method is being called on.  It then assumes you want a list of all the gods
whose primary key equals the god_id foreign key in the list of relationships it
got from the previous query.

What if you want a list of all the children and all the parents of a particular god? This sort of request is a self referential association. Instead of asking for associated data in another table as in humans associated with gods, you are asking for associated data within the same table as in gods associated with gods. This means the conventions above wont work because the assumptions explained above that rails makes will be wrong. Therefore, you have to explicitly tell rails where to look.

The Relationship model will need two additional lines as follows:

``` ruby
class Relationship < ActiveRecord::Base
  belongs_to :god
  belongs_to :human
  belongs_to :parent, class_name: "God"
  belongs_to :child, class_name: "God"
end
```

Here we are naming the relationship as ```parent``` and ```child``` but we don’t
want rails to look for the ```Parent``` class/model and the ```Child``` class/model because
those classes/models don’t exist. Therefore we have to explicitly tell rails to look
for the God class/model.

The God class will need these two additional lines of code:

```
has_many :children, class_name: "Relationship", foreign_key: :parent_id
has_many :parents, class_name: "Relationship", foreign_key: :child_id
```

We need to explicitly tell rails the class name is Relationship otherwise it will
look for the Child and Parent classes. Then we need to tell rails the foreign
key names we want. In this case we are looking for the parent_id foreign keys
in the ```relationships``` table and we want to access that data with by calling a method named ```children```. We are also looking for the child_id foreign keys in the relationships table and we want to access that data by calling a method named ```parents```


There is still a little convention at play. We are naming the methods
``children`` and ``parents`` since the Relationship model has ``belongs_to :parent, class_name: "God"``
and ``belongs_to :child, class_name: “God”`` which tell rails to associate the two classes.


This line

```
has_many :children, class_name: "Relationship", foreign_key: :parent_id
```

is essentially the same as writing this method with the exception that the method
below returns a Relationship::ActiveRecord_Relation and the convention returns
an ActiveRecord_Associations_CollectionProxy, both of which can be turned
into an array by calling ``to_a`` on them.

``` ruby
def children
  Relationship.where(parent_id: self.id)
end
```

With all this being said, I think this example shows the conventions in rails don't
always make the code easier to read. To me, it's easier to understand what is going on
in the method above than in the convention below:

```
has_many :children, class_name: "Relationship", foreign_key: :parent_id
```
















