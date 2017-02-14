---
layout: post
title: "haskell for rubyists: functors"
categories: programming ruby haskell
---

People have been paying me to write Ruby for about 5 years now. It wasn't my original programming language. But Ruby is a great language for the web, and I've used it very broadly: I've put JRuby in production, CRuby in production, written C extensions and Java extensions, and shipped I guess now thousands of features at large scale and small in it. I love Ruby's readability and simple central metaphor: everything is a message or a receiver. It's still the language I reach to when I want to quickly open a REPL and have the computer do simple tasks for me. (`JSON.parse(File.read('filepath'))` is etched on my brain.)

But Ruby frustrates me sometimes, because in no particular order: it has some functional-ish features without being functional (`compose` as an operator is still in the works as of this writing), it lacks an expressive type system, it has `nil` (how many hours do we spend debugging `NilClass` can't receive a method?), its core language[^ruby-async] async and concurrency primitives are way behind the times (comparing to Go or Java or Python or even Javascript), the global interpreter lock prevents parallelism, and its lack of import/export[^ruby-import] semantics mean it's really easy to overwrite methods (especially since nobody uses refinements) and a pain in the butt to make sure you've namespaced safely.

How does Haskell compare on these complaints? Well, it is definitely a truly functional programming language, it has an expressive type system, and it has import/export. For concurrency primitives, Haskell ships with thread support I don't see as particularly different from Ruby's. But in terms of parallelism, [Haskell can do multicore](https://wiki.haskell.org/Haskell_for_multicores) while Ruby cannot, so it's no contest there.

Although Haskell is infamous for its Monads, I actually want to just talk about [Functors](https://wiki.haskell.org/Typeclassopedia#Functor) to show a basic building block in Haskell that we really lack in Ruby, and some of the weirdness that results from that absence. Before your eyes glaze over, let me dangle a Ruby term in front of you: [Enumerable](http://ruby-doc.org/core-2.4.0/Enumerable.html).

In Ruby, we iterate over collections with `each` (or `map` or a bunch of other methods). `each` is from Enumerable:

> The class must provide a method each, which yields successive members of the collection.

In Ruby, `each` returns the original Array, while `map` returns the aggregation of return values of the block execution. Both of the main collection types (Hash and Array) in Ruby have an `each` and `map`. But in my experience, the duck typing of Ruby can lead to subtle, annoying errors when working with Hash, because 1. a unary block (you fail to destructure via `|k,v|`), via duck-typing, produces Arrays, and 2. because `map` _always returns an Array_!

Let's say I want to iterate over a Hash and change its values somehow.

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
=> ["BAR", "BAZ"]

# simply switching to each won't work either
# we only got the original receiver of the block
# at least with .each though, we got back the same type of thing we put in
h.each {|k, v| v.upcase }
=> {:foo=>"bar", :biz=>"baz"}

# so, we need to mutate our receiver
h.each {|k, v| h[k] = v.upcase }
=> {:foo=>"BAR", :biz=>"BAZ"}

# if we don't want to mutate the receiver, we have to use reduce to inject
# in a "memo" object to reconstruct the structure of our original object
# note I must return "memo" on each iteration of the block if I use []=
h.reduce({}) {|memo, (k,v)| memo[k] = v.upcase ; memo }
=> {:foo=>"BAR", :biz=>"BAZ"}

# or use merge! which returns the full hash on addition of a new pair
# though it performs somewhat worse (30% in my tests) and make sure
# you dont use merge -- it runs in quadratic time!
h.reduce(h) {|memo, (k,v)| memo.merge!({k => v.upcase}) }
=> {:foo=>"BAR", :biz=>"BAZ"}
```

In practice, it's pretty easy to forget for a moment what the type of your receiver is though and then you're sending a block that can only work on an Array to a Hash and getting back an error or a really weird coerced Hash-like Array (`[:foo, "bar"]`).

I can't tell you how many times I've found in professional production code coercions back to Hash via splat (`Hash[*[:foo, "bar"]]`) when the coder could've just properly formed their block or used a different iteration method in the first place. But I'm not trying to dump on other programmers. I'm pointing out that you need to take on some cognitive load to keep track of your object, because the language isn't doing it for you. That's what a good type system could be doing for you: telling you when different procs can actually be properly received.

I said above `each` retains some structure for you. Ok, so, `each` _almost_ provides a pretty useful guarantee: if it weren't for `delete`, the structure you put in is the structure you get back out, even if you can mutate your original variable out of that structure.

```ruby
h = {"foo": "bar", "biz":"baz"}
h.each {|k, v| h[:new_key] = [k,v]}
# RuntimeError: can't add a new key into hash during iteration

h.each {|k, v| h = "something" }
# return value is the original self
=> {:foo=>"bar", :biz=>"baz"}
h
# even though we mutated h to be a String!
=> "something"

# ok back to a real hash
h = {"foo": "bar", "biz":"baz"}
h.each {|k, v| h.delete(k) }
# all the elements are gone, but at least still a hash ¯\_(ツ)_/¯
=> {}
```

Unfortunately, even if `delete` didn't break it, that guarantee's usefulness is limited precisely because we need to use side-effects to achieve any interesting behavior with `each`. And that means we'd need to be able to refer to `each`'s receiver inline to return new values the way `map` does, but if you've ever printed out `self` from inside a block sent to `each` you've seen we don't:

```ruby
[1,2,3].each {|e| p "#{self} - #{e}" }
# "main - 1"
# "main - 2"
# "main - 3"
=> [1, 2, 3]
```

We need a generic variable name like `self` to target our receiver, across different receivers with different variable names. But we don't have it, so we're stuck needing to know our receiver's name:

```ruby
a = [1,2,3]
a.each {|e| p "#{a} - #{e}" }
# "[1, 2, 3] - 1"
# "[1, 2, 3] - 2"
# "[1, 2, 3] - 3"
=> [1, 2, 3]
```

Or doing some chicanery[^credit]:

```ruby
[1,2,3].instance_eval { each {|e| p "#{self} - #{e}" } }
"[1, 2, 3] - 1"
"[1, 2, 3] - 2"
"[1, 2, 3] - 3"
=> [1, 2, 3]

# if we want to return self instead of the aggregated values
# we have to hack map using instance_eval
h.instance_eval { map {|k, v| self[k] = v.upcase } ; self }
=> {:foo=>"BAR", :biz=>"BAZ"}
```

But even with that `instance_eval` we can't reuse our block as a Proc, which obviates the utility of our extractions in the first place:

```ruby
printer = proc {|e| p "#{self} - #{e}" }
# printer captured self as main, from the scope it was defined in
[1,2,3].instance_eval { map(&printer) }
# "main - 1"
# "main - 2"
# "main - 3"
=> ["main - 1", "main - 2", "main - 3"]
```

So, `each` could almost maintain structure, but requires mutation to be useful. Meanwhile, `map` doesn't necessitate mutating our original object but doesn't guarantee structure. (In fact, it almost guarantees coercion away from our structure.) At least `map` can be used more easily with blocks we extracted into Procs, while `each` can't:

```ruby
double = proc {|a| a * 2 }
a = [1,2,3,4,5]
a.each(&double)
=> [1, 2, 3, 4, 5] # nope, got back the receiver!
a.map(&double)
=> [2, 4, 6, 8, 10] # there we go
```

When we can extract to Procs, we can then also compose them (I'm using [proc_compose](https://github.com/mooreniemi/proc_compose#usage) here):

```ruby
double = proc {|a| a * 2 }
triple = proc {|a| a * 3 }
[1, 2, 3, 4, 5].map(&(double * triple))
=> [6, 12, 18, 24, 30]
```

That's equivalent to doing `[1, 2, 3, 4, 5].map(&double).map(&triple)`. And _that_ is one of the 2 Functor laws:

```haskell
fmap id = id
fmap (g . h) = (fmap g) . (fmap h)
```

But because Ruby's `map` doesn't enforce that structure is maintained, not all Enumerable classes are Functors. (Only Arrays.) With some work, we can make Hash's `map` structure preserving and give ourselves a lawful `fmap`:

```ruby
# note merge! is less performant than []=, but more readable imho
class Hash
  def fmap &block
    self.reduce({}) { |memo, (k,v)| memo.merge!({ k => block.call(v) }) }
  end
end
```

Now, with a real `fmap` we can do this:

```ruby
h.fmap(&:upcase)
=> {:foo=>"BAR", :biz=>"BAZ"}
# and this!
h.fmap(&(double * triple))
=> {:foo=>"barbarbarbarbarbar", :biz=>"bazbazbazbazbazbaz"}
```

So why _does_ Ruby's map do what it does? Why implement Hash's `map` to operate on values, like `fmap`, and then _also_ collect just those values into an Array? Seems very weird to me, but I only realized it having worked with a language like Haskell that exposes and specifies its abstractions in its type system. How does Haskell organize code such that you can be sure any collection (except Set) is a Functor? Let's tackle that in another post, but the term you should google is "type class".

[^ruby-async]: A lot of [extensions](https://github.com/ruby-concurrency/concurrent-ruby) exist, and Ruby 3 is going to have [guilds](http://olivierlacan.com/posts/concurrency-in-ruby-3-with-guilds/) to handle concurrency in core.
[^ruby-import]: You can get these semantics from the [cargo](https://github.com/soveran/cargo) gem, which I have never seen used in a production project yet, sadly.
[^credit]: Thanks to toretore on #ruby on irc.freenode for showing me this bit of badness.
