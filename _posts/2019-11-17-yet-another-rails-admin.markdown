---
layout: post
title:  "Yet another Rails admin?"
date:   2019-11-17 22:02:00 -0300
categories: [Backend, Frameworks]
tags: [Rails, Admin, Metaprogramming]
---
No, only a kadim!

[kadim](https://github.com/fnix/kadim) is derived from "cadim", an expression from the brazillian mineiro dialect, speaked in the central region of Minas
Gerais, that means a little bit. This region is most famous for its wonderful cheese bread, cachaça and feijoada!

Admins are all about CRUD, right?

When I first hear about [Administrate](https://github.com/thoughtbot/administrate) I though: "oh, finally, an admin to
rule all of them!". What get my attention was they guiding principles:

* No DSLs (domain-specific languages)
* Support the simplest use cases, and let the user override defaults with standard tools such as plain Rails controllers
  and views.
* Break up the library into core components and plugins, so each component stays small and easy to maintain.

Although you don't have a DSL, you still need to declare some
[data structures](http://administrate-prototype.herokuapp.com/customizing_dashboards) to define how data is presented.

The generated views are not what a Rails programmer would expect/write, this is an excerpt from a generated show view:

```erb
<section class="main-content__body">
  <dl>
    <% page.attributes.each do |attribute| %>
      <dt class="attribute-label" id="<%= attribute.name %>">
      <%= t(
        "helpers.label.#{resource_name}.#{attribute.name}",
        default: attribute.name.titleize,
      ) %>
      </dt>

      <dd class="attribute-data attribute-data--<%=attribute.html_class%>"
          ><%= render_field attribute, page: page %></dd>
    <% end %>
  </dl>
</section>
```

You know that `page.attributes` is related with the data structures pointed above, but you don't know how they are
available in the view without digging in the code.

I think that it supports the simplest uses cases, but it doesn't appear to be very flexible IMO. After working a long
time with RailsAdmin, flexibility is the aspect that I consider most important in this kind of solution.

You can think about scaffold and scaffold_controller generators as a cheap way to create admin areas, right? I also
think the generated code is more flexible than the one provided by Administrate generators.

So, getting back to my initial question, yeah, admins can be much more than CRUDs, but in its heart it's all about CRUD.
This led me to the question of how hard would it be to dynamically generate scaffolds? This was the spark for
[kadim](https://github.com/fnix/kadim). It will scaffold a controller with views for each model present in your
application, automatically updating controllers and views while your model changes.

For now it deals only with columns in your database. Relations, attachments and other ActiveRecord features are ignored.
It's still little, but it's enough to prove that it's possible to create a more dynamic admin. Get interested, take a
look at our [roadmap](https://github.com/fnix/kadim#roadmap).
