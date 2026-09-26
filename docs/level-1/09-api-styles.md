---
description: "REST vs gRPC vs GraphQL trade-offs, resource modeling, cursor pagination, versioning, and error design for APIs that survive growth."
---

# 09 · APIs: REST, gRPC & GraphQL

The API is the contract between a system and its clients. Once external clients depend
on it, changing it is slow and expensive, so API choices outlive most implementation
choices. This lesson compares the three dominant styles and then covers the parts that
matter regardless of style: pagination, versioning, and errors.

## REST

REST models the system as **resources** identified by URLs and manipulated with standard
HTTP methods.

```text
GET    /users/42                 fetch a user
GET    /users/42/orders?limit=20 list a user's orders
POST   /orders                   create an order
PATCH  /orders/981               change some fields
DELETE /orders/981               cancel/delete
```

**Strengths**: universally understood, works in every language and tool, benefits from
HTTP caching and CDNs, easy to debug with `curl`. **Weaknesses**: clients often need
several calls to assemble a screen (under-fetching) or receive fields they do not need
(over-fetching); JSON is verbose; no built-in schema unless you add one (OpenAPI).

## gRPC

gRPC defines services and messages in Protocol Buffers and generates client and server
code. It runs over HTTP/2.

```protobuf
syntax = "proto3";

service Orders {
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc WatchOrder (GetOrderRequest) returns (stream OrderEvent);  // server streaming
}

message GetOrderRequest { int64 order_id = 1; }
message Order {
  int64  order_id    = 1;
  int64  customer_id = 2;
  string status      = 3;
  int64  total_cents = 4;
}
message OrderEvent { string status = 1; int64 at_unix_ms = 2; }
```

**Strengths**: compact binary encoding, strongly typed contracts, generated clients,
streaming in both directions, deadlines propagated across calls. A strong default for
service-to-service traffic inside a company. **Weaknesses**: not directly callable from
browsers without a proxy layer (gRPC-Web), harder to inspect by eye, HTTP caching and
CDNs do not apply naturally, and L4 load balancing distributes it poorly (lesson 5).

## GraphQL

GraphQL exposes a typed schema; clients send a query describing exactly the shape of
data they want, usually to one endpoint.

```graphql
query {
  user(id: 42) {
    name
    orders(first: 3) { id status total }
  }
}
```

**Strengths**: one round trip per screen, no over-fetching, schema introspection, lets
frontend teams evolve screens without new endpoints. **Weaknesses**: the server must
guard against expensive queries (depth and cost limits), naive resolvers cause the
**N+1 query problem** (one query for the list, then one per item), responses are hard to
cache at the HTTP layer because everything is a POST to one URL, and authorization must
be enforced per field.

## Choosing

| Situation | Reasonable default |
|---|---|
| Public API for third-party developers | REST (+ OpenAPI spec) |
| Internal service-to-service calls, polyglot teams | gRPC |
| Many client types with differing data needs over one data graph | GraphQL (often in front of REST/gRPC services) |
| Heavily cacheable public content | REST behind a CDN |
| Streaming updates between services | gRPC streaming (or a message broker) |

Many organizations use all three: gRPC between services, a GraphQL or REST
"backend-for-frontend" layer for apps, and REST for partners.

## Pagination done right

**Offset pagination** (`?offset=4000&limit=20`) is simple but the database must still
walk past 4,000 rows, and if rows are inserted while a user pages, items get skipped or
repeated.

**Cursor (keyset) pagination** returns an opaque token encoding where the page ended:

```python
# cursor_pagination.py — keyset pagination over (created_at, id), stdlib only
import base64, json, sqlite3

db = sqlite3.connect(":memory:")
db.execute("CREATE TABLE posts(id INTEGER PRIMARY KEY, created_at INTEGER)")
db.execute("CREATE INDEX posts_recent ON posts(created_at DESC, id DESC)")
db.executemany("INSERT INTO posts VALUES (?, ?)", [(i, 1000 + i // 3) for i in range(1, 101)])

def encode(created_at, id_):
    return base64.urlsafe_b64encode(json.dumps([created_at, id_]).encode()).decode()

def page(cursor=None, limit=10):
    if cursor:
        c_at, c_id = json.loads(base64.urlsafe_b64decode(cursor))
        rows = db.execute(
            "SELECT id, created_at FROM posts WHERE (created_at, id) < (?, ?) "
            "ORDER BY created_at DESC, id DESC LIMIT ?", (c_at, c_id, limit)).fetchall()
    else:
        rows = db.execute("SELECT id, created_at FROM posts "
                          "ORDER BY created_at DESC, id DESC LIMIT ?", (limit,)).fetchall()
    next_cursor = encode(rows[-1][1], rows[-1][0]) if len(rows) == limit else None
    return [r[0] for r in rows], next_cursor

ids, cur = page()
print(ids)                  # [100, 99, 98, ..., 91]
ids, cur = page(cur)
print(ids)                  # [90, 89, ..., 81] — no overlap, no gaps
```

The `(created_at, id)` pair is the tiebreaker that makes ordering unique, and the index
lets the database seek straight to the cursor position. (Row-value comparison syntax
works in SQLite 3.15+ and PostgreSQL.)

## Versioning and evolution

- Prefer **additive changes**: new optional fields, new endpoints. Clients should ignore
  unknown fields; Protocol Buffers are designed for this (never reuse a field number).
- For breaking changes, version explicitly (`/v2/...` or a header) and run both versions
  during a migration window.
- Never change the meaning of an existing field silently.

## Errors

Return a correct status code plus a machine-readable body:

```json
{"error": {"code": "INSUFFICIENT_STOCK", "message": "Only 2 left", "retryable": false}}
```

Telling clients whether an error is retryable prevents both pointless retries and
silently dropped work.

## How It Actually Works

**Why protobuf is smaller and faster than JSON.** JSON repeats every field name as text
in every message and represents numbers as decimal strings. Protobuf encodes each field
as a small tag (field number + wire type) followed by the value; integers use variable
length encoding, so small numbers take one or two bytes. Field names never go on the
wire — both sides know them from the schema. This also explains the evolution rules:
a decoder skips tags it does not recognize, so adding fields is safe, but reusing a
number makes old decoders misinterpret new data.

**How GraphQL executes a query.** The server parses the query, validates it against the
schema, then walks the query tree calling a **resolver** function per field. For
`orders` under each of 50 users, a naive resolver issues 50 separate database queries.
The standard fix is a *DataLoader*-style batcher: resolvers request keys, the batcher
waits until the current tick of execution ends, then issues one `WHERE id IN (...)`
query and distributes the results.

## Common mistakes

- **Offset pagination on large, changing lists.**
- **Exposing database tables directly as the API**, so every schema change breaks
  clients.
- **GraphQL without query cost limits**, allowing one request to fan out to millions of
  rows.
- **gRPC behind a connection-level load balancer**, pinning all traffic to one backend.
- **Leaking internal errors** (stack traces, SQL) in error bodies.

## Exercise

1. Design the API for a library-booking system (search books, reserve a copy, cancel a
   reservation, list my reservations) in REST. Mark each endpoint's idempotency.
2. Write the same service as a `.proto` file.
3. Extend `cursor_pagination.py` to page in ascending order as well, and prove with a test
   that inserting new rows between page requests causes no duplicates.
4. For a mobile app home screen needing user profile, 5 recent orders, and 10 recommended
   products, compare the number of round trips under REST and GraphQL. When would you
   add a dedicated REST aggregate endpoint instead of GraphQL?
