---
layout: post
title: a (very) scenic tour of hnsw
date: 2022-12-04 22:21 -0500
usemathjax: true
toc: true
---

[WIP Post - 12/31/22] HNSW, which stands for "hierarchical navigable small
world" (graphs), is one of the first data structures for which I read not
just its paper but the trail of papers backward in time until I arrived at
one with a name I already knew. It was, in a very real sense, the first
academic hike I've taken through literature. Because I was not formally
educated in math or computer science, I was intimidated by this kind of
reading for too long. I hope these informal notes encourage others to try
a hike of their own. (If you do, email me your "scenic tour," I want to
read it!)

### the structure of this tour

If you're familiar with the terrain, you should feel free to skip to
wherever is new for you.

I start with inverted indices to ground and contrast approximate nearest
neighbors search.

Then I work the name "HNSW" backwards, moving from 1998 through to 2016
when the main paper is published.

### where i come from: inverted indices

I didn't start at zero when approaching the HNSW data structure. My
background meant I was quite familiar with inverted indices for search.

So let's start there as the initial setting. In "lexical" or "sparse"
search, documents are tokenized into terms and each term is a key in
a dictionary in which we keep some information about the term's importance
in the document. Historically, this information was used to calculate
[tf-idf](https://en.wikipedia.org/wiki/Tf%E2%80%93idf) or
[bm25](https://en.wikipedia.org/wiki/Okapi_BM25) ranking functions via
statistics of the underlying corpus, but it can also be importance
information calculated by a deep neural network, as is done in [deep
impact](https://arxiv.org/abs/2104.12016).

### a horizon i wanted to go towards: ann

To move to [k Nearest Neighbors
(KNN)](https://www.youtube.com/watch?v=HVXime0nQeI&ab_channel=StatQuestwithJoshStarmer),
that is, the setting in which HNSW is a meaningful approximation (thus ANN
for _Approximate_ Nearest Neighbors), we first make the great, beautiful
leap into space -- embedding space.

We need a model to encode the query and documents into vectors, rather
than tokenizing them into terms, and then we need a distance function to
measure the distance between these vectors. (Of course, we must have had
a distance function in mind when we embedded the vectors into "space" in
the first place.) If we like, we can do "exhaustive" search here which is
exact, where each document's vector is measured against our query vector,
but at scale, approximations are used since distance calculations are
relatively expensive.

As with topic sharding in traditional search, a technique to trade recall
for latency is to cluster similar things together and route the query only
to the most "promising" clusters. Commonly we'll see
[k-means](https://www.youtube.com/watch?v=4b5d3muPQmA&ab_channel=StatQuestwithJoshStarmer)
used for document vectors, then we average each of those clusters to
produce averaged "prototype" vectors, which we use much like terms in
something like an inverted index. Again, note that clustering offers
a trade-off away from exact search: we get faster search, but lower
recall, since we won't visit every cluster on every search. In the below,
the "blue box" for ANN is that set of "prototype" vectors representing
each of the clusters.

![](/images/lex-ann.png)

When we think about using a clustered nearest neighbors index versus an
inverted index for scoring scoring, we can see that in the scenario with
embeddings, although we have something "like" a dictionary, we have an
added step because we don't truly have the $$O(1)$$ key lookups. With an
inverted index, as soon as we have tokenized `"cats and dogs"` to
`["cat", "dog"]` we can use the tokens as keys directly to the lists of
document values, which are called posting lists. With clustered
embeddings, we first rank the cluster prototypes, and only then, for the
top clusters, we rank their members.

![](/images/lex-ann-2.png)

We ultimately want some balance between the length of the "blue box" of
cluster prototypes and the average (hopefully fairly even) length of the
"red boxes" containing the members of the cluster. Because we have two
places where we could "speed up" distance calculations, I find it's
sometimes confusing in practice to read documentation about HNSW. We can
use it to make ranking (searching) the blue box below fast (the cluster
prototypes), or we could even use it per each red box (the cluster
members). Or if we have no clusters at all, we can use HNSW across _all_
the embeddings, which would just be each of the red box rows concatenated
next to each other. In practice, no clustering is usually only done for
very "small" sets of embeddings, below say 10 million or so, which is
actually the most common setup I've seen overall.

This is why many `index_factory` settings in Faiss _start_ with `HNSW`:
they're laying out how the blue box can be made quickly searchable. Often
deployments I've seen prefer a "long" blue box with an HNSW data structure
to navigate it, a fairly small k representing the number of top clusters,
and then each cluster in the k set is scored exactly and exhaustively.

### steep climbing: how do we make ann faster?

Nearest neighbors implies a distance function on a space. As with most
traversal problems, if we pre-calculate ("index") some information about
our "space," we can navigate a it more efficiently at search "runtime."
(Paying the cost in [update and memory to speed up
read](https://stratos.seas.harvard.edu/files/stratos/files/rum.pdf).) The
analogy would be that you don't need to search every block for the Golden
Gate Bridge, since you know at least to start it needs to be somewhere in
California.

The question is, what information do we need to "navigate" quickly? In
fact, it turns out the term "navigable" in "Hierarchical Navigable Small
World Graphs" actually has a precise meaning, as does the mysterious and
almost informal sounding "small world." In order, the title is actually
the reverse chronology of the concepts. Since graphs go back to 1736,
we'll skip ahead to the 90s and start with "small world," just pausing to
note the features of graphs needed to understand.

### just a glancing view: the concrete origins of this problem

Part of what's fun across these papers, which I only touch on lightly here
and not in the rest of this tour, is that many of them refer to a real
life concrete phenomenon that appeared to be true: [the six degrees of
separation experiments run by Milgram in the
1960s](https://en.wikipedia.org/wiki/Six_degrees_of_separation#Small_world).
Which were actually proceeded by Manfred Kochen's Monte Carlo simulations
that predicted three. Why was it possible that people using only "local"
information (who they knew directly) were able to get a message through
across a "global" space by a relatively short path a non-negligible amount
of the time?

### graphs, at sea level

As a simplification, we'll put graphs on a spectrum. On one end, we have
random graphs. On the other end, regular graphs. Beyond the intuitive
naming, the graph types can be categorized with a clustering coefficient:

$$C(v)={e(v) \over deg(v)(deg(v)-1)/2}$$

> where $$e(v)$$ denotes the number of edges between the vertices in the
> $$v$$’s neighbourhood. (The degree [$$deg(v)$$] of a vertex is defined
> as the number of vertexes in its neighbourhood.)

In other words, the clustering coefficient is the number of edges of
a vertex divided roughly by the number of vertices immediately connected
to it (ie. $$v$$'s neighborhood). If most of the things connected to you
are near you, you're highly clustered. Regular graphs have this property,
while random graphs do not. But at the same time, a regular graph has
a high "diameter" (and a random graph low) because the maximum number of
edges in the shortest path connecting the vertices for any vertex is high
(or low, if you're random).

![](/images/ann_graphs.png)

A classic math question, given _this_ and _that_, is there something _in
between_ in a precise and meaningful way? Yes! The small world graph!

### 1/3rd of the way up, the "small world"

![](/images/small_world.png)

[Image from these excellent
slides](https://www.kth.se/social/upload/514c7450f276547cb33a1992/2-kleinberg.pdf)

As a sidebar, I was really tickled to find [Steven
Strogatz](https://podcasts.apple.com/us/podcast/the-joy-of-x/id1495067186)
worked on this paper since I've read some of his books and listened to his
podcast. It's a bit like seeing a celebrity in your local coffee shop.

#### 1998: ["Collective dynamics of ‘small-world’ networks"](https://www.nature.com/articles/30918)

> ...these systems can be highly clustered, like regular lattices, yet
> have small characteristic path lengths, like random graphs.

That is, it turns out with a small world graph you can get a high
clustering coefficient but a low diameter, since they have these
properties:

> most nodes (1) are not neighbors of one another, but the (2) neighbors
> of any given node are likely to be neighbors of each other and (3) most
> nodes can be reached from every other node by a small number of hops or
> steps.

How? Simply by "rewiring" some edges at random! More formally:

> With probability $$p$$, we reconnect this edge to a vertex chosen
> uniformly at random over the entire ring, with duplicate edges
> forbidden; otherwise we leave the edge in place.

Because they can parameterize probability $$p$$ between 0 for regular and
1 for random, they've found a way to make _this_ or _that_ into
a continuous space where there's a meaningful "between" between.

That parameterization can then allow you to graph the clustering
coefficient ($$C(p)$$) and path length ($$L(p)$$), and their relationship,
too:

![](/images/c_v_l.png)

I always love seeing a relatively simple model of a key trade-off. Here
it's the trade-off between clustering and path length.

### 2/3rds of the way up, navigable

Before we discuss "navigable" directly, first let's stop at a fun "cabin"
on our climb: P2P networks.

With peer-to-peer networks, definitionally, we need decentralized search.
With decentralization, greedy approaches are great because, implicitly,
they're using _local_ ("who are my immediate peers") not global ("who are
all the possible peers") information -- whereas centralization requires
global information. It also means:

> if one had full global knowledge of the local and long-range contacts of
> all nodes in the network, the shortest chain between two nodes could be
> computed simply by breadth-first search.

#### 2000 (May): ["The Small-World Phenomenon: An Algorithmic Perspective"](https://www.stat.berkeley.edu/~aldous/Networks/swn-1.pdf)

> Although recently proposed network models are rich in short paths, we
> prove that no decentralized algorithm, operating with local information
> only, can construct short paths in these networks with non-negligible
> probability. We then define an infinite family of network models that
> naturally generalizes the Watts-Strogatz model, and show that for one of
> these models, there is a decentralized algorithm capable of finding
> short paths with high probability.

Really read the above, it is saying: "lots of people have tried and
they're definitely wrong because I found literally the only right way to
do this." Math is just plain different from other kinds of writing.

Kleinberg's paper was cited in the work that finally began applying these
properties to nearest neighbor search with the note:

> The first algorithmic navigation model with a local greedy routing was
> proposed by J. Kleinberg [but] the exact mechanism that is responsible
> for formation of navigation properties in real-life networks remained
> unknown.

Funny enough, the word "greedy" doesn't even appear in his paper. And
neither does "navigable", only "navigation." But that summer, Kleinberg
goes on to write:

#### 2000 (August): ["Navigation in a small world"](https://www.nature.com/articles/35022643)

Where he says directly,

> These results indicate that efficient navigability is a fundamental
> property of only some small-world structures.

In other words, he defines "navigability" as when a "decentralized
algorithm can achieve a delivery time [of messages across the network]
bounded by any polynomial in logN."

And he comes up with a model that identifies when small worlds will be
navigable:

> A characteristic feature of small-world networks is that their diameter
> is exponentially smaller than their size, being bounded by a polynomial
> in logN, where N is the number of nodes. In other words, there is always
> a very short path between any two nodes. This does not imply, however,
> that a decentralized algorithm will be able to discover such short
> paths. My central finding is that there is in fact a unique value of the
> exponent $$\alpha$$ at which this is possible.

$$\alpha$$ is the only parameter for his model, which is again
a probability like the $$p$$ Strogatz used above.

> Long-range connections are added to a two-dimensional lattice controlled
> by a clustering exponent, $$\alpha$$, that determines the probability of
> a connection between two nodes as a function of their lattice distance.

He proved for a 2 dimensional lattice $$\alpha=2$$, and for d dimensional,
$$\alpha=d$$.

#### 2007: ["Peer to peer multidimensional overlays: approximating complex structures"](https://hal.inria.fr/inria-00164667/file/RR-6248.pdf)

I actually find the description of this paper by the next paper somewhat
more helpful for contextualizing it than reading the original paper:

> The first structure for solving ANN in $$E^d$$ with topology of small
> world networks is Raynet. It is an extension of earlier work by the same
> authors Voronet, which solved the problem of the exact NN in $$E^2$$.
> Originally Voronet was envisioned as a p2p network, where every node has
> coordinates in $$E^2$$. In Raynet every node has the coordinates in
> $$E^d$$. The system supports two levels of links - short for correct
> work of the greedy search algorithm and long - for logarithmic search.
> Short links correspond to edges of Delaunay graph, i.e. each object has
> references to objects that are neighbors of its Voronoi region. The main
> difference of Raynet from Voronet is that in Raynet every object doesn’t
> know all of its Voronoi neighbors, i.e. Raynet obtains neighborhood with
> approximately using the Monte Carlo method.

#### 2011: ["Approximate Nearest Neighbor Search Small World Approach"](https://www.iiis.org/CDs2011/CD2011IDI/ICTA_2011/PapersPdf/CT175ON.pdf)

> Raynet is the closest work to ours in terms of general concept. But
> unlike Raynet, we propose a structure that works with objects from
> arbitrary metric spaces.

#### 2013: ["Approximate nearest neighbor algorithm based on navigable small world graphs"](https://publications.hse.ru/pubs/share/folder/x5p6h7thif/128296059.pdf)

> The navigable small world is created simply by keeping old Delaunay
> graph approximation links produced at the start of construction.

#### 2016: ["Growing Homophilic Networks Are Natural Navigable Small Worlds"](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0158162)

> Navigability, an ability to find a logarithmically short path between
> elements using only local information...

### the summit: adding hierarchy

#### 2016: ["Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs"](https://arxiv.org/abs/1603.09320)

(Note: initially published in 2016, with revisions up to 2018.)

> Hierarchical NSW incrementally builds a multi-layer structure consisting
> from hierarchical set of proximity graphs (layers) for nested subsets of
> the stored elements.
