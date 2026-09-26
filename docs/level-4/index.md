---
description: "Level 4 of System Design: case studies in streaming, dispatch, file storage and payments, multi-region DR, cost, security, interviews, and design docs."
---

# Level 4 · Master <span class="level-badge">Whole systems</span>

The first three levels built a toolbox. Level 4 is about judgement: combining those tools
into complete systems, weighing cost and risk, and communicating the result to other
engineers — in a design review or an interview.

The four case studies are **reasoning exercises**, not reports of how any particular
company built its product. Each describes a plausible generic design derived from
requirements, and each makes its assumptions explicit. Where the lesson mentions publicly
documented industry practice, it is hedged as such; internal architectures change and are
rarely fully public.

## Modules

1. [Case Study: Video Streaming](01-case-video-streaming.md) — upload, transcoding pipelines, adaptive bitrate, and CDN-heavy delivery
2. [Case Study: Ride-Hailing Dispatch](02-case-ride-hailing.md) — geospatial indexing, location ingestion, matching, and trip state
3. [Case Study: Distributed File Storage](03-case-distributed-file-storage.md) — chunking, dedup, metadata, sync, and conflict handling
4. [Case Study: Payment System](04-case-payment-system.md) — ledgers, idempotency, external processors, and reconciliation
5. [Multi-Region & Disaster Recovery](05-multi-region-dr.md) — RPO/RTO, active-passive vs active-active, and failover drills
6. [Cost-Aware Design](06-cost-aware-design.md) — where money goes, unit economics, and trade-offs that save it
7. [Security Architecture](07-security-architecture.md) — authentication, authorization, secrets, encryption, and threat modeling
8. [The Interview Framework & Common Mistakes](08-interview-framework.md) — a minute-by-minute plan for a 45-minute round
9. [Writing Design Docs & Defending Trade-offs](09-design-reviews-tradeoffs.md) — structure, alternatives, and running a review
10. [Capstone — Your Own Design Doc](10-capstone-design-doc.md) — a full design of a system you choose, with a scoring rubric

## What you need before starting

- Levels 1–3. The case studies use vocabulary from every earlier lesson without
  re-explaining it.
- A notebook or document for your own designs. Level 4 has more writing than code; the
  skill being practised is explaining a design clearly enough that someone else could
  build it and critique it.

Finish with the capstone. It is the best evidence — for yourself or for a hiring panel —
that you can do this work.
