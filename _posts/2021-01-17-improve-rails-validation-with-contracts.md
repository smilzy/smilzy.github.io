---
title: dry-rb - Improve validations and code execution flow with contracts and dry-validation
header:
  image: /assets/images/suitcasescan-TA-1200.jpg
  teaser: /assets/images/suitcasescan-TA-teaser.jpg
tags:
  - ruby
  - architecture
  - design patterns
  - dry-rb
  - dry-validation
  - dry-struct
  - form objects
  - contracts
classes: wide
---

**Reorganize your application's operation flow and validation logic**

## Let's continue on the topic of dry-rb

Ruby is a dynamically typed language, which means that when we define a variable we don't have to specify what type of value it holds *(for example does it allow storing only a String or is it supposed to store an Integer)*, and we can dynamically reassign a value of a different type to the same variable.

```ruby
var = "I'm a string" # well, it's a string.
var = 42 # Suddenly it's the Answer to the Ultimate Question of Life, the Universe, and Everything
```

The Ruby programming language uses dynamic typing to encourage responsible use of *duck typing* among developers - which is pretty convenient, but can be a source of many headaches when overlooked and not used properly. For example when the program expects to receive a form object with *count* parameter as an Integer but user sends a String instead your code will go **BOOM** and throw an Exception. In Ruby you could check every parameter's type by calling `param.is_a?(Integer)` and manually handling the exception, but really - ain't nobody got time for that.

## That's where dry-types and dry-validation come to the rescue

