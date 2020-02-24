---
title: Migrating to Rails 6 - Keeping your assets
header:
  image: /assets/images/shane-hauser-18dsFU_VpZI-unsplash-x1200.jpg
tags:
  - ruby
  - assets
  - webpacker
  - architecture
  - deployment
classes: wide
---

**First step to migrating your app**

### Why Rails 6 - keeping the technological debt at minimum

*Rails 6* is the newest version of the Rails framework, it was released in 2019
and introduced a few important changes to the way apps are written in it. One of
the biggest changes was made to handling assets (stylesheets and javascript files)
by introducing *Webpacker* (preferably with *Yarn*) as the tool for maintaining
and compiling .js files.

### App assets / packages configuration

### Server setup for Rails 6 app or multiple apps with different tech stacks
First and foremost you have to make sure that the server is compatible with the
app. It is a good practice to configure *RVM* or *Rbenv* to match library versions
required by the application. It is useful to create a new *gemspec* for example:
*ruby-2.6.3@yourapp* and maintain proper versions of libraries there. This will
keep conflicts away on the line *server configuration -> library versions ->
Gemfile and app versioning -> deployment*.

Things you need to remember about:
* install nodejs,
```bash
curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
```
* install yarn - or else during deployment asset precompilation will fail but the
error message will not help at all with resolving the problem,
```bash
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo add-apt-repository ppa:chris-lea/redis-server
```
* update packages with `sudo apt-get update`,

## Maintaining multiple apps with different Ruby versions
[*Multiple Rubies with a single Passenger*](https://coderwall.com/p/x2_z4a/multiple-rubies-with-a-single-passenger).

In order to keep multiple applications on the same server that differ with their
Ruby versions we have to specify it in configuration. The best way to do so would
be to pass **PassengerRuby** to the *VirtualHost* config part of the app. Remember
that *PassengerDefaultRuby* is set globally, while *PassengerRuby* can be set per
project.
