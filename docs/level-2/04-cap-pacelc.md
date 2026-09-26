---
description: "CAP and PACELC explained correctly: what consistency and availability mean in the theorem, why CA is not a real choice, and latency trade-offs."
---

# 04 · CAP & PACELC, With Nuance

The CAP theorem is quoted in almost every system design discussion and misquoted in
most of them. "Pick two of three" is a slogan, not the theorem. This lesson states what
CAP actually says, where it applies, and why PACELC is more useful for everyday
design decisions.

## The definitions matter

CAP (conjectured by Eric Brewer, proved in a formal model by Gilbert and Lynch) uses
narrow definitions:

- **Consistency (C)** means **linearizability**: every read returns the most recent
  completed write, as if there were a single copy of the data and operations happened
  instantaneously in some order consistent with real time. This is *not* the "C" in ACID,
  which is about integrity constraints.
- **Availability (A)** means **every request received by a non-failed node must receive
  a non-error response** — eventually, with no bound on time. A system that answers 99.99%
  of requests is still not "available" in the CAP sense if some non-failed node refuses.
- **Partition tolerance (P)** means the system keeps operating even when the network
  drops or delays messages between nodes arbitrarily.

## What the theorem says

> If the network partitions, a system must choose between consistency and availability
> for requests that arrive during the partition.

Picture two replicas, X and Y, and a partition cuts them apart. A client writes to X.
Another client reads from Y. Y cannot know about the write. It can:

- answer with its (possibly stale) value → **available but not linearizable**, or
- refuse or wait until the partition heals → **linearizable but not available**.

There is no third option. That is the whole theorem.

## Why "CA" is not a real category

In any system spread over a network, partitions are not optional — cables fail, switches
misbehave, garbage-collection pauses make nodes look unreachable, and cloud networks have
bad minutes. You do not get to "choose not to have P". So for a distributed system the
real question is only: **when a partition happens, do we give up C or A?**

A single-node database is "CA" only in the trivial sense that there is no network between
replicas to partition. Once you add a replica, you are making the CP-vs-AP choice whether
you realize it or not — for instance, async failover that may lose writes has chosen
availability over consistency.

## It is per-operation, not per-product

Real systems rarely sit cleanly in one bucket:

- Many databases let you choose per request: a quorum or leader read (closer to CP) vs a
  read from any replica (closer to AP).
- A system can be CP for account balances and AP for product recommendations.
- Most systems also aren't strictly linearizable even without partitions, and many are
  not strictly available either. Labeling a product "CP" or "AP" is a rough description at
  best. Check what guarantees each operation actually provides.

## PACELC: the everyday trade-off

Partitions are relatively rare. Most of the time the network is fine, and a different
trade-off dominates. Daniel Abadi's PACELC formulation states:

> If there is a **P**artition, choose **A** or **C**; **E**lse (normal operation), choose
> **L**atency or **C**onsistency.

To make a write strongly consistent across replicas, you must wait for coordination —
a round trip to other replicas, possibly in other regions. To make it fast, you
acknowledge early and replicate afterward, accepting that some reads may be stale. That
latency-vs-consistency choice is paid on *every* request, not just during failures, so
it usually matters more to users.

| Configuration | During partition | Normal operation |
|---|---|---|
| Single leader, sync replication to a quorum | Minority side unavailable (PC) | Pays replication latency (EC) |
| Async replicas, read from any replica | Replicas keep serving (PA) | Fast but possibly stale (EL) |
| Dynamo-style with tunable quorums | Depends on R/W settings | Depends on R/W settings |

## Worked example: choosing for three features

An online store spans two regions.

1. **Inventory decrement at checkout.** Overselling a limited item is costly. Choose
   consistency: route stock decrements for an item to one authoritative leader. During a
   partition, the region that cannot reach the leader refuses checkout for that item (or
   offers "reserve and confirm later"). Accept extra latency for remote shoppers.
2. **Product reviews.** Showing a review a few seconds late harms nobody. Choose latency
   and availability: write locally, replicate asynchronously, read locally.
3. **Shopping cart.** Losing an "add to cart" is annoying; briefly showing a cart missing
   an item is tolerable. An AP design with merge-on-conflict (union of items, with
   tombstones for removals) keeps carts writable during a partition.

The design is not "a CP system" or "an AP system". It is a set of per-feature decisions
justified by the cost of each kind of error.

## Consistency models, briefly

Between linearizable and "anything goes" lies a spectrum:

- **Linearizable** — behaves like one copy in real-time order.
- **Sequential** — all nodes see operations in the same order, not necessarily real time.
- **Causal** — operations that are causally related are seen in order by everyone (a
  reply never appears before its question); concurrent ones may differ.
- **Read-your-writes / monotonic reads** — session guarantees (lesson 1).
- **Eventual** — if writes stop, replicas converge eventually. No bound on when, and no
  ordering promise in the meantime.

Causal consistency is notable because it can be provided while remaining available
during partitions, which makes it a useful middle ground.

## How It Actually Works

The proof is an argument about information flow. A linearizable read on replica Y must
reflect any write that completed before the read began. If a write completed on X during
a partition, the only way Y can know about it is a message from X, which the partition
blocks. So Y must either answer without that knowledge (violating linearizability) or
wait/refuse (violating availability). No protocol cleverness can move information across
a link that is not delivering messages.

The latency side of PACELC is the same argument without a partition: for Y to be sure it
has the latest write, some message must travel between replicas *during* the operation.
A quorum-based system, for example, needs a majority of replicas to acknowledge a write
before it counts as committed. If those replicas are in different regions, that is at
least one cross-region round trip — tens to hundreds of milliseconds — added to the write.
Physics sets the lower bound; design only chooses whether to pay it.

## Common mistakes

- **"We chose CA."** Not an option for a replicated system.
- **Confusing CAP consistency with ACID consistency.**
- **Treating CAP as a product label** instead of a per-operation property.
- **Ignoring PACELC** — optimizing for the rare partition while ignoring the latency cost
  paid on every request.
- **Assuming "eventual" means "fast".** It means "unbounded".

## Exercise

For a ride-hailing app with riders and drivers in one country, served from two regions:

1. Classify each operation — assigning a driver to a ride, updating a driver's location,
   showing ride history, charging the fare — as needing linearizability or tolerating
   staleness. Justify each with the cost of the error.
2. For the operation you marked most critical, describe precisely what happens during a
   network partition between regions.
3. Explain to a colleague, in under 150 words, why "our database is AP so we're highly
   available" is an incomplete statement.
