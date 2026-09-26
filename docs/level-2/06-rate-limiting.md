---
description: "Rate limiting algorithms compared with runnable code: token bucket, leaky bucket, fixed window, sliding log and sliding window counter, plus distributed limits."
---

# 06 · Rate Limiting Algorithms

A rate limiter caps how many requests a client (a user, an API key, an IP, a tenant) can
make in a period. It protects services from abuse and accidental overload, enforces
fairness between tenants, and backs commercial plan limits. The algorithm you choose
decides how bursts are treated, how much memory each client costs, and how accurate the
limit is.

## Where the limiter lives

- **At the edge / API gateway**: stops abusive traffic before it consumes app resources.
- **In each service**: protects that service's own dependencies (a database, a paid API).
- **In the client**: well-behaved clients self-limit to avoid being throttled.

When a request is rejected, return `429 Too Many Requests` with a `Retry-After` header,
and ideally headers describing the limit and remaining quota, so clients can back off
instead of hammering.

## The algorithms

### Fixed window counter

Count requests per key in the current window (e.g. the current minute). Reject once the
count exceeds the limit.

- One counter per key — tiny memory.
- **Boundary burst**: a client can send the full limit at 12:00:59 and again at 12:01:00,
  so up to twice the limit passes in about one second.

### Sliding window log

Store the timestamp of every accepted request; count those in the last 60 seconds.

- Exact.
- Memory grows with the limit: a 10,000/hour limit stores up to 10,000 timestamps per key.

### Sliding window counter

Keep counts for the current and previous fixed windows and weight the previous one by how
much of it still overlaps the sliding window:

```text
estimate = current_count + previous_count × (1 − elapsed_fraction_of_current_window)
```

