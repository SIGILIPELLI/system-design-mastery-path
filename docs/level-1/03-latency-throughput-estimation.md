---
description: "Latency vs throughput, percentiles, Little's Law, and back-of-envelope estimation with clearly labeled rough numbers for QPS, storage, and bandwidth."
---

# 03 · Latency, Throughput & Back-of-Envelope Estimation

Before choosing components, a designer needs to know roughly how big the problem is.
Is it 10 requests per second or 100,000? Gigabytes or petabytes? The answer is found
with arithmetic you can do on paper in five minutes, and it frequently changes the
design completely.

## Latency vs throughput

- **Latency** is how long one operation takes, from request to response.
- **Throughput** is how many operations complete per unit of time.

They are related but not the same. A highway can have high throughput (many cars per
hour) while each car still takes an hour to travel it. Adding lanes improves throughput
but not the trip time. In systems, adding servers usually increases throughput; reducing
latency usually requires doing less work, doing it closer to the user, or doing it in
parallel.

### Percentiles, not averages

Average latency hides pain. If 99 requests take 10 ms and one takes 2 seconds, the mean
is about 30 ms, which describes nobody. Designers talk in percentiles:

- **p50** (median): the typical request.
- **p99**: 1 in 100 requests is slower than this.
- **p99.9**: the tail.

Tails matter more than they look. If a page load fans out to 50 backend calls in
parallel and waits for all of them, the chance that *at least one* hits its p99 is
1 − 0.99⁵⁰ ≈ 39%. The page's typical latency is governed by the backends' tail.

## Rough numbers to carry around

!!! warning "Rules of thumb, not measurements"
    The figures below are order-of-magnitude approximations meant for quick reasoning.
    Actual values vary widely with hardware generation, configuration, cloud provider,
    and workload. Use them to compare options, never as a promise.

| Operation | Rough latency |
|---|---|
| Read from main memory | ~100 nanoseconds |
| Read 1 MB sequentially from memory | ~tens of microseconds |
| Round trip within one datacenter | ~0.5 ms |
| Random read from an SSD | ~0.1 ms (100 µs) |
| Read 1 MB sequentially from SSD | ~1 ms or less |
| Disk seek on a spinning hard drive | ~5–10 ms |
| Round trip between continents | ~100–200 ms |

The ratios are what matter: memory is roughly a thousand times faster than an SSD
random read, and a cross-continent round trip is hundreds of times slower than a
same-datacenter one.

Other handy rough figures:

- Seconds in a day: 86,400 — round to **~10⁵** for estimates.
- A simple, well-indexed key lookup on one relational database server: order of
  **thousands to tens of thousands** of queries per second, depending heavily on the
  query, hardware, and dataset size.
- One stateless app server handling light requests: order of **hundreds to a few
  thousand** requests per second, again very workload-dependent.
- Powers of two: 2¹⁰ ≈ 1 thousand (KB), 2²⁰ ≈ 1 million (MB), 2³⁰ ≈ 1 billion (GB),
  2⁴⁰ ≈ 1 trillion (TB).

## The estimation recipe

1. **Start from users and behavior.** Daily active users × actions per user per day.
2. **Convert to per-second.** Divide by ~10⁵. Multiply by a peak factor (2–10× is a
   common assumption; say which you chose).
3. **Split reads and writes.** They scale differently.
4. **Storage.** Writes per day × bytes per write × retention period. Add replication
   (often ×3).
5. **Bandwidth.** Requests per second × bytes per response.
6. **Sanity-check.** Does any number exceed what one machine does? That is where the
   design must distribute.

## Worked example: a photo-sharing app

Assumptions (state them explicitly — reviewers care more about these than about the
arithmetic):

- 10 million daily active users.
- Each user uploads 0.2 photos per day and views 50 photos per day.
- Average stored photo (after compression) is 500 KB; thumbnails are negligible here.
- Peak traffic is 3× average. Keep photos forever. Store 3 copies.

