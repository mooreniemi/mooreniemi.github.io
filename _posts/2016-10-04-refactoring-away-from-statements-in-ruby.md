---
layout: post
title: refactoring away from statements in Ruby
---

Since I've encountered exactly the same problem several times, and just
encountered it afresh today in a codebase at work, I want to dissect a bit
how I solved it once. Take this advice with a grain of salt. I'm positive
there's other, perhaps superior, ways to resolve this. But I thought this
was a useful set of observations regardless.

The situation is probably a familiar one: you want to filter some data. In
this case, in Rails. You've parsed some parameters and depending on those
parameters you want to call different methods and compose them to filter
down a result set.

Because
I [open-sourced](https://github.com/mooreniemi/Transbucket_Rails)
[Transbucket](http://www.transbucket.com) I can share an example directly.
The only thing you need to know is a "Pin" is a submission from a user
that represents a medical procedure. Other users want to see procedures of
the type they're considering or with the doctor they want or a few other
assorted criteria.

[Here](https://github.com/mooreniemi/Transbucket_Rails/blob/5661602eb25656a6618ec952ea0ecf38f4dc5b90/app/queries/pin_filter_query.rb)'s
where I started out, innocent enough, just a bit of branching:

```ruby
class PinFilterQuery
  attr_accessor :pins
  attr_accessor :procedures, :surgeons, :general

  def initialize(keywords)
    @procedures = keywords[:procedure]
    @surgeons = keywords[:surgeon]
    @general = format_scope(keywords.fetch(:scope,nil))
  end

  def filtered
    if surgeons.present?
      Rails.cache.fetch([surgeons].join(',')) do
        Pin.by_surgeon([surgeons])
      end
    elsif procedures.present?
      Rails.cache.fetch([procedures].join(',')) do
        Pin.by_procedure([procedures])
      end
    elsif general.present?
      args = general.join('.')
      Rails.cache.fetch(args) do
        Pin.instance_eval { eval args }
      end
    else
      Pin.none
    end
  end

  private

  def format_scope(scope)
    return if scope.nil?
    scope.collect!(&:parameterize).collect!(&:underscore).collect!(&:to_sym)
    scope
  end
end
```

I'd be surprised if you don't have a chunk of code like this somewhere in
an app you maintain or wrote. I could've moved towards a case statement,
but the net is about the same. I'd still be facing the same problem
I faced not long after I wrote the above: "hmm, I kinda want to filter on
surgeons AND procedures..."

This brings us up to our ["before"
shot](https://github.com/mooreniemi/Transbucket_Rails/blob/4600b8ccaed0c1e877d993db88685b87544cc68a/app/queries/pin_filter_query.rb):

```ruby
class PinFilterQuery
  attr_accessor :pins
  attr_accessor :procedures, :surgeons, :general

  def initialize(keywords)
    @procedures = keywords[:procedure]
    @surgeons = keywords[:surgeon]
    @general = format_scope(keywords.fetch(:scope,nil))
  end

  def filtered
    if surgeons.present? && procedures.present?
      if general.present?
        args = general.join('.')
        Rails.cache.fetch("very_specific:" + args + [surgeons + procedures].join(',')) do
          Pin.instance_eval { eval args }.by_surgeon([surgeons]).by_procedure([procedures])
        end
      else
        Rails.cache.fetch("surgeons_by_procedures:" + [surgeons + procedures].join(',')) do
          Pin.by_surgeon([surgeons]).by_procedure([procedures])
        end
      end
    elsif surgeons.present?
      Rails.cache.fetch("surgeons:" + [surgeons].join(',')) do
        Pin.by_surgeon([surgeons])
      end
    elsif procedures.present?
      Rails.cache.fetch("procedures:" + [procedures].join(',')) do
        Pin.by_procedure([procedures])
      end
    elsif general.present?
      args = general.join('.')
      Rails.cache.fetch("search_terms:" + args) do
        Pin.instance_eval { eval args }
      end
    else
      Pin.none
    end
  end

  private

  def format_scope(scope)
    return if scope.nil?
    scope.collect!(&:parameterize).collect!(&:underscore).collect!(&:to_sym)
    scope
  end
end
```

![grimace](/images/grimace.png)

You can see that everywhere I want to add another dimension by "and"-ing
filters, the nesting grows. There's a ton of repetition. It's not very
easy or fast to grok exactly what the separate dimensions even are here.
I knew this was bad, but I let it go in lieu of other priorities. When it
came time to add another thing to filter on (surgical complications)
I knew I couldn't let this get worse.

So let's work to diagnose how this got so bad in the first place. Well,
one observation to make is that there's really no need to be guarding on
the presence of parameters in this object! We shouldn't need to do:

```ruby
# [snip]
if surgeons.present?
  Pin.by_surgeon([surgeons])
elsif procedures.present?
  Pin.by_procedure([procedures])
# [snip]
```

Because we can move this down into the scopes themselves so that
they _still return a valid `Relation` even when nothing was put in_:

```ruby
def self.by_procedure(procedure)
  if procedure == [nil]
    where.not(procedure_id: procedure)
  else
    where(procedure_id: procedure)
  end
end
```

Once we've done that we can chain these suckers without fear!

```ruby
Pin.by_surgeon([surgeons]).
  by_procedure([procedures]).
  or_whatever([whatevers])
```

This is the general principle behind the entire refactor: we want to make
sure any individual method returns a value all the subsequent methods can
consume. For the scopes on the model, this means pushing the conditional
logic down into them but we can also do it in the calling object.

For general scopes (scopes without parameters), I used `instance_eval
{ eval args }`. This uses Ruby's own method dispatch and carries the risk
of malicious injection[^0]. Again, I needed to find a way to make that
a no-op when nothing was handed in to `args`. Well, it turns out you can
send a class name to itself and, voila, it'll return itself:

```ruby
String.instance_eval { eval 'String' }
=> String
String.instance_eval { eval 'self' } # also works
=> String
```

Again, you can see the trick here is that I'm using a conditional but I am
making sure to return something that can be an appropriate receiver of the
subsequent methods:

```ruby
args = general.present? ? general.join('.') : 'Pin'
```

Finally, I needed to handle the surgical complications. Complications I'd
implemented as tags using a popular gem. So I wanted to always be calling
`tagged_with`, but sometimes I wanted to not _actually_ filter by tags. I'm
using [acts_as_taggable](https://github.com/mbleigh/acts-as-taggable-on),
and it has this method:

```ruby
# Find users that has not been tagged with awesome or cool:
User.tagged_with(["awesome", "cool"], :exclude => true)
```

So I used a [sentinel value](https://en.wikipedia.org/wiki/Sentinel_value)
(just something that can [never itself be
a tag](https://en.wikipedia.org/wiki/Semipredicate_problem)) and again
guarded the construction of the params:

```ruby
complication_params.nil? ? [['SENTINEL'], exclude: true] : complication_params
```

Whew! Ok, now that I put the time in to make my method chain safe here's
[the "after"
shot](https://github.com/mooreniemi/Transbucket_Rails/blob/63f1361968f9ee29f932fbc45875b4df47513639/app/queries/pin_filter_query.rb):

```ruby
class PinFilterQuery
  attr_accessor :pins
  VALID_FILTERS = [:procedures, :surgeons, :general, :complications]
  PERMITTED_SCOPES = [:ftm, :mtf, :top, :bottom, :need_category]
  attr_reader(*VALID_FILTERS)

  def initialize(keywords)
    @procedures = keywords[:procedure]
    @surgeons = keywords[:surgeon]
    @complications = add_default(keywords[:complication])
    @general = format_scope(keywords.fetch(:scope,nil))
  end

  def filtered
    active_filters = []
    keywords = []
    VALID_FILTERS.each do |filter|
      next if send(filter).nil?
      keywords << send(filter)
      active_filters << filter
    end

    # this will make the instance_eval a no-op
    args = general.present? ? general.join('.') : 'Pin'
    Rails.cache.fetch(cache_key_for(active_filters, keywords)) do
      Pin.instance_eval { eval args }.
        tagged_with(*complications).
        by_procedure([procedures].flatten).
        by_surgeon([surgeons].flatten).
        recent
    end
  end

  private

  def cache_key_for(active_filters, keywords)
    "#{active_filters.zip(keywords)}"
  end

  def add_default(complication_params)
    complication_params.nil? ? [['SENTINEL'], exclude: true] : complication_params
  end

  def format_scope(scope)
    return if scope.nil?
    scope.collect!(&:parameterize).collect!(&:underscore).collect!(&:to_sym).
      collect! {|s| s if PERMITTED_SCOPES.include?(s) }
    scope
  end
end
```

The relevant portion of `#filtered` dropped from 27 LOC to 7, and the
amount of code we need to add to the body of this method for subsequent
scopes dropped from a quadratic factor to a linear factor. It's all
guaranteed to work no matter which methods receive parameters or not, but
don't take my word for it, here's the test [my fellow contributor on
Transbucket](https://github.com/corajr) wrote to prove it:

```ruby
it 'handles different combinations of query' do
  all_filters = {
    scope: 'top',
    surgeon: pin.surgeon.id.to_s,
    procedure: pin.procedure.id.to_s,
    complication: 'hematoma'
  }
  (1..all_filters.size).each do |size|
    all_filters.keys.combination(size).each do |keys|
      filter = {}
      keys.each { |k| filter[k] = [all_filters[k]] }
      expect(PinFilterQuery.new(filter).filtered.first).to eq(pin)
    end
  end
end
```

I don't want to claim this is _beautiful_ code, or amazing, or impressive.
I can, of course, in reviewing it now see things I'd improve
immediately.[^1] But it is a hell of a lot easier to work with. And in
every case the summary is moving away from a statement for control flow to
making everything an expression[^2] that is guaranteed to return a useful
value.

I suspect it has a formalistic underpinning: the power of not in
relational algebra paired with sentinel values? (Basically,
"not not everything == everything".) Using conditional
scaffolds around the parameters to make a kind of `Maybe` type? But even
without that formal aspect laid bare, I hope you can appreciate the
[difference in the "before" and "after"
shots](https://github.com/mooreniemi/Transbucket_Rails/commit/63f1361968f9ee29f932fbc45875b4df47513639).

[^0]: Like in this case, we want to make sure none of the `ActiveRecord` methods are available except scopes. Don't want folks using `delete_all` as a keyword!
[^1]: For instance, why didn't I move the cache key logic from 14-20 out entirely into a separate method to call inside `#filtered`?
[^2]: Not to be pedantic, but this is the _sweetspot_ of functional programming and one of its key benefits.
