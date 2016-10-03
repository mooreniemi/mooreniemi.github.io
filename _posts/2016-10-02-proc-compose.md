---
layout: post
title: Proc#compose
categories: ruby
---

After my [combinators
post](http://mooreniemi.github.io/2016/09/18/combinators.html) I was
curious why chaining `map` performs so much better than the composition
operator I wrote in Ruby. I thought to myself, maybe I just need to write
it in C? So, I wrote another C extension:
[Proc#compose](https://github.com/mooreniemi/proc_compose).

As I did last time, I started out with just a [basic
experiment](https://github.com/mooreniemi/compose) then extracted it into
a gem. The experiment includes some information about the main gem I found
offering `compose`: [funkify](https://github.com/banister/funkify). (It is
much slower than the Ruby version I wrote, but it's not a totally fair
comparison.)

Anyway, the C extension's performance is better than its Ruby
counterpart, but still not as good as just chaining:

![proc#compose](/images/proc_compose_performance.gif)

Unfortunately, I haven't yet answered my own question as to why it still
performs better to just chain `map` rather than use `compose`. If/when
I figure it out, I'll post an update. If you're curious as well, a good
place to start is my [comparison
script](https://github.com/mooreniemi/compose/blob/master/compose_performance.rb).
