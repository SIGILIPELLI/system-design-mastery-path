---
title: "System Design Interview Questions and Answers (179 Questions)"
description: "179 System Design interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the System Design Mastery Path."
---

# System Design Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free System Design Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What System Design Is &amp; How to Approach It](level-1/01-what-is-system-design.md)

**Q1.** What is the main goal of system design?

**Answer:** Meet functional and non-functional requirements (scale, latency, availability, cost) with explicit trade-offs

There is no single best design, only trade-offs against requirements.

**Q2.** Which step should come first in a design discussion?

**Answer:** Clarify requirements and constraints, then estimate scale

Requirements drive every later choice.

**Q3.** What is a non-functional requirement?

**Answer:** A quality such as latency, availability, durability or cost

They often decide the architecture.

**Q4.** Why state assumptions explicitly?

**Answer:** So estimates and trade-offs can be checked and challenged

Unstated assumptions hide risk.

### [02 · Client-Server &amp; HTTP Basics](level-1/02-client-server-http.md)

**Q5.** What does HTTP being stateless mean?

**Answer:** Each request carries all the information the server needs; the server keeps no client session by default

State lives in tokens, cookies or backing stores.

**Q6.** Which status class means a client error?

**Answer:** 4xx

5xx means server errors.

**Q7.** A request needs DNS 20 ms, TCP 30 ms, TLS 60 ms and 50 ms of server time. Assuming nothing is reused or cached, what is the minimum time to first byte (ms)?

```python
print(20+30+60+50)
```

**Answer:** 160

Connection reuse and caching remove the setup costs on later requests. Verified by running it.

**Q8.** What does keep-alive (connection reuse) save?

**Answer:** Repeated TCP and TLS handshakes

HTTP/2 multiplexes many requests on one connection.

### [03 · Latency, Throughput &amp; Back-of-Envelope Estimation](level-1/03-latency-throughput-estimation.md)

**Q9.** A service handles 50 million requests per day. What is the average queries per second (rounded)?

```python
print(round(50_000_000/86_400))
```

**Answer:** 579

There are 86,400 seconds in a day; peaks are usually 2-10x the average. Verified by running it.

**Q10.** With a 3x peak factor, what peak QPS do you plan for?

```python
print(round(50_000_000/86_400*3))
```

**Answer:** 1736

Plan for peak, not average. Verified by running it.

**Q11.** Each record is 2 KB and you store 100 million records. How many GB (using 1 GB = 10^9 bytes)?

```python
print(100_000_000*2_000/1e9)
```

**Answer:** 200.0

Then multiply by replicas and growth. Verified by running it.

**Q12.** Which latency number is roughly right: reading 1 MB sequentially from SSD or a network round trip within one datacentre?

**Answer:** They are both on the order of a millisecond or less; cross-continent round trips are around 100 ms

Know orders of magnitude, not exact values.

### [04 · Vertical vs Horizontal Scaling](level-1/04-vertical-horizontal-scaling.md)

**Q13.** What is vertical scaling?

**Answer:** Adding more power (CPU, RAM) to a single machine

Simple but limited and a single point of failure.

**Q14.** What is horizontal scaling?

**Answer:** Adding more machines and spreading load across them

Requires stateless services or partitioned state.

**Q15.** Why make web servers stateless?

**Answer:** Any instance can serve any request, so you can add or remove instances freely

Store sessions in a shared store or token.

**Q16.** What limits horizontal scaling of a single-primary database?

**Answer:** Write throughput on the primary; reads scale with replicas, writes need sharding

Plan for the write path.

### [05 · Load Balancing](level-1/05-load-balancing.md)

**Q17.** Round-robin over 4 servers: which server index (0-based) handles request 10?

```python
print(10 % 4)
```

**Answer:** 2

Index = request number mod server count. Verified by running it.

**Q18.** What do health checks let a load balancer do?

**Answer:** Stop sending traffic to failed instances

Active or passive.

**Q19.** What is a downside of sticky sessions?

**Answer:** Uneven load and lost sessions if the server dies

Prefer shared session stores or tokens.

**Q20.** What is the difference between L4 and L7 load balancing?

**Answer:** L4 routes by IP and port; L7 understands HTTP and can route by path, header or cookie

L7 costs more CPU but is smarter.

### [06 · Caching Fundamentals](level-1/06-caching-fundamentals.md)

**Q21.** A cache has a 90% hit rate. Cache reads take 1 ms and database reads 20 ms. What is the average read time (ms)?

```python
print(0.9*1+0.1*20)
```

**Answer:** 2.9

Average = hit rate x hit time + miss rate x miss time. Verified by running it.

**Q22.** What is cache-aside?

**Answer:** The app checks the cache first, loads from the database on a miss and populates the cache

The most common pattern.

**Q23.** What does TTL do?

**Answer:** Bounds how long an entry may be stale and limits memory use

A safety net for invalidation.

**Q24.** What is an LRU cache?

**Answer:** It evicts the least recently used entry when full

Good for temporal locality.

### [07 · SQL vs NoSQL Basics](level-1/07-sql-vs-nosql.md)

**Q25.** When is SQL a strong default?

**Answer:** Relational data needing joins, constraints and transactions

Relational databases scale further than most think.

**Q26.** What do many NoSQL stores trade for scale?

**Answer:** Rich queries, joins and sometimes strong consistency

Model data around access patterns.

