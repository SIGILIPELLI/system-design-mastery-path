---
description: "Database indexing for designers: B-tree vs LSM storage, composite and covering indexes, reading query plans, write costs, and indexes across shards."
---

# 07 · Indexing & Query Patterns

Most "the database is slow" incidents are not about the database being too small. They
are about a query reading far more data than it returns. An index is a secondary data
structure that lets the database find rows without scanning the whole table. Designing
indexes around your real query patterns is one of the highest-leverage skills in system
design, and it costs nothing but thought.

## What an index is

Think of the index at the back of a book: sorted terms, each pointing to pages. A
database index is a sorted structure mapping column values to row locations. With an
index on `email`, `WHERE email = 'a@x.io'` becomes a lookup of a few pages instead of a
scan of millions of rows.

Indexes are not free:

- **Every write updates every index** on the table. Ten indexes can make inserts several
  times slower.
- They consume storage and memory; an index that does not fit in memory is much slower.
- The query planner must choose among them, and sometimes chooses poorly.

## Composite indexes and column order

An index on `(customer_id, created_at)` is sorted by `customer_id`, then by `created_at`
within each customer. It efficiently serves:

- `WHERE customer_id = ?`
- `WHERE customer_id = ? AND created_at > ?`
- `WHERE customer_id = ? ORDER BY created_at DESC LIMIT 20`

It does **not** efficiently serve `WHERE created_at > ?` alone, because matching rows
are scattered across every customer's section — the **leftmost prefix** rule. A rule of
thumb for ordering columns: equality-filtered columns first, then the range or sort
column.

A **covering index** includes every column the query needs, so the database answers from
the index alone without visiting the table rows.

## Worked example: watching a query plan change

SQLite ships with Python and has `EXPLAIN QUERY PLAN`, so you can see index choices
locally.

```python
# indexing.py — observe scans vs index searches, and time the difference
import random, sqlite3, time

db = sqlite3.connect(":memory:")
db.execute("""CREATE TABLE orders(id INTEGER PRIMARY KEY, customer_id INT,
              status TEXT, total INT, created_at INT)""")
rng = random.Random(0)
db.executemany("INSERT INTO orders(customer_id,status,total,created_at) VALUES (?,?,?,?)",
               [(rng.randrange(50_000), rng.choice(["paid", "shipped", "refunded"]),
                 rng.randrange(100, 50_000), i) for i in range(500_000)])

q = ("SELECT id, total FROM orders WHERE customer_id = ? "
     "ORDER BY created_at DESC LIMIT 20")

def show(label):
    plan = db.execute("EXPLAIN QUERY PLAN " + q, (123,)).fetchall()
    t = time.perf_counter()
    for c in range(200):
        db.execute(q, (c,)).fetchall()
    ms = (time.perf_counter() - t) * 1000 / 200
    print(f"{label}: {[row[-1] for row in plan]}  ~{ms:.3f} ms/query")

show("no index        ")
db.execute("CREATE INDEX by_customer ON orders(customer_id)")
show("customer_id     ")
db.execute("CREATE INDEX by_customer_time ON orders(customer_id, created_at)")
show("(customer, time)")
db.execute("CREATE INDEX covering ON orders(customer_id, created_at, total)")
show("covering        ")
```

Expect the first plan to say `SCAN orders` and use a temporary B-tree for sorting; the
single-column index to `SEARCH` but still sort; the composite index to search *and*
return rows already in order; and the covering index to mention a `COVERING INDEX`. The
timings on your machine will vary, but the no-index case should be dramatically slower —
each query reads all 500,000 rows to return 20.

## Query patterns that defeat indexes

- **Functions on the indexed column**: `WHERE LOWER(email) = ?` cannot use an index on
  `email` (unless you create an expression index on `LOWER(email)`).
- **Leading wildcards**: `LIKE '%smith'` cannot use a normal B-tree index. Full-text
  search needs a different structure (Level 3 search lesson).
- **Low selectivity**: an index on `status` with three values often helps little; the
  planner may rightly scan instead.
- **Offset pagination** deep into results (Level 1, lesson 9).
- **`OR` across different columns**, and implicit type conversions, depending on the
  database.
- **N+1 queries**: fetching a list then querying once per item. Batch with `IN (...)` or
  a join.

## Indexes in a sharded world

When data is partitioned by `customer_id`, an index on `customer_id` is naturally local.
But a query by `email` must either:

- **Scatter-gather** to every shard's local index, or
- Use a **global secondary index** partitioned by `email`: one targeted lookup, but every
  user write must also update an index entry that may live on another shard. Such global
  indexes are usually updated asynchronously, so they can be briefly stale — a newly
  registered email might not be findable for a moment.

A common pattern is an explicit lookup table (`email → customer_id`) maintained by the
application or by change data capture, treated as a first-class part of the design.

## How It Actually Works

**B-trees.** A B-tree stores keys in fixed-size pages (often a few KB to 16 KB). Each
internal page holds hundreds of keys and child pointers, so the tree is very shallow —
three or four levels can index hundreds of millions of rows. A lookup reads one page per
level, and the top levels are almost always cached in memory, so a lookup often costs
one or two disk reads. Updates modify pages in place; when a page is full it splits in
two, and the split can propagate upward. Leaf pages are linked, making range scans a
walk along the leaves.

**LSM trees.** A log-structured merge tree buffers writes in an in-memory sorted
structure (a memtable), backed by a write-ahead log for durability. When the memtable
fills, it is written to disk as an immutable sorted file (an SSTable). Background
**compaction** merges files and discards overwritten or deleted values. Writes are
sequential and fast. Reads may check the memtable and several files; per-file **Bloom
filters** — compact probabilistic sets that can say "definitely not here" — let reads
skip most files. The trade-off: B-trees generally favor reads and predictable latency,
LSM trees favor write throughput, with compaction consuming background I/O.

**The planner.** The database estimates each candidate plan's cost from statistics
(row counts, value distributions) and picks the cheapest. Stale statistics produce bad
plans, which is why databases run periodic `ANALYZE`-style jobs.

## Common mistakes

- **Indexing every column "just in case"** and slowing every write.
- **Wrong column order** in composite indexes.
- **Never reading query plans** — guessing instead of checking.
- **Adding an index to a huge production table without an online build**, locking writes.
- **Forgetting global uniqueness** once sharded: a unique index is only unique per shard.

## Exercise

1. Run `indexing.py`. Then add `WHERE status = 'refunded'` to the query and design the
   best index for it. Verify with `EXPLAIN QUERY PLAN`.
2. Show that `WHERE customer_id + 0 = ?` stops using the index, and explain why.
3. Your users table is sharded by `user_id` across 32 shards. Login looks up by email.
   Compare scatter-gather, a global secondary index, and a lookup table on latency, write
   cost, and failure behavior. Pick one.
