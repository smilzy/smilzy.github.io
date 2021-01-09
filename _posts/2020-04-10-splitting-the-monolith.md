---
title: Splitting the monolith - Rails app architecture
header:
  image: /assets/images/jose-magana-XxzA5BMl-40-unsplash-x1200.jpg
  caption: "Photo credit: [**josemagana**](https://unsplash.com/@josemagana)"
  teaser: /assets/images/jose-magana-XxzA5BMl-40-unsplash-teaser.jpg
tags:
  - ruby
  - architecture
  - monolith
  - design patterns
  - microservices
  - engines
classes: wide
---

**Reorganize your application with Rails Mountable Engines**

## The Dream

*Deep breath...*

Ahh, that relaxing feeling. You start working on an application wondering how will it behave
when finished. The plan is there - make it easily maintainable and a pleasure to
work with. Everything has its place and runs like clockwork. The soothing feeling
you get when thinking how smooth working on this project will be reminds you of the
noise ocean waves hitting the shore make.

*But let's fast forward a few months.*

## The Rock

Wha... What happened? This didn't go as planned, did it? **A few temporary
solutions and some experimental features found their way into the project.** The
client (or the market) forces the product to quickly adapt to new requirements.
The foundation of your *dream application* stands firmly, but is accompanied by
a clutter of functionalities that make it a giant mess. Objects that should have
been separate entities are incorporated into bigger objects. **If you don't stop it
now it will grow stronger and ultimately transform into a scary spaghetti monster!**
<!-- ,a server room with cables everywhere and a big yellow sticker: *DO NOT TOUCH***. -->

{:refdef: style="text-align: center;"}
<!-- ![Cable clutter](/assets/images/martijn-baudoin-4h0HqC3K4-c-unsplash-x1200.jpg) -->
![Spaghetti in a bowl](/assets/images/stir-fry-noodles-in-bowl-2347311-x900_2.jpg)
{: refdef}
*Rawr! Tasty, but that's not what an application should be*
<!-- *Tasty, but impractical* -->
{: style="color:gray; font-size: 80%; text-align: center;"}

## The Crack

So here comes this moment. You've had enough and found some time for a bigger
refactoring task. You want to future-proof the application, so you started making
plans to tidy this mess.
**A whole section of your application can be extracted into an own being.** A new life
is born - a child that has to learn to communicate with its' parent.
In the back of your head you start hearing voices saying: *microservices - it's
the hot stuff*.

# BUT don't just go *rails new*!
{: style="text-align: center;"}
This approach screams: **problems!**

Don't throw yourself into the deep waters. Making a sudden change of this caliber
is not going to end well and may lead to you ripping your hair out of your head
from frustration. If your app is not well prepared it will end up in you having
to maintain two different applications with similar code bases and will
**slow development** majorly. Trust me - I've already been there and came to
realization that's a wrong order of operation to start with a new app.

<!-- we 1 -->

*So... How to do it without making an even bigger mess, and doubling
the workload instead?*
<!-- * **Plan for it** - decide which database tables should be transferred to -->
## The Solution

After spending some time on developing the new application I realized that it takes too long
to make it work like before. Whenever I thought that I'm close to finishing
a new problem popped up and slowed my development by another week. During this time
lots of features were introduced to the core application and that meant I had to
catch up. This seemed wrong and made my work unpleasant. We needed something quicker.
After another discussion with colleagues from *ReasonApps* we decided to take a
step back and I was tasked with finding another solution.

{:refdef: style="text-align: center;"}
![Steam engine train toy](/assets/images/gerold-hinzen-3OiH7mIqB78-unsplash-x900.jpg)
{: refdef}
*Not only a fun toy, but a great tool as well*
{: style="color:gray; font-size: 80%; text-align: center;"}

### Rails mountable engines
So I did some reading, I did some watching (*some smart stuff not Netflix BTW*) and found something
that immediately made me say: **that's it**. The previous idea was good, but order
of tasks should be reversed. First thing to be done was cleaning up our mess.
Rushing new architecture design without preparation is bound to bring problems,
and the finished product - migrating from monolithic architecture to microservices -
might end up looking like this:

![Migrating a shitty monolith ends up in shitty microservices](/assets/images/monolithic vs microservices.jpeg)

Instead I focused on making the application **modular**
So I started extracting the logic to a module inside the core application instead, and
worked out problems from there. I made it work more like a library - **a gem** -
which on it's own is very similar to a new application, but for now can still be
a part of the core app. Any progress made trying to extract logic into a new app
is useful as well.