**Q27.** What is a wide-column or key-value store good at?

**Answer:** Very high write throughput and simple lookups by key at large scale

Examples: Cassandra, DynamoDB.

**Q28.** What should drive the database choice?

**Answer:** Access patterns, consistency needs and operational maturity, not fashion

Often use more than one.

### [08 · CDNs &amp; Edge Caching](level-1/08-cdns.md)

**Q29.** What does a CDN do?

**Answer:** Caches content at edge locations close to users

Lower latency and origin load.

**Q30.** Which content suits a CDN best?

**Answer:** Static, cacheable assets such as images, scripts and video segments

Dynamic content can be cached with care.

**Q31.** What is cache invalidation on a CDN?

**Answer:** Purging or versioning objects so users get updated content

Prefer versioned URLs.

**Q32.** What is origin shielding?

**Answer:** A middle cache tier that reduces requests reaching the origin

Protects the origin under load.

### [09 · APIs: REST, gRPC &amp; GraphQL](level-1/09-api-styles.md)

**Q33.** What is REST organised around?

**Answer:** Resources identified by URLs and manipulated with standard HTTP methods

Stateless interactions.

**Q34.** When is GraphQL attractive?

**Answer:** When clients need to shape responses and avoid over- or under-fetching

Watch out for expensive queries.

**Q35.** What is gRPC good for?

**Answer:** Efficient, typed service-to-service calls over HTTP/2 with Protocol Buffers

Streaming support.

**Q36.** Why paginate list endpoints?

**Answer:** To bound response size and cost; cursor pagination is stable under inserts

Offset pagination gets slow at depth.

### [10 · Project — Design a URL Shortener](level-1/10-project-url-shortener.md)

**Q37.** A URL shortener issues 7-character base62 codes (a-z, A-Z, 0-9). How many distinct codes exist?

```python
print(62**7)
```

**Answer:** 3521614606208

62^7 is about 3.5 trillion, plenty for most services. Verified by running it.

**Q38.** At 100 million new URLs per year, how many years until 3.5 trillion codes are used?

```python
print(round(3_521_614_606_208/100_000_000))
```

**Answer:** 35216

Capacity is not the issue; avoiding collisions and hot keys is. Verified by running it.

**Q39.** How can you generate unique short codes without collisions?

**Answer:** A counter or ID generator encoded in base62, or random codes with a uniqueness check

Pre-generate ranges per server.

**Q40.** Why is a URL shortener read-heavy and how does that shape the design?

**Answer:** Many more redirects than creations, so cache hot mappings and use replicas

Optimise the read path.

## Level 2 · Intermediate

### [01 · Replication](level-2/01-replication.md)

**Q41.** What is replication?

**Answer:** Keeping copies of data on multiple nodes for availability and read scaling

Does not by itself increase write capacity.

**Q42.** What is replication lag?

**Answer:** The delay before a replica reflects a primary's writes, causing stale reads

Read-your-writes needs care.

**Q43.** What is the difference between sync and async replication?

**Answer:** Synchronous waits for replicas before acknowledging (safer, slower); asynchronous does not (faster, can lose recent writes)

Semi-sync is a compromise.

**Q44.** With 3 replicas and a quorum of W=2 writes and R=2 reads, do reads always overlap the latest write? (R + W &gt; N)

```python
N,W,R=3,2,2
print(R+W>N)
```

**Answer:** True

Overlapping read and write quorums guarantee seeing the latest acknowledged write. Verified by running it.

### [02 · Sharding &amp; Partitioning](level-2/02-sharding-partitioning.md)

**Q45.** What is sharding?

**Answer:** Splitting data across nodes so each holds a subset

Scales writes and storage.

**Q46.** What is a hot shard?

**Answer:** A shard receiving disproportionate load, often from a skewed key

Choose keys that distribute evenly.

**Q47.** Hash-modulo sharding with 4 shards: which shard holds key 1234567?

```python
print(1234567 % 4)
```

**Answer:** 3

Adding a shard remaps most keys, which is why consistent hashing exists. Verified by running it.

**Q48.** What makes cross-shard queries hard?

**Answer:** Joins and transactions span nodes, so they need scatter-gather or denormalisation

Design to keep related data together.

### [03 · Consistent Hashing](level-2/03-consistent-hashing.md)

**Q49.** Consistent hashing on a ring of 10 nodes: when you add one node, what fraction of keys moves on average?

```python
print(round(1/11,3))
```

**Answer:** 0.091

About K/N keys move, versus nearly all with modulo hashing. Verified by running it.

**Q50.** With modulo hashing, going from 4 to 5 nodes remaps what fraction of 1000 keys?

```python
moved=sum(1 for k in range(1000) if k%4 != k%5)
print(moved/1000)
```

**Answer:** 0.8

Modulo hashing reshuffles most keys on a resize. Verified by running it.

**Q51.** What do virtual nodes add to consistent hashing?

**Answer:** More even key distribution and smoother rebalancing

Each physical node owns many ring positions.

**Q52.** Where is consistent hashing used?

**Answer:** Distributed caches, sharded stores and load balancers

E.g. Dynamo-style systems.

### [04 · CAP &amp; PACELC, With Nuance](level-2/04-cap-pacelc.md)

**Q53.** What does the CAP theorem say about partitions?

**Answer:** During a network partition you must choose between consistency and availability

P is not optional in distributed systems.

