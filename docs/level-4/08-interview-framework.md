---
description: "A minute-by-minute framework for 45-minute system design interviews, what interviewers assess at each level, a worked mini-session, and the most common mistakes."
---

# 08 · The Interview Framework & Common Mistakes

A system design interview is a compressed design review: 45–60 minutes, an ambiguous
prompt, a whiteboard or diagramming tool, and an interviewer who is both stakeholder and
reviewer. There is no single correct answer. What is assessed is how you reason: whether
you clarify before building, quantify before choosing, and discuss trade-offs honestly.
Practices vary between companies, so treat this lesson as a robust default rather than a
guarantee of any specific interview's format.

## What interviewers are generally looking for

- **Problem scoping**: turning a vague prompt into concrete requirements.
- **Technical breadth**: knowing which building blocks exist and what each costs.
- **Depth**: going deep on at least one or two components with real mechanisms.
- **Trade-off reasoning**: naming alternatives and explaining why you chose one.
- **Communication**: structured, collaborative, responsive to hints.

Expectations scale with level. A mid-level candidate is expected to produce a sound
working design with some guidance; a senior candidate to drive the conversation, find the
hard parts unprompted, and discuss failure modes and operations; staff-level candidates
are often probed on ambiguity, organizational trade-offs, and evolution over time.

## A 45-minute plan

| Minutes | Phase | Output on the board |
|---|---|---|
| 0–5 | Clarify requirements | Functional list, non-functional targets, explicit out-of-scope |
| 5–10 | Estimate | QPS (read/write), storage, bandwidth — and the conclusion they imply |
| 10–15 | API and data model | 3–5 key endpoints, main entities with keys |
| 15–25 | High-level design | Request path for the 2–3 core operations |
| 25–40 | Deep dives | 1–3 components: data partitioning, the hardest consistency problem, the hot path |
| 40–45 | Wrap-up | Bottlenecks, failure modes, what you would do next with more time |

Say the plan out loud at the start ("I'll clarify requirements, do a quick estimate, then
sketch the design and go deep where it's hardest — does that work for you?"). It signals
structure and invites the interviewer to redirect early.

## Worked example: the first ten minutes of "design a rate-limited API gateway"

A condensed transcript-style sketch of good opening moves:

**Clarify.**

- "Is this a gateway in front of our own microservices, or a product sold to customers?"
  → our own services.
- "What are limits keyed on — API key, user, IP?" → per API key, per endpoint class.
- "Traffic scale?" → about 50,000 requests/s at peak, 200 backend services.
- "How strict must limits be? Is it OK to occasionally allow slightly over the limit?"
  → approximate is fine, but must hold within ~10% over a minute.
- "Other responsibilities?" → authentication, routing; caching is out of scope.

**Estimate.**

"50,000 requests/s; if each gateway instance handles a few thousand requests/s — an
assumption I'd verify with load tests — we need on the order of 20–30 instances plus
headroom. Every request needs a rate-limit check, so a central store would see ~50,000
operations/s: feasible for an in-memory store cluster, but it adds a network hop to every
request. That's the main design tension."

**Direction.**

"So I'll propose local token buckets per instance that sync with a central store every
few hundred milliseconds — accurate to within our tolerance and far fewer round trips.
I'll compare that with a fully central check during the deep dive."

In ten minutes the candidate has scoped the problem, found the core tension with
arithmetic, and set up a trade-off discussion. That is the pattern to practise.

## Deep-dive selection

Pick deep dives by asking: *where would this design fail first at the stated scale, or be
most subtle to get right?* Common candidates:

- The data model and partition key (almost always relevant).
- The consistency-critical operation (payments, bookings, assignment).
- The hottest path (feed reads, redirects, location updates).
- A known hard pattern (fan-out, ordering, exactly-once effects).

If the interviewer steers you somewhere else, follow — they are telling you what they want
to evaluate.

## Communicating trade-offs

Use a consistent shape: **options → criteria → choice → cost of the choice**.

> "For the timeline we could fan out on write or on read. Our reads outnumber writes 20 to
> 1 and we need fast feed loads, so fan-out on write for most users. The cost is write
> amplification for accounts with huge followings, which I'll handle with a hybrid:
> pull for accounts above a threshold."

Stating the cost of your own choice is a strong signal. Every choice has one.

## Common mistakes

1. **Jumping into boxes** before clarifying requirements.
2. **Estimation theatre** — computing numbers and never using them to decide anything.
3. **Buzzword architecture** — Kafka, Kubernetes, microservices, and three databases for a
   problem with 50 requests per second.
4. **Staying shallow everywhere** — ten components, none explained. Depth on a few beats
   breadth on many.
5. **Ignoring failure** — no discussion of what happens when a node, cache, or region fails.
6. **Hand-waving consistency** — "it's eventually consistent" without saying what users see.
7. **Monologuing** — not checking in, missing the interviewer's hints.
8. **Defending a mistake** — when the interviewer points out a flaw, acknowledge it and
   adapt; that is exactly what good engineers do in real reviews.
9. **Claiming specific companies' internals as fact** — "Company X uses exactly this"
   is risky unless well documented publicly; reasoning from requirements is more
   convincing anyway.

## How It Actually Works

The interview format works (to the extent it does) because it compresses the real job's
most important activity — making decisions under uncertainty and explaining them — into a
short session. Interviewers usually have a rubric with dimensions like those listed at the
top, and they take notes on evidence for each. That is why *saying* your reasoning matters:
a good decision made silently produces no evidence. It is also why a design that ends up
different from the interviewer's own mental model can still score very well — the rubric
rewards the quality of reasoning, not agreement.

Time management is the most common practical failure. Candidates who spend 20 minutes on
requirements or estimates run out of time before any deep dive, and the deep dive is where
depth — the dimension most correlated with seniority — is demonstrated. The timeboxes in
the plan above exist to protect it.

## Practice plan

- Do a timed session on each Level 1–4 project prompt, alone, with a 45-minute timer. Record
  yourself or write notes; review where time went.
- Practise out loud with a peer taking the interviewer role and interrupting with "what if
  this component fails?" and "what if traffic grows 10×?".
- Keep a one-page cheat sheet of your own rough numbers and building-block trade-offs —
  written in your own words, from this course's lessons.

## Exercise

1. Run a 45-minute mock on "design a web crawler" using the plan above. Afterwards, write
   down: time spent per phase, the one number that most shaped your design, and the deep
   dive you chose and why.
2. For each of the nine common mistakes, write one sentence you could say in an interview
   that demonstrates the opposite habit.
3. Take your URL-shortener design from Level 1 and present it to someone in exactly 10
   minutes. Note which questions they asked that your design did not answer.
