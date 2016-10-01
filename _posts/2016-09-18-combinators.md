---
layout: post
title: combinators in Ruby
---

[Strange Loop conference](http://www.thestrangeloop.com/) is outputting
videos at a steady clip. This one especially, by [Amar
Shah](https://twitter.com/amar47shah), caught my interest:

<iframe width="560" height="315"
src="https://www.youtube.com/embed/seVSlKazsNk" frameborder="0"
allowfullscreen></iframe>

This video resonated with me especially because recently [The Morning
Paper](https://blog.acolyer.org/) highlighted a paper called "[Why
Functional Programming
Matters](https://blog.acolyer.org/2016/09/14/why-functional-programming-matters/)".
It's not the first time I've seen the point made, but it's made well here:
we need to focus on the positive contributions of functional style, not
just what it restricts. The bit I want to focus in on is that modularity
is positive because it allows us to "glue" our program back together
differently:

> It is now generally accepted that modular design is the key to
> successful programming… However, there is a very important point that is
> often missed. When writing a modular program to solve a problem, one
> first divides the problem into subproblems, then solves the subproblems,
> and finally combines the solutions. The ways in which one can divide up
> the original problem depend directly on the ways in which one can glue
> solutions together. Therefore, to increase one’s ability to modularize
> a problem conceptually, one must provide new kinds of glue in the
> programming language.

I didn't think deeply on it at the time, but once I saw Shah's talk
I thought to myself: if functions are the bricks, then combinators are the
mortar ("glue"). Though perhaps confusingly, the mortar is just functions
too...

As always, I was able to rely on [my Haskell programmer in
residence](https://twitter.com/cora_jr) and used their help to walk
through some of the examples. Since Shah starts with `blackbird`, I'll
start there too.

## blackbird

Let's say we have a list of lists, and we want to get the sum of all the
lengths of the lists. So, we have a list like so: `l = [[1], [1, 2], [1,
2, 3]]` and since the lengths are respectively 1, 2, and 3, we get `6` for
the sum of all the lengths.

In the most **concrete** implementation of this behavior, I can simply do:
`l.map(&:length).reduce(0,:+)`. Now, what if I want to go more
**abstract** and to parameterize the function I `map` with? For instance,
if I wanted to just sum the first element of every list. I want something
like `l.map(parameter).reduce(0,:+)`. Well, I can do this:

```ruby
def aggregater
  proc {|f,l|
    l.map(&f).
      reduce(0, :+)
  }.curry
end
```

Now I can call it like so: `aggregater.(:length).(l) => 6` or
`aggregrator.(:first).(l) => 3`. (Note if I wasn't using `curry` I would
need to create nested procs for every variable I want to bring into
scope.) You've probably noticed at this point that I've unnecessarily
coupled the summation into the mapping. Let's break them out into two
independent `proc`s:

```ruby
sum = proc {|a| a.reduce(0, :+) }
# again we're just using curry here to save
# ourselves nesting procs for both f and a
# which is why we dont need it for sum
map = proc {|f,a| a.map &f }.curry
```

So we can say `aggregater` is a _combination_ of the two functions: `sum`
and `map`. It'd be neat if I could somehow just say `sum_map
= sum.combine.map` and then use `sum_map`. It turns out this is a well
known combinator named `blackbird`.

```ruby
aggregate = sum.blackbird.(map)
# this expectation passes :)
expect(aggregate.(:length).(list)).to eq(aggregater.(:length).(list))
```

`blackbird`, then is a way of capturing some of the plumbing we needed to
`compose` the functions.

```ruby
def compose
  proc do |i,x|
    self.(i.(x))
  end.curry
end

def * other
  self.compose.(other)
end

def blackbird
  proc {|g|
    # \f g x y -> f(g x y)
    self.compose * g
  }
end
```

This may seem like a lot of plumbing for not much gain in the calling
syntax, but keep in mind how we got `aggregater` and a huge
group of similarly shaped functions basically for free once we could use
`blackbird`. Now rather than rigidly and verbosely defining `aggregater`
such that it could only ever do a summation over a mapped function, we
have the mechanisms we need to create _any_ function that combines two
functions this way.

## interlude: method chaining vs function composition

In Ruby, most of the time we don't talk about function composition because
we talk about method chaining. For a Rubyist, a method chain passes values
along to be successively transformed but[^1] this has a real weakness: you
can only chain things which receive your methods. As soon as you return
a core Ruby type, you're either stuck using _only_ Ruby methods, going
back to nesting in functions, or adding some plumbing into the chain.


For example, say we have `c = Cat.new`:

```ruby
class Cat
  def meow
    "meow!"
  end
  def louder
    upcase + "!!!"
  end
end
```

It'd be cool if I could do `c.meow.louder` and get back `MEOW!!!!`. This
won't work though because the receiver becomes `String` between `meow` and
`louder`, resulting in: `NoMethodError: undefined method 'louder' for
"meow!":String`. We end up either making `meow` return `self` (forcing us
to use side-effects), parameterizing `louder` and calling `louder(meow)`,
or we use [tap](https://ruby-doc.org/core-2.3.1/Object.html#method-i-tap):

```ruby
c = Cat.new
=> #<Cat:0x007ff1ca9e0ef8>
c.meow.tap {|e| e }.louder
=> "MEOW!!!!"

# we could also make a convenience method
class Object
  def chain
    tap {|e| e }
  end
  alias :_ :chain
end
c.meow._.louder
=> "MEOW!!!!"
```

In contrast, what if we don't work through Ruby's central metaphor of
receivers and methods, but instead Make Everything A Function? Bear with
me here, but let's turn these methods into `proc`s:

```ruby
class Cat
  def meow
    proc {|cat| "#{cat} says: meow!" }
  end
  def louder
    proc {|noise| noise.upcase + "!!!" }
  end
end
```

Now if I want to use these functions I can compose them together. I'm
still composing `out * in`.

```ruby
(c.louder * c.meow).('kitty')
=> "KITTY SAYS: MEOW!!!!"

# or capture it in a method
class Cat
  attr_accessor :name
  def loud_meow
    # self is implicit here
    (louder * meow).(name)
  end
end

c.name = 'Kitty'
c.loud_meow
=> "KITTY SAYS: MEOW!!!!"
```

It actually doesn't matter now _what_ object was the receiver to a method.
We could compose methods on `Dog` (so long as they also returned `proc`s)
with anything in `Cat`. The receivers then just become a sort of
namespace:

```ruby
class Dog
  def bark
    proc {|dog| "#{dog} says: bark!" }
  end
end
d = Dog.new
=> #<Dog:0x007ff1caac8118>
(c.louder * d.bark).('doggo')
=> "DOGGO SAYS: BARK!!!!"
```

You see we've sort of squished the receiver all the way out to the
function parameter. In fact, all receivers will be squished out to the
end, meaning we won't ever be nesting
`receiver.method(like.this(receiver.method))`. I won't argue this is
_objectively_ better, but sometimes it can be nice to have all your inputs
in one place.

What I think we can say is objectively better is the performance benefit
we get from being able to `compose` functions.

In a functional language, we can take advantage of a theorem that looks
like this: `map f (map g xs) = map (f . g) xs`. So this will pass:

```ruby
  it 'obeys map f (map g xs) = map (f . g) xs' do
    expect([1,2].map(&double).map(&triple)).
      to eq([1,2].map(&(double * triple)))
  end
```

Notice now we're **only calling `map` once**! Theoretically, this isn't
a huge savings (just a constant factor), but it's free! Pretty cool,
I think. Unfortunately in Ruby, this doesn't [_actually_ perform
better](https://github.com/mooreniemi/experiments/blob/master/lib/compose.rb)
in real execution:

![performance is worse for composition](/images/map_vs_compose.gif)

Incidentally, what I wanted was to be able to do something more "normal"
for readability, like `meow.louder`. (Like we achieved using `tap`.) How
do I reverse function composition? One simple way is a combinator called
[`thrush`](https://codepen.io/Universalist/post/thrush-combinator-in-racket)!

```ruby
def thrush
  proc do |i,x|
    i.(self.(x))
  end.curry
end

def % other
  self.thrush.(other)
end
```

Armed with `thrush` I can call in the other order:

```ruby
(c.meow % c.louder).('i')
=> "I SAYS: MEOW!!!!"
```

## useful combinators, combining usefully

The simplest combinators might not be that impressive. But the "glue"
angle can have some handy results. Shah also covered `on` (`psi`), which
to me is where the plumbing starts to pay off.

Say I have two lists of lists, and I want to see if across the lists every
sublist has a matching sized sublist. So, for `a = [[1], [2,3], [4,5,6]]`
and `b = [[9], [8,7], [6,5,4]]`, we'd return `true`. We know we need to
`map(&:length)` over each sublist, and then we need to `==` each length.
(For simplicity these are pre-sorted lists.)

If I can accept some duplication, I can do something pretty clear[^2]:

```ruby
a.map(&:length) == b.map(&:length)
=> true
```

Hmm, any way I could spare myself needing to use `map(&:length)` twice?
And how can a combinator help? Let me show you how the code looks when we
use `on` (the `psi` combinator):

```ruby
describe '#psi (on)' do
  let(:compare) do
    proc {|a, b|
      a == b
    }
  end
  let(:map_length) do
    proc {|a|
      a.map(&:length)
    }
  end
  it 'can be used to compare two items by intermediation' do
    a = [[1], [2,3], [4,5,6]]
    b = [[9], [8,7], [6,5,4]]
    sublist_lengths = compare.on.(map_length)
    expect(sublist_lengths.(a).(b)).to eq(true)
  end
end
```

And here's `psi` itself:

```ruby
def psi
  proc {|g,x,y|
    # \f g x y -> f(g x) (g y)
    self.(g.(x), g.(y))
  }.curry
end
alias :on :psi
```

So if we just replace the body of `psi` with how it is used, here's an
example of the pattern we caught in an abstraction:

```ruby
compare.(map_length.(a), map_length.(b))
```

Exactly the prefix form of what we had originally! Not a huge _size_
savings, but IMHO `compare.on.(map_length)` reads a lot better, and when
I want to change the operation I'm doing to both lists, I only need to
change one reference.

```ruby
sublist_sums = compare.on.(map_sum)
sublist_sums.(a).(b)
=> false
```

Combinators glue functions together. In Ruby, that looks pretty weird (I
think most Rubyists would agree) because we don't deal in functions; we
deal in methods. But in a language like Haskell, it is very natural. If we
accept that modularity via function composition is a key improvement in
our programming lives our programming language can give us, I think we
have to admit something like Haskell might be strictly speaking better
than something like Ruby.

If you're curious to play with some of the (very) few Ruby combinators
I wrote they're [on
github](https://github.com/mooreniemi/experiments/blob/master/spec/compose_spec.rb).
If you want to see a complete library, [this Haskell
aviary](https://hackage.haskell.org/package/data-aviary-0.4.0/docs/Data-Aviary-Birds.html)
is super useful[^3].

[^1]: Personally I wish [refinements](https://ruby-doc.org/core-2.1.1/doc/syntax/refinements_rdoc.html) were more popular, because it'd help this a lot.
[^2]: If I don't care about clarity, and I want chaining with no repetition I can do something hairy with `zip`: `a.zip(b).collect {|e| e.map(&:length) }.map(&:uniq).all? {|e| e.size == 1 }`. Bit of a mouthful eh? And do you follow exactly what I'm doing here, like, intent-wise? [Here's the play-by-play](https://gist.github.com/mooreniemi/26771e58c3a4961ba1e2a7830b97351c).
[^3]: The type signatures basically led me through what I needed to write.
