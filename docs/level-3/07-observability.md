---
description: "Observability for distributed systems: metrics, structured logs, distributed traces, RED and USE methods, SLIs, SLOs, error budgets, and alerting on symptoms."
---

# 07 · Observability

A design is not finished when it works; it is finished when you can tell *whether* it is
working, and *why not* when it isn't. In a distributed system a single user request might
touch fifteen services, three caches, and two queues. When it is slow, "check the logs" is
not a plan. Observability is the set of signals and practices that let you answer new
questions about a running system without shipping new code.

## The three signal types

**Metrics** — numeric time series: request rate, error count, latency histograms, queue
depth, CPU. Cheap to store and query, ideal for dashboards and alerts. They tell you
*that* something is wrong and roughly where.

**Logs** — timestamped records of discrete events. Rich detail, expensive at volume. Make
them **structured** (JSON with consistent fields) so they can be queried, and include a
request or trace ID in every line.

**Traces** — the path of one request through the system, as a tree of **spans** (one per
operation) with timings and parent-child relationships. They answer "where did the 900 ms
go?" in a way neither metrics nor logs can.

```text
trace 7f3a…  GET /checkout                                   920 ms
├─ auth.verify_token                                          12 ms
├─ cart.get                                                   35 ms
│  └─ redis GET cart:42                                        2 ms
├─ pricing.quote                                             810 ms   ← here
│  ├─ db SELECT promotions                                   790 ms   ← missing index
│  └─ tax.calculate                                           15 ms
└─ render                                                     40 ms
```

## What to measure: RED and USE

**RED** for every service (request-driven work):

- **R**ate — requests per second.
- **E**rrors — failed requests per second (or ratio).
- **D**uration — latency distribution (percentiles from histograms, not averages).

**USE** for every resource (CPU, memory, disk, connection pools, queues):

- **U**tilization — fraction of time busy / capacity used.
- **S**aturation — work waiting (queue length, pool wait time).
- **E**rrors — resource-level errors.

Together they cover "is the service healthy?" and "is it starved of something?". Add a
handful of **business metrics** (orders placed per minute, messages delivered) — they often
reveal problems that technical metrics miss, like a silent bug that returns 200 but saves
nothing.

## SLIs, SLOs, and error budgets

- An **SLI** (service level indicator) is a measured ratio of good events: "fraction of
  checkout requests that succeed in under 500 ms".
- An **SLO** (objective) is the target: "99.9% over a rolling 28 days".
- The **error budget** is the allowed shortfall: 0.1% of requests — about 40 minutes of
  full outage-equivalent per 28 days.

The budget turns reliability into a decision tool: while budget remains, ship features
faster; when it is exhausted, prioritize reliability work. It also stops the impossible
goal of 100%, which no system achieves and which users cannot perceive past a point
(their own networks and devices fail more often).

## Worked example: percentiles, SLO burn, and a trace ID

```python
# observability_demo.py — histogram percentiles, SLO compliance, burn rate
import random, statistics, uuid, json, time

rng = random.Random(9)
# 10,000 request latencies: mostly fast, a slow tail, plus 0.3% errors
requests = []
for _ in range(10_000):
    latency = rng.lognormvariate(3.8, 0.45)            # ms, median ≈ 45
    if rng.random() < 0.02:
        latency += rng.uniform(300, 1500)              # slow dependency tail
    ok = rng.random() > 0.003
    requests.append((latency, ok))

lat = sorted(l for l, _ in requests)
pct = lambda p: lat[int(p / 100 * (len(lat) - 1))]
print(f"mean {statistics.mean(lat):.0f} ms | p50 {pct(50):.0f} | "
      f"p99 {pct(99):.0f} | p99.9 {pct(99.9):.0f}")

# SLI: success AND under 500 ms
good = sum(1 for l, ok in requests if ok and l < 500)
sli = good / len(requests)
slo = 0.99
budget_used = (1 - sli) / (1 - slo)
print(f"SLI {sli:.4f} vs SLO {slo} -> {budget_used:.0%} of error budget consumed")

# Structured log line carrying a trace id, as every service would emit
trace_id = uuid.uuid4().hex
print(json.dumps({"ts": time.time(), "level": "warn", "trace_id": trace_id,
                  "service": "pricing", "msg": "slow query", "duration_ms": 790}))
```

The mean hides the tail; the p99 shows it. The burn calculation shows how much of the
budget this window consumed. If the window is short (one hour) and the ratio is far above
1, you are burning a month's budget in hours — that is what a good alert fires on.

## Alerting well

- **Alert on symptoms users feel** (SLO burn, error rate, latency), not on every cause
  (CPU at 80% may be fine).
- **Multi-window burn-rate alerts**: page when budget is burning fast over both a short and
  a longer window (catches real incidents, ignores blips); open a ticket for slow burns.
- **Every page must be actionable** and link to a runbook. Alerts that are routinely ignored
  train people to ignore the important ones.
- Use cause-based metrics for **dashboards and diagnosis**, not for waking people up.

## How It Actually Works

**Trace propagation.** When service A calls B, it sends the trace ID and its current span
ID in request headers (the W3C Trace Context standard defines a `traceparent` header for
this). B creates a child span with A's span as parent, and passes its own IDs onward. Each
service reports finished spans asynchronously to a collector, which reassembles the tree
by trace ID. Because recording every trace at high volume is costly, systems **sample** —
either at the start of a request (head sampling, e.g. keep 1%) or after it completes
(tail sampling, e.g. keep all errors and slow requests), which is more useful but requires
buffering spans until the decision.

**Why histograms, not stored percentiles.** You cannot average percentiles: the p99 of
two servers is not the mean of their p99s. Metrics systems therefore record latency as
**histograms** — counts of requests per latency bucket. Buckets from many servers and time
windows can be summed exactly, and percentiles estimated from the merged histogram. The
estimate's precision depends on bucket boundaries, so choose them around your SLO
threshold.

**Cardinality.** Every unique combination of metric labels is a separate time series.
Adding `user_id` as a label to a request counter creates one series per user and can
overwhelm a metrics system. High-cardinality details belong in logs and traces; metrics
labels should have bounded value sets (endpoint, status class, region).

## Common mistakes

- **Averages instead of percentiles** for latency.
- **Unstructured logs without request/trace IDs**, making cross-service debugging
  guesswork.
- **Alerting on causes**, producing noisy pages that do not correspond to user pain.
- **High-cardinality metric labels.**
- **100% SLOs** or SLOs nobody uses to make decisions.
- **Observability added after the incident** rather than designed with the system.

## Exercise

1. Run `observability_demo.py`. Change the slow-tail probability from 2% to 0.5% and to 5%;
   record how mean, p50, and p99 move. Which statistic would you alert on, and why?
2. Define two SLIs and SLOs for the URL shortener from Level 1 (one for redirects, one for
   link creation). Compute each one's error budget in minutes per 30 days.
3. Sketch the trace you would expect for a news-feed load (Level 2 project). Mark where
   you would add custom spans and which attributes each span should carry.
