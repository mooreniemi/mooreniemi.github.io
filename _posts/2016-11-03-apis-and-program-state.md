---
layout: post
title: apis and program state
categories: apis web
---

I'm still at APIStrat. I attended the hypermedia breakout, and was happy to hear multiple people talk about APIs as graphs. "Hypermedia vs Graphs" was probably my favorite talk of today, and was the final one of that session. I rambled myself in my previous blog about how strange it was to contrast GraphQL and REST as being at odds, and this talk helped crystalize just how unnecessary that is as an attitude.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">My <a href="https://twitter.com/apistrat">@apistrat</a> talk from today &quot;Hypermedia vs Graphs: Best buddies or the next API battleground?&quot; |  <a href="https://t.co/xqpQKKCvTa">https://t.co/xqpQKKCvTa</a></p>&mdash; Gareth Jones (@garethj_msft) <a href="https://twitter.com/garethj_msft/status/794299567923662848">November 3, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

I'd only quibble with the characterization by Jones that hypermedia is waiting to happen. In fact, if you acknowledge that APIs that serve HTML are APIs, then hypermedia is as popular as it's always been: a lot of people write web pages with the canonical hypermedia affordances--those found in HTML! What still hasn't happened, though, is for those serving JSON to want to _use_ any hypermedia affordances. Well, more precisely, client developers still don't find the use value in affordances on JSON representations. Instead, the new hotness is Facebook's GraphQL. Why'd it turn out that way? I think Jones was very wise to ask "what problems were we trying to solve with hypermedia, and what problems were developers generally trying to solve?"

Although Jones' talk focused on distinguishing graph apis and hypermedia apis by their binding time (statically bound vs dynamically bound), I think there's another way to characterize it. GraphQL and similar efforts are way to make it easy to customize representations of resource state. Have a data model in your server's application that the client wants to implement somewhat differently? Then let them query the object however they want[^cache], and use those pieces to create their own object. I wonder if this has some analogy in the translation layer up from normalized database relations to objects (the [object relational impedance mismatch](http://www.agiledata.org/essays/impedanceMismatch.html) is the result).

Well, if graph apis are for expressing resource state in the way most flexible for clients, what is hypermedia's value prop? To my mind, it's that it's best at expressing application state.

What is resource state, and what is application state, and what's the difference? And why is the current community of frontend developers asking for more of the former than the latter? I think that last question actually really helps us understand the first one. Client developers, in an era of thick clients, are writing a _lot_ of application logic on their end. If you're writing application logic, you don't really want to integrate with other application logic; for the most part, people want to integrate with the type of integration layer that is most familiar to almost everyone. Persistent storage. Maybe it seems like I'm poking at our[^thankfully] limits, and well, I kinda am. But I think we have an opportunity for change soon.

Why? Because the web is reaching a maturity level where something like "serverless"[^serverless] is possible. [Eric Evans](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215) talks about [generic subdomains](http://blog.jonathanoliver.com/ddd-strategic-design-core-supporting-and-generic-subdomains/) in [DDD](http://domainlanguage.com/ddd/). A common one is authentication: do you really need to reimplement that? Is that a unique value prop of your company? Probably not. Now with identity providers via OAuth you can totally unburden yourself of that generic subdomain. So at very least you're already likely using resource state from hosts you don't manage. But why only treat other APIs like foreign databases? Why think about resources as models?

Right now, the only alternative in our industry's imagination is to treat other APIs like RPC namespaces. I won't knock that: that can be awesome. (I love functions--I just distrust putting the network between them.) But the alternative is to provide a representation of application state, instead. My way of describing application state is basically: in program state, what can the user do right now? (What might some viewer of this representation want to use the representation for?) But let me give you [RESTful Web API's](http://shop.oreilly.com/product/0636920028468.do) blurb too:

> Application state is kept on the client, but the server can manipulate it by sending representations—HTML documents, in this case—that describe the possible state transitions. Resource state is kept on the server, but the client can manipulate it by sending the server a representation—an HTML form submission, in this case—describing the desired new state.

Application state is the collection of state transitions a server will respect as of the moment it generated that contract. But in hypermedia's case not just the functions. You can also send them templates for formatting responses, or even use previous requests to give them entirely formed subsequent requests. (After all, what do you think a link or a form is?) In a sense, you're sending them the functions, and hopefully also the documentation around the functions, _and_ some default parameters (or contextually customized parameters).

So I'd like to engage you in a thought experiment, because it's one I'm entertaining myself. What if instead of thinking of APIs as wrappers around persistent data, we thought of them as discrete steps in workflows? What if rather than querying for an object, you were querying for a state in a state machine? How differently would you build or consume APIs? Would the resource-centric noun-y graph api approaches of today's moment suffice?

In an increasingly mature age of the web, will clients begin demanding more of their APIs than exposing data? Will we finally want to expose workflows? Until and unless **clients** want that, hypermedia will remain a niche area of research.


[^cache]: But I was left wondering this: how the heck do you cache all this?
[^thankfully]: That said, I consider it some evidence of rationality in our industry that totally transparent REST adapters of relational data haven't caught on in a big way. The server hasn't thinned down to nothing yet.
[^serverless]: Just like that old joke about the cloud, there is no serverless: just other people's servers.
