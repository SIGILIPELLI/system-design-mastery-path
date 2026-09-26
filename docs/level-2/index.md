---
description: "Level 2 of System Design: replication, sharding, consistent hashing, CAP and PACELC, queues, rate limiting, indexing, invalidation, and object storage."
---

# Level 2 · Intermediate <span class="level-badge">Distributing data</span>

Level 1 scaled the stateless parts of a system and leaned on one big database. Level 2
is about what happens when the data itself must be spread across machines — for
availability, for read capacity, or because it simply no longer fits.

That shift introduces the central tension of distributed data: copies can disagree, and
partitions must be found. Every lesson here is a variation on two questions. *Where does
this piece of data live?* And *what does a reader see while copies are catching up?*

## Modules

1. [Replication](01-replication.md) — leader-follower, multi-leader, and leaderless replication, and the anomalies each produces
2. [Sharding & Partitioning](02-sharding-partitioning.md) — range vs hash partitioning, choosing a shard key, and rebalancing
3. [Consistent Hashing](03-consistent-hashing.md) — why naive modulo hashing breaks and how rings with virtual nodes fix it
4. [CAP & PACELC, With Nuance](04-cap-pacelc.md) — what the theorems actually say, and what they do not
5. [Message Queues & Async Processing](05-message-queues.md) — queues vs logs, delivery semantics, back-pressure, and dead-letter queues
6. [Rate Limiting Algorithms](06-rate-limiting.md) — token bucket, leaky bucket, fixed and sliding windows, and distributed limits
7. [Indexing & Query Patterns](07-indexing-query-patterns.md) — B-trees, LSM trees, composite and covering indexes, and secondary indexes across shards
8. [Cache Invalidation Strategies](08-cache-invalidation.md) — TTLs, explicit invalidation, versioned keys, and the races that bite
9. [Blob & Object Storage](09-object-storage.md) — separating bytes from metadata, presigned uploads, multipart, and durability
10. [Project — Design a News Feed](10-project-news-feed.md) — fan-out on write vs read, ranking, pagination, and the celebrity problem

## What you need before starting

- Level 1, or equivalent comfort with load balancers, caches, and the estimation
  recipe.
- Python 3 for the simulations (consistent-hash ring, rate limiters, replication lag).
  Everything runs locally with the standard library.

Revisit your Level 1 URL-shortener design after lessons 1–3: you should be able to say
exactly how you would replicate and shard it, and what a user sees during failover.
