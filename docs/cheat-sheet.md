---
title: "System Design Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free System Design cheat sheet: 22 quick-reference tables from every level of the System Design Mastery Path, each linked to its full lesson."
---

# System Design Cheat Sheet

Quick-reference tables from **22 lessons** of the free System Design Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What System Design Is &amp; How to Approach It](level-1/01-what-is-system-design.md)

*Requirements come first*

| Quality | Example question to ask |
|---|---|
| Scale | How many users? Requests per second at peak? Data growth per year? |
| Latency | What response time is acceptable, and at which percentile (p50, p99)? |
| Availability | Is a few minutes of downtime a month acceptable? A few seconds? |
| Consistency | If a user writes something, must they (and others) see it immediately? |
| Durability | Can we ever lose accepted data? What about the last few seconds of it? |
| Cost | Is this a startup on a small budget or a platform with millions in spend? |
| Compliance | Personal data, payment data, data residency rules? |

### [02 · Client-Server &amp; HTTP Basics](level-1/02-client-server-http.md)

*HTTP in one table*

| Method | Meaning | Safe? | Idempotent? |
|---|---|---|---|
| GET | Read a resource | Yes | Yes |
| HEAD | GET without the body | Yes | Yes |
| POST | Create or trigger processing | No | No |
| PUT | Replace a resource at a known URL | No | Yes |
| PATCH | Partially modify a resource | No | Not necessarily |
| DELETE | Remove a resource | No | Yes |

### [03 · Latency, Throughput &amp; Back-of-Envelope Estimation](level-1/03-latency-throughput-estimation.md)

*Rough numbers to carry around*

| Operation | Rough latency |
|---|---|
| Read from main memory | ~100 nanoseconds |
| Read 1 MB sequentially from memory | ~tens of microseconds |
| Round trip within one datacenter | ~0.5 ms |
| Random read from an SSD | ~0.1 ms (100 µs) |
| Read 1 MB sequentially from SSD | ~1 ms or less |
| Disk seek on a spinning hard drive | ~5–10 ms |
| Round trip between continents | ~100–200 ms |

### [04 · Vertical vs Horizontal Scaling](level-1/04-vertical-horizontal-scaling.md)

*Making the app tier stateless*

| Hidden state | Move it to |
|---|---|
| User sessions in memory | Shared session store, or signed tokens |
| Uploaded files on local disk | Object storage |
| In-process caches assumed to be authoritative | Treat as disposable, or use a shared cache |
| Scheduled jobs that run "on the server" | A job scheduler with a lock or a single leader |
| WebSocket connections | Unavoidable — design routing for it (Level 3, lesson 6) |

### [05 · Load Balancing](level-1/05-load-balancing.md)

*Balancing algorithms*

| Algorithm | How it picks | Good for | Weak when |
|---|---|---|---|
| Round robin | Next server in order | Equal servers, similar requests | Requests vary widely in cost |
| Weighted round robin | Proportional to weight | Mixed machine sizes | Weights go stale |
| Least connections | Fewest open connections | Variable request duration | Many balancers each see partial counts |
| Least response time / EWMA | Fastest recently | Heterogeneous latency | Needs good measurements |
| Power of two choices | Pick 2 at random, choose the less loaded | Large pools, many balancers | — generally robust |
| Hash (IP, user ID, key) | Same key → same server | Session affinity, cache locality | Uneven keys; rehash on pool change |

### [06 · Caching Fundamentals](level-1/06-caching-fundamentals.md)

*Eviction policies*

| Policy | Evicts | Notes |
|---|---|---|
| LRU | Least recently used | Good general default; a single scan of cold keys can flush hot ones |
| LFU | Least frequently used | Keeps long-term popular items; slow to adapt when popularity shifts |
| FIFO | Oldest inserted | Simple, usually worse hit ratio |
| TTL | Expired items | Bounds staleness; often combined with LRU/LFU |
| Random | Random item | Surprisingly decent and cheap |

### [07 · SQL vs NoSQL Basics](level-1/07-sql-vs-nosql.md)

*The families*

| Family | Model | Typical strengths | Typical costs |
|---|---|---|---|
| Relational (PostgreSQL, MySQL) | Tables, rows, joins, SQL | Ad-hoc queries, joins, multi-row ACID transactions, constraints | Horizontal write scaling needs sharding work |
| Key-value (Redis, DynamoDB-style) | key → opaque value | Very fast lookups by key, simple to partition | Queries other than by key are hard |
| Document (MongoDB-style) | key → JSON-like document | Nested data read together, flexible schema | Cross-document joins and constraints are weaker |
| Wide-column (Cassandra-style) | Partition key → sorted rows | High write throughput, time series, huge scale | Must model tables around queries; limited ad-hoc querying |
| Graph (Neo4j-style) | Nodes and edges | Multi-hop relationship queries | Niche; scaling across machines is hard |
| Search (Elasticsearch/OpenSearch) | Inverted index | Full-text and faceted search | Usually a secondary index, not the source of truth |

