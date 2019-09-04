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

The first thing I would like to have more information upfront, was about the `--full` and `--mountable` options. This
write https://www.travisluong.com/ruby-on-rails-mountable-vs-full-engine/ from 2014 has some clarifications, but this
part "The mountable engine comes with its own layout, javascript, and css files, while a full engine does not." gives me
uncertainty, my generated --full engine has app/assets/{config,images,stylesheets}.

But okay, I can live with it's uncertainty.

One of the things that bogus me is why my engine controllers don't get autoload? And what worries me more is how I can
autoload my controllers?