I've learned about this approach first from watching Ruby conferences on YouTube -
[*Shopify journey to a Modular Monolith*](https://www.youtube.com/watch?v=ISYKx8sa53g) -
and reading Shopify blog posts with comments from one of the engineers.
{: style="color:gray; font-size: 80%;"}
<!-- [*Deconstructing the Monolith - Designing software that maximizes developer productivity blog post*](https://engineering.shopify.com/blogs/engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity) -->

## The steps
### 1. Plan for it
Make an Excel sheet on Google Docs or similar and share it with your team. It should
contain all models and parts of the application that you wish to extract from the
core app and their associations (connections) to the foundation. At this point
you'll realize there are many unnecessary things that clutter your app logic.
Decide which models should be transferred fully and what would link them to the core.
Good planning is very important and will help you in the long run, so don't underestimate it.
After that you can start creating an engine.
<!-- Decide which database tables can be enclosed in your new module. Then, after creating a new appication copy over the *db/schema.rb*
and leave the tables that should stay. Create the tables using *rake db:schema:load* -->
<!-- task. -->
<!-- * **Move logic** - copy config files, gems, model and linked business logic files, controllers and views. -->
### 2. Create a mountable engine with isolated namespace
<!-- Copy config files, gems, model and linked business logic files, controllers and views. -->
<!-- * **Document necessary changes** - it's time for debugging and looking over the new -->
Creating an engine is actually *VERY* easy, I was shocked - why didn't I know about
it earlier? ¯\\_ツ_/¯
{: style="white-space: nowrap;"}
```ruby
rails plugin new engine_name --mountable --skip-test --dummy-path=spec/dummy
```
The `--skip-test --dummy-path=spec/dummy` should be used only if you plan on using
RSpec. And voila - your rails engine is here! You should see a folder *engine_name*
in your application tree. Create a special folder **engines** and put your modules
there.
If you don't wish for your extracted models to require a table prefix go to
`engines/engine_name/lib/engine_name.rb` and modify it like this:
```ruby
require "engine_name/engine"

module EngineName
  # NOTE: To have table names without engine prefix
  def self.table_name_prefix; end
end
```
I highly advise you to get to know [*Getting Started with Engines*](https://guides.rubyonrails.org/engines.html)
page of Ruby on Rails docs. It explains basics such as how files should be nested
inside an engine, how to make engine configurable or inherit / require from the
core app.

Then I moved routes to the **routes.rb** in the engine and mounted the engine:
```ruby
Rails.application.routes.draw do
  mount EngineName::Engine, at: "/"
  # ...
end
```
Engine routes are accessible the same way as before, or you can specify an engine
route by prefixing the path with `engine_name.` like that: `engine_name.root_path`.

### 3. Move models, controllers and views
Now is the time for moving. First of all if you wish for your engine's
ApplicationController to be the same as core app ApplicationController then
make it inherit from it.

```ruby
module EngineName
  class ApplicationController < ::ApplicationController
  end
end
```

Then move your models / controllers / views and other files to engine like this:
`engines/engine_name/app/controllers/engine_name/orders_controller.rb`
`engines/engine_name/app/models/engine_name/order.rb`
`engines/engine_name/app/views/engine_name/orders/show.haml`
`engines/engine_name/app/models/concerns/engine_name/orderable.rb`
There will be lots of fixing involved, changing paths of rendered partials,
namespaces and things that you've missed if you skipped step 1 - but don't worry!
Any thing you miss can be discovered easily and rails will gladly throw an exception
at you. ;) Remember about proper file nesting, sometimes it can be unintuitive.

### 4. Configure gems, dependencies, config / initializers and setup testing environment
Your newly created engine is an own being. Think of it as a gem which uses some
other tools and has its' own dependencies - and those you manage via **gemspec**.
You can declare gems used by your models and views as `add_dependency`
(which is an alias for `add_runtime_dependency`) and gems that are only used
during development or testing with `add_development_dependency`.

```ruby
$:.push File.expand_path("../lib", __FILE__)

# Maintain your gem's version:
require "engine_name/version"

# Describe your gem and declare its dependencies:
Gem::Specification.new do |s|
  s.name        = "engine_name"
  s.version     = EngineName::VERSION
  s.authors     = ["Aleksander Woźniak"]
  s.email       = ["4wozniak@gmail.com"]
  s.homepage    = "https://wozniakalex.com"
  s.summary     = ": Summary of EngineName."
  s.description = ": Description of EngineName."
  s.license     = "MIT"

  s.files = Dir["{app,config,db,lib}/**/*", "MIT-LICENSE", "Rakefile", "README.md"]

  s.add_runtime_dependency "rails", "~> 5.1.4"
  s.add_runtime_dependency 'pg', '~> 0.18'

  # Gems used by models
  s.add_runtime_dependency 'aasm'
  s.add_runtime_dependency 'enumerize'
  s.add_runtime_dependency 'money-rails'
  # ...

  s.add_development_dependency 'redis-namespace'
  s.add_development_dependency 'rspec-sidekiq'
  s.add_development_dependency 'shoulda-callback-matchers'
  s.add_development_dependency 'shoulda-matchers'
end
```

```ruby
module EngineName
  class Engine < ::Rails::Engine
    isolate_namespace EngineName

    # NOTE: Expand engine's migration files to the core app
    initializer :append_migrations do |app|
      unless app.root.to_s.match(root.to_s)
        config.paths['db/migrate'].expanded.each do |expanded_path|
          app.config.paths['db/migrate'] << expanded_path
        end
      end
    end

    # NOTE: Tell rails rspec generators where to put files
    config.generators do |g|
      g.test_framework :rspec
      g.fixture_replacement :factory_bot
      g.factory_bot dir: 'spec/factories/engine_name'
    end
  end
end
```

At this point you're pretty much set and you can continue your journey with Rails
engines and modular monolithic applications. Your next step should be setting up
spec environment for your engine. In its' *spec* folder you will find a folder
named *dummy* which stands for a *dummy application*. Inside you'll find files
organized like a standalone Rails app. Everything that's placed here will be used by
your tests, so things like models and behavior from the core app. We don't want to
copy all those files in here and maintain them to be up to date so instead we'll
require dependencies in the `dummy/config/application.rb` file:

```ruby
require_relative 'boot'

require 'rails/all'

Bundler.require(*Rails.groups)
require "engine_name"

module Dummy
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 5.1
    config.time_zone = 'Warsaw'

    config.i18n.load_path += Dir[Rails.root.join('..', '..', '..', '..', 'config', 'locales', '**', '*.{rb,yml}')]
    config.i18n.enforce_available_locales = false
    config.i18n.available_locales = ["pl", "en"]
    config.i18n.default_locale = :pl
    config.active_record.default_timezone = :local

    config.active_record.time_zone_aware_attributes = false

    # NOTE: Load files from the core app
    config.autoload_paths << File.expand_path('../../../../../app/models', __dir__)
    config.autoload_paths << File.expand_path('../../../../../app/queries', __dir__)
    config.autoload_paths << File.expand_path('../../../../../app/queries/concerns', __dir__)
    config.autoload_paths << File.expand_path('../../../../../app/services', __dir__)
    config.autoload_paths << File.expand_path('../../../../../app/decorators', __dir__)
    config.autoload_paths << File.expand_path('../../../../../app/workers', __dir__)
  end
end
```
While working with engine for the first time some other problems might occur
with Rails autoloading, loading dependencies, Continuous Integration / Delivery
processes, running multiple spec environments. Most of it can be dealt with by
smart usage of `require`, but some require custom solutions. While running specs
on production we've run into some dependency problems which were solved by requiring
development dependencies in our `rails_helper.rb` in our engine.

```ruby
Gem.loaded_specs['engine_name'].development_dependencies.each do |d|
  require d.name
end
```

### 5. Create a communication layer
Hey, remember how at step 1 you were tasked with coming up with an idea how many
connections should there be between the engine and the core app? Now we need this
knowledge. We have to create an interface for our engine to communicate with core
app models and logic without having to be included in the same folder, and be more
**dynamic**. Good preparation is key here, but this will come in another episode.

### Summary
To sum it up - **Engines are the best way to start organizing your Rails application**,
and - in my opinion - are the first step to make, when planning on introducing
architecture changes. Cleaning our own mess first, and modularizing the application
improves development, keeps the code clean and helps maintain proper programming
principles. Lots of companies decide on introducing this approach to their business,
and all of them speak in superlatives. If you've never tried this approach before
- you should, so get going! :)  

Hungry for more?
[*Martin Fowler's article*](https://martinfowler.com/articles/break-monolith-into-microservices.html)
<!-- if spaghetti then TASTY BUT IMPRACTICAL -->




<!-- NOTE: Either use server room images and cable mess stuff, or look into
SPAGHETTI MONSTER theme and finish it with a clean looking photo of a tasty
spaghetti, and a caption along the lines of: HUNGRY FOR MORE? blah blah. -->