```text
Uploads/day   = 10M × 0.2          = 2M/day
Upload QPS    = 2M / 10^5          ≈ 20/s average, ~60/s peak
Views/day     = 10M × 50           = 500M/day
View QPS      = 500M / 10^5        ≈ 5,000/s average, ~15,000/s peak

New storage/day   = 2M × 500 KB    = 1 TB/day
Per year          = 365 TB ≈ 0.4 PB (raw), ~1.1 PB with 3 copies

Egress bandwidth  = 5,000/s × 500 KB = 2.5 GB/s average ≈ 20 Gbit/s
```

What the numbers tell us:

- Uploads are tiny (60/s). The write path does not need heroic engineering.
- Views are ~250× more frequent and cost **20 Gbit/s** of egress on average. That is the
  real problem, and it points directly to a CDN (lesson 8) and object storage.
- A petabyte-scale photo store is not a relational database's job. Metadata (who
  uploaded what, when) goes in a database; the bytes go in an object store (Level 2).

A script is a good way to keep estimates honest and to vary assumptions:

```python
# estimate.py — rough capacity model; every constant is an assumption
DAY = 86_400

def estimate(dau, uploads_per_user, views_per_user, photo_kb, peak=3, copies=3):
    up_qps = dau * uploads_per_user / DAY
    view_qps = dau * views_per_user / DAY
    tb_per_day = dau * uploads_per_user * photo_kb / 1e9     # KB -> TB
    egress_gbit = view_qps * photo_kb * 8 / 1e6               # KB/s -> Gbit/s
    return {
        "upload_qps_peak": round(up_qps * peak),
        "view_qps_peak": round(view_qps * peak),
        "raw_tb_per_year": round(tb_per_day * 365),
        "stored_tb_per_year": round(tb_per_day * 365 * copies),
        "egress_gbit_avg": round(egress_gbit, 1),
    }

print(estimate(10_000_000, 0.2, 50, 500))
# {'upload_qps_peak': 69, 'view_qps_peak': 17361, 'raw_tb_per_year': 365,
#  'stored_tb_per_year': 1095, 'egress_gbit_avg': 23.1}
```

The script uses 86,400 rather than 10⁵, so its numbers are slightly higher than the
hand estimate. Both lead to the same design conclusions, which is the point.

## How It Actually Works

**Little's Law** connects latency and throughput for any stable system:

```text
L = λ × W
concurrent requests in flight = arrival rate × average time in system
```

If a service receives 2,000 requests per second and each takes 50 ms, about
2,000 × 0.05 = 100 requests are in progress at any moment. If each request holds a
database connection for its whole duration, you need about 100 connections — or you
need requests to hold connections for less time. When a downstream dependency slows
from 50 ms to 500 ms, concurrency jumps tenfold to 1,000 at the same arrival rate.
Thread pools and connection pools run out, and the slowdown turns into an outage. This
is why slow dependencies are often more dangerous than dead ones.

**Why latency rises sharply near capacity.** A server that is 50% busy rarely makes
requests wait. As utilization approaches 100%, queues form and waiting time grows
non-linearly — in simple queueing models it scales roughly with 1 / (1 − utilization),
so going from 80% to 95% busy can multiply queueing delay several times. Designers
therefore plan capacity with headroom (often targeting well under full utilization at
peak) rather than sizing for 100%.

## Common mistakes

- **Forgetting the peak.** Average QPS is not what knocks systems over.
- **Mixing bits and bytes.** Network links are quoted in bits per second; storage in
  bytes. A factor of 8 error changes conclusions.
- **Doing precise arithmetic.** 86,400 vs 100,000 is irrelevant; getting the exponent
  wrong is not. Round aggressively and show your assumptions.
- **Estimating and then ignoring the result.** The estimate should decide something:
  whether to cache, shard, or use a CDN.
- **Quoting averages for latency SLOs.** State a percentile.

## Exercise

A messaging app has 50 million DAU. Each user sends 40 messages per day, averaging
200 bytes, and each message is delivered to an average of 3 recipients. Messages are
stored for 5 years with 3 replicas.

1. Compute send QPS (average and 5× peak) and delivery QPS.
2. Compute storage per year and for the full 5 years.
3. Use Little's Law: if delivering one message takes 80 ms end to end, how many
   deliveries are in flight at peak?
4. Which of these numbers most influences the design, and why? Modify `estimate.py` to
   model this app.
