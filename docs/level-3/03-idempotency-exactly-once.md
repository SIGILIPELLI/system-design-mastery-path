---
description: "Why exactly-once delivery is impossible over unreliable networks, and how idempotency keys, dedup tables, and transactional offsets deliver exactly-once effects."
---

# 03 · Idempotency & the Exactly-Once Myth

A client sends "charge $40" and the connection times out. Did the charge happen? The
client cannot know, so it has two choices: give up (and maybe lose a payment) or retry
(and maybe charge twice). Every reliable distributed system resolves this dilemma the
same way: **retry, and make the operation safe to repeat**. That property is
idempotency, and it is the practical foundation of "exactly once".

## Delivery vs effect

- **Exactly-once *delivery*** — the network delivers each message to the receiver exactly
  one time — cannot be guaranteed when messages or acknowledgements can be lost and nodes
  can crash. The sender must retry when it hears nothing, and sometimes the original did
  arrive. This is a consequence of the Two Generals problem: no finite exchange of
  messages over an unreliable channel lets both sides be certain the other knows.
- **Exactly-once *processing* or *effect*** — the observable result is as if each message
  was processed once — *is* achievable, by combining at-least-once delivery with
  deduplication or idempotent operations.

When a product advertises "exactly-once semantics", read the fine print: it almost always
means exactly-once effect *within that system's boundary* (for example, reading from and
writing to the same log with transactions). The moment your consumer calls an external
API or sends an email, you are responsible for idempotency again.

## Naturally idempotent operations

Some operations are idempotent by design:

- `SET balance_display = 'hidden'`, `PUT /users/7 {…full body…}`, `DELETE /carts/9`.
- "Set status to SHIPPED if it is PACKED" — a conditional transition.
- Upserts keyed by a natural identifier: `INSERT … ON CONFLICT (event_id) DO NOTHING`.

Others are not: `balance = balance - 40`, `POST /orders`, "send an SMS". Those need an
explicit mechanism.

## Idempotency keys

The client generates a unique key per *logical* operation (a UUID) and sends it with
every attempt. The server records the key with the result of the first attempt; later
attempts with the same key return the stored result instead of re-executing.

```http
POST /v1/payments
Idempotency-Key: 5f1c7b0e-2a55-4a84-9f0c-2d1f6f4b9a21
{"amount": 4000, "currency": "USD", "source": "card_abc"}
```

Implementation details that matter:

1. **Atomicity.** Recording the key and performing the effect must happen in the same
   database transaction, or a crash between them breaks the guarantee.
2. **Concurrent duplicates.** Two retries can arrive at the same time. A unique constraint
   on the key makes the second insert fail; it should then wait for or return the first
   one's result (or return `409 Conflict` — "in progress").
3. **Request fingerprint.** Store a hash of the request body; if the same key arrives with
   a different body, reject it — that is a client bug.
4. **Retention.** Keep keys long enough to cover realistic retry windows (hours to days),
   then expire them.

## Worked example: an idempotent payments endpoint

```python
# idempotency.py — idempotency keys with a transactional dedup table (SQLite)
import hashlib, json, sqlite3, uuid

db = sqlite3.connect(":memory:", isolation_level=None)
db.executescript("""
CREATE TABLE accounts(id TEXT PRIMARY KEY, balance INT NOT NULL);
CREATE TABLE idempotency(key TEXT PRIMARY KEY, request_hash TEXT, response TEXT);
INSERT INTO accounts VALUES ('alice', 10000);
""")

def charge(key, body):
    req_hash = hashlib.sha256(json.dumps(body, sort_keys=True).encode()).hexdigest()
    db.execute("BEGIN IMMEDIATE")                        # serialize writers
    try:
        row = db.execute("SELECT request_hash, response FROM idempotency WHERE key=?",
                         (key,)).fetchone()
        if row:
            db.execute("ROLLBACK")
            if row[0] != req_hash:
                return 422, {"error": "idempotency key reused with different request"}
            return 200, json.loads(row[1])               # replay stored result
        db.execute("UPDATE accounts SET balance = balance - ? WHERE id=?",
                   (body["amount"], body["account"]))
        response = {"payment_id": str(uuid.uuid4()), "amount": body["amount"]}
        db.execute("INSERT INTO idempotency VALUES (?,?,?)",
                   (key, req_hash, json.dumps(response)))
        db.execute("COMMIT")                             # effect + key, atomically
        return 201, response
    except Exception:
        db.execute("ROLLBACK")
        raise

key = str(uuid.uuid4())
body = {"account": "alice", "amount": 4000}
print(charge(key, body))           # 201, new payment
print(charge(key, body))           # 200, same payment_id replayed — no second debit
print(charge(key, {"account": "alice", "amount": 9999}))   # 422, mismatched body
print(db.execute("SELECT balance FROM accounts").fetchone())  # (6000,)
```

The debit and the key insertion commit together. If the process crashes before `COMMIT`,
neither exists and a retry performs the charge once. If it crashes after, the retry finds
the key and replays the response.

## Consumers of queues and streams

The same pattern applies to message consumers (Level 2, lesson 5):

- **Dedup table**: store processed message IDs in the same transaction as the consumer's
  database writes.
- **Store the offset with the output**: a consumer that writes results and its consumed
  offset in one transaction resumes exactly where its committed state ends, so replayed
  messages are either already reflected or not at all.
- **Idempotent sinks**: upserts keyed by event ID, or "apply if version is newer".

When the side effect is external and does not support idempotency keys (some email or SMS
providers), you cannot get a perfect guarantee. Choose the least harmful failure: record
"attempting" before sending and accept a rare duplicate, or record "sent" first and accept
a rare miss — and say which you chose in the design.

## How It Actually Works

Why can't the transport layer just fix this? TCP does deduplicate and order bytes — but
only *within one connection*. When a connection dies and the client reconnects, TCP has no
memory of what the previous connection delivered to the application or whether the
application processed it. The ambiguity moves up a layer, to the application, which is
the only place that knows what "the same operation" means. That is the end-to-end
argument: guarantees like "exactly once" must be implemented at the endpoints that
understand the operation, with lower layers providing optimization only.

An idempotency key works because it turns an ambiguous *event* ("a request arrived") into
a named *fact* ("operation 5f1c… has happened, with this result"). Facts can be checked
and recorded atomically alongside the effect in one local transaction. Retries then become
lookups of that fact. The network can still deliver messages many times — but the effect
is applied once.

## Common mistakes

- **Generating the idempotency key on the server** (a new key per attempt defeats the
  purpose). The client must reuse it across retries of the same logical operation.
- **Checking the key, then performing the effect, then storing the key** in separate
  transactions.
- **Assuming a broker's "exactly-once" covers your external side effects.**
- **Deduplicating by payload** — two legitimate $40 charges look identical.
- **Keys that expire before clients stop retrying.**

## Exercise

1. Run `idempotency.py`. Then write a test that fires the same key from 10 threads at once
   (use separate connections to a file-based database) and confirms exactly one debit.
2. Design idempotency for "transfer money between two accounts that live in different
   services". Where does the key live, and how does it interact with the saga from lesson 2?
3. A consumer reads order events and increments a daily revenue counter in Redis. Make it
   idempotent under redelivery. What must you store, and where?
