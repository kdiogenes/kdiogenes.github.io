---
layout: post
title:  "Turbolinks Gotchas"
date:   2020-08-24 12:36:57 -0300
categories: rails turbolinks
---
If you see Turbolinks loading a link twice it can be related with
https://github.com/turbolinks/turbolinks#reloading-when-assets-change. In my specific case I was using different layouts
with one of them having an extra script tag annotated with `data-turbolinks-track="reload"`. Reading the first sentence
of this section again, it seems pretty obvious that a reload will occur:

> Turbolinks can track the URLs of asset elements in <head> from one page to the next and automatically issue a full
> reload if they change.

Adding or removing an element with `data-turbolinks-track="reload"` is a change in the assets URLs, right?

I expected it to only load the extra asset, but this makes sense? Let's see what the maintainers think: put an issue
link here...
