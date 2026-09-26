---
description: "Cost-aware system design: where cloud spend goes, unit economics per request and per user, right-sizing, storage tiering, egress, and cost as a design constraint."
---

# 06 · Cost-Aware Design

Every design in this course has a price, and the price is part of whether the design is
good. A system that meets its SLOs at ten times the necessary cost is a poor design — and
at scale, cost trade-offs are among the most consequential decisions engineers make.
This lesson treats cost the same way as latency: estimate it, find the dominant term, and
design around it.

## Where the money usually goes

In typical cloud-hosted systems, spend clusters into a few categories:

| Category | Drivers | Typical levers |
|---|---|---|
| Compute | Instance hours, over-provisioning, idle capacity | Right-sizing, autoscaling, spot/interruptible for batch, efficient code |
| Storage | GB-months, replicas, snapshots, retention | Tiering, lifecycle deletion, compression, dedup |
| Data transfer | Egress to the internet, cross-region, sometimes cross-zone | CDNs, compression, keeping chatty traffic local |
| Managed services | Per-request or per-capacity pricing of databases, queues, etc. | Batching, caching, choosing capacity vs on-demand modes |
| Observability | Log volume, metric cardinality, trace retention | Sampling, log levels, retention tiers |

Prices vary by provider, region, and contract, and change over time — so this lesson
uses **ratios and structure**, not specific price quotes. Always look up current pricing
for your provider before committing a design.

## Unit economics

Total monthly spend is hard to reason about. **Cost per unit of value** is not:

- cost per 1,000 requests;
- cost per monthly active user;
- cost per GB stored, per video-hour streamed, per order processed.

Once you know a unit cost, you can compare it to revenue per unit, predict how spend scales
with growth, and evaluate a design change by its effect on the unit cost.

## Worked example: a cost model you can change

The script below uses **placeholder prices** (clearly not real quotes) to show the method:
model each component as usage × unit price, then find the dominant term.

```python
# cost_model.py — structure of a monthly cost estimate; prices are PLACEHOLDERS
PRICES = {                       # illustrative only — replace with your provider's current rates
    "vm_hour": 0.10,             # per app instance-hour
    "storage_gb_month": 0.02,    # object storage, hot tier
    "egress_gb": 0.05,           # internet egress via CDN
    "db_hour": 0.50,             # managed database instance-hour
    "log_gb": 0.50,              # log ingestion
}
HOURS = 730

def monthly(app_instances, db_instances, stored_tb, egress_tb, logs_gb):
    items = {
        "app compute": app_instances * HOURS * PRICES["vm_hour"],
        "database":    db_instances * HOURS * PRICES["db_hour"],
        "storage":     stored_tb * 1000 * PRICES["storage_gb_month"],
        "egress":      egress_tb * 1000 * PRICES["egress_gb"],
        "logging":     logs_gb * PRICES["log_gb"],
    }
    total = sum(items.values())
    for k, v in sorted(items.items(), key=lambda kv: -kv[1]):
        print(f"  {k:12s} {v:>10,.0f}  ({v / total:.0%})")
    return total

print("baseline:")
base = monthly(app_instances=40, db_instances=3, stored_tb=200, egress_tb=500, logs_gb=20_000)
print("after: images re-encoded 30% smaller, debug logs sampled 10x, app right-sized:")
new = monthly(app_instances=28, db_instances=3, stored_tb=160, egress_tb=350, logs_gb=2_000)
print(f"change: {new / base - 1:+.0%}")
```

With these placeholder numbers, egress dominates and logging — easy to overlook — comes
second. Shrinking media (fewer bytes per view) and sampling logs each save several times
more than right-sizing compute does. Change the prices to your provider's
real ones and the ranking may change, which is the point: **find your dominant term before
optimizing**.

## Design levers, by category

**Compute**

- Right-size instances to measured utilization; many fleets run at low average CPU.
- Autoscale stateless tiers on real load signals; scale to zero where feasible.
- Use interruptible/spot capacity for fault-tolerant batch work (transcoding, analytics)
  that can be retried — never for things that cannot be interrupted.
- Performance work *is* cost work: halving CPU per request halves the fleet.

**Storage**

- Lifecycle policies: hot → infrequent → archive → delete (Level 2, lesson 9).
- Retention that matches actual need — logs rarely need to stay hot for a year.
- Compression and efficient formats (columnar formats for analytics data).
- Erasure coding vs replication for cold data.

**Data transfer**

- Serve static and media bytes via CDN; compress text responses.
- Keep chatty service-to-service traffic within a zone or region where your provider
  charges for crossing those boundaries (check the pricing model).
- Avoid cross-region replication of data that does not need it.

**Architecture**

- Caching trades memory cost for database cost — worth it only when hit ratios are high.
- Async and batch processing smooth peaks, so you provision for the average plus buffer
  instead of the peak.
- Managed services cost more per unit but save engineering time; self-hosting saves money
  only if your team's time is cheaper than the difference. Count both.

## Cost as a requirement

Put cost into design docs alongside SLOs: "Target: under X per 1,000 feed loads at launch
scale; under Y at 10× scale." Then each design alternative gets a cost estimate, and
reviewers can weigh "40 ms faster p99" against "35% more expensive" explicitly.

## How It Actually Works

Cloud costs follow the same arithmetic as capacity planning, multiplied by a price. The
reason cost optimization so often focuses on a few items is that cost distributions across
components are usually heavily skewed — much like request traffic across keys. A system's
bill typically has one or two dominant lines, and changes to anything else barely register.
The capacity-planning habit from Level 1 (estimate, find the biggest number, design for
it) transfers directly.

Idle capacity is the other hidden driver. A service provisioned for peak and running at
15% average utilization is paying for 85% idle. Autoscaling, batching, and queues all
work by moving provisioned capacity closer to actual demand — which is also why
utilization targets for cost and headroom targets for reliability (lesson 5) pull in
opposite directions and must be balanced deliberately.

## Common mistakes

- **Optimizing compute** when egress or storage dominates.
- **Quoting prices from memory** — they vary and change.
- **Unbounded log and metric retention.**
- **Cross-zone or cross-region chatter** nobody noticed on the architecture diagram.
- **Spot capacity for stateful or latency-critical services.**
- **Ignoring engineering time** when comparing managed and self-hosted options.

## Exercise

1. Look up your cloud provider's current prices for the five placeholders and rerun
   `cost_model.py`. Which component dominates now?
2. Estimate the monthly cost per 1,000 DAU for the news feed from Level 2, listing each
   assumption. Which single design change would reduce it most?
3. Write a "cost" section for the URL-shortener design from Level 1: dominant cost at 10×
   scale, and two ways to cut it by at least 30%.
