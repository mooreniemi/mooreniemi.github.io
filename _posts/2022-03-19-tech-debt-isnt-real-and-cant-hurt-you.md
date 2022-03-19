---
layout: post
title: tech debt isnt real and cant hurt you
date: 2022-03-19 18:06 -0400
---

On every team I've worked on, including at Amazon, engineers bring up
"tech debt" as a concern. When I ask "what is tech debt" other engineers
give me different responses that I want to summarize here because
I despise the ubiquity and uselessness of this term.

# 5 theses on tech debt

1. Tech debt is the lack of technical _confidence_ in the system this
   moment. That's how engineers _feel_ using this term.
2. Lack of technical confidence (ignoring individual psychology), comes
   from a system being opaque and difficult to change and grow.
3. A system is opaque and difficult to change when ambiguous or
   contradictory requirements were not or could not be reconciled,
   yielding complexity to manage competing trade-offs.
4. You will never, **ever** work on a meaningful real-life system that
   does useful work with _zero_ "technical debt."
5. Tech debt can be prevented, and it is your fault it got this bad.

# how to minimize tech debt

Tech debt has its commonly recognized symptoms: high operational burden,
bugs, slow velocity, kvetching. These things can be measured. An immature
manager will focus on minimizing these metrics, generally by focusing on
slow velocity by "worrying" at their engineers excessively via progress
tracking meetings. The manager's confidence in the engineer is gone, the
engineer's confidence in themself leaves shortly thereafter; a death
spiral ensues. I can't fix that for you.

That said, I think there are some related causes and comorbidities that
get less air time and have more leverage.

1. You have little to no "test coverage." (Depending on your system, what
   a test is can look different, don't be so literal.) Tests are an
   overhead and I am not evangelical about them, but without them, the
   system is not exhaustively exercised except in its runtime environment
   where variable isolation is ~infinitely harder.
2. You have high friction with build tools, so it's hard to impossible for
   you to have a tight feedback loop. Making a tiny, incremental change to
   test your assumption takes 10 minutes to compile? You won't do it.
   You'll batch a set of assumptions together, and if it looks "fine"
   you'll ship not realizing one of the dozen is a rotten egg.
3. You have bad product to market fit in the first place. If the product
   doesn't have a strong fit, increasingly desperate and chaotic efforts
   to diversify the product to find a fit will be reflected directly in
   the code. I see engineers can be very naive about this. Sometimes
   things are on fire because people are being childish with matches and
   you should leave.
4. You have no metrics or don't know how and why to read them. Your system
   should have a "cockpit" type view for you potentially at multiple
   levels and you should know all the profiling and analysis tools for
   your area. You should also have touchpoints to the BI (business
   intelligence) tools and preferably all these tools should produce data
   you can _directly join on_ **and** _visualize_. Yes, EDA (exploratory
   data analysis) matters.

I would personally spurn "make a big catch all Epic for all the ops
issues," "just document it better," and "just spend more time on design"
as "solutions" to tech debt. For the first, that's called a garbage can
and your confidence will drop the more you see that can overflow. It's an
unbounded queue and will become infinite.

As for the other strategies, I value both practices highly and would make
you do them if I could, but I don't think they help with the root issue.
Why? With documentation, as a mechanism it is almost never in sync with
code, so it has an error rate first of translation and then, continuously,
of drift. With design, the best designs are _downstream of well defined
requirements and the ability to test your assumptions_, which are
themselves downstream of tight feedback loops.

If I could give you one piece of advice to increase your technical
confidence, it would simply be to minimize the friction of your feedback
loop in testing your assumptions.

Verify the ice is thick enough, and you will skate.
