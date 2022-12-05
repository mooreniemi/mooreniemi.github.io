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

![](images/lex-ann.png)

When we think about using these two different data structures for scoring,
we can see that in the scenario with embeddings, although we have
something "like" a dictionary, we have an added step. With an inverted
index, as soon as we have tokenized "cats and dogs" to ["cat", "dog"] we
can use the tokens as keys directly to the lists of document values, which
are called postings lists. With clustered embeddings, we first rank the
cluster prototypes, and for the top clusters, we rank their members. It's
sometimes confusing in practice to read documentation about HNSW, in part
because we can use it in more than one place. We can use it to make
ranking (searching) the blue box below fast (the cluster prototypes), or
we could even use it per each red box (the cluster members). Or if we have
no clusters at all, we can use it across all embeddings. In practice,
that's usually only done for very "small" sets of embeddings.

![](images/lex-ann-2.png)


### steep climbing: how do we make ann faster?


["Efficient and robust approximate nearest neighbor search using
Hierarchical Navigable Small World
graphs"](https://arxiv.org/abs/1603.09320) was initially published in
2016, with revisions up to 2018.

> Hierarchical NSW incrementally builds a multi-layer structure consisting from hierarchical set of proximity graphs
(layers) for nested subsets of the stored elements.
