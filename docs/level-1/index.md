---
description: "Level 1 of System Design: how to approach a design, HTTP, estimation, scaling, load balancing, caching, databases, CDNs, and API styles."
---

# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: build the vocabulary and the habits that every later design rests on. By the end
of this level you should be able to take a vague prompt like "design a pastebin",
turn it into concrete requirements, estimate the load with a few lines of arithmetic,
and sketch a single-region architecture where every box is there for a reason you can
state out loud.

Nothing in this level is exotic. Load balancers, caches, and relational databases run
most of the software you use every day. The point is to understand them well enough to
know *when* each one helps, what it costs, and how it fails.

## Modules

1. [What System Design Is & How to Approach It](01-what-is-system-design.md) — requirements, constraints, and a repeatable process from prompt to diagram
2. [Client-Server & HTTP Basics](02-client-server-http.md) — what really happens between a browser and your server, and why it matters for design
3. [Latency, Throughput & Back-of-Envelope Estimation](03-latency-throughput-estimation.md) — rough numbers every designer should carry, and how to use them
4. [Vertical vs Horizontal Scaling](04-vertical-horizontal-scaling.md) — bigger machines vs more machines, and why statelessness is the price of the second
5. [Load Balancing](05-load-balancing.md) — L4 vs L7, algorithms, health checks, and the load balancer as a single point of failure
6. [Caching Fundamentals](06-caching-fundamentals.md) — cache-aside, hit ratios, eviction policies, and what a cache can never fix
7. [SQL vs NoSQL Basics](07-sql-vs-nosql.md) — choosing by access pattern and consistency needs rather than by fashion
8. [CDNs & Edge Caching](08-cdns.md) — pushing bytes closer to users, cache keys, TTLs, and purges
9. [APIs: REST, gRPC & GraphQL](09-api-styles.md) — the real trade-offs between three API styles, plus pagination and versioning
10. [Project — Design a URL Shortener](10-project-url-shortener.md) — a complete end-to-end design, from requirements to failure modes

## What you need before starting

- Comfort with at least one programming language and with the idea of a web request.
  Code samples are in Python 3 and use only the standard library unless stated.
- Basic familiarity with a relational database (a `SELECT` with a `WHERE` clause).
- No cloud account. Everything that can run locally is written to run locally; where
  something only exists as a managed service, the lesson says so.

By the end of this level you will have produced one full design (the URL shortener)
and a handful of smaller sketches in the exercises. Keep them — you will revisit
several in Level 2 when you add replication and sharding.
