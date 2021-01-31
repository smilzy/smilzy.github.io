---
title: dry-rb - Improve Rails operation flow with monads
header:
  image: /assets/images/benjamin-voros-hfueW9tLWtc-unsplash-1200.jpg
  teaser: /assets/images/benjamin-voros-hfueW9tLWtc-unsplash-teaser.jpg
  caption: "Photo credit: [**Benjamin Voros**](https://unsplash.com/@vorosbenisop)"
tags:
  - ruby
  - architecture
  - design patterns
  - dry-rb
  - dry-monads
  - monads
  - operations
classes: wide
---

**Reorganize your application's code flow and error handling**

## Let's finish this month's dry-rb rally with a little thing to wrap it all up ;)

Monads. What are they, and how to use them in Ruby? Not many people understand this concept, and it's not really that strange - utilised by functional programming languages such as Haskell - monads are not commonly used in OO languages and frameworks like Ruby on Rails. Why would you want to use monads? A quote from *wikipedia* will shine a little light on the answer:

> *With a monad, a programmer can turn a complicated sequence of functions into a succinct pipeline that abstracts away auxiliary data management, control flow, or side-effects.*

What does that mean? It means that **monads help keep code execution fluent and help massively with exceptions, error handling, while using an easy to follow standard**. I'm not going to focus on theory - it's easier to explain using code samples than plain text, so let's get to work!

## Result monad

In order to execute a chain of functions, where execution of one function may depend on another function's successful response the **Result** monad comes in handy. **Instead of chaining multiple if statements, using .try or &. operator we define adequate methods which contain the logic and result in a *Success* or *Failure* of given code**.

```ruby
require 'dry/monads'

class CreateOrderOperation
  include Callee
  include Dry::Monads[:result, :do]

  option :form
  option :contract, default: -> { NewOrderContract.new }

  def call
    yield check_errors

    user = yield find_user
    address = yield extract_address(user: user)
    distance = yield calculate_distance(address: address)
    order = yield create_order(distance: distance)

    Success(order)
  end

  private

  def check_errors
    if errors.any?
      Failure(errors)
    else
      Success()
    end
  end

  def errors
    @errors ||= contract.call(form.to_h).errors
  end

  def find_user
    user = User.find_by(id: form.to_h[:user_id])

    if user
      Success(user)
    else
      Failure(:user_not_found)
    end
  end

  def extract_address(user:)
    if user.address.present?
      Success(user.address)
    else
      Failure(:address_not_found)
    end
  end

  def calculate_distance(address:)
    CalculateDistanceService.call(address: address)
    # Returns Success(distance) or Failure(:distance_calculation_error)
  end

  def create_order(distance:)
    # Some example logic here
    order = Order.new(form.to_h)
    order.distance = distance

    if order.save
      Success(order)
    else
      Failure(:order_save_error)
    end
  end
end
```

Okay, we've got a few things going here, but the code should feel easy to understand. By incorporating [*dry-monads*](https://dry-rb.org/gems/dry-monads/1.3/){:target="_blank"} to our codebase we can take advantage of monads workflow. By following an example class structure from [previous dry-rb post]({% post_url 2021-01-17-improve-rails-validation-with-contracts %}) and using the [*do notation*](https://dry-rb.org/gems/dry-monads/1.3/do-notation/){:target="_blank"} combined with [*result*](https://dry-rb.org/gems/dry-monads/1.3/result/){:target="_blank"} monad we receive a clean example of monads in work.

First step is to `require 'dry/monads'` in our file, then `include Dry::Monads[:result, :do]` inside of our class. The part in square brackets lets us take only the behaviour we want to use - **:result** which stands for Result monad (`Success()`, `Failure()`), and **:do** which stands for *do notation* (`yield`).

The way our code works now is:
1. Whenever we use `yield` and pass it a function, which returns a `Dry::Monads::Result::Success` object **it unwraps the value from the Result object**,
2. If the returned value is a `Dry::Monads::Result::Failure` object - further code execution is stopped, and we return from the main execution flow with received `Failure()` object,
3. We make sure to return a `Dry::Monads::Result` object as the ending value of our class, so it may be used in a *monad-like way* in other parts of our application.

Handling Success or Failure is made fairly easy for example with a *case* statement. **Our Failure() objects can be customized to contain error codes, error messages and much more, which helps greatly with error handling and exception handling**. Example use of this approach in a codebase might look like this:

```ruby
require 'dry/monads'

class OrdersController
  include Dry::Monads[:result]

  # ...

  def create
    result = CreateOrderOperation.call(
      form: OrderForm.new(params: order_params),
      contract: NewOrderContract.new
    )

    case result
    when Success
      flash[:notice] = 'Order created successfully'
      redirect_to orders_path
    when Failure
      flash.now[:error] = "Operation failed with code: #{result.failure}"
      render :new
    end
  end

  # ...
end
```

As seen above monads help us keep every single piece of logic separated. **This allows for easier testing, and helps maintain the Single Responsibility Principle**. Most common monad in our system is the *Result* monad, but there are other components of **dry-monads** which you may find useful.


## Maybe monad

This type of monad is useful when one of the values in our code flow might be a `nil`. It either returns a `Dry::Monads::Maybe::Some` object, or a `Dry::Monads::Maybe::None` object. Both of these can be transformed into a *Result* monad via the `.to_result` method.

```ruby
require 'dry/monads'

class CalculationsService
  include Callee
  include Dry::Monads[:maybe, :do, :result]

  option :params

  def call
    maybe_user = yield find_user
    maybe_address = yield extract_address(user: maybe_user)
    maybe_distance = yield calculate_distance(address: maybe_address)

    Some(maybe_distance)
    # Returns Some(distance) or None()
  end

  private

  def find_user
    Maybe(User.find_by(id: params[:user_id]))
  end

  def extract_address(user:)
    Maybe(user.address)
  end

  def calculate_distance(address:)
    CalculateDistanceService.call(address: address)
  end
end
```

This way we've received similar behaviour to previous code, which used the *Result* monad, but we've lost the ability to personalize exceptions - we just get the *None()* object.

Handling the `Dry::Monads::Maybe::Some` and `Dry::Monads::Maybe::None` objects can be done the same way as before - with `case` statement, or using the `or` operator.

```ruby
result = CalculationsService.call(params: params)
result.value_or(0.0) # Returns maybe_distance or default value (0.0) if None()
```

The *Maybe* monad might be more useful in chaining functions when making calculations, or digging through data, where something along the way may be empty.

> *This is similar to how the &. operator works in Ruby but does wrapping* - [*the docs*](https://dry-rb.org/gems/dry-monads/1.3/maybe/#code-maybe-code){:target="_blank"}

```ruby
Maybe(User.find_by(id: params[:user_id])).maybe(&:address).maybe(&:house_number).or(Some('Invalid address'))
# Returns Some(house_number) or Some('Invalid address')
```

For full documentation and more use cases please refer to [*dry-monads documentation*](https://dry-rb.org/gems/dry-monads/1.3/){:target="_blank"}, and please keep in mind what is written there:

>**Just don't wrap everything with Maybe, come up with conventions.**

Thank you.
