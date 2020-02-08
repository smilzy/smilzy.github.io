---
title: Splitting the monolith - Rails app architecture
header:
  image: /assets/images/jose-magana-XxzA5BMl-40-unsplash-x1200.jpg
  caption: "Photo credit: [**josemagana**](https://unsplash.com/@josemagana)"
tags:
  - ruby
  - architecture
  - monolith
  - design patterns
  - microservices
  - api
classes: wide
---

**Organize your application**

## The Dream

*Deep breath...*

Ahh, the dream. You start working on an application wondering how will it behave
when finished. The plan is there - make it easily maintainable and a pleasure to
work with. Everything has its place and runs like clockwork. The soothing feeling
you get when thinking how smooth working on this project will be reminds of the
noise ocean waves make when hitting the shore.

*But let's fast forward a few months.*

## The Rock

Wha... What happened? This doesn't look like planned, does it? A few temporary
solutions and some experimental features found their way into the project. The
client (or the market) forces the product to quickly adapt to new requirements.
The foundation of your *dream application* stands firmly, but is accompanied by
a clutter of functionalities that make it a giant mess. Objects that should have
been separate entities are incorporated into bigger objects. If you don't stop it
now it will grow stronger and ultimately transform into a scary spaghetti monster,
a server room with cables everywhere and a big yellow sticker: *DO NOT TOUCH*.

{:refdef: style="text-align: center;"}
<!-- ![Cable clutter](/assets/images/martijn-baudoin-4h0HqC3K4-c-unsplash-x1200.jpg) -->
![Spaghetti in a bowl](/assets/images/stir-fry-noodles-in-bowl-2347311-x900_2.jpg)
{: refdef}
*Tasty, but that's not what an application should be*
<!-- *Tasty, but impractical* -->
{: style="color:gray; font-size: 80%; text-align: center;"}

## The crack

So here comes this moment. You've had enough and found some time for a bigger
refactoring task. You want to future-proof the application, so you started making
plans to tidy this mess.
A whole section of your application can be extracted into an own being. A new life
is born - a child that has to learn to communicate with its' parent.

### rails new ...

But wait, how to do it without making an even bigger mess, and doubling
the workload instead?
<!-- * **Plan for it** - decide which database tables should be transferred to -->
### 1. Plan for it
Decide which database tables should be transferred to
the new application. Then, after creating a new appication copy over the *db/schema.rb*
and leave the tables that should stay. Create the tables using *rake db:schema:load*
task.
<!-- * **Move logic** - copy config files, gems, model and linked business logic files, controllers and views. -->
### 2. Move logic
Copy config files, gems, model and linked business logic files, controllers and views.
<!-- * **Document necessary changes** - it's time for debugging and looking over the new -->
### 3. Document necessary changes
It's time for debugging and looking over the new
application files step by step deleting unused parts of code and writing comments
like *FIXME:* with descriptions of changes needed to make it work again. Keeping
note of it in *readme.md* helps as well. Continue doing so trying to complete the
flow of the application by fixing and refactoring parts of code, adding missing
columns to tables and transferring parts of code that you missed first time.
Then - decide how to build the API.
<!-- * **Create a communication layer** - -->
### 4. Create a communication layer
Create a new version of API in your applications 


Hungry for more?
[*Martin Fowler's article*](https://martinfowler.com/articles/break-monolith-into-microservices.html)
<!-- if spaghetti then TASTY BUT IMPRACTICAL -->




<!-- NOTE: Either use server room images and cable mess stuff, or look into
SPAGHETTI MONSTER theme and finish it with a clean looking photo of a tasty
spaghetti, and a caption along the lines of: HUNGRY FOR MORE? blah blah. -->
