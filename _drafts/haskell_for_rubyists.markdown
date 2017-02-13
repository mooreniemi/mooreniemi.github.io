---
layout: post
title: "haskell for rubyists"
categories: programming ruby haskell
---

People have been paying me to write Ruby for about 5 years now. It wasn't my original programming language; I started out in Python and R, because I was doing NLP and data processing at the start of my career. Ruby is a great language for the web, and I've used it very broadly: I've put JRuby in production, CRuby in production, written C extensions and Java extensions, and shipped I guess now thousands of features at large scale and small in it. I love Ruby's readability and simple central metaphor: everything is a message or a receiver. It's still the language I reach to when I want to quickly open a REPL and have the computer do simple tasks for me. (`JSON.parse(File.read('filepath'))` is etched on my brain.)

But Ruby frustrates me sometimes, because in no particular order: it has some functional-ish features without being functional (`compose` as an operator is still in the works as of this writing), it lacks an expressive type system, it has `nil` (how many hours do we spend debugging `NilClass` can't receive a method?), its core language[^ruby-async] async and concurrency primitives are way behind the times (comparing to Go or Java or Python or even Javascript), the global interpreter lock prevents parallelism, and its lack of import/export[^ruby-import] semantics mean it's really easy to overwrite methods (especially since nobody uses refinements) and a pain in the butt to make sure you've namespaced safely.

How does Haskell compare on these complaints? Well, it is definitely a truly functional programming language, it has an expressive type system, and it has import/export. For concurrency primitives, Haskell ships with thread support I don't see as particularly different from Ruby's. But in terms of parallelism, [Haskell can do multicore](https://wiki.haskell.org/Haskell_for_multicores) while Ruby cannot, so it's no contest there.

Although Haskell is infamous for its monads, I actually want to just talk about [functors](https://wiki.haskell.org/Typeclassopedia#Functor) to start. Before your eyes glaze over, let me dangle a Ruby term in front of you: [Enumerable](http://ruby-doc.org/core-2.4.0/Enumerable.html).

In Ruby, we iterate over collections with `each` (or `map` or a bunch of other methods). `each` is from Enumerable:

> The class must provide a method each, which yields successive members of the collection.

In Ruby, `each` returns the original Array, while `map` returns the aggregation of return values of the block execution. Both of the main collection types (Hash and Array) in Ruby have an `each` and `map`. But in my experience, the duck typing of Ruby can lead to subtle, annoying errors when working with Hash, because a unary block, via duck-typing, produces Arrays.

Let's say I want to map over a Hash and change its values somehow.

```ruby
h = { "foo": "bar", "biz": "baz" }

# if we ignore the structure of hash in our block,
# it provides a single local variable to our block,
# and then it coerces to Array and fails
h.map(&:upcase)
# => NoMethodError: undefined method `upcase' for [:foo, "bar"]:Array

# we can try to fix the block we'll map with by parameterizing it...
h.map {|k, v| v.upcase }
# but map still can't work, because we lost the original structure
# see, we only got our return values of the block
# => ["BAR", "BAZ"]

# simply switching to each won't work either
# we only got the original receiver of the block
# at least with .each though, we got back the same type of thing we put in
h.each {|k,v| v.upcase }
# => {:foo=>"bar", :biz=>"baz"}

# so, we need to mutate our receiver
h.each {|k,v| h[k] = v.upcase }
# => {:foo=>"BAR", :biz=>"BAZ"}
```

In practice, it's pretty easy to forget for a moment what the type of your receiver is though and then you're sending a block that can only work on an Array to a Hash and getting back an error or a really weird coerced Hash-like Array (`[:foo, "bar"]`).

I can't tell you how many times I've found in professional production code coercions back to Hash via splat (`Hash[*[:foo, "bar"]]`) when the coder could've just properly formed their block in the first place. But I'm not trying to dump on other programmers. I'm pointing out that you need to take on some cognitive load to keep track of your object, because the language isn't doing it for you. That's what a good type system could be doing for you: telling you when different procs can actually be properly received.

In Ruby, Enumerable objects provide `each`, and `each` _almost_ provide a pretty useful guarantee: the structure you put in is the structure you get back out, even if you can mutate your original variable out of that structure. The exception seems to be `delete`, and I'm curious if it isn't a bug.

```ruby
h = {"foo": "bar", "biz":"baz"}
h.each {|k, v| h[:new_key] = [k,v]}
# RuntimeError: can't add a new key into hash during iteration

h.each {|k,v| h = "something" }
# return value is the original self
# => {:foo=>"bar", :biz=>"baz"}
h
# even though we mutated h to be a String!
# => "something"

# ok back to a real hash
h = {"foo": "bar", "biz":"baz"}
h.each {|k,v| h.delete(k) }
# all the elements are gone, but still a Hash!
# => {}
```

Unfortunately, even if `delete` didn't break it, that guarantee's usefulness is limited precisely because we need to use side-effects to achieve any interesting behavior with `each`. And that means we'd need to be able to refer to `each`'s receiver inline to return new values the way `map` does, but if you've ever printed out `self` from inside a block sent to `each` you've seen we don't:

```ruby
[1,2,3].each {|e| p "#{self} - #{e}" }
# "main - 1"
# "main - 2"
# "main - 3"
# => [1, 2, 3]
```

This means we're stuck knowing our receiver's name:

```ruby
a = [1,2,3]
a.each {|e| p "#{a} - #{e}" }
# "[1, 2, 3] - 1"
# "[1, 2, 3] - 2"
# "[1, 2, 3] - 3"
# => [1, 2, 3]
```

Or doing some chicanery[^credit]:

```ruby
[1,2,3].instance_eval { each {|e| p "#{self} - #{e}" } }
"[1, 2, 3] - 1"
"[1, 2, 3] - 2"
"[1, 2, 3] - 3"
=> [1, 2, 3]
```

Or using our Hash example:

```ruby
# if we want to return self instead of the aggregated values
# we have to hack map using instance_eval
h.instance_eval { map {|k, v| self[k] = v.upcase } ; self }
# => {:foo=>"BAR", :biz=>"BAZ"}
```

Without that `instance_eval` we can't reuse our block as a Proc; we need a generic variable name like `self` to target our receiver, across different receivers with different variable names. So, `each` could almost maintain structure, but requires mutation to be useful. Meanwhile, `map` doesn't guarantee structure but doesn't necessitate mutating our original object. That means `map` can be used more easily with blocks we extracted into Procs, while `each` can't:

```ruby
double = proc {|a| a * 2 }
a = [1,2,3,4,5]
a.each(&double)
# => [1, 2, 3, 4, 5] # nope, got back the receiver!
a.map(&double)
# => [2, 4, 6, 8, 10] # there we go
```

When we can extract to Procs, we can then also compose them (I'm using [proc_compose](https://github.com/mooreniemi/proc_compose#usage) here):

```ruby
double = proc {|a| a * 2 }
triple = proc {|a| a * 3 }
[1, 2, 3, 4, 5].map(&(double * triple))
# => [6, 12, 18, 24, 30]
```

That's equivalent to doing `[1, 2, 3, 4, 5].map(&double).map(&triple)`. And _that_ is one of the 2 Functor laws:

```haskell
fmap id = id
fmap (g . h) = (fmap g) . (fmap h)
```

But because Ruby's `map` doesn't guarantee that structure is maintained, Ruby's Enumerable classes aren't Functors.

[^ruby-async]: A lot of [extensions](https://github.com/ruby-concurrency/concurrent-ruby) exist, and Ruby 3 is going to have [guilds](http://olivierlacan.com/posts/concurrency-in-ruby-3-with-guilds/) to handle concurrency in core.
[^ruby-import]: You can get these semantics from the [cargo](https://github.com/soveran/cargo) gem, which I have never seen used in a production project yet, sadly.
[^credit]: Thanks to toretore on #ruby on irc.freenode for showing me this bit of badness.
