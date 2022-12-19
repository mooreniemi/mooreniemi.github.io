---
layout: post
title: scenic tour of hnsw
date: 2022-12-04 22:21 -0500
---

[WIP Post - 12/4/22] HNSW, which stands for "hierarchical navigable small
world" (graphs), is one of the first data structures for which I read not
just its paper but the trail of papers backward in time until I arrived at
one with a name I already knew. I really enjoyed this tour, and so I'd
like to share it with you as I walked it.

### where i come from: inverted indices

I didn't start at zero when approaching the HNSW data structure. My
background meant I was quite familiar with inverted indices for search. So
let's start there as the initial setting. In "lexical" or "sparse" search,
documents are tokenized into terms and each term is a key in a dictionary
in which we keep some information about the term's importance in the
document. Historically, this information was used to calculate
[tf-idf](https://en.wikipedia.org/wiki/Tf%E2%80%93idf) or
[bm25](https://en.wikipedia.org/wiki/Okapi_BM25) ranking functions via
statistics of the underlying corpus, but it can also be importance
information calculated by a deep neural network, as is done in [deep
impact](https://arxiv.org/abs/2104.12016).

### a fork in the road, ann

To move to Approximate Nearest Neighbors (ANN), that is, the setting in
which HNSW is meaningful, we first make the great, beautiful leap into
space -- embedding space.

We need a model to encode the query and documents into vectors, rather
than tokenizing them into terms, and then we need a distance function to
measure the distance between these vectors. (Of course, we must have had
a distance function in mind when we embedded the vectors into "space" in
the first place.) If we like, we can do "exhaustive" search here, where
each document's vector is measured against our query vector, but typically
a further step is applied that clusters similar document vectors together,
averages those clusters, and organizes the average (sometimes called
"prototype" vector) and cluster pairs into something like a dictionary as
well. Clustering offers a trade-off: faster search, but lower recall,
since we won't visit every cluster on every search.

![](/images/lex-ann.png)

When we think about using these two different data structures for scoring,
we can see that in the scenario with embeddings, although we have
something "like" a dictionary, we have an added step. With an inverted
index, as soon as we have tokenized "cats and dogs" to ["cat", "dog"] we
can use the tokens as keys directly to the lists of document values, which
are called postings lists. With clustered embeddings, we first rank the
cluster prototypes, and only then, for the top clusters, we rank their
members. (The analogy I'd make here is to get to posting lists in ANN
can't be as cheap as it is for an inverted index.) It's sometimes
confusing in practice to read documentation about HNSW because we can use
it in more than one place. We can use it to make ranking (searching) the
blue box below fast (the cluster prototypes), or we could even use it per
each red box (the cluster members). This is why many `index_factory`
settings in Faiss _start_ with `HNSW`: they're laying out how the blue box
can be made quickly searchable. (Of course, if we have no clusters at all,
we can use HNSW across all the embeddings, which would just be each of the
red box rows concatenated next to each other. In practice, that's usually
only done for very "small" sets of embeddings.)

![](/images/lex-ann-2.png)


### steep climbing: how do we make ann faster?

Nearest neighbors implies a distance function on a space. As with most
problems, if we pre-calculate some information, we can navigate a space
more efficiently at "runtime." (Paying the cost in [update and memory to
speed up
read](https://stratos.seas.harvard.edu/files/stratos/files/rum.pdf).) The
analogy would be that you don't need to search every block for the White
House, since you know at least to start it needs to be somewhere in
Washington D.C.

The question is, what information do we need to "navigate" quickly? In
fact, it turns out the term "navigable" in "Hierarchical Navigable Small
World Graphs" actually has a precise meaning, as does the mysterious and
almost informal sounding "small world." In order, the title is actually
the reverse chronology of the concepts. Since graphs go back to 1736,
we'll skip ahead to the 90s and start with "small world," just pausing to
note the features of graphs needed to understand.

As a simplification, we'll put graphs on a spectrum. On one end, we have
random graphs. On the other end, regular graphs.

With a random graph edges are randomly assigned, meaning path lengths are
short, but also that it is likely nodes are direct neighbors. For
a regular graph, eg. for a lattice, vertices are laid out in grid with
edges in cardinal directions and long path length likely (think Manhattan
distance given the grid constraint). Note also regular sets hard limit on
neighbor list per node (all have same number).

![](/images/ann_graphs.png)

A classic math question, given _this_ and _that_, is there something _in
between_ in a precise and meaningful way? Yes! The small world!

![](/images/small_world.png)

1998: ["Collective dynamics of ‘small-world’
networks"](https://www.nature.com/articles/30918)

> ...these systems can be highly clustered, like regular lattices, yet
> have small characteristic path lengths, like random graphs.

I was really tickled to find [Steven
Strogatz](https://podcasts.apple.com/us/podcast/the-joy-of-x/id1495067186)
worked on this paper since I've read some of his books and listened to his
podcast.

Small worlds are a way to characterize a graph, “in which most nodes (1)
are not neighbors of one another, but the (2) neighbors of any given node
are likely to be neighbors of each other and (3) most nodes can be reached
from every other node by a small number of hops or steps.”

2013: ["Approximate nearest neighbor algorithm based on navigable small world
graphs"](https://publications.hse.ru/pubs/share/folder/x5p6h7thif/128296059.pdf)

> The navigable small world is created simply by keeping old Delaunay
> graph approximation links produced at the start of construction.

2016: ["Growing Homophilic Networks Are Natural Navigable Small
Worlds"](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0158162)

> Navigability, an ability to find a logarithmically short path between
> elements using only local information...

2016: ["Efficient and robust approximate nearest neighbor search using
Hierarchical Navigable Small World
graphs"](https://arxiv.org/abs/1603.09320) was initially published in
2016, with revisions up to 2018.

> Hierarchical NSW incrementally builds a multi-layer structure consisting
> from hierarchical set of proximity graphs (layers) for nested subsets of
> the stored elements.
