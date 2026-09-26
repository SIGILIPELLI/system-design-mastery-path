---
description: "Handling skewed load: detecting hot keys, request coalescing, key splitting, local caching, noisy-neighbor isolation, and fair scheduling across tenants."
---

# 09 · Hot Keys, Skew & Multi-Tenancy

Partitioning (Level 2) spreads *keys* evenly. Real traffic is not spread evenly across
keys. A viral post, a flash-sale product, a celebrity account, or one enormous customer
can concentrate a large share of load on a single key, partition, or tenant. When that
happens, the rest of the cluster sits idle while one node melts. This lesson is about
recognizing skew and handling it without redesigning everything.

## Kinds of skew

| Kind | Example | Symptom |
|---|---|---|
| Hot read key | A viral post's metadata | One cache node or shard at 100% CPU / network |
| Hot write key | A global "likes" counter on that post | Lock contention on one row; one partition's write queue grows |
| Hot partition | Time-based range partition receiving all current writes | Newest shard overloaded, others idle |
| Large tenant | One enterprise customer = 30% of traffic | Their batch jobs slow down everyone else ("noisy neighbor") |

## Detecting it

Aggregate metrics hide skew — cluster-wide CPU at 30% can mean one node at 100%. Watch
**per-node and per-partition** metrics (max vs median), and sample request keys to find the
top offenders. A space-efficient way to track heavy hitters in a stream is a **Count-Min
Sketch** or the Space-Saving algorithm, which estimate top-k frequencies with fixed memory.

```python
# heavy_hitters.py — Space-Saving top-k tracking in fixed memory
import random

def space_saving(stream, k):
    counts = {}
    for key in stream:
        if key in counts:
            counts[key] += 1
        elif len(counts) < k:
            counts[key] = 1
        else:
            victim = min(counts, key=counts.get)       # evict the smallest counter
            counts[key] = counts.pop(victim) + 1       # inherit its count (overestimate)
    return sorted(counts.items(), key=lambda kv: -kv[1])

rng = random.Random(5)
stream = [f"post:{rng.randrange(100_000)}" for _ in range(200_000)]
stream += ["post:viral"] * 30_000 + ["post:trending"] * 8_000
rng.shuffle(stream)
for key, est in space_saving(stream, k=50)[:3]:
    print(key, est)
```

With only 50 counters over more than 100,000 distinct keys, the two heavy keys surface at
the top. The algorithm can overestimate counts (a key inherits its victim's count), but
any key whose true frequency exceeds N/k is guaranteed to be tracked — enough to find hot
keys worth acting on.

## Handling hot reads

1. **Local (in-process) caching** of the hottest keys on every app server, with a short
   TTL. The load spreads across all app servers instead of one cache node. Staleness of a
   second or two is usually fine for viral content.
2. **Replicate the hot key** across several cache nodes (`post:viral#0..#7`) and pick one
   at random on read.
3. **Request coalescing**: when many concurrent requests miss on the same key, let one
   fetch and the rest wait for its result (Level 1, lesson 6).
4. **CDN caching** of the public representation.

## Handling hot writes

Writes to a single key are harder: they must all agree on one value.

1. **Split the counter**: keep N sub-counters (`likes:post:viral:0..15`), increment a
   random one, and sum them on read (or periodically). Contention drops by N; reads cost
   more.
2. **Buffer and batch**: aggregate increments in memory or a queue and apply `+N`
   periodically (the click-counter design from the Level 1 project).
3. **Relax precision**: show "1.2M likes" — an approximate, periodically updated number
   is often what the product wanted anyway.
4. **Append instead of update**: record each event in an append-only log partitioned
   randomly; compute totals asynchronously.

## Multi-tenancy and noisy neighbors

A multi-tenant system serves many customers on shared infrastructure. Sharing is what
makes it economical — and what lets one tenant hurt others.

Isolation tools, from light to heavy:

- **Per-tenant rate limits and quotas** (Level 2, lesson 6), sized by plan.
- **Fair queueing**: instead of one FIFO queue for all background jobs, keep per-tenant
  queues and serve them round-robin (or weighted). One tenant's 1M-job backlog no longer
  delays another tenant's single job.
- **Priority classes**: interactive requests ahead of batch work.
- **Cell or shard isolation**: place large tenants on dedicated shards or dedicated
  "cells" (complete, independent copies of the stack serving a subset of tenants), so their
  load and their failures stay contained.
- **Tenant-aware observability**: metrics tagged by tenant tier (not per tenant ID as a
  metric label — see the cardinality warning in lesson 7; use logs and traces for per-tenant
  detail) so you can tell which tenant is causing a problem.

## Worked example: fair queueing vs FIFO

```python
# fair_queue.py — per-tenant round-robin vs a single FIFO queue
from collections import deque, OrderedDict

jobs = [("big", i) for i in range(1000)] + [("small", 0)]   # small arrives last

fifo = deque(jobs)
pos_fifo = next(i for i, j in enumerate(fifo) if j[0] == "small")

queues = OrderedDict()
for tenant, job in jobs:
    queues.setdefault(tenant, deque()).append(job)
order = []
while queues:
    for tenant in list(queues):
        order.append((tenant, queues[tenant].popleft()))
        if not queues[tenant]:
            del queues[tenant]
pos_fair = next(i for i, j in enumerate(order) if j[0] == "small")

print(f"small tenant's job runs at position {pos_fifo} with FIFO, {pos_fair} with fair queueing")
```

With FIFO, the small tenant waits behind all 1,000 of the big tenant's jobs; with per-tenant
round-robin it runs second. The big tenant's total completion time barely changes.

## How It Actually Works

Why does one hot key overload one node even with perfect hashing? A hash function maps
each key to exactly one partition — deterministically, which is the whole point. Hashing
balances the *number of keys* per node; the traffic per key is a property of the
workload. If one key receives 20% of requests, its node receives at least 20% of cluster
traffic regardless of cluster size. Adding nodes does nothing for that key. The only
remedies change the mapping for that key specifically (replicate it, split it, cache it
elsewhere) or reduce its traffic (coalesce, batch, approximate).

Fair queueing works by changing *which* job is served next rather than how many are
served. Throughput is the same; what changes is the distribution of waiting time. Weighted
variants (deficit round robin, weighted fair queueing) generalize this so a paying tier can
receive, say, twice the share of a free tier — without either being able to monopolize
workers.

## Common mistakes

- **Monitoring averages across nodes**, so the one overloaded node is invisible.
- **Adding nodes to fix a hot key.**
- **Exact global counters** on viral objects, updated synchronously.
- **One shared FIFO queue** for all tenants' background work.
- **No per-tenant limits** until the first large customer takes down everyone else.

## Exercise

1. Run `heavy_hitters.py` with `k=10`, `50`, and `500`. How reliable is the top-3 at each
   size?
2. Implement a sharded counter with 16 sub-keys and a reader that sums them. Compare the
   maximum writes per sub-key against a single key for 100,000 increments.
3. Your SaaS has 5,000 small tenants and 3 enterprise tenants who each generate 15% of
   load. Design their placement (shared shards, dedicated shards, or cells), limits, and how
   you would move a tenant that grows into the enterprise tier.