**Q54.** What does PACELC add?

**Answer:** Even without partitions there is a trade-off between latency and consistency

Else Latency vs Consistency.

**Q55.** Which is a CP-style behaviour?

**Answer:** Refusing writes on the minority side of a partition to avoid divergence

Consensus systems often do this.

**Q56.** Which is an AP-style behaviour?

**Answer:** Accepting writes on both sides and reconciling later (eventual consistency)

Conflicts need resolution rules.

### [05 · Message Queues &amp; Async Processing](level-2/05-message-queues.md)

**Q57.** Why use a message queue?

**Answer:** Decouple producers and consumers, absorb bursts and enable retries

Async processing.

**Q58.** What delivery guarantee is typical?

**Answer:** At-least-once, so consumers must be idempotent

Exactly-once is built from idempotency.

**Q59.** What is a dead-letter queue?

**Answer:** Where messages go after repeated failures for inspection

Alert on it.

**Q60.** A queue gets 1000 msgs/s and consumers each handle 50 msgs/s. How many consumers keep up?

```python
import math
print(math.ceil(1000/50))
```

**Answer:** 20

Add headroom for bursts. Verified by running it.

### [06 · Rate Limiting Algorithms](level-2/06-rate-limiting.md)

**Q61.** A token bucket with capacity 10 refills 5 tokens/s. A client sends 10 requests at t=0 and 10 more at t=1. How many of the second burst are allowed?

```python
cap,rate=10,5
tokens=cap-10
tokens=min(cap,tokens+rate*1)
print(int(tokens))
```

**Answer:** 5

After the first burst the bucket is empty and refills 5 tokens in a second. Verified by running it.

**Q62.** What does a fixed-window limiter suffer from?

**Answer:** Bursts at window boundaries can double the allowed rate

Sliding windows smooth this.

**Q63.** Where do you enforce rate limits in a distributed system?

**Answer:** At the edge or gateway with a shared counter (for example Redis) keyed by client

Per-instance counters are inconsistent.

**Q64.** What should a rate-limited response include?

**Answer:** 429 and a Retry-After hint

Be explicit.

### [07 · Indexing &amp; Query Patterns](level-2/07-indexing-query-patterns.md)

**Q65.** What does a database index trade?

**Answer:** Faster reads for slower writes and more storage

Index what you query.

**Q66.** What does a composite index (a, b) serve well?

**Answer:** Queries filtering on a, or on a and b (the leftmost prefix)

Order matters.

**Q67.** Why can LIKE '%term' not use a normal B-tree index?

**Answer:** The leading wildcard prevents prefix lookups

Use full-text or trigram indexes.

**Q68.** What is a covering index?

**Answer:** An index containing all columns a query needs, avoiding table lookups

Faster reads.

### [08 · Cache Invalidation Strategies](level-2/08-cache-invalidation.md)

**Q69.** What makes cache invalidation hard?

**Answer:** Knowing when cached data is stale across many readers and writers

Prefer TTLs plus explicit invalidation on writes.

**Q70.** What is write-through caching?

**Answer:** Writes go to the cache and the database together so the cache stays fresh

Costs write latency.

**Q71.** What is a cache stampede and a fix?

**Answer:** Many requests miss at once on a hot key; use single-flight locking or jittered TTLs

Protect the database.

**Q72.** What does versioned keys give you?

**Answer:** Invalidation by changing the version so old entries are never read

Old entries expire naturally.

### [09 · Blob &amp; Object Storage](level-2/09-object-storage.md)

**Q73.** What is object storage good for?

**Answer:** Large, immutable blobs such as images and backups, with high durability and cheap capacity

Examples: S3, GCS.

**Q74.** How do clients upload large files safely?

**Answer:** Direct to storage with pre-signed URLs and multipart uploads

Keeps the app tier light.

**Q75.** A 5 GB file is uploaded in 100 MB parts. How many parts?

```python
import math
print(math.ceil(5_000/100))
```

**Answer:** 50

Parts can be retried independently. Verified by running it.

**Q76.** Why store metadata in a database and bytes in object storage?

**Answer:** Each system does what it is best at, and queries stay fast

Keep a key reference.

### [10 · Project — Design a News Feed](level-2/10-project-news-feed.md)

**Q77.** What is fan-out on write for a news feed?

**Answer:** Pushing a new post into followers' feeds at write time for fast reads

Costly for users with many followers.

**Q78.** What is fan-out on read?

**Answer:** Building a feed at read time by merging followed users' posts

Slower reads, cheaper writes.

**Q79.** How are celebrity accounts typically handled?

**Answer:** A hybrid: push for normal users and pull for accounts with huge follower counts

Avoid write amplification.

**Q80.** A user with 2 million followers posts; fan-out on write creates how many feed entries?

```python
print(2_000_000)
```

**Answer:** 2000000

That write amplification is why hybrids exist. Verified by running it.

## Level 3 · Advanced

### [01 · Consensus Basics: Raft Conceptually](level-3/01-consensus-raft.md)

**Q81.** What problem does consensus solve?

**Answer:** Getting a group of nodes to agree on a value or log order despite failures

Raft and Paxos.

**Q82.** A 5-node Raft cluster needs how many nodes for a majority?

```python
print(5//2+1)
```

**Answer:** 3

It tolerates 2 failures. Verified by running it.

**Q83.** What does a Raft leader do?