### [08 · CDNs &amp; Edge Caching](level-1/08-cdns.md)

*What a CDN accelerates*

| Content | CDN benefit |
|---|---|
| Static assets (JS, CSS, fonts, images) | Very high — cache for a long time with versioned filenames |
| Video segments | Very high — the same segments are watched by many users |
| Public API responses that are identical for everyone | High, with short TTLs |
| Personalized pages / per-user API responses | Little caching; still gains connection setup near the user |
| Writes (POST, uploads) | No caching; some benefit from terminating TLS nearby and faster routing |

### [09 · APIs: REST, gRPC &amp; GraphQL](level-1/09-api-styles.md)

*Choosing*

| Situation | Reasonable default |
|---|---|
| Public API for third-party developers | REST (+ OpenAPI spec) |
| Internal service-to-service calls, polyglot teams | gRPC |
| Many client types with differing data needs over one data graph | GraphQL (often in front of REST/gRPC services) |
| Heavily cacheable public content | REST behind a CDN |
| Streaming updates between services | gRPC streaming (or a message broker) |

### [10 · Project — Design a URL Shortener](level-1/10-project-url-shortener.md)

*Step 4 — Generating short codes*

| Approach | How | Pros | Cons |
|---|---|---|---|
| Hash the URL | Take first 7 chars of base62(SHA-256(url)) | Same URL → same code | Collisions must be handled; same URL from two users shares a code (maybe unwanted) |
| Counter + base62 | Global counter, encode in base62 | No collisions, short | Sequential → enumerable; a single counter is a bottleneck/SPOF |
| Random | 7 random base62 chars, insert with uniqueness check | Not enumerable, simple, no coordination | Collision retry needed (rare while space is sparse) |
| Pre-generated key pool | Background job creates unused random codes; servers claim batches | Fast issue, no collision on hot path | Extra component; must not hand the same batch out twice |

## Level 2 · Intermediate

### [02 · Sharding &amp; Partitioning](level-2/02-sharding-partitioning.md)

*Choosing a partition key*

| System | Good key | Why | Watch out for |
|---|---|---|---|
| Multi-tenant SaaS | `tenant_id` | Almost all queries and transactions are per tenant | One huge tenant |
| Chat messages | `conversation_id` | "Load this conversation" is the hot query | Giant group chats |
| Social posts | `user_id` of author | Profile pages read one user's posts | Celebrity accounts |
| Orders | `customer_id` | Customer history + per-customer transactions | Global "all orders today" reports |

### [04 · CAP &amp; PACELC, With Nuance](level-2/04-cap-pacelc.md)

*PACELC: the everyday trade-off*

| Configuration | During partition | Normal operation |
|---|---|---|
| Single leader, sync replication to a quorum | Minority side unavailable (PC) | Pays replication latency (EC) |
| Async replicas, read from any replica | Replicas keep serving (PA) | Fast but possibly stale (EL) |
| Dynamo-style with tunable quorums | Depends on R/W settings | Depends on R/W settings |

### [05 · Message Queues &amp; Async Processing](level-2/05-message-queues.md)

*Two shapes: queues and logs*

| | Work queue | Log |
|---|---|---|
| After consumption | Message deleted | Message retained until retention expires |
| Multiple independent readers | Needs one queue per reader (fan-out exchange/topic) | Native (consumer groups) |
| Ordering | Often best-effort | Ordered within a partition |
| Replay old messages | No | Yes (within retention) |
| Per-message retry/delay | Natural | Awkward — a slow message blocks its partition |

### [08 · Cache Invalidation Strategies](level-2/08-cache-invalidation.md)

*Choosing*

| Data | Suggested strategy |
|---|---|
| Public, tolerant of minutes of staleness | TTL with jitter |
| User-visible edits (profile, settings) | Delete-on-write + TTL backstop; read-your-writes from DB right after edit |
| Derived/aggregate views | Versioned keys |
| Many writers, many code paths | CDC-driven invalidation + TTL |
| Money, inventory at checkout | Do not serve from cache for the decision; read the source |

### [09 · Blob &amp; Object Storage](level-2/09-object-storage.md)

*Durability, availability, and cost tiers*

| Tier | Use | Trade-off |
|---|---|---|
| Standard / hot | Frequently accessed content | Highest per-GB price, cheap access |
| Infrequent access | Backups, older media | Cheaper storage, per-GB retrieval fees, minimum storage durations |
| Archive / cold | Compliance archives | Very cheap storage, retrieval can take minutes to hours |

## Level 3 · Advanced

### [06 · Real-Time Systems: WebSockets &amp; Pub/Sub](level-3/06-real-time-systems.md)

*Four ways to push*