- Two counters per key; smooths the boundary burst.
- An approximation (it assumes the previous window's requests were spread evenly), which
  is usually acceptable.

### Token bucket

A bucket holds up to **capacity** tokens and refills at **rate** tokens per second. Each
request takes a token; with no tokens, it is rejected.

- Allows bursts up to `capacity`, then enforces the average `rate` — often exactly what
  you want for APIs ("100 requests/minute, bursts of 20 allowed").
- Two numbers per key (tokens, last refill time). Refill is computed lazily on each
  request — no background timer needed.

### Leaky bucket

Requests enter a queue that drains at a constant rate; if the queue is full, new
requests are dropped. It **smooths** output to a steady rate — useful in front of a
downstream that cannot handle bursts — at the cost of queueing delay.

## Worked example: all four, side by side

```python
# limiters.py — compare rate limiting algorithms on the same traffic
from collections import deque

class FixedWindow:
    def __init__(self, limit, window):
        self.limit, self.window, self.start, self.count = limit, window, 0.0, 0
    def allow(self, now):
        if now - self.start >= self.window:
            self.start, self.count = now - (now % self.window), 0
        if self.count < self.limit:
            self.count += 1
            return True
        return False

class SlidingLog:
    def __init__(self, limit, window):
        self.limit, self.window, self.log = limit, window, deque()
    def allow(self, now):
        while self.log and self.log[0] <= now - self.window:
            self.log.popleft()
        if len(self.log) < self.limit:
            self.log.append(now)
            return True
        return False

class SlidingCounter:
    def __init__(self, limit, window):
        self.limit, self.window = limit, window
        self.cur_start, self.cur, self.prev = 0.0, 0, 0
    def allow(self, now):
        start = now - (now % self.window)
        if start != self.cur_start:
            self.prev = self.cur if start - self.cur_start == self.window else 0
            self.cur_start, self.cur = start, 0
        overlap = 1 - (now - start) / self.window
        if self.cur + self.prev * overlap < self.limit:
            self.cur += 1
            return True
        return False

class TokenBucket:
    def __init__(self, capacity, rate):
        self.capacity, self.rate, self.tokens, self.last = capacity, rate, capacity, 0.0
    def allow(self, now):
        self.tokens = min(self.capacity, self.tokens + (now - self.last) * self.rate)
        self.last = now
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False

# Traffic: 10 requests at t=59.9s, then 10 at t=60.1s (straddling a window boundary)
traffic = [59.9] * 10 + [60.1] * 10
for limiter in [FixedWindow(10, 60), SlidingLog(10, 60),
                SlidingCounter(10, 60), TokenBucket(10, 10 / 60)]:
    allowed = sum(limiter.allow(t) for t in traffic)
    print(f"{type(limiter).__name__:15s} allowed {allowed} of 20 within 0.2 s")
```

Output:

```text
FixedWindow     allowed 20 of 20 within 0.2 s
SlidingLog      allowed 10 of 20 within 0.2 s
SlidingCounter  allowed 11 of 20 within 0.2 s
TokenBucket     allowed 10 of 20 within 0.2 s
```

The fixed window lets twice the limit through across the boundary; the others hold the
line. The sliding counter admits one extra request because 0.1 s into the new window it
weights the previous window's 10 requests at 99.8%, giving an estimate of 9.98 — a small
taste of its approximation, discussed below. (The token bucket starts full here; in practice it also allows an initial burst of
`capacity`, by design.)

## Distributed rate limiting

With 20 gateway instances, a per-instance limit of 100/min lets a client get up to
2,000/min by spreading requests. Options:

1. **Central store**: all instances update counters in a shared in-memory store such as
   Redis. The check-and-increment must be **atomic** — a server-side script or atomic
   increment-with-expiry — or two instances can both read "99" and both allow. Adds a
   network round trip per request; decide whether to **fail open** (allow traffic when the
   store is down) or **fail closed**.
2. **Local limits with a share of the global quota**: each instance enforces
   limit ÷ instances. No coordination, but inaccurate when traffic is uneven across
   instances.
3. **Hybrid**: local token buckets that periodically sync usage with a central store.
   Slightly over-admits between syncs; far fewer round trips.

For most APIs, approximate is fine; a limit of 1,000/min that occasionally lets through
1,030 is not a problem. For hard limits tied to money (a paid third-party API quota), be
conservative.

## How It Actually Works

The token bucket's lazy refill is the trick that makes it cheap. Instead of a timer adding
tokens every tick for millions of keys, each key stores `tokens` and `last_refill_time`.
When a request arrives, the elapsed time since `last_refill_time` times the rate gives the
tokens earned since then, capped at capacity. That single multiply-and-min is equivalent
to having refilled continuously. In a shared store, this read-compute-write sequence runs
as one atomic server-side script so that concurrent requests for the same key cannot
interleave.

The sliding window counter's weighting is a linear interpolation. It assumes requests in
the previous window were uniformly spread; if they were all at the very end of that
window, it underestimates, and if they were all at the beginning, it overestimates. In
exchange it needs two integers instead of a log of timestamps, which is why large-scale
limiters commonly use it.

## Common mistakes

- **Non-atomic read-then-write** on a shared counter, letting concurrent requests exceed
  the limit.
- **Limiting only by IP**, which punishes many users behind one NAT and does little
  against distributed abuse. Prefer API keys or user IDs where available.
- **No `Retry-After`**, so clients retry immediately and make the load worse.
- **Rate limiter as a hard dependency** with no fail-open/closed decision.
- **One global limit for all endpoints**, when an expensive search endpoint deserves a
  much lower limit than a cheap read.

## Exercise

1. Add a leaky-bucket limiter to `limiters.py` that queues up to 5 requests and drains 1
   every 6 seconds. Report both drops and the maximum queueing delay.
2. Simulate 20 gateway instances, each with a local fixed window of 5/min, and a client
   spreading 100 requests evenly across them. How many get through?
3. Design limits for a public API with a free tier (60/min) and a paid tier (1,000/min
   with bursts of 200). Which algorithm, which key, which store, and what happens when the
   store is unreachable?