**Answer:** Accepts writes and replicates log entries to followers, committing when a majority has them

Followers redirect clients.

**Q84.** Why use an odd number of nodes?

**Answer:** Even counts tolerate no more failures than the next lower odd count

4 nodes tolerate 1, like 3.

### [02 · Distributed Transactions &amp; Sagas](level-3/02-distributed-transactions-sagas.md)

**Q85.** Why are distributed transactions hard?

**Answer:** Partial failures and network delays make atomic commit across services costly and blocking

Avoid when possible.

**Q86.** What is a saga?

**Answer:** A sequence of local transactions with compensating actions to undo earlier steps on failure

Eventual consistency across services.

**Q87.** What is a compensating action for 'charge card'?

**Answer:** Refund the charge

It must itself be idempotent.

**Q88.** What is two-phase commit's main drawback?

**Answer:** Coordinator failure can block participants holding locks

Sagas avoid the lock holding.

### [03 · Idempotency &amp; the Exactly-Once Myth](level-3/03-idempotency-exactly-once.md)

**Q89.** What does idempotent mean for an API?

**Answer:** Repeating the same request has the same effect as sending it once

PUT and DELETE are idempotent by definition.

**Q90.** How do you make a payment POST safe to retry?

**Answer:** Require an idempotency key and store the result per key

The server returns the stored response.

**Q91.** A handler processes messages at-least-once. If message ids [1,2,2,3,1] arrive and duplicates are ignored, how many are processed?

```python
seen=set(); n=0
for i in [1,2,2,3,1]:
    if i not in seen:
        seen.add(i); n+=1
print(n)
```

**Answer:** 3

Deduplicate by a stable id. Verified by running it.

**Q92.** How is 'exactly once' usually achieved in practice?

**Answer:** At-least-once delivery plus idempotent processing

Effectively-once.

### [04 · Event-Driven Architecture &amp; CDC](level-3/04-event-driven-cdc.md)

**Q93.** What does change data capture (CDC) do?

**Answer:** Streams database changes (from the log) to other systems

Debezium reads the write-ahead log.

**Q94.** What is the outbox pattern?

**Answer:** Writing the event to an outbox table in the same transaction as the state, then publishing it reliably

Avoids dual-write inconsistencies.

**Q95.** What is event sourcing?

**Answer:** Storing a sequence of events as the source of truth and deriving state by replay

Gives audit and replay.

**Q96.** What is a downside of event-driven designs?

**Answer:** Harder reasoning about ordering, duplicates and eventual consistency

Invest in observability.

### [05 · Search Systems](level-3/05-search-systems.md)

**Q97.** What data structure powers full-text search?

**Answer:** An inverted index mapping terms to the documents containing them

Built by Lucene, Elasticsearch.

**Q98.** Posting lists: 'cat' -&gt; docs {1,2,4}, 'dog' -&gt; docs {2,3,4}. Which docs match 'cat AND dog'?

```python
print(sorted({1,2,4} & {2,3,4}))
```

**Answer:** [2, 4]

AND intersects posting lists. Verified by running it.

**Q99.** Why keep search indexes separate from the primary database?

**Answer:** Different access patterns, scaling and eventual consistency via CDC

Search is a derived view.

**Q100.** What does ranking (e.g. BM25) do?

**Answer:** Orders matching documents by relevance

Beyond simple matching.

### [06 · Real-Time Systems: WebSockets &amp; Pub/Sub](level-3/06-real-time-systems.md)

**Q101.** Which protocol suits server push to browsers?

**Answer:** WebSockets (or Server-Sent Events for one-way streams)

Persistent connections.

**Q102.** What is a challenge of WebSocket scale?

**Answer:** Holding many long-lived connections, so state and routing need a shared pub/sub backbone

Sticky routing and presence.

**Q103.** What does long polling do?

**Answer:** The client holds a request open until the server has data, then reconnects

Simpler fallback.

**Q104.** What is a presence system?

**Answer:** Tracking which users are online, typically with heartbeats and TTLs

Expire entries on missed heartbeats.

### [07 · Observability](level-3/07-observability.md)

**Q105.** What are the three pillars of observability?

**Answer:** Metrics, logs and traces

Each answers different questions.

**Q106.** A service has an SLO of 99.9% over 30 days. How many minutes of error budget is that? (rounded)

```python
print(round(30*24*60*0.001))
```

**Answer:** 43

43,200 minutes x 0.1% = 43.2. Verified by running it.

**Q107.** What are the golden signals?

**Answer:** Latency, traffic, errors and saturation

Start dashboards there.

**Q108.** Why alert on symptoms rather than causes?

**Answer:** Symptoms indicate user impact and reduce noise

Causes go on dashboards.

### [08 · Reliability Patterns](level-3/08-reliability-patterns.md)

**Q109.** What is a circuit breaker?

**Answer:** Stops calling a failing dependency for a while so it can recover and callers fail fast

Protects resources.

**Q110.** Exponential backoff starting at 100 ms doubling each retry: what is the wait before the 4th retry (ms)?

```python
print(100*2**3)
```

**Answer:** 800

100, 200, 400, 800. Verified by running it.

**Q111.** Why add jitter to retries?

**Answer:** So clients do not retry in lockstep and overload a recovering service

Random spread.

**Q112.** What is a bulkhead?

**Answer:** Isolating resources so one failing component cannot exhaust everything

