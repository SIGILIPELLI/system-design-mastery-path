---
description: "Multi-region architecture and disaster recovery: RPO and RTO, backup vs pilot light vs warm standby vs active-active, data replication, and failover drills."
---

# 05 · Multi-Region & Disaster Recovery

Everything so far has survived the loss of a server, a rack, or an availability zone. A
**region** — a whole geographic cluster of datacenters — can also become unavailable:
power or cooling failures, network cuts, a bad configuration change pushed region-wide,
or a provider control-plane incident. Whether to design for that, and how far, is a
business decision expressed in two numbers.

## RPO and RTO

- **RPO (recovery point objective)**: how much recently written data you can afford to lose,
  measured in time. "RPO = 5 minutes" means losing up to the last five minutes of writes is
  acceptable in a disaster.
- **RTO (recovery time objective)**: how long the service may be down before it is restored.

Lower numbers cost more — often much more. A payments ledger may need RPO near zero; an
internal analytics dashboard may accept RPO of a day and RTO of a few hours. Set them per
system, with the business, before choosing an architecture.

## The spectrum of strategies

| Strategy | What runs in the second region | Typical RPO | Typical RTO | Cost |
|---|---|---|---|---|
| Backup & restore | Nothing; backups copied there | Hours (last backup) | Hours to days | Lowest |
| Pilot light | Replicated data; minimal or no compute | Minutes (replication lag) | Tens of minutes to hours (scale up compute) | Low |
| Warm standby | Scaled-down full stack, live replication | Seconds to minutes | Minutes | Medium |
| Active-active | Full stack serving live traffic in both | Near zero to seconds (depends on data design) | Seconds to minutes | Highest, and most complex |

The RPO/RTO ranges are indicative, not guarantees; they depend entirely on implementation
and, above all, on practice.

## Data is the hard part

Stateless tiers can simply run in both regions. Data decides the design:

- **Async cross-region replication** (the common choice): writes commit locally and ship to
  the other region. Normal latency is unaffected, but on failover, unreplicated writes are
  lost — your RPO equals your replication lag at the moment of failure. Monitor that lag
  as a first-class metric.
- **Synchronous or consensus-based cross-region writes**: RPO near zero, but every write pays
  a cross-region round trip (Level 2, lesson 4's PACELC). Typically reserved for data where
  loss is intolerable, often with three or more regions so a majority survives losing one.
- **Partitioned "home region" data**: each user or tenant lives in one region (which also
  helps with data-residency rules); their data replicates elsewhere for DR only. Writes
  stay local; cross-region conflicts are avoided because each record has one writer region.

Active-active with **both** regions accepting writes to the *same* records requires
conflict resolution (Level 2, lesson 1 — multi-leader). Prefer designs where any given
record has a single writer region at a time.

## Routing traffic between regions

- **DNS-based** failover or latency routing: simple, but clients and resolvers cache DNS
  records, so switching takes at least the TTL and often longer.
- **Anycast / global load balancers**: one IP announced from many places; traffic shifts
  faster.
- **Client-side**: mobile apps with a list of regional endpoints can fail over themselves.

## Worked example: estimating data loss and capacity on failover

```python
# failover_math.py — RPO exposure and survivor capacity for an N-region deployment
def rpo_exposure(write_rate_per_s, replication_lag_s):
    return write_rate_per_s * replication_lag_s

def survivor_utilization(regions, per_region_util):
    # traffic from the failed region spreads over the survivors
    return per_region_util * regions / (regions - 1)

print("writes at risk with 2s lag at 3,000 writes/s:", rpo_exposure(3_000, 2))
print("writes at risk with 45s lag (lag spike):     ", rpo_exposure(3_000, 45))
for n, util in [(2, 0.45), (2, 0.60), (3, 0.60)]:
    u = survivor_utilization(n, util)
    print(f"{n} regions at {util:.0%} -> survivors at {u:.0%}"
          + ("  <- overloaded" if u > 0.85 else ""))
```

Two lessons fall out. First, RPO is not a constant: a replication lag spike during the
incident (which is common, since incidents stress systems) multiplies data at risk.
Second, **active-active only helps if the survivors can carry the load**: two regions each
running at 60% become one region at 120%. N-region designs must reserve roughly 1/N
headroom — or plan to shed non-critical load during failover.

## Failover and failback procedure

A sketch of a region evacuation:

1. **Decide** — who can declare a regional failover, and on what signals? Automatic
   failover for regional events risks flapping on false alarms; many organizations keep a
   human decision with a well-rehearsed runbook.
2. **Fence the old region** — stop writes there (revoke its write role) to avoid split brain
   if it comes back mid-procedure.
3. **Promote** replicas in the target region; confirm replication position; record what
   may have been lost.
4. **Shift traffic** and scale up the target region.
5. **Reconcile** writes that were accepted in the old region but never replicated, once it
   returns — they may need manual or automated replay.
6. **Fail back** deliberately later, as a planned operation.

## Test it, or it does not exist

A DR plan that has never been exercised usually fails when needed: expired credentials,
missing capacity quotas in the standby region, a dependency hard-coded to the primary
region, a runbook nobody has read. Run **game days** — planned failovers, first in staging,
then in production during low traffic — and measure the achieved RTO and RPO against the
targets.

## How It Actually Works

Regional failures are rarely clean. A region often degrades partially: some services
fail, some are slow, and the control plane you would use to fail over may itself be
impaired. That is why mature designs aim for **static stability**: the standby region should
be able to take traffic *without* needing to create new resources at the moment of the
disaster (because API calls to provision capacity may be the thing that is failing).
Pre-provisioned capacity, pre-established replication, and data-plane-only failover
mechanisms (flip a routing weight, promote an already-running replica) are more reliable
than "launch everything when needed".

Similarly, shared dependencies silently re-couple regions: a single global configuration
service, identity provider, or DNS zone can take down "independent" regions together.
Multi-region design includes an audit of every global dependency.

## Common mistakes

- **Choosing active-active by default** without an RPO/RTO requirement that justifies it.
- **Forgetting that async replication means data loss on failover**, and not telling the
  business.
- **No headroom** in surviving regions.
- **Standby regions that depend on the primary** (images, secrets, config pulled from it).
- **Never testing failover.**
- **Backups never restored in a test** — a backup is only proven by a restore.

## Exercise

1. Run `failover_math.py` and choose per-region utilization targets for 2- and 3-region
   active-active designs that keep survivors under 80% after one region fails.
2. For the chat system from Level 3, choose RPO and RTO, pick a strategy from the table,
   and write the failover runbook (at least eight steps).
3. List every global dependency you can think of in a typical web stack (DNS, certificates,
   identity, CI/CD, container registry, feature flags…) and describe how each would behave
   if the primary region disappeared.
