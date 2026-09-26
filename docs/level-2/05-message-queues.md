---
description: "Message queues and async processing: queues vs logs, at-most/at-least-once delivery, acknowledgements, back-pressure, ordering, and dead-letter queues."
---

# 05 · Message Queues & Async Processing

Not every piece of work needs to happen while the user waits. Sending a welcome email,
resizing an uploaded image, updating a search index, recomputing a recommendation — all
can happen a few seconds later. A **message queue** lets one component hand off work to
another without both being available at the same moment, at the same speed.

## What queues buy you

- **Decoupling in time.** The producer does not need the consumer to be up right now.
- **Load leveling.** A burst of 50,000 uploads in a minute becomes a backlog that workers
  drain at their own steady pace, instead of a spike that knocks them over.
- **Independent scaling.** Add consumers when the backlog grows.
- **Fan-out.** One event ("order placed") can feed several independent consumers
  (email, analytics, fulfilment).

The costs: another system to operate, eventual rather than immediate results, harder
debugging (work happens "somewhere else, later"), and delivery semantics you must
understand.

## Two shapes: queues and logs

**Work queues** (RabbitMQ-style brokers, cloud queue services): a message is delivered to
one consumer; once acknowledged it is removed. Great for distributing tasks.

**Logs** (Kafka-style, cloud streaming services): messages are appended to a partitioned,
durable log and retained for a period. Consumers track their own **offset** and can
re-read history. Multiple consumer groups read the same log independently. Great for
event streams, fan-out, and replay.

| | Work queue | Log |
|---|---|---|
| After consumption | Message deleted | Message retained until retention expires |
| Multiple independent readers | Needs one queue per reader (fan-out exchange/topic) | Native (consumer groups) |
| Ordering | Often best-effort | Ordered within a partition |
| Replay old messages | No | Yes (within retention) |
| Per-message retry/delay | Natural | Awkward — a slow message blocks its partition |

## Delivery semantics

Networks lose acknowledgements, so a broker cannot always know whether a consumer
finished its work.

- **At-most-once**: acknowledge *before* processing. A crash mid-processing loses the
  message. Acceptable for metrics you can afford to drop.
- **At-least-once**: acknowledge *after* processing. A crash after processing but before
  the ack causes redelivery — the message is processed twice. This is the practical
  default.
- **Exactly-once**: the *effect* of each message happens once. Achieved end to end only
  by combining at-least-once delivery with idempotent or transactional processing — a
  subject Level 3 lesson 3 treats in depth.

Design rule: **assume every message may arrive more than once, and make consumers
idempotent.**

## Worked example: an at-least-once worker with retries and a DLQ

```python
# queue_worker.py — at-least-once processing, retries, and a dead-letter queue
import queue, random

jobs = queue.Queue()
dead_letters = []
processed_ids = set()           # idempotency record (a DB table in real life)
MAX_ATTEMPTS = 3
rng = random.Random(4)

def handle(msg):
    """Send a receipt email. Fails randomly; msg 7 is 'poison' and always fails."""
    if msg["id"] == 7 or rng.random() < 0.3:
        raise RuntimeError("SMTP timeout")
    if msg["id"] in processed_ids:
        return "skipped duplicate"
    processed_ids.add(msg["id"])
    return "sent"

for i in range(1, 11):
    jobs.put({"id": i, "attempts": 0})
jobs.put({"id": 3, "attempts": 0})         # a duplicate delivery of message 3

while not jobs.empty():
    msg = jobs.get()
    try:
        result = handle(msg)
        print(f"msg {msg['id']}: {result}")    # ack happens here, after success
    except RuntimeError as e:
        msg["attempts"] += 1
        if msg["attempts"] >= MAX_ATTEMPTS:
            dead_letters.append(msg)
            print(f"msg {msg['id']}: moved to DLQ after {msg['attempts']} attempts ({e})")
        else:
            jobs.put(msg)                       # redeliver later

print("DLQ:", [m["id"] for m in dead_letters])
```

Three production patterns appear here:

1. **Retries** handle transient failures. Real systems add delay between attempts
   (exponential backoff — Level 3, lesson 8) instead of retrying instantly.
2. **Dead-letter queue (DLQ)**: a message that keeps failing — a "poison message" — is set
   aside after N attempts so it does not block or consume workers forever. Someone
   inspects the DLQ, fixes the cause, and replays.
3. **Idempotency**: the duplicate delivery of message 3 is detected and skipped. Note the
   subtle bug left for the exercise: this check happens *after* the part that could fail,
   and in real life the "send" and the "record" are not atomic.

## Ordering

Global ordering across a high-throughput queue is expensive. Logs provide order **within
a partition**, and messages with the same key (for example, `order_id`) go to the same
partition. So per-entity ordering is achievable; global ordering generally is not. With
retries, even per-key order can break if a failed message is retried after its successor
succeeded. If order matters, either process a key strictly sequentially (and accept that
one stuck message blocks that key) or make consumers tolerate reordering (version numbers,
"apply only if newer").

## Back-pressure

If producers outpace consumers indefinitely, the backlog grows without bound. A queue
hides this for a while — then runs out of disk or causes hours of delay. Monitor **queue
depth and age of the oldest message**, autoscale consumers on them, and decide what
happens at the limit: reject new work (with a 429 or 503), shed low-priority work, or
slow producers down. A bounded queue that pushes back is healthier than an unbounded one
that hides a problem.

## How It Actually Works

A work-queue broker keeps each message in one of a few states: ready, in-flight
(delivered but not acknowledged), or deleted. When it delivers a message, it starts a
**visibility timeout** or waits on the consumer's connection; if no acknowledgement
arrives before the timeout or the connection drops, the message returns to "ready" and is
delivered again. That timer is precisely why at-least-once delivery produces duplicates:
a consumer that is merely *slow* looks identical to one that crashed.

A log-based system works differently. Each partition is an append-only file on disk,
replicated to other brokers. Producers append at the end; the broker assigns each record
an increasing offset. Consumers read sequentially from their saved offset and periodically
commit the offset they have processed. On restart, a consumer resumes from its last
committed offset, reprocessing anything after it — again, at-least-once. Because the data
is not deleted on read, sequential disk reads and the OS page cache make logs efficient at
high throughput, and replay is just "reset the offset".

## Common mistakes

- **Non-idempotent consumers** under at-least-once delivery → duplicate emails, double
  charges.
- **No DLQ**, so one malformed message is retried forever and blocks a partition.
- **Unbounded queues with no depth alerts.**
- **Visibility timeout shorter than processing time**, causing the same message to be
  processed concurrently by two workers.
- **Using a queue where a synchronous answer is required** — the user needs to know now
  whether their payment succeeded.
- **Assuming global ordering.**

## Exercise

1. Run `queue_worker.py`. Then fix the idempotency ordering bug: design how you would make
   "send email" and "record as sent" safe against a crash between them. (Hint: can the
   email provider accept an idempotency key? If not, what is the least-bad option?)
2. Add exponential backoff with jitter: a failed message becomes eligible again only after
   `base * 2**attempts` plus random jitter.
3. An image-processing pipeline receives uploads in bursts of 10,000 per minute; each resize
   takes 2 seconds of CPU. How many workers are needed to clear a burst within 5 minutes?
   What metric would you autoscale on?
