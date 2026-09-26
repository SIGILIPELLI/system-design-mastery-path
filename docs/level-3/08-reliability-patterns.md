---
description: "Reliability patterns with runnable code: timeouts, retries with exponential backoff and jitter, retry budgets, circuit breakers, bulkheads, and load shedding."
---

# 08 · Reliability Patterns

Most large outages are not caused by one component failing. They are caused by the rest
of the system reacting badly to it: callers waiting forever, retries multiplying load,
thread pools filling with stuck requests, and a slow dependency dragging down services
that never needed it. Reliability patterns exist to **contain** failure — to make a
partial failure stay partial.

## Timeouts: the first line

Every network call needs a timeout. Without one, a hung dependency holds a thread,
connection, or memory indefinitely; by Little's Law (Level 1, lesson 3), rising latency
times constant arrival rate means rising concurrency until something runs out.

- Base timeouts on the dependency's observed latency (e.g. somewhat above its p99), not on
  a round number picked once.
- Use **deadlines** that propagate: if the user's request has 800 ms left, a downstream
  call should not be given 2 seconds. gRPC propagates deadlines natively; with HTTP, pass
  the remaining budget in a header.

## Retries — carefully

Retries recover from transient failures (a dropped packet, a restarting pod). Done
naively, they are the most common way to turn a small incident into a large one.

Rules:

1. **Only retry idempotent operations** (or ones protected by idempotency keys — lesson 3).
2. **Only retry retryable errors**: timeouts, 503, connection resets. Not 400 or 404.
3. **Exponential backoff with jitter**: wait `base × 2^attempt`, capped, randomized. Without
   jitter, all clients that failed together retry together, in synchronized waves.
4. **Cap attempts** and respect the overall deadline.
5. **Retry at one layer only.** If each of 4 layers retries 3 times, one user request can
   become 3⁴ = 81 calls to the bottom service — during exactly the moment it is struggling.
6. **Retry budgets**: allow retries only up to a fraction (say 10%) of normal traffic, so
   retries cannot multiply load during an outage.

## Circuit breakers

A circuit breaker wraps calls to a dependency and tracks failures:

- **Closed**: calls pass through; failures are counted.
- **Open**: after the failure rate crosses a threshold, calls fail *immediately* without
  touching the dependency, for a cool-down period. Callers get a fast error or a fallback
  instead of waiting on timeouts, and the struggling dependency gets breathing room.
- **Half-open**: after the cool-down, a few trial calls are allowed; success closes the
  circuit, failure reopens it.

## Worked example: backoff with jitter and a circuit breaker

```python
# resilience.py — retries with full-jitter backoff + a circuit breaker (stdlib only)
import random

class CircuitOpen(Exception):
    pass

class CircuitBreaker:
    def __init__(self, threshold=5, cooldown=10.0):
        self.threshold, self.cooldown = threshold, cooldown
        self.failures, self.state, self.opened_at = 0, "closed", 0.0
    def call(self, fn, now):
        if self.state == "open":
            if now - self.opened_at < self.cooldown:
                raise CircuitOpen()                     # fail fast
            self.state = "half-open"                    # allow a trial call
        try:
            result = fn(now)
        except Exception:
            self.failures += 1
            if self.state == "half-open" or self.failures >= self.threshold:
                self.state, self.opened_at = "open", now
            raise
        self.failures, self.state = 0, "closed"
        return result

def backoff_delays(attempts, base=0.1, cap=5.0, rng=random.Random(2)):
    # "full jitter": uniform between 0 and the exponential ceiling
    return [rng.uniform(0, min(cap, base * 2 ** a)) for a in range(attempts)]

print("jittered delays:", [round(d, 3) for d in backoff_delays(5)])

# A dependency that is down from t=0 to t=30, then healthy
def dependency(now):
    if now < 30:
        raise TimeoutError("upstream timeout")
    return "ok"

breaker = CircuitBreaker(threshold=5, cooldown=10)
calls_reaching_dependency = 0
outcomes = []
for t in range(0, 60):                                   # one request per second
    try:
        calls_reaching_dependency += breaker.state != "open" or \
            (t - breaker.opened_at >= breaker.cooldown)
        outcomes.append(breaker.call(dependency, t))
    except CircuitOpen:
        outcomes.append("fast-fail")
    except TimeoutError:
        outcomes.append("timeout")

print("timeouts:", outcomes.count("timeout"),
      "| fast-fails:", outcomes.count("fast-fail"),
      "| ok:", outcomes.count("ok"),
      "| calls that hit the dependency:", calls_reaching_dependency, "of 60")
print("first success at t =", outcomes.index("ok"))
```

