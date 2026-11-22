---
title: "Final Quiz"
description: "Test what you have learned across all four levels of System Design Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this estimate print for peak QPS and daily storage?\n```python\nreqs=200_000_000\navg=reqs/86_400\npeak=avg*4\nstore_gb=reqs*0.5/1e6\nprint(round(avg), round(peak), store_gb)\n```\n",
    "options": [
      "2315 2315 100.0",
      "2315 9259 1000.0",
      "23148 92593 100.0",
      "2315 9259 100.0"
    ],
    "correctIndex": 3,
    "explanation": "Average QPS is requests over 86,400 seconds; peak uses a multiplier; 0.5 KB per request is 100 GB per day. Verified by running it."
  },
  {
    "question": "A cache with a 95% hit rate has 1 ms hits and 40 ms misses. What is the mean read time (ms)?\n```python\nprint(0.95*1+0.05*40)\n```\n",
    "options": [
      "1.0",
      "2.95",
      "20.5",
      "40.0"
    ],
    "correctIndex": 1,
    "explanation": "Hit rate dominates average latency. Verified by running it."
  },
  {
    "question": "With N=5 replicas, which (W, R) pairs guarantee read-after-write overlap (R + W > N)?\n```python\nN=5\nprint([(w,r) for w in range(1,6) for r in range(1,6) if r+w>N and w==3])\n```\n",
    "options": [
      "[(3, 3)]",
      "[(3, 3), (3, 4), (3, 5)]",
      "[(1, 1), (2, 2)]",
      "[(3, 1), (3, 2)]"
    ],
    "correctIndex": 1,
    "explanation": "With W=3 any R of at least 3 overlaps every write quorum. Verified by running it."
  },
  {
    "question": "Hash-modulo sharding from 4 to 5 shards moves what fraction of keys 0..999?\n```python\nprint(sum(1 for k in range(1000) if k%4!=k%5)/1000)\n```\n",
    "options": [
      "0.2",
      "0.8",
      "0.5",
      "0.05"
    ],
    "correctIndex": 1,
    "explanation": "Modulo resharding remaps most keys; consistent hashing moves about 1/N. Verified by running it."
  },
  {
    "question": "A 3-node Raft cluster has one node down. Can it still commit writes?\n```python\nnodes,down=3,1\nprint(nodes-down >= nodes//2+1)\n```\n",
    "options": [
      "True",
      "None",
      "Only reads",
      "False"
    ],
    "correctIndex": 0,
    "explanation": "Two of three is a majority. Verified by running it."
  },
  {
    "question": "A service sees double charges after client timeouts and retries. What is the fix?",
    "options": [
      "Require an idempotency key and return the stored result for repeated keys",
      "Disable retries",
      "Increase the timeout only",
      "Add a cache"
    ],
    "correctIndex": 0,
    "explanation": "Retries must be safe."
  },
  {
    "question": "Which pattern keeps a database write and an event publish consistent without distributed transactions?",
    "options": [
      "Publish first",
      "The outbox: write the event in the same local transaction and publish it from there",
      "Hope",
      "Write twice"
    ],
    "correctIndex": 1,
    "explanation": "CDC or a relay publishes the outbox."
  },
  {
    "question": "Why do hot keys matter in sharded caches?",
    "options": [
      "One node absorbs disproportionate traffic and becomes a bottleneck",
      "Speed",
      "Law",
      "Cost"
    ],
    "correctIndex": 0,
    "explanation": "Replicate or split the key."
  },
  {
    "question": "What is the right response to a thundering herd on an expired hot key?",
    "options": [
      "Disable caching",
      "Add a server",
      "Longer database timeout",
      "Single-flight rebuilds with jittered TTLs so only one request recomputes"
    ],
    "correctIndex": 3,
    "explanation": "Protect the database."
  },
  {
    "question": "When does a queue between services help most?",
    "options": [
      "To remove retries",
      "To absorb bursts and decouple speeds, accepting asynchronous results",
      "To avoid idempotency",
      "For synchronous reads"
    ],
    "correctIndex": 1,
    "explanation": "Monitor lag."
  },
  {
    "question": "Why prefer cursor pagination over offset for large feeds?",
    "options": [
      "Offsets get slow with depth and shift when rows are inserted; cursors are stable",
      "Smaller",
      "Required",
      "Shorter"
    ],
    "correctIndex": 0,
    "explanation": "Use an indexed ordering key."
  },
  {
    "question": "Which consistency does a CP system sacrifice during a partition?",
    "options": [
      "Availability: the minority side stops accepting writes",
      "Durability",
      "Latency only",
      "Consistency"
    ],
    "correctIndex": 0,
    "explanation": "CAP."
  },
  {
    "question": "What does PACELC add to CAP?",
    "options": [
      "Security",
      "Cost",
      "Even without partitions you trade latency against consistency",
      "Nothing"
    ],
    "correctIndex": 2,
    "explanation": "Else-case trade-off."
  },
  {
    "question": "What is the main benefit of virtual nodes in consistent hashing?",
    "options": [
      "More even load and smoother rebalancing",
      "Fewer nodes",
      "Encryption",
      "Speed only"
    ],
    "correctIndex": 0,
    "explanation": "The challenge shows the ratio improving."
  },
  {
    "question": "A sliding-window limiter stops a boundary burst that a fixed window allows. What is the cost?",
    "options": [
      "None",
      "Law",
      "More memory and work per client to track timestamps (or approximations)",
      "Less accuracy"
    ],
    "correctIndex": 2,
    "explanation": "Token buckets are cheaper and allow controlled bursts."
  },
  {
    "question": "How do you decide between fan-out on write and on read?",
    "options": [
      "Always push",
      "Always pull",
      "Random",
      "By follower distribution: push for typical users, pull for celebrities (hybrid)"
    ],
    "correctIndex": 3,
    "explanation": "Avoid write amplification."
  },
  {
    "question": "What does an SLO error budget let you do?",
    "options": [
      "Trade reliability against change velocity with an explicit allowance for failures",
      "Stop deploys always",
      "Hide errors",
      "Skip tests"
    ],
    "correctIndex": 0,
    "explanation": "Spend it deliberately."
  },
  {
    "question": "Which belongs in every design doc besides the design itself?",
    "options": [
      "Marketing",
      "Goals and non-goals, alternatives considered, risks and a rollout/rollback plan",
      "Code",
      "Salaries"
    ],
    "correctIndex": 1,
    "explanation": "Make decisions reviewable."
  },
  {
    "question": "What is the weakest justification for a technology choice in a design review?",
    "options": [
      "Cost analysis",
      "Measured fit",
      "Operational maturity",
      "It is popular or what we already know without evidence it fits the requirements"
    ],
    "correctIndex": 3,
    "explanation": "Tie choices to requirements."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A consistent-hash ring with virtual nodes

Implement a consistent-hashing ring (bisect over sorted virtual-node points) and use it to show how load balance improves with more virtual nodes, that adding a node moves only about 1/N of the keys (all to the new node) and that removing a node leaves nothing behind. This is a self-contained Python model of the idea used by distributed caches and Dynamo-style stores.

**Starter code**

```python
import bisect, hashlib

def h(s):
    return int(hashlib.md5(s.encode()).hexdigest(), 16)

# TODO: class Ring(nodes=(), vnodes=100)
#   add(node) / remove(node): place `vnodes` points per node at h(f"{node}#{i}") on a sorted list (use bisect.insort), remember each point's owner
#   lookup(key): the owner of the first point clockwise from h(key) (wrap around with % len)

keys = [f"user:{i}" for i in range(10_000)]

def spread(ring):
    counts = {}
    for k in keys:
        n = ring.lookup(k)
        counts[n] = counts.get(n, 0) + 1
    return counts

ring = Ring(["a", "b", "c", "d"])
before = {k: ring.lookup(k) for k in keys}
c = spread(ring)
print("spread:", {n: c[n] for n in sorted(c)})
for vn in (5, 100, 2000):
    cc = spread(Ring(["a", "b", "c", "d"], vnodes=vn))
    print(f"vnodes={vn:4} max/min load ratio = {max(cc.values()) / min(cc.values()):.2f}")

ring.add("e")
moved = sum(1 for k in keys if ring.lookup(k) != before[k])
print("moved after adding e:", round(100 * moved / len(keys)), "% (ideal 20%)")
only_to_new = all(ring.lookup(k) == "e" for k in keys if ring.lookup(k) != before[k])
print("every moved key went to the new node:", only_to_new)

modulo_moved = sum(1 for k in keys if h(k) % 4 != h(k) % 5)
print("modulo hashing 4->5 would move:", round(100 * modulo_moved / len(keys)), "%")

ring.remove("b")
after_remove = {k: ring.lookup(k) for k in keys}
print("keys left on removed node:", sum(1 for v in after_remove.values() if v == "b"))
```

**Expected output**

```text
spread: {'a': 2207, 'b': 2911, 'c': 2726, 'd': 2156}
vnodes=   5 max/min load ratio = 2.74
vnodes= 100 max/min load ratio = 1.35
vnodes=2000 max/min load ratio = 1.09
moved after adding e: 21 % (ideal 20%)
every moved key went to the new node: True
modulo hashing 4->5 would move: 80 %
keys left on removed node: 0
```

<details><summary>Show solution and explanation</summary>

<pre><code>import bisect, hashlib

def h(s):
    return int(hashlib.md5(s.encode()).hexdigest(), 16)

class Ring:
    def __init__(self, nodes=(), vnodes=100):
        self.vnodes, self.keys, self.owner = vnodes, [], {}
        for n in nodes:
            self.add(n)

    def add(self, node):
        for i in range(self.vnodes):
            p = h(f&quot;{node}#{i}&quot;)
            bisect.insort(self.keys, p)
            self.owner[p] = node

    def remove(self, node):
        for i in range(self.vnodes):
            p = h(f&quot;{node}#{i}&quot;)
            self.keys.remove(p)
            del self.owner[p]

    def lookup(self, key):
        i = bisect.bisect(self.keys, h(key)) % len(self.keys)
        return self.owner[self.keys[i]]

keys = [f&quot;user:{i}&quot; for i in range(10_000)]

def spread(ring):
    counts = {}
    for k in keys:
        n = ring.lookup(k)
        counts[n] = counts.get(n, 0) + 1
    return counts

ring = Ring([&quot;a&quot;, &quot;b&quot;, &quot;c&quot;, &quot;d&quot;])
before = {k: ring.lookup(k) for k in keys}
c = spread(ring)
print(&quot;spread:&quot;, {n: c[n] for n in sorted(c)})
for vn in (5, 100, 2000):
    cc = spread(Ring([&quot;a&quot;, &quot;b&quot;, &quot;c&quot;, &quot;d&quot;], vnodes=vn))
    print(f&quot;vnodes={vn:4} max/min load ratio = {max(cc.values()) / min(cc.values()):.2f}&quot;)

ring.add(&quot;e&quot;)
moved = sum(1 for k in keys if ring.lookup(k) != before[k])
print(&quot;moved after adding e:&quot;, round(100 * moved / len(keys)), &quot;% (ideal 20%)&quot;)
only_to_new = all(ring.lookup(k) == &quot;e&quot; for k in keys if ring.lookup(k) != before[k])
print(&quot;every moved key went to the new node:&quot;, only_to_new)

modulo_moved = sum(1 for k in keys if h(k) % 4 != h(k) % 5)
print(&quot;modulo hashing 4-&gt;5 would move:&quot;, round(100 * modulo_moved / len(keys)), &quot;%&quot;)

ring.remove(&quot;b&quot;)
after_remove = {k: ring.lookup(k) for k in keys}
print(&quot;keys left on removed node:&quot;, sum(1 for v in after_remove.values() if v == &quot;b&quot;))
</code></pre>

<p>Each node owns many points on the ring; a key belongs to the first point clockwise from its hash. Adding a node only takes keys from neighbours, unlike modulo hashing which remaps about 80% of keys when going from 4 to 5 nodes. More virtual nodes smooth the load. Verified by running the solution (MD5 is deterministic, so the numbers are reproducible).</p>
</details>

### Rate limiters compared on a boundary burst

Implement fixed-window, sliding-window and token-bucket limiters with an explicit clock argument and compare them on a burst straddling a window boundary and on steady and heavy clients. This is a deterministic simulation, not a distributed limiter; production versions keep the counters in a shared store such as Redis.

**Starter code**

```python
# TODO: FixedWindow(limit, window).allow(t)   counter that resets at each window boundary (start = t - t % window)
# TODO: TokenBucket(capacity, rate).allow(t)   refill (t - last) * rate tokens up to capacity, spend 1 per request
# TODO: SlidingWindow(limit, window).allow(t)  keep timestamps newer than t - window; allow if fewer than limit

# 10 requests per second allowed; a burst of 10 just before a window boundary and 10 right after
burst = [0.90 + i * 0.005 for i in range(10)] + [1.00 + i * 0.005 for i in range(10)]
for name, lim in [("fixed window", FixedWindow(10, 1.0)), ("sliding window", SlidingWindow(10, 1.0)), ("token bucket", TokenBucket(10, 10.0))]:
    ok = sum(lim.allow(t) for t in burst)
    print(f"{name:15} allowed {ok} of {len(burst)}")

# steady 5 req/s for 4 seconds is fine for everyone; a client hammering 50 req/s is limited
steady = [i * 0.2 for i in range(20)]
heavy = [i * 0.02 for i in range(200)]
for name, make in [("fixed window", lambda: FixedWindow(10, 1.0)), ("token bucket", lambda: TokenBucket(10, 10.0))]:
    print(f"{name:15} steady {sum(make().allow(t) for t in steady)}/20  heavy {sum((lambda l: [l.allow(t) for t in heavy])(make()))}/200")
```

**Expected output**

```text
fixed window    allowed 20 of 20
sliding window  allowed 10 of 20
token bucket    allowed 11 of 20
fixed window    steady 20/20  heavy 40/200
token bucket    steady 20/20  heavy 49/200
```

<details><summary>Show solution and explanation</summary>

<pre><code>class FixedWindow:
    def __init__(self, limit, window):
        self.limit, self.window, self.start, self.count = limit, window, 0, 0

    def allow(self, t):
        if t - self.start &gt;= self.window:
            self.start, self.count = t - (t % self.window), 0
        if self.count &lt; self.limit:
            self.count += 1
            return True
        return False

class TokenBucket:
    def __init__(self, capacity, rate):
        self.capacity, self.rate, self.tokens, self.last = capacity, rate, capacity, 0.0

    def allow(self, t):
        self.tokens = min(self.capacity, self.tokens + (t - self.last) * self.rate)
        self.last = t
        if self.tokens &gt;= 1:
            self.tokens -= 1
            return True
        return False

class SlidingWindow:
    def __init__(self, limit, window):
        self.limit, self.window, self.hits = limit, window, []

    def allow(self, t):
        self.hits = [x for x in self.hits if t - x &lt; self.window]
        if len(self.hits) &lt; self.limit:
            self.hits.append(t)
            return True
        return False

# 10 requests per second allowed; a burst of 10 just before a window boundary and 10 right after
burst = [0.90 + i * 0.005 for i in range(10)] + [1.00 + i * 0.005 for i in range(10)]
for name, lim in [(&quot;fixed window&quot;, FixedWindow(10, 1.0)), (&quot;sliding window&quot;, SlidingWindow(10, 1.0)), (&quot;token bucket&quot;, TokenBucket(10, 10.0))]:
    ok = sum(lim.allow(t) for t in burst)
    print(f&quot;{name:15} allowed {ok} of {len(burst)}&quot;)

# steady 5 req/s for 4 seconds is fine for everyone; a client hammering 50 req/s is limited
steady = [i * 0.2 for i in range(20)]
heavy = [i * 0.02 for i in range(200)]
for name, make in [(&quot;fixed window&quot;, lambda: FixedWindow(10, 1.0)), (&quot;token bucket&quot;, lambda: TokenBucket(10, 10.0))]:
    print(f&quot;{name:15} steady {sum(make().allow(t) for t in steady)}/20  heavy {sum((lambda l: [l.allow(t) for t in heavy])(make()))}/200&quot;)
</code></pre>

<p>The fixed window lets 20 requests through in 0.1 seconds because the counter resets at the boundary, the sliding window enforces the true rate, and the token bucket allows a controlled burst (11 here: the full bucket plus a little refill). Verified by running the solution.</p>
</details>
