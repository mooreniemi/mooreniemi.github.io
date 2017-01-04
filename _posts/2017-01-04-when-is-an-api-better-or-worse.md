---
layout: post
title: when is an API better or worse?
category: apis
---

If an API is not simply a data persistence surface or a list of functions,
what is it? For me, the underutilized model is the API as a [state
machine](http://blog.markshead.com/869/state-machines-computer-science/)
or graph[^graph].

> In simple terms, a state machine will read a series of inputs.  When it
> reads an input it will switch to a different state.  Each state
> specifies which state to switch for a given input.

Based on this description, you may be asking: "but a web API must be
stateless, how can a state machine describe it?"

A state machine has a set of states and a set of state transitions. In
graph terms, these are nodes and edges. In web terms, these are resources
and controls (links). There's really no contradiction in using this
depiction to model our API. Both the server and client can be maintain
their own independent state (the server likely managing some storage, and
the client rendering a page). They only transfer state by representations
of the state. That's REST.

That's all very nice, but what do we gain by thinking about the API as
a state machine that we can't get out of it from our other models?

I'm going to extend an example I saw used by [API
Fortress](http://apifortress.com/) during
[APIStrat2016](boston2016.apistrat.com/schedule/). It's based on
a brilliant insight about Etsy.

![etsy's navigation](/images/etsy_navigation.gif)

Etsy's navigation is based on categories. This makes a lot of sense, as it
matches user intent. But what if a product has no category? It's
effectively unreachable. For API Fortress, their point is that the data
integrity of the API is bad. This isn't wrong. But our state machine model
of APIs can take us even further in diagnosing what is "bad" about this,
not just what is invalid.

Here's a simplification of Etsy's API as a state machine.

![etsy's graph](/images/etsy_graph.svg)

For every edge, I've annotated a link relation on the state transition
from state to state. States, or nodes, may or may not be full page
reloads. They may or may not even be new requests. If we use transclusion,
this entire graph could be loaded at the initial page load. For simplicity
though, let's say in this case every edge represents a necessary user
interaction like a click.[^hover]

We can see by this graph how many interactions it takes and what paths
actually exist, to get to the revenue generating target of the API. Armed
with this, we can compare 2 different additions to our API:

![bad graph](/images/bad_etsy_graph.svg) ![good graph](/images/good_etsy_graph.svg)

In the first graph, we try to fix our data problem up in our API by making
a new user path to products not dependent on categories. We imagine we
have a way of categorizing vendors, such that we can show collections of
vendors and then their products. The new link from the homepage is the
dashed red line. The solid red lines are existing links we're now making
use of.

In the second graph, we instead accept products must be categorized or
have a default category of "uncategorized". This solves our original
problem since it means all products will be reachable. This is strictly
better than the vendor solution in the previous graph, because we can
compare the edge length; the new path has length 6 versus our current
ideal case (when a product has a category) of 4.

But then we decide, heck, can we do better than our ideal case and create
a new optimal path? So we choose some "top" vendors and then make sure
each of those vendors likewise has some "top" products. By doing this,
we've reduced the edge length in our ideal case to 3!

This doesn't render the client obsolete though: it's still up to them
whether they will make edges "hard" (require a click) or "soft" (require
a hover). They may also decide to not render an edge at all, or to
forcibly create some new edges that aren't explicitly supported on the
server. Based on what they decide, we need to make sure our API can either
embed all content in a single representation or just effectively link to
further representations.

Of course, if _we_ can eyeball such a graph, then certainly a machine
should be able to traverse it for us and do the calculations. To make this
possible though, we need to make sure some version of our
representations[^version] annotates these state transitions such that we
can follow the edges of our program graph. Consider the below two
representations:

```json
{
  "name": "item",
  "actions": [
    {
      "name": "add-item"
    }
  ]
}
```
```json
{
  "name": "add-item",
  "actions": [
    {
      "name": "purchase"
    }
  ]
}
```

A machine could easily reassemble these fragments into the following
graph: item → add-item → purchase. Using that graph we can calculate by
edges that at this state in the API (viewing an item) it takes 2 clicks to
reach our revenue target! Based on this, we might decide to implement
a featured product on our homepage.

We can also get some nice benefits:

* We can easily detect "dead" unreachable nodes.
* We don't have to write a separate specification for our API. Our API
   representation set _is_ the specification. From just those
   representations we gain documentation, sure, but with an execution
   graph, we could be generating nearly a whole application ready for use.
   (It'd just be missing your application logic!)
* It forces some size constraints. [Graphs get really complex, really
   fast.](https://i.stack.imgur.com/JxZGM.png) Without some restrictions
   to insist that, say, some subgraphs can only be entered through one
   node (so that you could consider subgraphs semi-independently) you'd
   quickly get overwhelmed. I think this is a positive, but it's not
   unarguable.
* Because race conditions are so often a matter of sequence, forcing
   a common sequence through business processes can help keep your app
   safe.

I think this is a really useful tool in conceiving of and building APIs.
But it's largely a process change at first, not a technology one. You have
to model your API as a graph with the help of media formats (or
conventions in your representations) that make that graph machine and
human traversable. I'll explore how to do that in a future post.

---

[^graph]: By graph, I _do not_ mean object graphs. Objects in GraphQL or other graph APIs aren't even objects in the object oriented sense: they don't have methods or hide implementation. They're just instances of values in a schema.
[^hover]: In reality, especially with a transcluded page, we could have other more passive interactions like hovering.
[^version]: Clients will often want to receive the most abbreviated or concise version of the representation possible. We should still endeavor to make that optional for them.
