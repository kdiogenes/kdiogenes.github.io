---
layout: post
title:  "My first Rails engine"
date:   2019-08-18 12:36:57 -0300
categories: rails programming engine gem
---
Writing a Rails engine, also know as a Ruby Gem, appears to be a very scary task, don't you think?

The documentation is not so good as the Rails one, although it's give you some good bits of informations to get started.

After writing some bits of code, you perceive that things don't behave as you mind, autoloading is different, there is
no much documentation about the possibilities, how to structure things, etc.

The first thing I would like to have more information upfront, was about the `--full` and `--mountable` options. I find
this write https://www.travisluong.com/ruby-on-rails-mountable-vs-full-engine/ from 2014 to have some good
clarifications, but this part "The mountable engine comes with its own layout, javascript, and css files, while a full
engine does not." gave me uncertainty, my generated --full engine has app/assets/{config,images,stylesheets}.

But okay, I can live with this uncertainty.

The second chapter of Getting Started with Engines, Generating an engine, from Ruby on Rails guide has a good
description of what will be generated with --full and --mountable options, but the guide lacks to give hints about when
to choose one or another.

My hint would be:
* --mountable: when you want an isolated Rails application inside a hosting Rails application
* --full: when you want more integration with the hosting Rails application

You can google more about (I would say that you should), but I also would like to say some more words about this
distinction.

A mountable engine is almost identical a Rails application, except that it will run inside another Rails application.
Use it when you need the full power of Rails inside your engine. Some examples of engines providing mountable solutions
are [Sidekiq](https://github.com/mperham/sidekiq), [RailsAdmin](https://github.com/sferik/rails_admin) and
[GraphiQL-Rails](https://github.com/rmosolgo/graphiql-rails).

A full engine is a kind of custom logic that you want to enhance in your hosting Rails applications. Some popular
examples are [Simple Form](https://github.com/heartcombo/simple_form), [bootsnap](https://github.com/Shopify/bootsnap)
and most of Rails gems like [Active Record](https://github.com/rails/rails/tree/master/activerecord),
[Active Storage](https://github.com/rails/rails/tree/master/activestorage), etc. They are mostly solutions to makes
programmer life better.

There are some gems that appear to be mountable, like
[Comfortable Mexican Sofa](https://github.com/comfy/comfortable-mexican-sofa), but it's not mountable by design, it
needs to interact heavily with the hosting application. So, a full engine is not only solutions to support development,
it can also be used to make Rails like applications.

In the end you have to reason about how much your engine will need to interact with the hosting application. Less
interaction indicates a mountable and more a full engine. Remember, this is just an advice, you will be able to interact
with the hosting application in both ways, so be prepared to experiment a bit to decide what feels right for your
engine.
