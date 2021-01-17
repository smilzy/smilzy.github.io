---
title: dry-rb - Improve Rails development with dry-initializer and callee
header:
  image: /assets/images/alan-j-hendry-su3FE5qTEjw-unsplash-1200.jpg
  caption: "Photo credit: [**Alan J. Hendry**](https://unsplash.com/@imedianamibia){:target='_blank'}"
  teaser: /assets/images/alan-j-hendry-su3FE5qTEjw-unsplash-teaser.jpg
tags:
  - ruby
  - architecture
  - design patterns
  - dry-rb
  - functional programming
classes: wide
---

**Improve your project's quality and readability with dry-rb and callee gem**

## Are you happy with how your Rails code reads?

Of course you are.

*- My code is perfect!*

You say while brushing your old, messy code under the rug :)

Let's face it - there's always room for improvement, especially when the time comes to look at the code we put under the rug earlier. To make this process less unpleasant we could invest some time in better preparation, applying clean code principles and looking for code design improvements on the broad ocean of ideas which is The Internet. On one of these e-boat trips I stumbled upon [**dry-rb**](https://dry-rb.org/){:target="_blank"} - a set of libraries designed to help you neatly organize your Ruby code into easily maintainable components.

This is what they describe the toolset on their webpage:
> *dry-rb helps you write clear, flexible, and more maintainable Ruby code. Each dry-rb gem fulfils a common task, and together they make a powerful platform for any kind of Ruby application.*

It wasn't until much later when we finally introduced *dry-rb* to our *DeliGoo* project at *ReasonApps*, and looking at it now - I regret we hadn't done that earlier. **It helped us improve readability, introduced tools that helped our team standardize approach to handling problems and writing code.** For example the process of creating a new class for saving an object to database in our project would always follow the same steps:

* create a new class in *operations* folder with *_operation.rb* suffix,
* its' public interface should consist of only one method - **call** - which would accept *keyword arguments* defined on top with *option* class method (*dry-initializer* and *callee*),
* require a form object (*dry-struct*, *dry-types*) and a contract for validating the form (*dry-validation*),
* use functional programming approach to make sure each step is completed successfully and returns **Success()** object - or break code execution on failure and return **Failure()** object (*dry-monads*).

```ruby
require 'dry/monads'

module Addresses
  class CreateOperation
    include Callee
    include Dry::Monads[:result, :do]

    option :form_data
    option :contract, default: -> { Addresses::CreateContract.new }

    def call
      yield check_errors

      address = yield create_address

      Success(address)
    end

    private

    def check_errors
      return Failure(errors: errors) if errors.any?

      Success(nil)
    end

    def errors
      @errors ||= contract.call(form_data).errors.to_h
    end

    def create_address
      address = AddressFactory.call(params: form_data)

      if address.save
        Success(address)
      else
        Failure(address: address, errors: Errors::ValidationValue.new(key: :address).errors)
      end
    end
  end
end
```

Of course we didn't rush and throw everything they have at our codebase - here's how we did it in small iterations.

## The Initializer - Callee

To start things off mildly we focused on the beginning of the lifecycle - initialization of an object.

> *[dry-initializer](https://dry-rb.org/gems/dry-initializer/3.0/){:target="_blank"} is a simple mixin of class methods params and options for instances.*

Extending Dry::Initializer in our Ruby class lets us define standard arguments or keyword arguments for the `initialize` method as simple one liners on top of the file.

```ruby
# We can go from this:
class Thing
  def initialize(keyword_argument:, standard_param)
    @keyword_argument = keyword_argument
    @standard_param = standard_param
  end
end

# To this:
class Thing
  extend Dry::Initializer

  option :keyword_argument
  param :standard_param
end

# And the application is the same:
Thing.new(keyword_argument: 'test', 'simple string')
```

It helped us separate the initialization part of an object, keep the looks of it cleaner, and allow for additional behaviour such as *type constraints* and more.

{:refdef: style="text-align: center;"}
![An old phone](/assets/images/quino-al-xhGMQ_nYWqU-unsplash-900.jpg)
{: refdef}
*Time to make a call*
{: style="color:gray; font-size: 80%; text-align: center;"}

Another important step here was deciding to standardize the public interface for our services and operations. We use [*callee*](https://github.com/dreikanter/callee){:target="_blank"} gem which is built on top of *dry-initializer* in order to make our classes initialize via the *call* method (similar behaviour to my *DelegateCallToNew* module from Query Objects post). This lets us use them as procs and keeps things cleaner - **all operation classes and most services are callable, and are coded in similar manner.**

```ruby
class PerformanceCalculator
  include Callee

  option :employee
  option :date, default: -> { Date.today }

  def call
    (work_done / target) * 100
  end

  private

  def work_done
    employee.work_done.for_date(date)
  end

  def target
    ...
  end
end

# Used like:
performance = PerformanceCalculator.call(employee: user, date: Date.yesterday)
```

When our team grew and every person was used to writing code in a little different manner we have decided to enforce some rules. We focused on separating responsibilities of classes and reorganizing our code into better structures which belong in adequate folders. **The way that *callee* made us construct classes appealed to us instantly and was very easy to adjust to.** Ability to specify types of parameters passed to classes found use in some parts of our application - and possibly skipped us some headaches. If you need help figuring out how to make more advanced logic work - [*dry-rb* keeps their documentation well maintained](https://dry-rb.org/gems/dry-initializer/master/){:target="_blank"}.

In Gemfile:
```ruby
# dry-rb
gem 'dry-initializer'
# or
gem 'callee' # it includes dry-initializer as a dependency
```
*And that's all you need to start*
{: style="color:gray; font-size: 80%; text-align: center;"}

As of today - **I can't really imagine not using *callee* in my project** - it's so convenient that I have to recommend using it to everyone :)