Separate thread pools or pools per dependency.

### [09 · Hot Keys, Skew &amp; Multi-Tenancy](level-3/09-multi-tenant-hot-keys.md)

**Q113.** What is the noisy neighbour problem?

**Answer:** One tenant consuming shared resources and degrading others

Use quotas and isolation.

**Q114.** What is a hot key?

**Answer:** A key receiving disproportionate traffic that overloads one node

Mitigate with replication or key splitting.

**Q115.** How can you mitigate a hot cache key?

**Answer:** Replicate it across nodes or add a short local cache in front

Split with random suffixes.

**Q116.** How do you isolate tenants?

**Answer:** Quotas, rate limits and, for large tenants, dedicated resources

Fairness and security.

### [10 · Project — Design a Chat System](level-3/10-project-chat-system.md)

**Q117.** What storage and flow does a chat system need for message history?

**Answer:** A durable, ordered store partitioned by conversation plus a push channel for live delivery

Cassandra-style stores fit well.

**Q118.** How do you order messages in a conversation?

**Answer:** A per-conversation sequence number assigned by the server

Clocks disagree.

**Q119.** How do offline users get messages?

**Answer:** From stored history on reconnect, with push notifications as a nudge

Track delivery and read receipts.

**Q120.** 1 million active users send 40 messages/day each of 200 bytes. How many GB per day of raw message data? (10^9)

```python
print(1_000_000*40*200/1e9)
```

**Answer:** 8.0

Plus metadata and replication. Verified by running it.

## Level 4 · Master

### [01 · Case Study: Video Streaming](level-4/01-case-video-streaming.md)

**Q121.** What does a video streaming pipeline do before delivery?

**Answer:** Transcodes into multiple bitrates and segments for adaptive streaming

HLS and DASH.

**Q122.** What is adaptive bitrate streaming?

**Answer:** The player switches quality per segment based on bandwidth

Smooth playback.

**Q123.** Why does a CDN matter most for video?

**Answer:** Bandwidth is huge and latency-sensitive; edges serve the bulk of traffic

Reduces origin cost.

**Q124.** What is the biggest cost driver of a streaming service?

**Answer:** Egress bandwidth and storage, so efficiency and peering matter

Optimise encoding.

### [02 · Case Study: Ride-Hailing Dispatch](level-4/02-case-ride-hailing.md)

**Q125.** What does geospatial indexing (geohash, quadtree, S2) do?

**Answer:** Maps locations to cells so nearby searches are efficient

Find drivers near a rider.

**Q126.** How are driver locations updated at scale?

**Answer:** Frequent lightweight updates into an in-memory geo store, with short TTLs

Redis GEO is common.

**Q127.** What is a trade-off in matching riders and drivers?

**Answer:** Optimal global matching versus fast local greedy matching

Latency matters.

**Q128.** How do you handle a surge of requests in one area?

**Answer:** Shard by region, rate limit and apply dynamic pricing as demand control

Demand shaping.

### [03 · Case Study: Distributed File Storage](level-4/03-case-distributed-file-storage.md)

**Q129.** How are large files stored in a distributed file system?

**Answer:** Split into chunks replicated across nodes, with a metadata service tracking locations

Like GFS/HDFS.

**Q130.** A 1 GB file in 64 MB chunks needs how many chunks? (round up, 1 GB = 1024 MB)

```python
import math
print(math.ceil(1024/64))
```

**Answer:** 16

Each chunk is replicated, commonly 3 times. Verified by running it.

**Q131.** Why is the metadata service a bottleneck risk?

**Answer:** All lookups go through it, so it needs replication, caching and partitioning

Keep it small and fast.

**Q132.** What is deduplication?

**Answer:** Storing identical chunks once, referenced by content hash

Saves space.

### [04 · Case Study: Payment System](level-4/04-case-payment-system.md)

**Q133.** What is the most important property of a payment system?

**Answer:** Correctness and consistency of money movement, including idempotency and auditability

Never lose or double-charge.

**Q134.** Why use a double-entry ledger?

**Answer:** Every transaction records equal debits and credits, making errors detectable

Immutable entries.

**Q135.** What does reconciliation do?

**Answer:** Compares internal records with external provider statements to find mismatches

Run daily.

**Q136.** Why is the idempotency key essential for payment APIs?

**Answer:** Retries after timeouts must not charge twice

Store key to result.

### [05 · Multi-Region &amp; Disaster Recovery](level-4/05-multi-region-dr.md)

**Q137.** What is the difference between multi-AZ and multi-region?

**Answer:** Multi-AZ protects against a data centre failure; multi-region protects against regional failure

Region adds latency and cost.

**Q138.** What are RPO and RTO?

**Answer:** Acceptable data loss and acceptable time to recover

Drive the DR strategy.

**Q139.** Which DR strategy has the fastest recovery at highest cost?

**Answer:** Active-active across regions

Warm standby is in between.

**Q140.** Why test failover regularly?

**Answer:** Untested recovery plans fail when needed

Game days.

### [06 · Cost-Aware Design](level-4/06-cost-aware-design.md)

**Q141.** A service runs 20 instances at $0.10/hour. What is the monthly cost (30 days)?

```python
print(20*0.10*24*30)
```

**Answer:** 1440.0

Rightsize and use autoscaling. Verified by running it.

**Q142.** Which saves most for steady baseline load?

**Answer:** Reserved or committed-use capacity

