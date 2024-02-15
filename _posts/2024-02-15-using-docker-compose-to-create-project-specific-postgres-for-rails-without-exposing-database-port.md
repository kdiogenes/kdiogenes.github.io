---
layout: post
title:  "Using docker-compose to create project-specific PostgreSQL for Rails without exposing database port"
date:   2024-02-15
categories: rails postgres docker-compose
---

When working with Rails applications, it's common to use PostgreSQL as the database. While it's easy to set up a PostgreSQL server on your local machine, it can be a bit cumbersome to manage different versions of PostgreSQL for different projects. Additionally, you might not want to expose the database port so you can be able to run multiple project databases without having to deal with ports.

![Managing PostgreSQL with Docker Compose](/assets/images/2024-02/dynamic-and-visually-engaging-scene-developer-working-on-a-Rails-project.png)

One way to solve this problem is to use `docker-compose` to create a project-specific PostgreSQL instance. This allows you to run the database server in a container without exposing the database port to the host machine. Here's how you can do it:

### Step 1: Create a `docker-compose.yml` file

Create a file named `docker-compose.yml` in the root of your Rails project. This file will define the services that make up your application, including the PostgreSQL database.

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16.2
    environment:
      POSTGRES_USER: $USER
      POSTGRES_HOST_AUTH_METHOD: trust
    volumes:
      - ./.docker-volumes/pgdata:/var/lib/postgresql/data
      - ./.docker-volumes/pgsockets:/var/run/postgresql
```

In this file, we define a service called `postgres` that uses the `postgres:16.2` image. We also set the `POSTGRES_USER` environment variable to the current user and set `POSTGRES_HOST_AUTH_METHOD` to `trust` to allow connections without a password. We then define two volumes to persist the database data and sockets.

### Step 2: Start the PostgreSQL service

To start the PostgreSQL service, run the following command in the root of your Rails project:

```bash
docker-compose up -d postgres
```

This will start the PostgreSQL service in the background. You can verify that the service is running by running the following command:

```bash
docker-compose ps
```

### Step 3: Configure your Rails application

To use the PostgreSQL service in your Rails application, you need to update the `config/database.yml` file to point to the PostgreSQL service. Here's nice configuration (you just need to add the host key to the default entry of a newly generated project that uses PostgreSQL):

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  host: <%= Rails.root.join ".docker-volumes/pgsockets" %>
```

This configuration tells Rails to connect to the PostgreSQL service using the Unix socket located at `.docker-volumes/pgsockets` in the root of your project. It's also have the advantage to be generic enough to work in any folder, so you don't need to change it when you move your project around or sharing your code with others.

Happy hacking!