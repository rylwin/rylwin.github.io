---
layout: post
title: "Stop writing Rails applications"
date: 2013-09-29 16:01
comments: true
categories: ruby rails
---

## TL;DR

Don't just throw all your code in `app/models`; don't just build a Rails app.
Have another directory that contains all of your domain-specific logic. Put some
thought into how you organize the files in that directory. And there's a rule to
follow: files in this directory must not invoke any Rails-provided methods.

## Write an application for your domain, not a Rails app

Don't get me wrong. **Rails is fantastic**--I love it! But my past Rails
projects all eventually suffer from poor architecture and slow tests. **It's not
Rails' fault, it's mine. I've been leaning too heavily on Rails this whole
time.** Now I'm finally learning how to build great apps that I enjoy working on
long beyond the prototype stage.

The key to building an application with Rails that stands the test of time is to
not use Rails as the defining feature of the application.  **Rails is just a
delivery mechanism and a persistence layer**.  The domain logic should be
separate from Rails. (Thanks to Bob Martin for
[motivating me to confront this this issue](http://youtu.be/WpkDN78P884).)

At the same time, **Rails provides a lot of default goodness out of the box. I
do not want a lot of extra code that re-invents anything** already provided
by Rails. The goal is to achieve an improved architecture without sacrificing
the ease provided by Rails.

I've split this post into two parts (ignoring this intro). The first is a
high-level explanation of the architecture that I'm proposing.  In the second
section, I'll show some of the code I'm using to get this all to work.

## The Architecture (from 20k ft)

The short version: `app/models` contains my ActiveRecord models and `app/lib`
contains application / domain-specific logic.

### app/models

The **classes in `app/models` should be ActiveRecord classes or classes that
extensively use the ActiveRecord API** (e.g., query-helper classes).
Additionally, code in **`app/models` should know nothing of our domain logic**.
The classes should be focused only on data retrieval and storage. One way to
think of `app/models` classes is as **facades to the ActiveRecord API**.

I use ActiveRecord models throughout the application, but **I don't write any
code outside of `app/models` that directly uses an ActiveRecord method**. This
means that, in code outside of `app/models`, I only invoke standard field
getters/setters and methods that I've defined on the models.

This approach is a balancing act. For example, I let scaffolded controllers
invoke ActiveRecord methods--I'm not going to spend time updating the default
controllers.  This goes to the point of not creating extra work / reinventing
the wheel.  After all, one could have PORO classes to represent each
ActiveRecord model to the rest of the application, but achieving this would
require a decent amount of work for what may be an almost entirely academic
benefit.

### app/lib

The **`app/lib` directory contains the business/domain logic**. The code in
`app/lib` must have **no knowledge of Rails**--none at all.  `app/lib` classes
(all POROs) can still use the model classes, but they are not allowed to use any
ActiveRecord API methods (except the field getters/setters). This means direct
invocation of `save`, `create`, `update_attributes`, all querying, etc. is off
limits.

By separating the application logic into a separate directory, I've found it
much **easier to make good architectural decisions**. A nice benefit of this
approach is that it makes it very easy to write
[fast_specs](http://arrrrcamp.be/videos/corey-haines/fast-rails-tests) since I
know that code in `app/lib` does not have dependencies on Rails or the database.
All models in `app/lib` have their specs located in a `fast_spec` directory.

The classes within `app/lib` are organized into modules. The modules you define
will vary depending on your domain, but I'll share a few of mine from a project
I'm working on:

#### Integration - interaction with remote data sources

Integration classes are responsible for interacting with remote data sources.
These classes have no dependencies on our other classes. The external sources
may be remote APIs, uploaded data files, etc.

#### Render - renders documents in various formats

Classes in the render module are responsible for rendering non-HTML/JSON views
(e.g., render a PDF, XLS, etc). It is often useful to be able to generate these
files outside of the standard request/response cycle and it's much easier to
test the result when it's just plain Ruby.

#### Service - encapsulates user story logic

Service classes encapsulate the logic of our user stories (or parts of stories).
The service classes decrease complexity and coupling in the system by their
organization and by providing interaction between the persistence layer
(`app/models`) and the domain logic layer (`app/lib`). Service classes may
invoke other service classes to build more complex behaviors.

#### Util - code not relevant to our domain

Any code unrelated to our specific domain gets placed here (e.g., I have a class
that converts XLSB files to XLS and another that compares hashes). Anything you
throw in here might be a good candidate for a gem!

## This sounds awesome. How can I do it? (i.e., the code)

### Setting up `app/lib`

Add the following to config/application.rb:

```ruby
config.autoload_paths += %W(#{config.root}/app/lib)
```

That's all you need to do in order to begin adding code into `app/lib`! Here's
what my `app/lib` looks like:

```bash
integration/    render/     service/    util/
integration.rb  render.rb   service.rb  util.rb
```

Each folder (module) has a simple file with the same name that creates the
module. E.g.,:

```ruby service.rb
module Service; end
```

### Setting up `fast_spec`

For the corresponding fast_specs, add a `fast_spec` directory at the top level
of your project. Then add a file `fast_spec/spec_fast_helper.rb`:

```ruby fast_spec/spec_fast_helper.rb
require 'bundler'
# require 'other gems from your bundle'

root = File.join(File.dirname(__FILE__), '..')

# Set up any configuration here...
# I18n.load_path = [File.join(root, 'config', 'locales', 'en.yml')]

# Require your app/lib folder. You may need to require some classes first
# depending on how you've set up your class hierarchy.
Dir[File.join(root, 'app', 'lib', '**', '*.rb')].sort.each do |lib|
  require lib
end
```

I also added a rake task so that I can `rake fast`:

```ruby lib/tasks/000_fast.rake
require 'rspec'

desc "Run fast specs"
RSpec::Core::RakeTask.new(:fast) do |task|
  task.pattern = 'fast_spec/**/*_spec.rb'
  task.rspec_opts = '-Ifast_spec'
end

task :default => :fast
```

Now in each fast_spec file just `require 'spec_fast_helper'`. If you like guard,
check out [guard-fast_spec](https://github.com/shutl/guard-fast_spec). And if
you haven't been using guard because your tests have been too slow, try it
again.  When your tests are this fast you'll love it.

When I need to include ActiveRecord models in the tests, I just use stubs. The
danger here is that your specs become out of sync with your models, but a decent
integration test suite should catch these issues.

Since all the complicated logic is contained in `fast_spec`, the specs in `spec`
are all simple. Really simple. Which means that those specs run decently fast:

```bash
$ time rake spec
# output truncated
...............................................................................
...............................................................................
...............................................................................
...............................................................................
..................................

Finished in 6.82 seconds
350 examples, 0 failures

rake spec  19.44s user 1.62s system 93% cpu 22.624 total
```

And the `fast_specs`?

```bash
$ time rake fast
# output truncated
...............................................................................
...............................................................................
........

Finished in 1.09 seconds
166 examples, 0 failures

rake fast  7.91s user 0.77s system 98% cpu 8.798 total
```

It's still a young app so there aren't a ton of specs, but I think you get the
idea.

## Minimal changes, big results

This simple organizational change (along with the associated rules) makes it
easier for me to employ good OOP design principles in my projects that use
Rails, yet I'm still able to reap the benefit of Rails (which is plenty!).

Let me know if you like this idea. If there's interest, I'll write some
follow-up posts with code samples that demonstrate how I'm using this `app/lib`
organization to write cleaner, more modular code.