Use spot for tolerant jobs.

**Q143.** What hidden cost surprises teams?

**Answer:** Cross-AZ and egress data transfer

Design for locality.

**Q144.** What is cost-aware design?

**Answer:** Making cost a design constraint and measuring it per feature or request

Tag and attribute.

### [07 · Security Architecture](level-4/07-security-architecture.md)

**Q145.** What is defence in depth?

**Answer:** Layered controls so one failure does not expose everything

Network, identity, data.

**Q146.** What does least privilege mean?

**Answer:** Granting only the permissions needed

Scope service accounts.

**Q147.** Where do you terminate TLS?

**Answer:** At the edge and re-encrypt internally for sensitive systems (mTLS)

Encrypt in transit.

**Q148.** What belongs in a threat model?

**Answer:** Assets, entry points, attackers and mitigations

STRIDE is one aid.

### [08 · The Interview Framework &amp; Common Mistakes](level-4/08-interview-framework.md)

**Q149.** What is a good order for a design interview?

**Answer:** Requirements, estimates, API and data model, high-level design, deep dives, trade-offs

Show structured thinking.

**Q150.** What do interviewers look for?

**Answer:** Clarifying questions, trade-off reasoning and handling of scale and failure

Justify choices.

**Q151.** What should you do when requirements are ambiguous?

**Answer:** Ask and state assumptions explicitly

Narrow scope.

**Q152.** Why discuss failure modes?

**Answer:** Real systems fail; resilience is part of the design

Mention monitoring.

### [09 · Writing Design Docs &amp; Defending Trade-offs](level-4/09-design-reviews-tradeoffs.md)

**Q153.** What is an ADR?

**Answer:** An architecture decision record capturing context, decision and consequences

Keep a trail.

**Q154.** How should a design review be run?

**Answer:** Share the doc early, focus on risks and trade-offs and record decisions

Blameless and constructive.

**Q155.** What is a good way to compare options?

**Answer:** A table of criteria (cost, risk, complexity, fit) with weights or notes

Make trade-offs visible.

**Q156.** What does reversibility mean for decisions?

**Answer:** Prefer choices that are cheap to change and take care over one-way doors

Two-way vs one-way doors.

### [10 · Capstone — Your Own Design Doc](level-4/10-capstone-design-doc.md)

**Q157.** What belongs in a design doc?

**Answer:** Context, goals and non-goals, design, alternatives, risks and rollout plan

Write for reviewers.

**Q158.** Why include non-goals?

**Answer:** They limit scope and prevent misunderstandings

Be explicit.

**Q159.** Why include a rollout and rollback plan?

**Answer:** Changes can fail; recovery must be planned

Feature flags and canaries.

**Q160.** How do you show a design works?

**Answer:** Capacity estimates, failure analysis and a plan to measure with SLOs

Evidence over assertion.

## Mixed questions across all levels

**Q161.** What does this estimate print for peak QPS and daily storage?

```python
reqs=200_000_000
avg=reqs/86_400
peak=avg*4
store_gb=reqs*0.5/1e6
print(round(avg), round(peak), store_gb)
```

**Answer:** 2315 9259 100.0

Average QPS is requests over 86,400 seconds; peak uses a multiplier; 0.5 KB per request is 100 GB per day. Verified by running it.

**Q162.** A cache with a 95% hit rate has 1 ms hits and 40 ms misses. What is the mean read time (ms)?

```python
print(0.95*1+0.05*40)
```

**Answer:** 2.95

Hit rate dominates average latency. Verified by running it.

**Q163.** With N=5 replicas, which (W, R) pairs guarantee read-after-write overlap (R + W &gt; N)?

```python
N=5
print([(w,r) for w in range(1,6) for r in range(1,6) if r+w>N and w==3])
```

**Answer:** [(3, 3), (3, 4), (3, 5)]

With W=3 any R of at least 3 overlaps every write quorum. Verified by running it.

**Q164.** Hash-modulo sharding from 4 to 5 shards moves what fraction of keys 0..999?

```python
print(sum(1 for k in range(1000) if k%4!=k%5)/1000)
```

**Answer:** 0.8

Modulo resharding remaps most keys; consistent hashing moves about 1/N. Verified by running it.

**Q165.** A 3-node Raft cluster has one node down. Can it still commit writes?

```python
nodes,down=3,1
print(nodes-down >= nodes//2+1)
```

**Answer:** True

Two of three is a majority. Verified by running it.

**Q166.** A service sees double charges after client timeouts and retries. What is the fix?

**Answer:** Require an idempotency key and return the stored result for repeated keys

Retries must be safe.

**Q167.** Which pattern keeps a database write and an event publish consistent without distributed transactions?

**Answer:** The outbox: write the event in the same local transaction and publish it from there

CDC or a relay publishes the outbox.

**Q168.** Why do hot keys matter in sharded caches?

**Answer:** One node absorbs disproportionate traffic and becomes a bottleneck

Replicate or split the key.

**Q169.** What is the right response to a thundering herd on an expired hot key?

**Answer:** Single-flight rebuilds with jittered TTLs so only one request recomputes

Protect the database.

**Q170.** When does a queue between services help most?

**Answer:** To absorb bursts and decouple speeds, accepting asynchronous results

Monitor lag.

**Q171.** Why prefer cursor pagination over offset for large feeds?

