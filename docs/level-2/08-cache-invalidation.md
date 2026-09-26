---
description: "Cache invalidation strategies and their race conditions: TTLs, delete-on-write, versioned keys, CDC-driven invalidation, leases, and stampede control."
---

# 08 · Cache Invalidation Strategies

Level 1 introduced caching. The hard part was deferred: when the underlying data
changes, how does the cache find out? Every strategy trades freshness, complexity, and
load on the source. And several obvious strategies contain race conditions that leave
stale data in the cache *indefinitely* — the worst kind of bug, because it is silent.

## Strategy 1: TTL only

Every entry expires after a fixed time. No invalidation code at all.

- Staleness is bounded by the TTL. Simple and robust.
- Short TTLs mean more misses; long TTLs mean more staleness.
- Good for data where "up to N seconds old" is acceptable: product listings, public
  profiles, feed rankings.

Adding **jitter** (e.g. TTL of 300 ± 30 seconds) prevents large batches of keys cached at
the same moment from all expiring at the same moment.

## Strategy 2: delete on write

After updating the database, delete the cache key; the next read repopulates it.

```text
write path:  UPDATE db  →  DELETE cache[key]
read path:   GET cache[key] → on miss: SELECT db → SET cache[key]
```

Why *delete* instead of *update* the cache? Two concurrent writers updating the cache can
apply in the opposite order from the database, leaving the older value cached. Deleting
is order-insensitive: any later read refetches the truth.

### The race that remains

Even delete-on-write has a classic race:

```text
t1  Reader: cache miss for key
t2  Reader: SELECT → gets OLD value
t3  Writer: UPDATE db to NEW value
t4  Writer: DELETE cache[key]        (nothing to delete)
t5  Reader: SET cache[key] = OLD      ← stale, and stays until TTL
```

The reader's slow write-back lands after the invalidation. It is rare (the read must be
slower than the entire write + delete), but at high traffic, rare things happen daily.
Mitigations:

- **Always keep a TTL** as a backstop, so no staleness lasts forever.
- **Leases / versioned sets**: on a miss, the cache issues a token; the later `SET` is
  accepted only if no delete happened since the token was issued.
- **Delayed double delete**: delete, then delete again a short time later (a pragmatic
  heuristic, not a guarantee).

## Strategy 3: versioned keys

Include a version in the key: `user:42:v17`. Writes increment the version (stored in the
database row or a small, authoritative counter); readers build the key from the current
version. Old entries are never invalidated — they just stop being requested and get
evicted. This sidesteps the delete races entirely, at the cost of reading the version
first. It is ideal for derived data (rendered fragments, computed aggregates).

## Strategy 4: invalidation from the change stream

Rather than having every code path remember to invalidate, subscribe to the database's
change log (change data capture — Level 3, lesson 4). A consumer sees every committed
change and deletes or refreshes the affected keys.

- No code path can forget to invalidate — even manual fixes and migrations are covered.
- Invalidation is asynchronous (typically sub-second lag), and you need a reliable
  consumer and a mapping from rows to cache keys.

## Worked example: reproducing and fixing the race

```python
# invalidation_race.py — deterministic replay of the delete-on-write race, then leases
db = {"price:1": 100}

class Cache:
    def __init__(self):
        self.data, self.leases, self.next_token = {}, {}, 0
    def get(self, k):
        return self.data.get(k)
    def lease(self, k):                      # issued on a miss
        self.next_token += 1
        self.leases[k] = self.next_token
        return self.next_token
    def set_with_lease(self, k, v, token):   # accepted only if lease still valid
        if self.leases.get(k) == token:
            self.data[k] = v
            del self.leases[k]
            return True
        return False
    def delete(self, k):
        self.data.pop(k, None)
        self.leases.pop(k, None)             # invalidates outstanding leases

def run(use_lease):
    cache = Cache()
    # t1: reader misses, (optionally) gets a lease
    token = cache.lease("price:1") if use_lease else None
    # t2: reader reads OLD value from DB
    old = db["price:1"]
    # t3/t4: writer updates DB and deletes key
    db["price:1"] = 120
    cache.delete("price:1")
    # t5: reader writes back what it read
    if use_lease:
        cache.set_with_lease("price:1", old, token)
    else:
        cache.data["price:1"] = old
    db["price:1"] = 100                      # reset for next run
    return cache.get("price:1")

print("plain cache-aside ->", run(False))   # 100: stale value stuck in cache
print("with leases       ->", run(True))    # None: stale write rejected, next read refetches
```

The lease approach is the idea behind the "lease" mechanism described in published work on
large Memcached deployments, and similar compare-and-set tokens exist in several caches.

## Stampedes on invalidation

Invalidating a hot key sends every concurrent reader to the database at once. Options:

- **Request coalescing**: only the lease holder fetches; others wait briefly or get the
  stale value.
- **Serve stale while revalidating**: keep the old value marked stale, return it, and
  refresh in the background.
- **Refresh ahead**: refresh popular keys shortly before expiry.

## Choosing

| Data | Suggested strategy |
|---|---|
| Public, tolerant of minutes of staleness | TTL with jitter |
| User-visible edits (profile, settings) | Delete-on-write + TTL backstop; read-your-writes from DB right after edit |
| Derived/aggregate views | Versioned keys |
| Many writers, many code paths | CDC-driven invalidation + TTL |
| Money, inventory at checkout | Do not serve from cache for the decision; read the source |

## How It Actually Works

Staleness bugs come from the fact that "read from DB then write to cache" is not atomic
with respect to "write to DB then invalidate cache". Two independent sequences of
operations interleave, and the cache has no idea which value is newer — it simply keeps
whatever was written last. Every fix introduces some notion of *ordering* the cache can
check: a lease token (was there an invalidation since you read?), a version number (is
this value newer than what I have?), or a single ordered stream of changes (CDC applies
invalidations in commit order). TTLs do not fix ordering; they bound how long a mistake
survives, which is why they are the universal backstop.

In multi-region deployments the problem compounds: each region's cache must be
invalidated, and a region's cache can be refilled from a local replica that has not yet
received the change. A common approach is to invalidate from each region's own replication
stream, so invalidation in a region happens only after the region's replica has the new
data.

## Common mistakes

- **Updating the cache on write instead of deleting**, creating ordering bugs between
  concurrent writers.
- **No TTL at all** — any missed invalidation becomes permanent.
- **Invalidating before committing the transaction**: a reader repopulates the cache with
  the old value before the commit lands.
- **Invalidating in one region only.**
- **Forgetting derived keys**: updating a user's name but not the cached team page that
  embeds it.

## Exercise

1. Run `invalidation_race.py`. Then implement versioned keys as a third strategy and show
   it also avoids the race.
2. Move the `delete` to *before* the database update in the plain version. Construct an
   interleaving that leaves stale data, and explain why deleting after commit is the
   correct order.
3. A product page cache includes price, stock count, and reviews. Assign an invalidation
   strategy and TTL to each part, and decide whether they should be one cache entry or
   three.
