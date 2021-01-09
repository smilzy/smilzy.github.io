---
title: Form Objects in Ruby - Dynamic nested fields (nested-form)
header:
  image: /assets/images/wim-arys-gDlpMyInsak-unsplash-1200x.jpg
  caption: "Photo credit: [**wimarys**](https://unsplash.com/@wimarys)"
  teaser: /assets/images/wim-arys-gDlpMyInsak-unsplash-teaser.jpg
tags:
  - ruby
  - form objects
  - forms
  - design patterns
  - associations
classes: wide
---

**Step up your form object wizardry**

### Picking it up where we left it
*This post is a continuation of [Form Objects in Ruby - Multi-Step Wizard](https://smilzy.github.io/form-objects-in-ruby-part-1/).*

Thus far custom form object from previous article handles multi-step validations
and associations in the basic way. That already covers most uses of creating
or updating associations through forms. But when the need for dynamically associating
objects (which there could be *n* of) appears, the code so far won't be sufficient.
Usage of custom JavaScript solutions or gems and the way they are handled by Rails
will create problems when trying to validate it with custom form object. For example
when using [*Nested Form*](https://github.com/ryanb/nested_form) an error appears
after submitting the form informing about abundance of specified association for
the form object. `undefined method 'reflect_on_association' for NilClass:Class`

### What to do then?

No worries though, because I've already been there and done that, so I can present
you a one-line solution for this problem.


```ruby
def self.reflect_on_association(args)
  # NOTE: Fix for association presence check by nested form
  OpenStruct.new(klass: args.to_s.classify.constantize)
end
```

This small snippet of code tricks Rails by returning an object which responds to
*klass* method and returns the class name of nested association. It's *clean* and
it gets the job done only for the problematic case without making a mess in the project.

To make it work the association name must be implemented as an attribute *attr_reader*
and setup in the constructor of the form. Params generated for associated object
in the nested form take form of an array of key-value objects, so we handle them
accordingly. Getting rid of items with *_destroy* parameter set to true before
further validation makes the process easier.

Full code example is presented below:

```ruby
class OrderCreateForm
  include ActiveModel::Model
  attr_reader :items

  def initialize(attrs = {})
    attr_accessors
    super(attrs)
    @items = order.items.any? order.items : [order.items.build]
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
      item_attrs = item_attributes.delete_if { |k, v| v[:_destroy] == '1' }
      item_attributes.any? && ! [:count, :price].all? { |attr| item_attrs[attr].present? }
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

  def self.reflect_on_association(args)
    # NOTE: Fix for association presence check by nested form
    OpenStruct.new(klass: args.to_s.classify.constantize)
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

### Update action form object

You could read and wonder *I know how to handle create action, but what about updating?*

Well, nothing easier than that, the only difference between a Form Object create action
presented above and update action lays in the code for *save* action and actually
looking up the object specified by *id* parameter. While creating a new object is
defined, validated and finally created with given parameters the update is performed
on an existing object and is validated and then updated with given parameters
(between steps or on final step - the choice is yours).

```ruby
class OrderUpdateForm
  include ActiveModel::Model
  attr_reader :items

  def initialize(attrs = {})
    attr_accessors
    super(attrs)
    @items = order.items.any? order.items : [order.items.build]
  end

  def save
    return false unless valid?
    client.update(params_cleaned_up)
  end

  def order
    @order ||= Order.find_by_id(id)
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
      item_attrs = item_attributes.delete_if { |k, v| v[:_destroy] == '1' }
      item_attributes.any? && ! [:count, :price].all? { |attr| item_attrs[attr].present? }
    end
  end

  class Step3 < Step2
    validates :price, numericality: { greater_than_or_equal_to: 0 }

    def last_step?
      true
    end
  end

  def self.reflect_on_association(args)
    # NOTE: Fix for association presence check by nested form
    OpenStruct.new(klass: args.to_s.classify.constantize)
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

### Form objects structure

This summarizes most use cases for form objects in Ruby:
* basic form behaviour
* multi-step wizard
* between-step validations
* between-step callbacks
* nested associations
* dynamic object creation & validation

This approach lets us create only fully prepared objects without generating drafts
in semi-ready stages, which later on clog the database by being abandoned objects that
will never reach their final form.

Having full control over a form object allows creating complex forms with easily
maintainable object state, clean testing environment, and gives a lot of room for
customization and improvements.
Calling external services, performing validations, testing and finally saving
only the fully prepared object is a walk in the park with this approach.

Final Base Form Object structure sufficient enough to be the ancestor for specific
form objects is presented below:
```ruby
class BaseForm
  include ActiveModel::Model
  include Rails.application.routes.url_helpers

  def initialize(attrs = {})
    attr_accessors
    super(attrs)
  end

  def save
    valid?
  end

  def last_step?
    true
  end

  private

  def permitted_attributes
    {}
  end

  def attributes_hash
    {}.tap { |hash| permitted_attributes.each { |attr| hash[attr] = send(attr) } }
  end

  def attr_accessors
    permitted_attributes.each { |attr| class_eval { attr_accessor attr } }
  end

  def params_cleaned_up
    params = attributes_hash
    params
  end
end
```