**Answer:** Offsets get slow with depth and shift when rows are inserted; cursors are stable

Use an indexed ordering key.

**Q172.** Which consistency does a CP system sacrifice during a partition?

**Answer:** Availability: the minority side stops accepting writes

CAP.

**Q173.** What does PACELC add to CAP?

**Answer:** Even without partitions you trade latency against consistency

Else-case trade-off.

**Q174.** What is the main benefit of virtual nodes in consistent hashing?

**Answer:** More even load and smoother rebalancing

The challenge shows the ratio improving.

**Q175.** A sliding-window limiter stops a boundary burst that a fixed window allows. What is the cost?

**Answer:** More memory and work per client to track timestamps (or approximations)

Token buckets are cheaper and allow controlled bursts.

**Q176.** How do you decide between fan-out on write and on read?

**Answer:** By follower distribution: push for typical users, pull for celebrities (hybrid)

Avoid write amplification.

**Q177.** What does an SLO error budget let you do?

**Answer:** Trade reliability against change velocity with an explicit allowance for failures

Spend it deliberately.

**Q178.** Which belongs in every design doc besides the design itself?

**Answer:** Goals and non-goals, alternatives considered, risks and a rollout/rollback plan

Make decisions reviewable.

**Q179.** What is the weakest justification for a technology choice in a design review?

**Answer:** It is popular or what we already know without evidence it fits the requirements

Tie choices to requirements.

## Practical coding questions

### A consistent-hash ring with virtual nodes

Implement a consistent-hashing ring (bisect over sorted virtual-node points) and use it to show how load balance improves with more virtual nodes, that adding a node moves only about 1/N of the keys (all to the new node) and that removing a node leaves nothing behind. This is a self-contained Python model of the idea used by distributed caches and Dynamo-style stores.

Starter code:

```
import bisect, hashlib

def h(s):
    return int(hashlib.md5(s.encode()).hexdigest(), 16)

# TODO: class Ring(nodes=(), vnodes=100)
#   add(node) / remove(node): place `vnodes` points per node at h(f"{node}#{i}") on a sorted list (use bisect.insort), remember each point's owner
#   lookup(key): the owner of the first point clockwise from h(key) (wrap around with % len)

keys = [f"user:{i}" for i in range(10_000)]

def spread(ring):
    counts = {}
    for k in keys:
        n = ring.lookup(k)
        counts[n] = counts.get(n, 0) + 1
    return counts

ring = Ring(["a", "b", "c", "d"])
before = {k: ring.lookup(k) for k in keys}
c = spread(ring)
print("spread:", {n: c[n] for n in sorted(c)})
for vn in (5, 100, 2000):
    cc = spread(Ring(["a", "b", "c", "d"], vnodes=vn))
    print(f"vnodes={vn:4} max/min load ratio = {max(cc.values()) / min(cc.values()):.2f}")

ring.add("e")
moved = sum(1 for k in keys if ring.lookup(k) != before[k])
print("moved after adding e:", round(100 * moved / len(keys)), "% (ideal 20%)")
only_to_new = all(ring.lookup(k) == "e" for k in keys if ring.lookup(k) != before[k])
print("every moved key went to the new node:", only_to_new)

modulo_moved = sum(1 for k in keys if h(k) % 4 != h(k) % 5)
print("modulo hashing 4->5 would move:", round(100 * modulo_moved / len(keys)), "%")

ring.remove("b")
after_remove = {k: ring.lookup(k) for k in keys}
print("keys left on removed node:", sum(1 for v in after_remove.values() if v == "b"))
```

Expected output:

```text
spread: {'a': 2207, 'b': 2911, 'c': 2726, 'd': 2156}
vnodes=   5 max/min load ratio = 2.74
vnodes= 100 max/min load ratio = 1.35
vnodes=2000 max/min load ratio = 1.09
moved after adding e: 21 % (ideal 20%)
every moved key went to the new node: True
modulo hashing 4->5 would move: 80 %
keys left on removed node: 0
```

??? success "Show a solution"

    ```
    import bisect, hashlib

    def h(s):
        return int(hashlib.md5(s.encode()).hexdigest(), 16)

    class Ring:
        def __init__(self, nodes=(), vnodes=100):
            self.vnodes, self.keys, self.owner = vnodes, [], {}
            for n in nodes:
                self.add(n)

        def add(self, node):
            for i in range(self.vnodes):
                p = h(f"{node}#{i}")
                bisect.insort(self.keys, p)
                self.owner[p] = node

        def remove(self, node):
            for i in range(self.vnodes):
                p = h(f"{node}#{i}")
                self.keys.remove(p)
                del self.owner[p]

        def lookup(self, key):
            i = bisect.bisect(self.keys, h(key)) % len(self.keys)
            return self.owner[self.keys[i]]

    keys = [f"user:{i}" for i in range(10_000)]

    def spread(ring):
        counts = {}
        for k in keys:
            n = ring.lookup(k)
            counts[n] = counts.get(n, 0) + 1
        return counts

    ring = Ring(["a", "b", "c", "d"])
    before = {k: ring.lookup(k) for k in keys}
    c = spread(ring)
    print("spread:", {n: c[n] for n in sorted(c)})
    for vn in (5, 100, 2000):
        cc = spread(Ring(["a", "b", "c", "d"], vnodes=vn))
        print(f"vnodes={vn:4} max/min load ratio = {max(cc.values()) / min(cc.values()):.2f}")

    ring.add("e")
    moved = sum(1 for k in keys if ring.lookup(k) != before[k])
    print("moved after adding e:", round(100 * moved / len(keys)), "% (ideal 20%)")
    only_to_new = all(ring.lookup(k) == "e" for k in keys if ring.lookup(k) != before[k])
    print("every moved key went to the new node:", only_to_new)

    modulo_moved = sum(1 for k in keys if h(k) % 4 != h(k) % 5)
    print("modulo hashing 4->5 would move:", round(100 * modulo_moved / len(keys)), "%")

    ring.remove("b")
    after_remove = {k: ring.lookup(k) for k in keys}
    print("keys left on removed node:", sum(1 for v in after_remove.values() if v == "b"))
    ```

    Each node owns many points on the ring; a key belongs to the first point clockwise from its hash. Adding a node only takes keys from neighbours, unlike modulo hashing which remaps about 80% of keys when going from 4 to 5 nodes. More virtual nodes smooth the load. Verified by running the solution (MD5 is deterministic, so the numbers are reproducible).