| Technique | How | Latency | Cost / complexity |
|---|---|---|---|
| Short polling | Client asks every N seconds | Up to N seconds | Wasteful: most responses are empty |
| Long polling | Server holds the request until data arrives or timeout | Near real time | One request per message; reconnect overhead |
| Server-Sent Events (SSE) | One long-lived HTTP response streaming events | Real time, server→client only | Simple, works with HTTP infrastructure, auto-reconnect with last event ID |
| WebSockets | Upgraded persistent, bidirectional connection | Real time, both directions | Stateful connections; needs L7 support for upgrades and sticky routing |

### [09 · Hot Keys, Skew &amp; Multi-Tenancy](level-3/09-multi-tenant-hot-keys.md)

*Kinds of skew*

| Kind | Example | Symptom |
|---|---|---|
| Hot read key | A viral post's metadata | One cache node or shard at 100% CPU / network |
| Hot write key | A global "likes" counter on that post | Lock contention on one row; one partition's write queue grows |
| Hot partition | Time-based range partition receiving all current writes | Newest shard overloaded, others idle |
| Large tenant | One enterprise customer = 30% of traffic | Their batch jobs slow down everyone else ("noisy neighbor") |

## Level 4 · Master

### [05 · Multi-Region &amp; Disaster Recovery](level-4/05-multi-region-dr.md)

*The spectrum of strategies*

| Strategy | What runs in the second region | Typical RPO | Typical RTO | Cost |
|---|---|---|---|---|
| Backup & restore | Nothing; backups copied there | Hours (last backup) | Hours to days | Lowest |
| Pilot light | Replicated data; minimal or no compute | Minutes (replication lag) | Tens of minutes to hours (scale up compute) | Low |
| Warm standby | Scaled-down full stack, live replication | Seconds to minutes | Minutes | Medium |
| Active-active | Full stack serving live traffic in both | Near zero to seconds (depends on data design) | Seconds to minutes | Highest, and most complex |

### [06 · Cost-Aware Design](level-4/06-cost-aware-design.md)

*Where the money usually goes*

| Category | Drivers | Typical levers |
|---|---|---|
| Compute | Instance hours, over-provisioning, idle capacity | Right-sizing, autoscaling, spot/interruptible for batch, efficient code |
| Storage | GB-months, replicas, snapshots, retention | Tiering, lifecycle deletion, compression, dedup |
| Data transfer | Egress to the internet, cross-region, sometimes cross-zone | CDNs, compression, keeping chatty traffic local |
| Managed services | Per-request or per-capacity pricing of databases, queues, etc. | Batching, caching, choosing capacity vs on-demand modes |
| Observability | Log volume, metric cardinality, trace retention | Sampling, log levels, retention tiers |

### [08 · The Interview Framework &amp; Common Mistakes](level-4/08-interview-framework.md)

*A 45-minute plan*

| Minutes | Phase | Output on the board |
|---|---|---|
| 0–5 | Clarify requirements | Functional list, non-functional targets, explicit out-of-scope |
| 5–10 | Estimate | QPS (read/write), storage, bandwidth — and the conclusion they imply |
| 10–15 | API and data model | 3–5 key endpoints, main entities with keys |
| 15–25 | High-level design | Request path for the 2–3 core operations |
| 25–40 | Deep dives | 1–3 components: data partitioning, the hardest consistency problem, the hot path |
| 40–45 | Wrap-up | Bottlenecks, failure modes, what you would do next with more time |

### [09 · Writing Design Docs &amp; Defending Trade-offs](level-4/09-design-reviews-tradeoffs.md)

*Trade-off tables*

| Criterion (weight) | A: Central rate-limit store | B: Local buckets + periodic sync | C: Per-instance only |
|---|---|---|---|
| Accuracy (high) | Exact | Within ~10% | Poor under uneven load |
| Added latency per request (high) | +1 network round trip | ~0 | 0 |
| Behavior if store fails (medium) | Must fail open or closed | Continues with local limits | Unaffected |
| Complexity (medium) | Low | Medium | Lowest |
| **Decision** | | **Chosen** | |

### [10 · Capstone — Your Own Design Doc](level-4/10-capstone-design-doc.md)

*Choose a system*

| Prompt | The hard part to confront |
|---|---|
| Ticket booking for concerts with flash sales | Inventory contention, fairness, bot traffic, holds that expire |
| Collaborative document editor | Concurrent edits (OT/CRDTs), presence, offline, history |
| Distributed job scheduler (cron at scale) | Exactly-once-effect execution, leader election, missed runs |
| Metrics/monitoring platform | Write-heavy ingestion, time-series storage, high-cardinality queries |
| Food-delivery order tracking | Multi-party state machine, live location, ETAs, partner integrations |
| Web crawler | Politeness, URL frontier prioritization, dedup at scale, traps |
| Notification service (email/SMS/push) | Fan-out, provider failures, idempotency, user preferences, rate limits |
| Hotel/airline inventory and reservations | Overbooking policy, consistency across channels, sagas with external systems |

---

Go deeper with the full [System Design Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
