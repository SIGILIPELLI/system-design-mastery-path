---
description: "Capstone: write a complete design doc for a system you choose, with suggested prompts, milestones, a 100-point scoring rubric, and a self-review checklist."
---

# 10 · Capstone — Your Own Design Doc

The capstone is a complete design document for a system of your choice, written to the
standard of a real engineering review. It should take several focused sessions, not one
sitting. When finished, it is a portfolio piece: concrete evidence that you can take an
ambiguous problem to a defensible, reviewable design.

## Choose a system

Pick something with at least one genuinely hard part. Either choose from these prompts or
propose your own of similar scope:

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

Avoid prompts identical to this course's projects (URL shortener, news feed, chat) unless
you substantially extend them.

## Deliverable

A document following the structure from lesson 9, typically 6–15 pages, containing:

1. **Summary** — the proposal and its central trade-off in one paragraph.
2. **Context, goals, non-goals.**
3. **Requirements** — functional; non-functional with numbers (percentile latency,
   availability, durability, RPO/RTO, cost target).
4. **Estimates** — traffic, storage, bandwidth, with every assumption labeled as such; state
   explicitly which conclusion each number drives.
5. **API** — the key endpoints or messages, with idempotency marked.
6. **Data model** — entities, keys, partition keys, indexes, and which store holds each.
7. **Architecture** — a diagram plus narrated request paths for the 2–3 core operations.
8. **Deep dives** — at least two, each with alternatives and a trade-off table.
9. **Consistency and correctness** — for each important operation, the guarantee provided
   and what users see under concurrency and failure.
10. **Failure modes** — a table: component, failure, detection, user impact, mitigation,
    recovery.
11. **Observability** — SLIs/SLOs, key dashboards, the alerts that page.
12. **Security** — trust boundaries, authN/authZ model, sensitive data handling.
13. **Cost** — dominant cost term and a unit-cost estimate (with placeholder prices clearly
    marked if you do not look up real ones).
14. **Evolution** — what changes at 10× scale; what you would build first (milestones).
15. **Open questions and risks.**

Optional but valuable: a small runnable prototype of the hardest mechanism (in the spirit
of this course's Python snippets) — for example, a simulation of your seat-hold expiry, your
scheduler's leader election, or your notification dedup.

## Suggested milestones

1. **Session 1** — choose the prompt; write requirements, non-goals, and estimates. Stop and
   ask: which number decides the architecture?
2. **Session 2** — API, data model, first architecture diagram, request paths.
3. **Session 3** — deep dives with alternatives; consistency analysis.
4. **Session 4** — failure modes, observability, security, cost.
5. **Session 5** — self-review with the rubric; get a peer review if possible; revise.

## Scoring rubric (100 points)

Score yourself honestly, or ask a peer to. Each band describes what earns the points.

| Dimension | Points | Full marks look like |
|---|---|---|
| Requirements & scope | 10 | Specific, quantified NFRs; sensible non-goals; ambiguities resolved explicitly |
| Estimation | 10 | Clear assumptions; correct orders of magnitude; conclusions explicitly drawn and used |
| API & data model | 10 | Entities, keys, and partition keys justified by access patterns; idempotency addressed |
| Architecture | 15 | Every component justified by a requirement; request paths narrated; no unexplained boxes |
| Deep dives & trade-offs | 20 | ≥2 deep dives with real mechanisms; genuine alternatives; weighted trade-off tables; costs of chosen options stated |
| Correctness & failure | 15 | Guarantees stated per operation; failure-mode table with detection and recovery; no silent data loss unaddressed |
| Operations, security, cost | 10 | SLOs and alerts; trust boundaries and authZ; dominant cost term identified |
| Communication | 10 | Summary-first; readable diagrams; concise; honest about weaknesses and open questions |

Rough interpretation: **85+** — ready to present in a senior-level design review;
**70–84** — solid, with identifiable gaps to close; **below 70** — revisit the lessons
corresponding to your weakest dimensions and revise.

## Worked example: scoring a weak and a strong answer on one dimension

**Dimension: Correctness & failure — for a ticket-booking design.**

*Weak (5/15):* "We use a database with transactions so seats won't be double-booked. If
the server fails, the load balancer routes to another one."

*Strong (14/15):* "Seat holds are rows with `(event_id, seat_id)` unique and a
`hold_expires_at`. Placing a hold is a conditional insert; two buyers racing for a seat
cannot both succeed. Payment runs as a saga: hold → payment intent (idempotent key =
hold ID) → confirm. If payment times out, the intent is resolved by querying the
processor; the hold is extended once while resolving, then released. If the booking
service crashes after payment but before confirmation, a recovery job finds `SUCCEEDED`
intents with unconfirmed holds and confirms them. Failure table rows cover the database
primary (bookings pause, ~30 s failover, holds preserved), the queue (confirmation emails
delayed, not lost — outbox), and the payment processor outage (new checkouts fail fast via
circuit breaker; existing holds extended)."

The difference is not length; it is that every claim names a mechanism, and every failure
names what the user experiences.

## A self-review checklist

Before calling the doc finished:

- [ ] Could a reader state my central trade-off after reading only the summary?
- [ ] Does every component in the diagram trace back to a requirement?
- [ ] Is every number in the estimates used for at least one decision?
- [ ] For each write operation: is it idempotent, and what happens on retry?
- [ ] For each read path: what is the staleness bound, and is it acceptable?
- [ ] For each stateful component: what happens when it fails, and how would we know?
- [ ] Is any "exactly once", "real time", or "infinitely scalable" claim left unqualified?
- [ ] Are back-of-envelope figures labeled as rough assumptions rather than facts?
- [ ] Have I claimed how a specific real company built something? If so, is it public and
      hedged — or should it be removed?

## How It Actually Works

A capstone like this is effective practice because it exercises the full loop of design
work — requirements, estimation, decision, justification, failure analysis — in writing,
where gaps are visible. Interviews and reviews test the same loop under time pressure; the
capstone builds the underlying fluency without it. The rubric's weights mirror what
reviewers tend to care about most: whether the design is *correct under failure* and
whether its *trade-offs are explicit*, rather than how many technologies appear on the
diagram.

## Common mistakes

- **Picking a system with no hard part**, producing a CRUD app with a load balancer.
- **Technology-first writing** ("we'll use Kafka") instead of requirement-first.
- **A failure-mode section that lists failures but not user impact or detection.**
- **No alternatives**, or alternatives dismissed in one line.
- **Never getting outside review.**

## Exercise

This lesson's exercise *is* the capstone:

1. Choose a prompt and write the full design doc.
2. Build a small prototype or simulation of its hardest mechanism.
3. Score it with the rubric; revise until you reach at least 80.
4. If possible, have a peer score it independently and discuss every dimension where your
   scores differ by more than 3 points.