[**dry-validation**](https://dry-rb.org/gems/dry-validation/1.5/) is a gem from the *dry-rb* stable, which makes use of [**dry-types**](https://dry-rb.org/gems/dry-types/1.2/) in order to help you validate the correctness of parameters you want to handle. **It is especially useful in validating form data and not only does it check for parameter type, but it also allows for specifying custom validation rules and error messages**. The way it works is you create a *contract* which defines a *schema* - information about parameters, whether or not are they required, what type are they expected to be and do they fulfill the custom validations.

In my projects I fully migrated over from standard ActiveRecord validations to *dry-validation* contracts. It helps with **separation of concerns** - so our validations are kept outside of the model - and this solves problems when your application evolves, and validation logic is changed, but objects already existing in the database aren't. If you suddenly add `validates :birth_date, presence: true` to your model - all of the objects already in the database that don't have *birth_date* set will be invalid. Any other classes that are built on this logic of an object having an attribute will be broken, and you won't know this until it's too late.

This approach allows us for creating validations for specific scenarios, so we break free from rails validations chains and create our own logic.

```ruby
require 'dry-validation'

class ApplicationContract < Dry::Validation::Contract
  config.messages.default_locale = :pl # default I18n-compatible locale identifier
  config.messages.top_namespace = 'contracts' # the key in the locale files under which messages are defined, by default it's dry_validation

  # Check more config options at https://dry-rb.org/gems/dry-validation/1.5/configuration/
end

class NewPaymentContract < ApplicationContract
  params do
    required(:price).filled(:decimal)
    required(:payment_form).filled(:string)
    required(:email).filled(:string)

    optional(:phone).filled(:string)
  end

  rule(:price) do
    key.failure('must be greater than or equal 0') if value < 0
  end
end

contract = NewPaymentContract.new

contract.call(price: -5.0, payment_form: 'card', email: 'test@example.com').errors.to_h
# => {:price=>["must be greater than or equal 0"]}
```

### Make custom rules reusable

To improve it further, custom contract rules can be reused between your contracts. In order to *dry* it out a bit we can register a **macro**, put it in ApplicationContract and it will be accessible in all classes, which inherit from this base class.

```ruby
require 'dry-validation'

class ApplicationContract < Dry::Validation::Contract
  config.messages.default_locale = :pl
  config.messages.top_namespace = 'contracts'

  # For below macro we could import_predicates_as_macros which has :gteq? method built in
  register_macro(:greater_than_or_equal) do |macro:|
    number = macro.args[0]

    key.failure("must be greater than or equal #{number}") if value < number
  end
end

class NewPaymentContract < ApplicationContract
  params do
    required(:price).filled(:float)
    required(:payment_form).filled(:string)
    required(:email).filled(:string)

    optional(:phone).filled(:string)
  end

  rule(:price).validate(greater_than_or_equal: 0)
end

contract = NewPaymentContract.new

contract.call(price: -5.0, payment_form: 'card', email: 'test@example.com').errors.to_h
# => {:price=>["must be greater than or equal 0"]}
```

Contracts can be used to validate any data structure and **should** be used to validate data sent by form objects which will greatly improve your form building capabilities.

## The form object

[**dry-struct**](https://dry-rb.org/gems/dry-struct/1.0/) is the tool we'll use to build a form object. It uses **dry-types** to make sure parameters passed to the form are of correct type. Later on we pass that form object to the operation it was designed for and use contract object to validate the form again, but this time taking into account custom validation rules. If there are no errors we proceed with the operation - e.g. updating our object with form data.

```ruby
require 'dry-struct'

class ApplicationForm < Dry::Struct
end

class NewPaymentForm < ApplicationForm
  # Required attributes
  attribute :price, Types::Coercible::Float
  attribute :payment_form, Types::Coercible::String
  attribute :email, Types::Coercible::String

  # Optional attributes
  attribute? :phone, Types::Coercible::String
end

form = NewPaymentForm.new(price: 19.99, payment_form: 'card')
# => Dry::Struct::Error ([NewPaymentForm.new] :email is missing in Hash input)

form = NewPaymentForm.new(price: 'wrong_type', payment_form: 'card', email: 'test@example.com')
# => Dry::Struct::Error ([NewPaymentForm.new] "wrong_type" (String) has invalid type for :price violates constraints (invalid value for Float(): "wrong_type" failed))

form     = NewPaymentForm.new(price: 19.99, payment_form: 'card', email: 'test@example.com')
contract = NewPaymentContract.new

contract.call(form.to_h).errors.to_h
# => {}
```

What **dry-validation** is better at than standard Rails validations is handling complex attribute logic / nested attributes, which can be represented by hashes or arrays, and should be constructed according to a scheme. For example if in our form we expect an array of custom hash objects as the `data` attribute, we can write:

```ruby
UserResultSchema = Types::Hash.schema(
  name: Types::Coercible::String,
  result: Types::Coercible::Float
)

class ResultsForm < ApplicationForm
  attribute :data, Types::Array.of(
    UserResultSchema
  )
end

form = ResultsForm.new(
  data: [
    {
      name: 'John Doe',
      result: 99.0
    },
    {
      name: 'Jane Doe',
      result: nil
    }
  ]
)
# => Dry::Struct::Error ([ResultsForm.new] [{:name=>"John Doe", :result=>99.0}, {:name=>"Jane Doe", :result=>nil}] (Array) has invalid type for :data violates constraints (nil (NilClass) has invalid type for :result violates constraints (can't convert nil into Float failed) failed))
```

The above *UserResultSchema* is a recipe for how our UserResult object should look like, and - as you can see - **this let's us create custom schemas, which can be further reused in our application**.

## Wrapping it up together in an operation

In order to fully reap the benefits of forms and contracts written that way we have to use them in a place where they belong - a service or operation as I'll call it - which takes in concrete data and performs an action only if the data is valid.

> Below class uses the **callee** gem, which I wrote about before in [dry-rb - Improve Rails development with dry-initializer and callee]({% post_url 2021-01-06-improve-rails-development-with-dry-rb-and-callee %}) article


```ruby
class CreatePaymentOperation
  include Callee

  option :form
  option :contract, default: -> { NewPaymentContract.new }

  def call
    return errors if errors.any?

    create_payment
  end

  private

  def errors
    @errors ||= contract.call(form.to_h).errors
  end

  def create_payment
    # Some logic here
  end
end
```

And voilà - it's clean and easy! Most of my operations follow the same scheme: they take form data and a contract and first thing they do is check for form data integrity. Only if the data is valid the operation proceeds to do what it was designed for. To further refine our application's logic flow I use **monads** to help me ensure each step of the operation is completed successfully, and stop the operation and return proper errors if any of the steps fail. Monads may seem a little bit **exotic** at first for Ruby developers, but - as I'll explain in another article - are very easy to pick up on and use :)

Bye until next time!
