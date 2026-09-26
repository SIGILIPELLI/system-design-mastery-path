---
description: "SQL vs NoSQL by access pattern: relational, key-value, document, wide-column, and graph stores, transactions, and how to choose without hype."
---

# 07 · SQL vs NoSQL Basics

"Should we use SQL or NoSQL?" is a poorly formed question. "NoSQL" covers several very
different families of database, and modern relational databases have absorbed many
features (JSON columns, replication, partitioning) that once distinguished them. The
useful question is: **what are the access patterns, what consistency do they need, and
how will the data grow?**

## The families

| Family | Model | Typical strengths | Typical costs |
|---|---|---|---|
| Relational (PostgreSQL, MySQL) | Tables, rows, joins, SQL | Ad-hoc queries, joins, multi-row ACID transactions, constraints | Horizontal write scaling needs sharding work |
| Key-value (Redis, DynamoDB-style) | key → opaque value | Very fast lookups by key, simple to partition | Queries other than by key are hard |
| Document (MongoDB-style) | key → JSON-like document | Nested data read together, flexible schema | Cross-document joins and constraints are weaker |
| Wide-column (Cassandra-style) | Partition key → sorted rows | High write throughput, time series, huge scale | Must model tables around queries; limited ad-hoc querying |
| Graph (Neo4j-style) | Nodes and edges | Multi-hop relationship queries | Niche; scaling across machines is hard |
| Search (Elasticsearch/OpenSearch) | Inverted index | Full-text and faceted search | Usually a secondary index, not the source of truth |

The product names are examples of the family, not endorsements; features differ
between products and versions, so check the specific system's documentation.

## Start with the access patterns

Write down every important query before choosing:

```text
Q1  get order by order_id                                 (very frequent)
Q2  list a customer's orders, newest first, paginated      (frequent)
Q3  revenue per product per day for the finance report     (daily batch)
Q4  place an order: insert order + decrement stock atomically
```

- Q1 is a key lookup — any database handles it.
- Q2 is "partition by customer, sort by time" — natural in both a relational table with
  an index on `(customer_id, created_at)` and a wide-column table keyed by customer.
- Q3 is an aggregation over many rows — relational or a separate analytics store.
- Q4 needs an atomic multi-row update — native in relational databases; in many NoSQL
  stores it needs conditional writes, limited transactions, or a redesign.

Here a relational database fits naturally. If instead the dominant workload were
"append 500,000 sensor readings per second and read the last hour per sensor", a
wide-column or time-series store would fit better, and the relational database would
need significant partitioning effort.

## Worked example: modeling the same data two ways

A blog with posts and comments.

**Relational (normalized):**

```sql
CREATE TABLE posts (
  post_id    BIGINT PRIMARY KEY,
  author_id  BIGINT NOT NULL,
  title      TEXT NOT NULL,
  body       TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL
);
CREATE TABLE comments (
  comment_id BIGINT PRIMARY KEY,
  post_id    BIGINT NOT NULL REFERENCES posts(post_id),
  author_id  BIGINT NOT NULL,
  body       TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL
);
CREATE INDEX comments_by_post ON comments(post_id, created_at);

-- A post page:
SELECT * FROM posts WHERE post_id = 7;
SELECT * FROM comments WHERE post_id = 7 ORDER BY created_at LIMIT 50;
```

**Document (denormalized):**

```json
{
  "_id": 7,
  "author_id": 3,
  "title": "Why queues",
  "body": "...",
  "created_at": "2026-09-01T10:00:00Z",
  "comments": [
    {"author_id": 9, "body": "Nice", "created_at": "2026-09-01T11:00:00Z"}
  ]
}
```

The document version reads a post page in one fetch. But a post with 40,000 comments
becomes a huge document rewritten on every new comment, and "all comments by user 9"
requires scanning every post. A common compromise: keep recent or top comments embedded
and store the full list in a separate collection. The relational version handles both
queries with indexes but needs two queries (or a join) per page.

Neither is "right"; each makes some queries cheap and others expensive. Choosing a
model *is* choosing which queries you are optimizing for.

## Transactions and consistency

Relational databases offer **ACID** transactions:

- **Atomicity** — all changes in a transaction apply, or none do.
- **Consistency** — constraints (foreign keys, uniqueness, checks) hold after commit.
- **Isolation** — concurrent transactions do not see each other's partial work (to a
  degree set by the isolation level).
- **Durability** — once committed, the change survives a crash.

Many NoSQL systems historically offered weaker guarantees (single-item atomicity,
eventual consistency between replicas) in exchange for easier partitioning. Several now
offer transactions too, often with limits on scope or performance. Always check what a
specific database guarantees rather than assuming by category.

## How It Actually Works

**Why relational databases are hard to scale for writes.** A single-node relational
database gives you joins and transactions cheaply because all the data is on one
machine: a transaction takes locks (or uses multi-version concurrency control) in one
memory space and commits by flushing one write-ahead log. Spread the rows across
machines and every cross-machine transaction needs a coordination protocol (Level 3),
every join may need network round trips, and a uniqueness constraint must be checked
across partitions. Those costs are why sharded relational setups usually restrict
transactions and joins to one shard.

**Why key-value and wide-column stores scale out easily.** They give up cross-key
operations by design. If every operation touches exactly one partition key, the system
can hash the key to a node and never coordinate between nodes. Wide-column stores
typically store each partition's rows sorted on disk (often using a log-structured
merge tree), so "latest N rows for this key" is a sequential read. The price: queries
that do not start from the partition key require a separate table or index, which you
must keep in sync yourself.

**Storage engines.** Many relational engines use **B-trees**, which keep data sorted in
pages and update them in place — good for reads and range scans. Many write-heavy stores
use **LSM trees**, which buffer writes in memory, flush them as sorted immutable files,
and merge those files in the background — good for write throughput, at the cost of
reads sometimes checking several files. Level 2's indexing lesson goes deeper.

## Common mistakes

- **Choosing NoSQL "for scale"** at a scale a single relational database handles easily,
  then rebuilding joins and constraints in application code.
- **Choosing relational and then storing everything in one JSON column**, giving up
  the benefits you chose it for.
- **Not listing queries first.** Especially fatal with wide-column stores, where tables
  must be designed per query.
- **Assuming "eventually consistent" means "consistent within milliseconds".** It means
  no bound is guaranteed.
- **Using the search index as the primary store.**

## Exercise

For each system, list its three most important queries, choose a database family, and
justify it in two sentences. Name one query your choice makes painful.

1. A ride-sharing trip history ("show my last 20 trips").
2. A bank's ledger of account movements.
3. A product catalog with highly variable attributes per category.
4. A "people you may know" feature based on friends-of-friends.
5. An IoT platform ingesting temperature readings every second from 2 million devices.
