---
title: Form Objects in Ruby - Multi-Step Wizard
header:
  image: /assets/images/nazli-mozaffari-df2Vb3Iwe_4-unsplash-x1200.jpg
  caption: "Photo credit: [**Nazli Mozaffari**](https://unsplash.com/@mzfn)"
tags:
  - ruby
  - form objects
  - forms
  - design patterns
classes: wide
---

**Rethink your approach towards forms**
<!-- ### Scratching the surface -->
### Standard Rails Form

Building basic forms in Ruby is a straightforward task:
  * Create a view with a form
  * Specify permitted parameters in the controller
  * Handle success and failure in controller's form submit action

And that's it! Up until you need to make some customizations like dynamically setting some attribute depending on other values or multi-step wizards with nested resource creation and in-between step validations. Writing all that in your controller will mess it up and it also breaks the Single Responsibility Principle.

### What to do then?

A simple solution to this problem would be creating a new class which would handle these custom actions. If it's a small change maybe a helper method or a Service Object would be sufficient, but we want to future proof it. That's where the Form Object design pattern comes into play.

```ruby
class OrderCreateForm
  include ActiveModel::Model

  validates :kind, presence: true
  validates :price, numericality: { greater_than_or_equal_to: 0 }

  def initialize(attrs = {})
    attr_accessors
    super(attrs)
  end

  def save
    return false unless valid?
    order.save
  end

  def order
    @order ||= Order.new(params_cleaned_up)
  end

  private

  def permitted_attributes
    [ :kind,
      :price,
      :description ]
  end

  def attributes_hash
    {}.tap { |hash| permitted_attributes.each { |attr| hash[attr] = send(attr) } }
  end

  def attr_accessors
    permitted_attributes.each { |attr| class_eval { attr_accessor attr } }
  end

  def params_cleaned_up
    params = attributes_hash
    params[:points] = (kind == :premium ? price * 2 : price)
    params
  end
end
```
This file can be placed in *app/forms/order_create_form.rb*

The code above is associated with initializing, validating and creating a new Order object. The code requires us to replace `@order` instance variable in new and create actions in the controller.
```ruby
class OrdersController < ApplicationController
  def new
    @order = OrderCreateForm.new
  end

  def create
    @order = OrderCreateForm.new(order_params)
    if @order.save
      redirect_to orders_path
    else
      render :new
    end
  end

  private

  def order_params
    params.permit(:order).require(:kind, :price, :description)
  end
end
```

The OrderCreateForm class gains the ability to validate using the validates option because of including ActiveModel::Model module. When initializing a new form object the class automatically creates attr_accessors for fields described in permitted_attributes array, allowing access to `@order.kind; @order.value; @order.description` methods. The save method makes the form object similar to a standard Rails form object - it validates the attributes as specified and then sends the request to save final object to the model (where it is validated again).


### Multi-Step Wizard and nested attributes

This approach is useful when the process of creating an object is complicated and consists of multiple independent steps, so the form could be split into parts. Fortunately expanding from previous solution is very easy. As an addition it includes a way of handling nested attributes for creating associated objects.

```ruby
class OrderCreateForm
  include ActiveModel::Model

  def initialize(attrs = {})
    attr_accessors
    super(attrs)
  end

  def save
    return false unless valid?
    next_step!
  end

  def next_step
    @step = @step.to_i + 1
  end

  def order
    @order ||= Order.new(params_cleaned_up)
  end

  class Step1 < OrderCreateForm
    validates :kind, presence: true
  end

  class Step2 < Step1
    validate  :item_attributes_validator

    private

    def item_attributes_validator
      errors.add(:item_attributes, "can't be blank.") if item_attributes_condition
    end

    def item_attributes_condition
      item_attributes.any? && ! [:count, :price].all? { |attr| item_attributes[attr].present? }
    end
  end

  class Step3 < Step2
    validates :price, numericality: { greater_than_or_equal_to: 0 }

    def save
      return false unless valid?
      order.save
    end

    def last_step?
      true
    end
  end

  def last_step?
    false
  end

  private

  def permitted_attributes
    [ :kind,
      :price,
      :description,
      :item_attributes
      :step ]
  end

  def attributes_hash
    {}.tap { |hash| permitted_attributes.each { |attr| hash[attr] = send(attr) } }
  end

  def attr_accessors
    permitted_attributes.each { |attr| class_eval { attr_accessor attr } }
  end

  def params_cleaned_up
    params = attributes_hash
    params[:points] = (kind == :premium ? price * 2 : price)
    params.delete(:step)
    params
  end
end
```

In the code above step subclasses are introduced inheriting from the original class or previous steps to preserve validations. Other additions are allowing our form object to take item_attributes param and custom validation on Step2.

```ruby
= simple_nested_form_for @order, as: :order, url: orders_path do |f|
  = render 'orders/new/step1', f: f
  = render 'orders/new/step2', f: f if @order.step > 1
  = render 'orders/new/step3', f: f if @order.step > 2
  = f.input_field :step, value: f.object.step || 1, as: :hidden
```

While code in step partials looks similar:

```ruby
# step 1
= f.input :kind
= f.input :description
# step 2
= f.fields_for :items do |fi|
  = fi.input :amount
  = fi.input :price
# step 3
= f.input :price
```

And the controller code changed as well:

```ruby
class OrdersController < ApplicationController
  def new
    @order = OrderCreateForm.new
  end

  def create
    @order = "OrderCreateForm::Step#{order_params[:step]}".constantize.new(order_params)
    if @order.save
      if @order.last_step?
        redirect_to orders_path, notice: 'Order created'
      else
        render :new
      end
    else
      render :new
    end
  end

  private

  def order_params
    params.require(:order).permit(:kind, :price, :description, :step, items_attributes: [:count, :price])
  end
end
```

The code responsible for successful / failure response could be further improved and moved into the form object itself, which tidies up the controller.
