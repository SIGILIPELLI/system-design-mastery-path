---
description: "Sharding explained: range vs hash partitioning, picking a shard key, hot spots, cross-shard queries, and rebalancing without downtime."
---

# 02 · Sharding & Partitioning

When a dataset or its write load outgrows one machine, you split it into **partitions**
(also called shards), each holding a subset of the data and living on a different node.
Every record belongs to exactly one partition. Combined with replication, each partition
typically has its own leader and followers.

Partitioning buys near-linear write and storage scaling — *if* the load spreads evenly
and most operations touch one partition. Both conditions depend almost entirely on one
decision: the **partition key**.

## Range partitioning

Assign contiguous key ranges to partitions, like volumes of an encyclopedia:

```text
partition 1: user names A–F
partition 2: G–M
partition 3: N–S
partition 4: T–Z
```

- Range queries are efficient: "all orders between two dates" touches few partitions.
- Ranges can be split when they grow, which is how many distributed databases rebalance.
- **Hot spots** are the danger. If the key is a timestamp, *all* current writes land in
  the newest range. The fix is often a compound key: `(sensor_id, timestamp)` spreads
  writes across sensors while keeping each sensor's data sorted.

## Hash partitioning

Hash the key and assign by hash value:

```text
partition = hash(user_id) mod number_of_partitions   (naive — see next lesson)
```

- Spreads keys evenly, even if key values are clustered.
- Destroys ordering: a range query must ask every partition.
- Naive `mod N` reshuffles almost every key when N changes; lesson 3's consistent
  hashing and the fixed-partition scheme below avoid that.

## Choosing a partition key

Ask three questions:

1. **Do most queries include this key?** Then they route to one partition.
2. **Does it have high cardinality and even load?** Many distinct values, none dominant.
3. **Do transactions stay inside one key's data?** Then you keep local ACID.

| System | Good key | Why | Watch out for |
|---|---|---|---|
| Multi-tenant SaaS | `tenant_id` | Almost all queries and transactions are per tenant | One huge tenant |
| Chat messages | `conversation_id` | "Load this conversation" is the hot query | Giant group chats |
| Social posts | `user_id` of author | Profile pages read one user's posts | Celebrity accounts |
| Orders | `customer_id` | Customer history + per-customer transactions | Global "all orders today" reports |

## Worked example: measuring skew

```python
# shard_skew.py — how even is the load under different keys?
import hashlib, random
from collections import Counter

def shard_of(key, n):
    h = int(hashlib.md5(str(key).encode()).hexdigest(), 16)
    return h % n

rng = random.Random(3)
N = 8
# 100,000 orders from 20,000 customers; one "enterprise" customer places 15% of orders
orders = [("ent", i) if rng.random() < 0.15 else (rng.randrange(20_000), i)
          for i in range(100_000)]

by_customer = Counter(shard_of(c, N) for c, _ in orders)
by_order    = Counter(shard_of(o, N) for _, o in orders)

def report(name, counts):
    loads = [counts[s] for s in range(N)]
    print(f"{name:12s} max/avg = {max(loads) / (sum(loads) / N):.2f}  {loads}")

report("customer_id", by_customer)
report("order_id", by_order)
```

Partitioning by `order_id` spreads load almost perfectly (max/avg near 1.0). Partitioning
by `customer_id` puts the enterprise customer's 15% on one shard, making it roughly twice
as loaded as average. But `order_id` makes "all orders for a customer" hit every shard.
Typical resolutions: keep `customer_id` and give the few giant tenants dedicated shards,
or add a suffix to split one hot key into several sub-keys (`ent#0` … `ent#7`) and
combine them on read.

## Cross-partition operations

Things that get harder once data is partitioned:

- **Queries without the partition key** become scatter-gather: send to all partitions,
  merge results. Latency is set by the slowest partition, and cost grows with partition
  count.
- **Secondary indexes** must either be local to each partition (writes are cheap, reads
  scatter) or global and partitioned by the indexed value (reads are targeted, but a
  write may update an index partition on another node, usually asynchronously).
- **Transactions across partitions** need distributed commit protocols or sagas
  (Level 3), both costly. Design keys so common transactions stay local.
- **Unique constraints** on anything other than the partition key require a separate
  lookup table or coordination.

## Rebalancing

Nodes are added and removed; data must move without downtime.

- **Fixed number of partitions** (for example, 1,024 logical partitions on 8 nodes): when
  a node joins, it takes over whole partitions from others. Keys never change partition;
  only partition → node assignments change. Simple and widely used. Choose the partition
  count large enough for future growth.
- **Dynamic splitting**: a partition that grows past a size threshold splits in two
  (natural for range partitioning).
- **Consistent hashing**: next lesson.

A routing layer — a config service, a proxy, or a smart client — keeps the current
partition map. During a move, the old owner keeps serving until the new one has copied
the data and caught up on changes made during the copy; then ownership switches.

## How It Actually Works

A sharded system has a **routing** step before every query. The router hashes (or
range-looks-up) the key to find the partition, looks up which node currently owns that
partition in the partition map, and forwards the request. The partition map is small
(thousands of entries), so every router can hold a full copy; it is typically stored in
a strongly consistent coordination service so that all routers agree on ownership after
a move.

Moving a partition live usually follows a copy-then-catch-up pattern: snapshot the
partition to the new node, stream changes made since the snapshot (from the replication
log), and once the new copy is nearly current, briefly pause writes to that partition,
apply the final changes, flip ownership in the map, and resume. The pause is short
because only the tail of changes remains. Writes that arrive at the old node after the
flip are rejected with a "wrong owner" error that tells the client to refresh its map.

## Common mistakes

- **Sharding by a monotonically increasing key** with range partitioning (all writes on
  the last shard).
- **Picking a key that most queries do not use**, turning every request into
  scatter-gather.
- **Too few logical partitions**, so you cannot spread load when you add nodes.
- **Ignoring celebrity keys** and assuming hash partitioning makes load uniform. It
  spreads *keys* evenly, not *traffic* per key.
- **Sharding before you need to**, paying the complexity for years without benefit.

## Exercise

1. Run `shard_skew.py`, then implement key salting for the enterprise customer (split into
   8 sub-keys) and show the new max/avg ratio. What does a "list all orders for `ent`"
   query cost now?
2. Choose partition keys for a URL shortener's `links` and `click_counts` tables. Justify
   them against the three questions above.
3. Your ride-sharing trips table is partitioned by `rider_id`. Drivers now need "my trips
   this week". Propose two designs and compare their write cost and read cost.
