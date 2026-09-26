---
description: "Consistent hashing from first principles: why mod-N rehashing fails, the hash ring, virtual nodes, a runnable implementation, and rendezvous hashing."
---

# 03 · Consistent Hashing

Suppose you spread cache keys across servers with `hash(key) % N`. It works — until N
changes. Add a fifth server to four and almost every key maps to a different server. For
a cache that means a near-total miss storm; for a database it means moving almost all
the data. Consistent hashing is a family of techniques that make adding or removing a
node move only about **1/N** of the keys.

## The problem, measured

```python
import hashlib
def h(s): return int(hashlib.md5(s.encode()).hexdigest(), 16)

keys = [f"user:{i}" for i in range(100_000)]
moved = sum(1 for k in keys if h(k) % 4 != h(k) % 5)
print(f"{moved / len(keys):.0%} of keys move when going from 4 to 5 servers")  # ~80%
```

With modulo, a key stays put only if its hash gives the same remainder for both N and
N+1, which is true for only about 1/(N+1) of keys. Going 4 → 5 moves about 80%.

## The ring

Map both **nodes** and **keys** onto the same circular hash space (say 0 to 2³²−1).
A key belongs to the first node found moving clockwise from the key's position.

```text
             0
        N3 ●   ● k1 → belongs to N1
      ●           ●  N1
   k3            
      ●           ● k2 → belongs to N2
        N2 ●   ●
```

- **Adding a node** places it at one point on the ring. It takes over only the keys
  between its predecessor and itself — from a single neighbor. Everything else stays.
- **Removing a node** hands its keys to its clockwise successor only.

## Virtual nodes

With one point per node, the arcs between points are random lengths, so some nodes own
far more of the ring than others. And when a node leaves, *one* neighbor absorbs all its
load. The fix: give each physical node many points ("virtual nodes", say 100–200), by
hashing `"nodeA#0"`, `"nodeA#1"`, and so on. Load then averages out across many small
arcs, and a departing node's keys are spread over many neighbors. Weighting is easy too:
a machine with twice the capacity gets twice the virtual nodes.

## Worked example: a ring with virtual nodes

```python
# ring.py — consistent hashing with virtual nodes (standard library only)
import bisect, hashlib
from collections import Counter

def _hash(s):
    return int(hashlib.md5(s.encode()).hexdigest()[:8], 16)   # 32-bit position

class HashRing:
    def __init__(self, vnodes=150):
        self.vnodes = vnodes
        self.positions = []            # sorted ring positions
        self.owner = {}                # position -> physical node

    def add(self, node):
        for i in range(self.vnodes):
            p = _hash(f"{node}#{i}")
            bisect.insort(self.positions, p)
            self.owner[p] = node

    def remove(self, node):
        for i in range(self.vnodes):
            p = _hash(f"{node}#{i}")
            self.positions.remove(p)
            del self.owner[p]

    def lookup(self, key):
        p = _hash(key)
        i = bisect.bisect_right(self.positions, p) % len(self.positions)  # wrap around
        return self.owner[self.positions[i]]

keys = [f"user:{i}" for i in range(100_000)]
ring = HashRing()
for n in ["A", "B", "C", "D"]:
    ring.add(n)
before = {k: ring.lookup(k) for k in keys}
print("load with 4 nodes:", sorted(Counter(before.values()).items()))

ring.add("E")
after = {k: ring.lookup(k) for k in keys}
moved = sum(before[k] != after[k] for k in keys)
print(f"moved after adding E: {moved / len(keys):.1%} (ideal: {1/5:.0%})")
print("all moved keys went to E:", all(after[k] == "E" for k in keys if before[k] != after[k]))
```

Running it shows roughly even load across A–D, about 20% of keys moving when E joins,
and every moved key going to E — exactly the property we wanted. (Hash collisions
between virtual-node positions are ignored here for brevity; production code handles
them.)

## Rendezvous (highest random weight) hashing

An alternative with the same "move only 1/N" property and no ring: for each key,
compute `score = hash(key + node)` for every node and pick the node with the highest
score. When a node is removed, only its keys move, each to its own second-best node.

```python
def rendezvous(key, nodes):
    return max(nodes, key=lambda n: _hash(f"{key}|{n}"))
```

It costs O(N) per lookup, which is fine for tens of nodes and avoids virtual-node
bookkeeping. It also makes "top K nodes for this key" (for placing replicas) trivial:
take the K highest scores.

## Where it is used

Consistent hashing appears wherever a changing set of nodes shares keys without a central
directory: client-side sharding across cache servers, request routing to cache-friendly
backends, and data placement in Dynamo-style databases, where a key's replicas are
typically the next few distinct physical nodes clockwise on the ring. Many systems
instead use a fixed number of logical partitions with an explicit assignment table
(lesson 2), which achieves similar movement with more control. Both are legitimate; the
ring shines when there is no coordinator to hold the table.

## How It Actually Works

The "1/N" property follows from the geometry. Every key's owner is determined by the
nearest node position clockwise. Inserting a new position changes the nearest position
only for keys in the arc it now covers — keys elsewhere have the same nearest neighbor as
before. With V virtual nodes per physical node, the new node inserts V points, each
capturing one small arc from whichever node previously owned it, so it takes about
1/(N+1) of the ring in total, collected from many donors.

Load balance depends on the variance of arc lengths. With one point per node the largest
arc can easily be several times the average. With V points per node, each node's total
share is a sum of V random arcs, and its relative spread shrinks roughly like 1/√V —
which is why a hundred or so virtual nodes per server typically gives balance within
several percent.

The lookup is a binary search over the sorted positions: O(log(N·V)), microseconds even
for thousands of positions.

Note what consistent hashing does *not* solve: a single hot key still lands on one node.
Keys move evenly; traffic per key does not.

## Common mistakes

- **Using `hash(key) % N` for caches or shards that resize**, causing mass misses or mass
  data movement.
- **Too few virtual nodes**, producing badly uneven load.
- **Language-default hash functions** that differ between processes (Python's `hash()` on
  strings is randomized per process). Use a stable hash like MD5, xxHash, or MurmurHash
  — cryptographic strength is not needed, stability is.
- **Clients with different ring views** during membership changes, sending the same key
  to different nodes. Membership must be distributed consistently.
- **Assuming it fixes hot keys.**

## Exercise

1. Run `ring.py` with `vnodes=1`, `10`, `150`, and `1000`. Tabulate max/avg load for each.
2. Remove node B and verify that only B's keys move, and that they spread across several
   remaining nodes rather than one.
3. Implement weights: node D has double capacity. Confirm it receives about twice the keys.
4. Implement replica placement: return the first 3 *distinct physical* nodes clockwise
   from a key. Why must you skip virtual nodes belonging to a node already chosen?
