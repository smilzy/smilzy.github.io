---
title: Query Objects - Organize your ActiveRecord scopes
header:
  image: /assets/images/shane-hauser-18dsFU_VpZI-unsplash-x1200.jpg
tags:
  - ruby
  - database
  - SQL
  - queries
  - design patterns
  - architecture
classes: wide
---

**Upgrade your query interface**

### What is a Query Object

*Query Object* is an instance of a class which sole purpose is constructing a
database call. Rails ecosystem and ActiveRecord got everyone used to the simplicity
of making database queries by using our models directly. That's why controllers
are cluttered with lines such as:
```ruby
@orders = Order.where('created_at >= ? AND status = ?', Date.today, :completed)
```
If you're any more experienced then
these clauses would be extracted into scopes and put in the model file:

```ruby
class Order < ApplicationRecord
  scope :today,     -> { where('created_at >= ?', Date.today) }
  scope :completed, -> { where(status: :completed) }
end
```

And that's sufficient. If the model and its' uses are simple and so are the database
queries, then writing scopes in the model file is OK. This, however, often changes
when the application grows and the data is used to create detailed reports with
advanced filters, or lots of scopes that have small differences between them are
used in different areas all over the project. Then your model file might end up
looking like this:

![Scope mess in a model](/assets/images/rails-scopes-example.jpg)

And it's only a part of it. It's hard to follow and understand well, and it's
even harder to modify it to fit new conditions and requirements. So instead we
keep making more of those scopes, similar to each other, yet so different. This
code violates the *open-closed principle* of Clean Code, so the code is open for
extension but is not closed for modification.

### Let's start improving this

For starters, let's create a *queries* folder in our *app* directory. Here's
where Query Objects will have their home.

Next, let's create a class in */app/queries/orders_query.rb* which will be used
to construct database queries for the model and call them.

```ruby
class OrdersQuery
  attr_reader :scope

  def initialize(scope: Order.all)
    @scope = scope
  end

  def today
    scope.where('created_at >= ?', Date.today)
  end
end
```

Class constructor is injected with the scope of the model we will be query'ing.
It allows for more flexibility and connecting multiple query objects together
which will be shown further down the post. The way this query object should be
used now is like this:

```ruby
class OrdersController < BaseController
  def index
    @orders = OrdersQuery.new.today
  end
end
```

Which is very similar to a standard `Order.today` scope, but the database call
layer is decoupled from the model file. And we're not finished yet - because there's
still room for improvement.

Let's say our Report class, which displays information about completed or failed
orders from each day, needs to perform a call to database in order to grab
associated orders.
In such case let's create a new file in */app/queries/orders_query/report_orders.rb*.

```ruby
class OrdersQuery::ReportOrders < OrdersQuery
  def call(date: Date.today)
    scope
      .where('created_at >= ? AND created_at <= ?', date.beginning_of_day, date.end_of_day)
      .where(status: [:completed, :failed])
  end
end
```

*We're naming our function **call** which is no accident and it will be relevant
later on.*

From now on instead of writing this piece of code in the controller (or a class
method) we can use `OrdersQuery::ReportOrders.new.call` to access associated objects
and this query can be referenced from multiple places of the application. The code
is clean, easy to read and understand, allows for easy modification and is stored
in only one place - with other query objects.

### Extract parts that repeat quite often

Right now we can see that each query object method has a life of its own and stores
all the necessary data inside itself. But basic queries that are used by multiple
scopes repeat themselves in at least a few places. That's why we extract them to
own instances and put in the base orders query file.

```ruby
class OrdersQuery
  attr_reader :scope

  def initialize(scope: Order.all)
    @scope = scope
  end

  class CompletedOrFailed < OrdersQuery
    def call
      scope
        .where(orders: { status: [:completed, :failed] })
    end
  end

  class ForDay < OrdersQuery
    def call(date:)
      scope
        .where("created_at >= ? AND created_at <= ?", date.beginning_of_day, date.end_of_day)
    end
  end
end
```

And we can use them in other query objects like this:

```ruby
class OrdersQuery::ReportOrders < OrdersQuery
  def call(date: Date.today)
    @scope = ForDay.new(scope: scope).call(date: Date.today)
    @scope = CompletedOrFailed.new(scope: scope).call
  end
end
```

This way we make sure the behaviour stays the same between queries that use
these conditions, and we are keeping the code DRY.

Another improvement would be getting rid of calling *new* each time we want to
access the Query Object and delegate *call* to a *new* instance of the class.
We can do it that way:

```ruby
class CompletedOrFailed < OrdersQuery
  class << self
    delegate :call, to: :new
  end

  def call
    scope
      .where(orders: { status: [:completed, :failed] })
  end
end
```
This results in calling queries like this: `OrdersQuery::CompletedOrFailed.call`,
so unless we're planning on passing a custom scope to the constructor there's no
need to use *new*.

### Next step - utilizing model scopes

Right now model scopes are used all over the application and would require switching
to Query Object calls, but there's a way to deal with that. We can still use them.
In order to make the scopes defined in model files still usuable we can delegate
them to the corresponding Query Objects. In order to do se we can dig into how
*scopes* work in Rails and we'll learn about a *scoping* method. The code below
is taken from Rails library.

*Rails 5.1.7*
```ruby
def scope(name, body, &block)
  unless body.respond_to?(:call)
    raise ArgumentError, 'The scope body needs to be callable.'
  end

  # ...
  extension = Module.new(&block) if block

  if body.respond_to?(:to_proc)
    singleton_class.send(:define_method, name) do |*args|
      scope = all.scoping { instance_exec(*args, &body) }
      scope = scope.extending(extension) if extension

      scope || all
    end
  else
    singleton_class.send(:define_method, name) do |*args|
      scope = all.scoping { body.call(*args) }
      scope = scope.extending(extension) if extension

      scope || all
    end
  end
end
```

*Rails 5.2.3 and newer*
```ruby
def scope(name, body, &block)
  unless body.respond_to?(:call)
    raise ArgumentError, 'The scope body needs to be callable.'
  end

  # ...
  extension = Module.new(&block) if block

  if body.respond_to?(:to_proc)
    singleton_class.define_method(name) do |*args|
      scope = all._exec_scope(name, *args, &body)
      scope = scope.extending(extension) if extension
      scope
    end
  else
    singleton_class.define_method(name) do |*args|
      scope = body.call(*args) || all
      scope = scope.extending(extension) if extension
      scope
    end
  end
end
```

In newer rails versions the *scoping* method is still used if body is a *proc* and
it's hidden in the *_exec_scope* method.

The scope method takes a name of the scope and a *proc* or an object that responds
to the method *call* as the *body* param. That's why we name our Query Object
methods that way. While *scoping* method calls the *body* query on top of an
already existing scope, making a connection. This knowledge allows us to write
scopes in Rails models like this:

```ruby
scope :completed_or_failed, OrdersQuery::CompletedOrFailed
```

I learned about this approach here: [*Delegating to Query Objects through ActiveRecord scopes*](http://craftingruby.com/posts/2015/06/29/query-objects-through-scopes.html)

### Summary

Using Query Objects in your Rails applications is full of benefits like:
* standardizes database query logic,
* removes query logic from model files,
* keeps query logic in one place,
* makes complicated queries easy to construct, read and change,
* are really flexible and modular,
* are compatible with standard Rails model scopes,
* clean solution open for extension, closed for modification.

If your application has grown a lot lately, or is database query heavy then
Query Objects are your best friends.
