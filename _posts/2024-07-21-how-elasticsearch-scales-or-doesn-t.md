---
layout: post
title: how elasticsearch scales (or doesn't)
usemathjax: true
date: 2024-07-21 22:48 -0400
---

If you Google ["adding replicas can't improve search latency elasticsearch"](https://www.google.com/search?q=adding+replicas+can%27t+improve+search+latency+elasticsearch&oq=adding+replicas+can%27t+improve+search+latency+elasticsearch&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTIHCAEQIRigATIHCAIQIRigATIHCAMQIRigATIHCAQQIRigATIHCAUQIRigAdIBCDY2NjFqMGoxqAIAsAIA&sourceid=chrome&ie=UTF-8), you'll find [Distinguished Software Engineer Adrien Grand](https://discuss.elastic.co/t/getting-worse-search-performance-with-a-replica-shard/60880/3) saying something that should be true:

> **Replicas do not help with latency,** they can only help with throughput by replicating the data on more nodes.

For a p0 baseline latency (that is, latency under no overlapping requests), it is true that adding replica shards can't improve your search time. But let's say we have compute capacity available (not yet at significant cpu utilization), and we have enough load where it's possible more than one request might overlap, will adding another replica improve latency?

In Elasticsearch? Yes!

### modeling performance

Elasticsearch is a sharded search engine. You choose the number of primary shards at the cluster setup time and (with some caveats) can not change the primary shard count without reindexing. Your unit of parallelism is your number of primary shards. That is, if you have a corpus of 100 documents, and you shard it into 10, your system can go no faster than searching 10 documents. This is the sense in which Adrien's comment holds true - it's an instantiation of Amdahl's Law:

$$
S = \frac{1}{(1 - P) + \frac{P}{N}}
$$

Where:
- $$S$$ is the speedup of the system.
- $$P$$ is the proportion of the system that can be parallelized.
- $$1 - P$$ is the proportion of the system that remains serial.
- $$N$$ is the number of processors (or parallel units).

This law is saying:

$$
S = \frac{\text{Execution Time with 1 Processor}}{\text{Execution Time with } N \text{ Processors}}
$$

Which naively we'd just put down as:

$$
S = \frac{\text{Execution Time with 1 Shard}}{\text{Execution Time with } N \text{ Shards}}
$$

But that is not the entire model of performance: what about the potential $$1-P$$? 

Even without any formalism, you should be suspicious that something is missing, because if having more parallelism is simply better, you'd want to shard as small as possible, but [you're not advised to do so](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/size-your-shards.html#shard-count-recommendation) and you'd find performance sufffers. Why? Because **something must aggregate the outputs of the shards** for your final result set. The more shards, the more work the aggregator must do. (Think of the limiting case, if each document was in a single shard, the coordinating node would still need to rerank all of the documents.)

How do we express this balance?

This is a simple model, better described [in these excellent slides](https://speakerdeck.com/emfree/queueing-theory?slide=73), here translated to humble Python:

```python
agg_time = lambda shards: results * shards
scan_time = lambda shards: documents / shards
total = lambda shards: agg_time(shards) + scan_time(shards)
```

Does `agg_time` dominate `scan_time` as `shards` changes?

![](/images/star_time.png)

Yes. Searching over shards in parallel is faster than searching the entire corpus sequentially, but as we shard further and further aggregation costs dominate.

We can also see throughput if we do `1 / total_time`[^0]:

![](/images/star_thrpt.png)

Of course the times plotted here are not exact in any sense, they're not grounded on real measurements. 

They just show that **given this topology**, this model predicts there is an optimal shard size beyond which further sharding slows down your system because each search request is really two pieces; one of the two pieces is an aggregation over a star graph. Just glance at the below: every incoming edge requires an aggregation. 

![](/images//star_graph.png)

When Elasticsearch receives a request at a [coordinating node](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#coordinating-node), this coordinating node is responsible for doing all aggregation. It forms a star graph with all other shards, typically with itself usually also being a shard.[^1] 

Admittedly, these aggregations may be cheap theoretically, since you should just be merging heaps, but in practice you're also paying for sending payloads over the network and the variability that entails[^2]. I've measured significant impacts to latency.

But this is where the oddity of the shared role comes in to disrupt our expectations, because adding replica nodes means adding coordinating nodes, **adding replica nodes also increases the capacity of your aggregators.** This extra capacity reduces utilization, and the [reduced utilization lowers latency](https://erikbern.com/2018/03/27/waiting-time-load-factor-and-queueing-theory.html). Remember, we said we have _compute_ capacity available - it's not yet at significant cpu utilization, and we're in reality, where most non-trivial workloads include overlapping work units.

### what other topologies are possible?

Of course, adding more aggregators can only help so much, because their work is growing with each added shard. And the point of Elasticsearch is to add more document volume and scale elastically, right? Alright, then how can we make ElasticElasticsearch? (Elastic$$^2$$search? Actually $$\log_2$$search would be even better.)

The answer is actually pretty simple. We need to somehow **shard the aggregators**. Hmm. How would that work? How do we make each aggregator responsible for $$\frac{\text{Total Aggregation Work Needed}}{\text{One Unit of Aggregation}}$$?

We need to actually change the **structure**, the __topology__ of the cluster.

Here's 4 different cluster arrangements. The first is our familiar star graph. Then we add aggregator nodes which serve the function of splitting the total aggregation work.

![](/images/topologies.png)

You can see there are fewer red aggregator nodes the higher our degree or fan-out. Obviously, the opposite is also true -- increasing the parallelism you dedicate reaches an effective max in fanning out as a binary tree. You can probably sense I am already setting up another trade-off, but before we go there, let's confirm.

Is this better than just using the star graph? Yes!

![](/images/cluster_comparison.png)

```python
agg_time_two = lambda shards: results * math.log(shards, fanout)
total_two = lambda shards: agg_time_two(shards) + scan_time(shards)
```

Now what's the trade-off? Cost (as in money) of the additional capacity for aggregators. The more aggregators, the more you're paying. Thus in practice you aim for the appropriate trade-off between dollars and latency for your use case. How many aggregators do you need for the best latency (and worst cost)? Number of leaf nodes (data shards) - 1, or in other words, and not very surprisingly if you think about it, you need almost $$2N\alpha$$ the capacity, where $$N$$ is your number of data shards, and $$\alpha$$ is a factor representing your $$\text{Aggregation} \propto \text{Scan}$$ compute.


---

[^0]: Why do we say we can improve throughput but not latency? Throughput is really $$T = \frac{\text{Concurrent Work Units}}{\text{Latency per Work Unit}}$$, where we typically use 1 as a numerator for simplicity (assume a single processor can process a single unit at a time). So if our latency per work unit can't improve, all we can do is add capacity. Of course, the more subtle argument is that __any queuing at all__ increases latency non-linearly -- so we always need to run at under full capacity utilization for best latency -- is not obvious from this formula.
[^1]: It's possible to make nodes coordination nodes only but in practice I've actually never seen deployments where coordinating nodes are not also data nodes. Further, I think it may be possible to set up coordinating-only nodes to talk to other coordinating-only nodes and finally talk to data-only nodes (forming a tree with fan-out), but have not seen this in the wild. If you've seen it, email me!
[^2]: And there, if you were wise, you'd want something like [Power-of-Two-Choices](https://web.stanford.edu/class/cs265/Lectures/Lecture6/l6.pdf) to offset.