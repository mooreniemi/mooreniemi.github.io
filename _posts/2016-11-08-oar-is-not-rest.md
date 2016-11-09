---
layout: post
title: OAR is not REST
categories: rest apis
---

I hate scare quotes. But I've used them a few times describing 'REST'. I'm not really driving for the "world's most boring pedant" lifestyle. So let me try coining some other acronym for this pattern: OAR. Object as resource.

You've doubtless seen the OAR pattern out in the wild. I'd define its constraints this way: 1. a resource maps 1:1(-ish) to an object in your server's application logic, 2. uris and http methods are given conventions mapping to CRUD operations. A soft 3rd constraint I've seen is returning JSON, though I imagine some folks are still returning XML. When people describe an architecture as 'RESTful' what they're always describing is OAR. Rails[^rails] epitomizes OAR to me.

OAR exposes data as a surface to clients. Clients can't call methods on the data that has been sent over to them. These objects are not true objects anymore, because they can't encompass data _and_ behavior anymore. As soon as we serialize them for the wire, they are frozen in JSON like brittle snowflakes. Almost immediately, the object schema on the client and the server diverge and skew away from each other. This divergence chafes, and boils appear: ad hoc endpoints on the server, byzantine property checks littered around the client... One solution is a [BFF](http://samnewman.io/patterns/architectural/bff/). Another, I think, is GraphQL.

Yet another is, well, REST. But you might not actually like or want REST. REST explicitly involves a representation of state _and_ state changes. Here's Fielding on hypertext, _the_ original media type of REST:

> When I say hypertext, I mean the simultaneous presentation of information and controls such that the information becomes the affordance through which the user (or automaton) obtains choices and selects actions.

RPC just presents controls. OAR just presents information. REST presents _both_.

Perhaps unsurprisingly, every non-trivial OAR API I've worked with has **had** to implement some RPC, because it is never enough to _just_ give information. Sometimes you need to give controls. This makes me think that most apps likely need as big a toolbox as the web itself needed. And the web was 3 technologies: http, urls, and html (hypertext).

[^rails]: Rails might be the worst, because it encourages table relations mapped to models. This leads people to exposing their database schema as their API.