Without a breaker, all 30 requests during the outage would wait for a timeout and hit the
failing dependency. With it, only the first five plus one trial call per cool-down reach the
dependency; the rest fail fast. The first success comes at the first half-open trial after
the dependency recovers — a little after t=30, which is the cost of the cool-down.

## Bulkheads

Ships have watertight compartments so one breach does not sink the vessel. In software:
**isolate resources per dependency or per workload** so one cannot exhaust what others need.

- Separate thread/connection pools per downstream: if the recommendations service hangs,
  it fills only its own pool of 20 threads, and checkout calls still have theirs.
- Separate clusters or queues for different tenants or priorities (lesson 9).
- Separate critical and non-critical traffic: health checks and admin endpoints should
  never queue behind user traffic.

## Graceful degradation and load shedding

- **Fallbacks**: if recommendations fail, show bestsellers; if personalization fails,
  show the generic page. Decide in advance which features are optional.
- **Load shedding**: when saturated, reject some work *early and cheaply* (a fast 503 or
  429) rather than accepting everything and serving everyone slowly. Prioritize: shed
  prefetches and analytics before checkouts.
- **Queue deadlines**: drop requests that have waited longer than the client will wait —
  processing them is wasted work.

## How It Actually Works

The failure mode these patterns fight is the **retry storm / metastable failure**: a
system that is stable under normal load gets pushed over by a trigger (a brief spike, a
cache flush, a dependency blip), and then *stays* broken even after the trigger is gone,
because the recovery attempts themselves — retries, reconnects, cache refills — keep load
above capacity. Each layer's retries multiply; timed-out requests are retried while their
original copies are still being processed; queues fill with work no one is waiting for.

Each pattern breaks a link in that loop. Timeouts bound how long any request holds
resources. Jitter de-synchronizes clients so retries spread out instead of arriving in
waves. Retry budgets and single-layer retries cap amplification. Circuit breakers cut load
on a failing dependency to near zero so it can recover. Load shedding keeps goodput (useful
completed work) high instead of letting all requests become equally slow. Designing for
recovery means asking not only "what fails?" but "what does everyone else do when it
fails?"

## Common mistakes

- **No timeouts**, or timeouts longer than the caller's own deadline.
- **Retrying at every layer** and **retrying without jitter.**
- **Retrying non-idempotent writes.**
- **Circuit breakers without a fallback**, turning every dependency blip into a hard error
  for users anyway (sometimes right, but decide deliberately).
- **Shared pools** so one slow dependency starves all others.
- **Never testing failure** — use fault injection (latency, errors, killed instances) in
  staging and, carefully, in production.

## Exercise

1. Run `resilience.py`. Then remove the breaker (call `dependency` directly with a 2-second
   timeout per call) and compute total time spent waiting during the outage.
2. Model retry amplification: 3 layers, each retrying up to 3 times on failure. If the
   bottom service fails 50% of requests, how many calls does it receive per user request on
   average? Then apply a 10% retry budget at each layer and recompute.
3. For an e-commerce product page that calls pricing, inventory, reviews, and
   recommendations, classify each dependency as critical or optional, and specify timeout,
   retry policy, breaker, and fallback for each.
