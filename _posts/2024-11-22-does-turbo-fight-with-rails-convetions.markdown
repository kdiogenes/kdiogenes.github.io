---
title: Does Turbo Fight with Rails Conventions?
date: 2024-11-22 16:50:00 -0300
categories: [Backend, Frameworks]
tags: [Rails, Turbo, Conventions, Hotwired]
---

Let's say I have a `transcriptions/index.html.erb` with the following code:

```erb
<div class="px-2 py-4">
  <ul role="list" class="divide-y divide-gray-100">
    <%= render @transcriptions %>
  </ul>
</div>
```

By the Rails convention, the partial `transcriptions/_transcription.html.erb` will get rendered for each transcription.

Let's assume that the `transcriptions/_transcription.html.erb` partial has the following content:

```erb
<li id="<%= dom_id transcription %>">
  <!-- Actual content here -->
</li>
```

And also consider that we have a model called `Transcription` with the following the code:

```ruby
class Transcription < ApplicationRecord
  after_update_commit :broadcast_updates

  private
    def broadcast_updates
      broadcast_replace
    end
end
```

According to the [broadcast_replace documentation](https://www.rubydoc.info/gems/turbo-rails/Turbo%2FBroadcastable:broadcast_replace) it's the same as `broadcast_replace_to`, but the designed stream is automatically set to the current model. Here is what the [broadcast_replace_to documentation](https://www.rubydoc.info/gems/turbo-rails/Turbo%2FBroadcastable:broadcast_replace_to) says:

```ruby
# Sends <turbo-stream action="replace" target="clearance_5"><template><div id="clearance_5">My Clearance</div></template></turbo-stream>
# to the stream named "identity:2:clearances"
clearance.broadcast_replace_to examiner.identity, :clearances

# Sends <turbo-stream action="replace" target="clearance_5"><template><div id="clearance_5">Other partial</div></template></turbo-stream>
# to the stream named "identity:2:clearances"
clearance.broadcast_replace_to examiner.identity, :clearances, partial: "clearances/other_partial", locals: { a: 1 }

# Sends <turbo-stream action="replace" method="morph" target="clearance_5"><template><div id="clearance_5">Other partial</div></template></turbo-stream>
# to the stream named "identity:2:clearances"
clearance.broadcast_replace_to examiner.identity, :clearance, attributes: { method: :morph }, partial: "clearances/other_partial", locals: { a: 1 }
```

This means that when my model callback gets executed, this is what will happen:

```ruby
# Sends <turbo-stream action="replace" target="transcription_5"><template><li id="transcription_5">My Transcription Content</li></template></turbo-stream>
# to the stream named "transcription:5"
```

Considering that my index will have various transcriptions, how can I send this HTML to the index page? You need to create a stream channel for each transcription? Ok, you can add something like this to index page:

```erb
<% @transcriptions.each do |transcription| %>
  <%= turbo_stream_from transcription %>
<% end %>
```

But if you are also implementing pagination, for example, the turbo channels is one more thing that you should keep updated.

This made me feel that the implementation, or the documentation, is a bit counter-intuitive. It's like there is a conflict between the **broadcasts** model methods that sets the stream automatically and the conventions long used on views.

Maybe I'm a bit addicted to using Rails conventions in a certain way, but so far I haven't been able to see a way how these methods can represent a convention. In my head, something that you don't need to specify parameters cover a general use case.

Anyway, for this case, the best solution I found was to use a specific channel name with `turbo_stream_from` in `transcriptions/index.html.erb`:

```erb
<%= turbo_stream_from :transcriptions %>

<div class="px-2 py-4">
  <ul role="list" class="divide-y divide-gray-100">
    <%= render @transcriptions %>
  </ul>
</div>
```

And use `broadcast_replace_to` in `Transcription` model:

```ruby
class Transcription < ApplicationRecord
  after_update_commit :broadcast_updates

  private
    def broadcast_updates
      broadcast_replace_to :transcriptions
    end
end
```

I'm only starting my journey of using Turbo Stream on a real application, so if you think I'm overlooking something, let me know!