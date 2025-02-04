---
title: Dev, don't put Rails on the whale
date: 2018-07-23
categories: [Backend, DevOps]
tags: [Rails, Docker, Development Environment, PostgreSQL, Redis, ElasticSearch]
---

![Rails on Docker](/assets/images/2018-07/rails-on-docker.webp)
_If you need, or if you just want to do it, go on!_

In my project I use Rails, docker and docker-compose. I install Rails local and use a Procfile to start other services through docker-compose.

The services used in this project are PostgreSQL, Redis and ElasticSearch.

For me this is the perfect configuration for a development environment. Tools like rvm and rbenv are a breeze to manage project specific gems and after years working with them, I never find a gem with native extensions that work on my machine, but not on production environments.

These tools also allows for multiple Ruby/Rails versions without any layer beyond YARV. Why I would want to use docker to replace any of these tools?

The benefits of not using docker goes beyond this. Without it, you can easily use spring, pry, bundle open, bundle install works smooth and other debug tools that needs an interactive shell.

The benefits of not using docker goes beyond this. Without it, you can easily use spring, pry, bundle open, bundle install works smooth and other debug tools that needs an interactive shell.

Docker is cool, but above all, it need to be useful.