### Rate limiters compared on a boundary burst

Implement fixed-window, sliding-window and token-bucket limiters with an explicit clock argument and compare them on a burst straddling a window boundary and on steady and heavy clients. This is a deterministic simulation, not a distributed limiter; production versions keep the counters in a shared store such as Redis.

Starter code:

```
# TODO: FixedWindow(limit, window).allow(t)   counter that resets at each window boundary (start = t - t % window)
# TODO: TokenBucket(capacity, rate).allow(t)   refill (t - last) * rate tokens up to capacity, spend 1 per request
# TODO: SlidingWindow(limit, window).allow(t)  keep timestamps newer than t - window; allow if fewer than limit

# 10 requests per second allowed; a burst of 10 just before a window boundary and 10 right after
burst = [0.90 + i * 0.005 for i in range(10)] + [1.00 + i * 0.005 for i in range(10)]
for name, lim in [("fixed window", FixedWindow(10, 1.0)), ("sliding window", SlidingWindow(10, 1.0)), ("token bucket", TokenBucket(10, 10.0))]:
    ok = sum(lim.allow(t) for t in burst)
    print(f"{name:15} allowed {ok} of {len(burst)}")

# steady 5 req/s for 4 seconds is fine for everyone; a client hammering 50 req/s is limited
steady = [i * 0.2 for i in range(20)]
heavy = [i * 0.02 for i in range(200)]
for name, make in [("fixed window", lambda: FixedWindow(10, 1.0)), ("token bucket", lambda: TokenBucket(10, 10.0))]:
    print(f"{name:15} steady {sum(make().allow(t) for t in steady)}/20  heavy {sum((lambda l: [l.allow(t) for t in heavy])(make()))}/200")
```

Expected output:

```text
fixed window    allowed 20 of 20
sliding window  allowed 10 of 20
token bucket    allowed 11 of 20
fixed window    steady 20/20  heavy 40/200
token bucket    steady 20/20  heavy 49/200
```

??? success "Show a solution"

    ```
    class FixedWindow:
        def __init__(self, limit, window):
            self.limit, self.window, self.start, self.count = limit, window, 0, 0

        def allow(self, t):
            if t - self.start >= self.window:
                self.start, self.count = t - (t % self.window), 0
            if self.count < self.limit:
                self.count += 1
                return True
            return False

    class TokenBucket:
        def __init__(self, capacity, rate):
            self.capacity, self.rate, self.tokens, self.last = capacity, rate, capacity, 0.0

        def allow(self, t):
            self.tokens = min(self.capacity, self.tokens + (t - self.last) * self.rate)
            self.last = t
            if self.tokens >= 1:
                self.tokens -= 1
                return True
            return False

    class SlidingWindow:
        def __init__(self, limit, window):
            self.limit, self.window, self.hits = limit, window, []

        def allow(self, t):
            self.hits = [x for x in self.hits if t - x < self.window]
            if len(self.hits) < self.limit:
                self.hits.append(t)
                return True
            return False

    # 10 requests per second allowed; a burst of 10 just before a window boundary and 10 right after
    burst = [0.90 + i * 0.005 for i in range(10)] + [1.00 + i * 0.005 for i in range(10)]
    for name, lim in [("fixed window", FixedWindow(10, 1.0)), ("sliding window", SlidingWindow(10, 1.0)), ("token bucket", TokenBucket(10, 10.0))]:
        ok = sum(lim.allow(t) for t in burst)
        print(f"{name:15} allowed {ok} of {len(burst)}")

    # steady 5 req/s for 4 seconds is fine for everyone; a client hammering 50 req/s is limited
    steady = [i * 0.2 for i in range(20)]
    heavy = [i * 0.02 for i in range(200)]
    for name, make in [("fixed window", lambda: FixedWindow(10, 1.0)), ("token bucket", lambda: TokenBucket(10, 10.0))]:
        print(f"{name:15} steady {sum(make().allow(t) for t in steady)}/20  heavy {sum((lambda l: [l.allow(t) for t in heavy])(make()))}/200")
    ```

    The fixed window lets 20 requests through in 0.1 seconds because the counter resets at the boundary, the sliding window enforces the true rate, and the token bucket allows a controlled burst (11 here: the full bucket plus a little refill). Verified by running the solution.

---

More free practice: the [System Design Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
