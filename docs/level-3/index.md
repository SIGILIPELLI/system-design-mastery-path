---
description: "Level 3 of System Design: Raft consensus, sagas, idempotency, event-driven design and CDC, search, real-time systems, observability, and reliability."
---

# Level 3 · Advanced <span class="level-badge">Distributed correctness</span>

Levels 1 and 2 were about capacity: how to serve more requests and store more data. Level
3 is about **correctness when things go wrong** — which, at scale, is all the time. Nodes
crash mid-operation, messages arrive twice, clocks disagree, and a slow dependency drags
down everything that calls it.

These lessons cover the mechanisms that let systems agree, coordinate, and recover:
consensus for a single source of truth, sagas for workflows spanning services,
idempotency for surviving retries, event streams for keeping systems in sync, and the
observability and reliability patterns that let you run all of it.

## Modules

1. [Consensus Basics: Raft Conceptually](01-consensus-raft.md) — leader election, log replication, terms, and why majorities matter
2. [Distributed Transactions & Sagas](02-distributed-transactions-sagas.md) — two-phase commit, its blocking problem, and saga compensation
3. [Idempotency & the Exactly-Once Myth](03-idempotency-exactly-once.md) — idempotency keys, dedup windows, and what "exactly once" really means
4. [Event-Driven Architecture & CDC](04-event-driven-cdc.md) — events vs commands, the outbox pattern, and change data capture
5. [Search Systems](05-search-systems.md) — inverted indexes, relevance scoring, indexing pipelines, and sharded search
6. [Real-Time Systems: WebSockets & Pub/Sub](06-real-time-systems.md) — polling vs SSE vs WebSockets, connection gateways, and presence
7. [Observability](07-observability.md) — metrics, logs, traces, SLOs, and error budgets
8. [Reliability Patterns](08-reliability-patterns.md) — timeouts, retries with backoff and jitter, circuit breakers, bulkheads, load shedding
9. [Hot Keys, Skew & Multi-Tenancy](09-multi-tenant-hot-keys.md) — detecting and handling uneven load and noisy neighbours
10. [Project — Design a Chat System](10-project-chat-system.md) — one-to-one and group messaging, delivery guarantees, ordering, and presence

## What you need before starting

- Levels 1–2, particularly replication, partitioning, queues, and CAP/PACELC.
- Python 3 for the simulations (a toy Raft election, a saga orchestrator, a circuit
  breaker). As before, only the standard library is required.

This is the densest level. Expect to reread lessons 1–3; they are the foundation of
every serious distributed-systems discussion.
