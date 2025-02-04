---
layout: post
title:  Digging on ActiveStorage attached methods
date:   2019-09-18 13:23:00 -0300
categories: [Backend, Frameworks]
tags: [Rails, ActiveStorage, Metaprogramming]
---
I read [Metaprogramming Ruby 2](https://pragprog.com/book/ppmetr2/metaprogramming-ruby-2) a long time ago, but I
remember to read it more for the challenge than for the fun. The explanation about the Ruby object model is fantastic,
difficult to grasp, but what to expect from a complex subject? I never find a better explanation on the topic.

Looking again to its topic contents, I don't remember much about it, but I remembered the chapter *Metaprogramming Is
Just Programming* while reading [`has_one_attached`](https://github.com/rails/rails/blob/master/activestorage/lib/active_storage/attached/model.rb#L35).

Look at it:

```ruby
def has_one_attached(name, dependent: :purge_later)
  generated_association_methods.class_eval <<-CODE, __FILE__, __LINE__ + 1
    def #{name}
      @active_storage_attached_#{name} ||= ActiveStorage::Attached::One.new("#{name}", self)
    end
    def #{name}=(attachable)
      attachment_changes["#{name}"] =
        if attachable.nil?
          ActiveStorage::Attached::Changes::DeleteOne.new("#{name}", self)
        else
          ActiveStorage::Attached::Changes::CreateOne.new("#{name}", self, attachable)
        end
    end
  CODE

  has_one :"#{name}_attachment", -> { where(name: name) }, class_name: "ActiveStorage::Attachment", as: :record, inverse_of: :record, dependent: :destroy
  has_one :"#{name}_blob", through: :"#{name}_attachment", class_name: "ActiveStorage::Blob", source: :blob

  scope :"with_attached_#{name}", -> { includes("#{name}_attachment": :blob) }

  after_save { attachment_changes[name.to_s]&.save }

  after_commit(on: %i[ create update ]) { attachment_changes.delete(name.to_s).try(:upload) }

  ActiveRecord::Reflection.add_attachment_reflection(
    self,
    name,
    ActiveRecord::Reflection.create(:has_one_attached, name, nil, { dependent: dependent }, self)
  )
end
```

If you use Rails, you probably know about `has_one`, `scope`, `after_save` and `after_commit`. Probably you already
figured out most of what `has_one_attached` is doing? Use it is like transforming this:

```ruby
class MyModel < ApplicationRecord
  has_one_attached :file
end
```

To:

```ruby
class MyModel < ApplicationRecord
  def file
    @active_storage_attached_file ||= ActiveStorage::Attached::One.new("file", self)
  end
  def file=(attachable)
    attachment_changes["file"] =
      if attachable.nil?
        ActiveStorage::Attached::Changes::DeleteOne.new("file", self)
      else
        ActiveStorage::Attached::Changes::CreateOne.new("file", self, attachable)
      end
  end

  has_one :"file_attachment", -> { where(name: :file) }, class_name: "ActiveStorage::Attachment", as: :record, inverse_of: :record, dependent: :destroy
  has_one :"file_blob", through: :"file_attachment", class_name: "ActiveStorage::Blob", source: :blob

  scope :"with_attached_file", -> { includes("file_attachment": :blob) }

  after_save { attachment_changes[:file.to_s]&.save }

  after_commit(on: %i[ create update ]) { attachment_changes.delete(:file.to_s).try(:upload) }

  ActiveRecord::Reflection.add_attachment_reflection(
    self,
    :file,
    ActiveRecord::Reflection.create(:has_one_attached, :file, nil, { dependent: dependent }, self)
  )
end
```

What struck me most about this code is its simplicity and readability. Ruby has a great support for metaprogramming,
this code shows how receptive the language is to change code at runtime.

The secret to understand this code, is understanding how the Ruby object model works. The book above is incredible to
explain all the details, but I think I can provide some fuel for the curious.

When the Ruby interpreter sees `class MyModel < ApplicationRecrod`, it's an instruction to change the value of `self` to
the object `MyModel` (yeah, in Ruby classes are very live objects). So when the interpreter finds
`has_one_attached :file` it's the same as `self.has_one_attached :file`. Every time you don't specify a receiver, it
will be `self`!

A good way to understand it better is with a simple example using `rails c` or `irb`:

```ruby
"ruby".upcase # upcase receiver is the "ruby" string - explicit
puts "ruby" # what is the receiver of puts? self! - implicit
# where is puts implemented?
self.class # output: Object
self.class.ancestors # output: [Object, Kernel, BasicObject]
```

`puts` is implemented in the [Kernel module](https://ruby-doc.org/core-2.6.4/Kernel.html#method-i-puts), the console
`self` is an instance of `Object`, so this is the reason why you don't need to specify a receiver in this case. You can
explore this even further by writing `puts self.inspect` in you class, or use [pry](http://pryrepl.org/) and writing
`binding.pry` in your class. `pry` is fantastic to explore your Ruby objects.

Great, this explains why all these `ActiveRecord` methods can be used inside `has_one_attached`: in this context, `self`
is a class object inheriting from `ApplicationRecord`, so you can access all the ActiveRecord API. I didn't realize
about metaprogramming this way while reading the book, maybe because I was not very experient with Ruby or because I was
not seeking for the fun.

Probably I have to revisit this book looking for the fun, but it's also a reminder to keep learning the concepts, keep
coding and above all keep reading code. Realize that metaprogramming can be this readable is so:

![Mind blown!](/assets/images/mind-blown.gif)
{: style="text-align: center"}

Lastly, these are great powers, use wisely and remember Uncle Ben.
