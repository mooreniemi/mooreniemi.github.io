---
layout: post
title: rum in rust
date: 2022-12-31 13:30 -0500
---

Whenever possible, it's nice to know the theoretical limits of what you're
doing. As I've spent the last couple years helping write a search engine
from scratch, access methods on data have been top of mind for me, and so
I intuited the [RUM
Conjecture](https://stratos.seas.harvard.edu/files/stratos/files/rum.pdf),
as probably many programmers have, before I found out it had a name.

> **The RUM Conjecture.** An access method that can set an upper
bound for two out of the read, update, and memory overheads, also
sets a lower bound for the third overhead.

In other words, choose 2 of 3: read speed, update speed, and memory.

The paper gives some simple ratios to measure Overhead:

> Read Overhead (RO). The RO of an access method is given by the data
> accesses to auxiliary data when retrieving the necessary base data. We
> refer to RO as the read amplification: the ratio between the total
> amount of data read including auxiliary and base data, divided by the
> amount of retrieved data.

> Update Overhead (UO). The UO is given by the amount of updates applied
> to the auxiliary data in addition to the updates to the main data. We
> refer to UO as the write amplification: the ratio between the size of
> the physical updates performed for one logical update, divided by the
> size of the logical update.

> Memory Overhead (MO). The MO is the space overhead induced by storing
> auxiliary data. We refer to MO as the space amplification, defined as
> the ratio between the space utilized for auxiliary and base data,
> divided by the space utilized for base data.

For example, when thinking about how a search engine is updated, you have
much more complexity than a key-value store. (Elasticsearch alludes to
this in their [index splitting
documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-split-index.html#incremental-resharding).)
It's the case that a single document update will require updating many
different posting lists (one for each of its terms), and those posting
lists are now commonly stored as compressed blocks, which have to be
decompressed and recompressed to change. So the Update Overhead or write
amplification of changing a single document could be seen as all the
physical updates required across these posting lists divided by one (since
the document is the logical unit). Through the RUM lens, search indexing
is about trading Update Overhead for faster reads.

While thinking about how best to make search data mutable in Rust, I came
across another good example of a RUM dilemma, which is common to any
concurrent read/update situations. I was playing with using
[`notify`](https://crates.io/crates/notify) crate to [watch the file
system and reload some search data in
a server](https://github.com/mooreniemi/notify/blob/segs/examples/hot_reload_tide/src/main.rs#L108).
I was working along the design lines of [Lucene
Segments](https://lucene.apache.org/core/3_0_3/fileformats.html#Segments)
(but with dummy structs), so that parts of an entire shard can be loaded
and unloaded incrementally. I expect read traffic to be high, but updates
to be relatively rare (maybe every few minutes or every day). What data
structure should I use for wrapping the Segments (which are themselves
immutable) data structure so that it can be read and updated concurrently?

The options I found were:

1. `Arc<RwLock<T>>` (from `std` or
   [`parking_lot`](https://docs.rs/parking_lot/latest/parking_lot/type.RwLock.html))
2. [`arc_swap`](https://docs.rs/arc-swap/latest/arc_swap/)
3. [`rcu_clean`](https://docs.rs/rcu-clean/latest/rcu_clean/)
4. [`left_right`](https://docs.rs/left-right/latest/left_right/)
5. [`dashmap`](https://docs.rs/dashmap/latest/dashmap/)
6. [`lockfree-cuckhoohash`](https://docs.rs/lockfree-cuckoohash/latest/lockfree_cuckoohash/)

The first 3 options can just wrap a `HashMap`, which is a simple refactor.
The next 3 require switching to a different container than `HashMap`.

With both `ArcSwap` and `ArcRcu`, the mechanism that allows fast reads and
slower updates is essentially to use Memory Overhead and keep two copies
of the data and swap. This MO cost is also how `left_right` works, with
the added cost of keeping an `oplog`.

In common cases where a configuration is being reloaded in total each
time, this works totally fine. But in my case I wanted to try and maintain
a map or list of Segments and add and remove them piecemeal over time,
explicitly for enabling incremental changes that require less memory
consumption on my host. But the naive
[Read-Copy-Update](https://en.wikipedia.org/wiki/Read-copy-update#Name_and_overview)
pattern can't preserve the incrementalism here: we have to swap the entire
map or list at once.

That is, I'd like to be able to do:

```
let reader = std::io::BufReader::new(file);
let segment: Segment = serde_json::from_reader(reader).expect("segment");
segments.insert(p.to_str().expect("valid path").to_string(), segment);
```

But `segments` needs to be shared across threads, so I put it in
`Arc<RwLock<T>>`:

```
let reader = std::io::BufReader::new(file);
let segment: Segment = serde_json::from_reader(reader).expect("segment");
// we don't lock until AFTER we have already loaded the structure
segments.write().unwrap().insert(p.to_str().expect("valid path").to_string(), segment);
```

This is really the best I can do with wrapping types, because if I want to
use `ArcSwap`, I no longer have a `write` method but only a `store` method
which must take an entire copy of the structure. Here's an example where
`ArcSwap` works really well, switching out a lightweight configuration
file in a watcher:

```
// initial_config is just a simple map
let config = Arc::new(ArcSwap::from_pointee(initial_config));
let c_config = Arc::clone(&config);

// inside the watcher
if event.kind.is_modify() {
    match load_config(CONFIG_PATH) {
        Ok(new_config) => {
            log::info!("loaded new_config: {:?}", new_config);
            // no way to call .insert() on c_config
            c_config.store(Arc::new(new_config));
```

And though it's not as immediately obvious this is happening with `ArcRcu`
too, it is, and somewhat worse because it will keep _all_ old copies until
[you call `clean` or the smart pointer itself is
gone](https://docs.rs/rcu-clean/latest/rcu_clean/#cleaning) (not what we
expect for long-lived data structure like in my case).

```
let reader = std::io::BufReader::new(file);
let segment: Segment = serde_json::from_reader(reader).expect("segment");
// no lock, but is actually copying whole map to just change one segment
*segments.update().insert(p.to_str().expect("valid path").to_string(), segment);
```

That is, not only does it have MO, but it has a pretty big memory leak
foot-gun in that you may forget to call `clean`.

Moving on to the options that require more significant refactoring,
there's the `left_right` technique, which was originally announced in [a
brief paper in
2015](https://hal.archives-ouvertes.fr/hal-01207881/document).

> The Left-Right is a concurrency control technique with two identical
> objects or data structures, that allows an unlimited number of threads
> (Readers) to access one instance in read-only mode, while a single
> thread (Writer) modifies the other instance. The Writer starts by
> working on the right-side instance (rightInst) while the Readers read
> the left-side instance (leftInst), and once the Writer completes the
> modification, the two instances are switched and new Readers will read
> from the rightInst. The Writer will wait for all the Readers still
> running on the leftInst instance to finish, and then repeat the
> modification on the leftInst, leaving both instances up-to-date. It us
> up to the Writer to ensure that Readers are always running on the data
> structure that is currently not being modified. The synchronization
> between Writers is achieved with an exclusive lock that is used to
> protect write-access (writersMutex).

Of course, like RCU, this requires high MO. One can potentially reduce
this by making only the "head" of your data structure mutable and keep
a tombstone filter on the "long tail" which remains static, but that
requires even more MO, and also periodic vacuuming. In other words, it
doesn't allow us to read all the Segments and update all of them without
MO.

With `dashmap`, [the author explains the strategy in this Hacker News
comment](https://news.ycombinator.com/item?id=22703269) in contrast to
`CHashMap`:

> CHashMap is essentially a table behind an rwlock where each slot is also
> behind its own rwlock. This is great in theory since it allows decently
> concurrent operations providing they don't need to lock the whole table
> for resizing. In practice this falls short quickly because of the
> contention on the outer lock and the large amount of locks and unlocks
> when traversing the map **dashmap works by splitting into an array of
> shards, each shard behind its own rwlock.** The shard is decided from
> the keys hash. This will only lock and unlock once for any one shard and
> allows concurrent table locking operations provided they are on
> different shards. Further, there is no central rwlock each thread must
> go thru which improves performance significantly.

That is, `DashMap` switches from `RwLock<HashMap>` to (figuratively)
`Vec<RwLock<HashMap>>`. This is nice for the Segments case given you're
reducing how many Segments are impacted by a write lock on one. It could
be strictly better, provided the Segments really are falling across
different shards.

Finally, there is `LockFreeCuckooHash`, which [this post explains in
depth](https://eourcs.github.io/LockFreeCuckooHash/). The key thing is
that it does not use reader-write locks, like `DashMap`, and so we should
expect it to perform (much) better under contention. When the table runs
out of space, however, "[most] implementations today [including this one]
implement resizing by locking the whole table and copying all of the data
to a new table." For my toy case, which is such a small map, this isn't
a show-stopper, but makes it unsuitable for many other cases.

In sum then, for our original options, which looked like 6, in RUM we find
are really just 2:

1. Accept a read penalty so that we can reduce memory overhead
2. Accept a memory penalty so that we can increase read speed

It'd be fun and surprising if the RUM conjecture turns out to be false. In
my day to day working life I treat it as true, and that lets me navigate
the design space faster.
