---
description: "The client-server model and HTTP from a designer's view: DNS, TCP and TLS setup, methods, status codes, keep-alive, and stateless servers."
---

# 02 · Client-Server & HTTP Basics

Nearly every system you design will have clients (browsers, mobile apps, other
services) sending requests to servers over HTTP. You do not need to memorize the RFCs,
but you do need to know where time goes in a request, which parts are cacheable, and
what the protocol does and does not promise. Those facts shape designs directly.

## The life of a request

When a browser fetches `https://api.example.com/orders/42`, roughly this happens:

1. **DNS lookup.** Resolve `api.example.com` to an IP address. Often cached by the OS
   or browser; a cold lookup can cost one or more round trips to resolvers.
2. **TCP handshake.** SYN, SYN-ACK, ACK — one round trip before any data flows.
3. **TLS handshake.** Negotiate keys and verify the certificate. TLS 1.3 needs one
   round trip for a new connection (and can resume in fewer).
4. **HTTP request and response.** One more round trip, plus server processing time.
5. **Connection reuse.** With keep-alive (default in HTTP/1.1) and multiplexing
   (HTTP/2, HTTP/3), later requests skip steps 1–3.

For a user 100 ms of round-trip time away from your server, a cold request needs about
three round trips before the first response byte: roughly 300 ms of pure network
latency before your code does anything. This is why CDNs terminate connections near
users (lesson 8), and why APIs that make a client perform ten sequential calls feel slow.

## HTTP in one table

| Method | Meaning | Safe? | Idempotent? |
|---|---|---|---|
| GET | Read a resource | Yes | Yes |
| HEAD | GET without the body | Yes | Yes |
| POST | Create or trigger processing | No | No |
| PUT | Replace a resource at a known URL | No | Yes |
| PATCH | Partially modify a resource | No | Not necessarily |
| DELETE | Remove a resource | No | Yes |

"Idempotent" means repeating the request has the same effect as sending it once.
This matters because clients and proxies retry. A retried `PUT /users/7 {name: "Ana"}`
is harmless; a retried `POST /payments` can charge twice unless you design for it
(Level 3, lesson 3).

Status codes you will reason about constantly:

- `200 OK`, `201 Created`, `204 No Content` — success.
- `301/308` permanent redirect, `302/307` temporary redirect (the URL-shortener project
  depends on this distinction).
- `304 Not Modified` — the client's cached copy is still valid.
- `400` bad request, `401` not authenticated, `403` not allowed, `404` not found,
  `409` conflict, `429` too many requests (rate limiting).
- `500` server bug, `502` bad gateway (upstream returned garbage), `503` unavailable
  (overloaded or down), `504` gateway timeout (upstream too slow).

## Statelessness

HTTP is stateless: each request carries everything the server needs to process it.
A server does not "remember" the previous request from the same client unless you
build that memory somewhere.

This property is what makes horizontal scaling possible. If any app server can handle
any request, a load balancer can spread traffic freely and a failed server can be
replaced with no loss. When application code keeps session data in local memory, you
lose that — requests must be pinned to the server holding the session.

The standard fix is to move state *out* of the app tier:

- Session data in a shared store (a database or an in-memory store such as Redis), keyed
  by a session ID in a cookie.
- Or a signed token (for example a JWT) that carries the claims itself, verified with a
  key on every request. Tokens avoid a lookup but are hard to revoke before expiry.

## Worked example: a minimal HTTP server and client

This runs with only the Python standard library. It shows statelessness, idempotency,
and status codes in about 40 lines.

```python
# server.py — run with: python3 server.py
import json
from http.server import BaseHTTPRequestHandler, HTTPServer

STORE = {}  # stands in for a database; NOT shared across processes

class Handler(BaseHTTPRequestHandler):
    def _send(self, code, body=None):
        data = json.dumps(body).encode() if body is not None else b""
        self.send_response(code)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(data)))
        self.end_headers()
        self.wfile.write(data)

    def do_GET(self):
        key = self.path.strip("/")
        if key in STORE:
            self._send(200, {"key": key, "value": STORE[key]})
        else:
            self._send(404, {"error": "not found"})

    def do_PUT(self):
        key = self.path.strip("/")
        length = int(self.headers.get("Content-Length", 0))
        value = json.loads(self.rfile.read(length) or b"null")
        created = key not in STORE
        STORE[key] = value                     # same result no matter how often
        self._send(201 if created else 200, {"key": key, "value": value})

HTTPServer(("127.0.0.1", 8080), Handler).serve_forever()
```

```python
# client.py — run while the server is running
import json, urllib.request, urllib.error

def call(method, path, body=None):
    data = json.dumps(body).encode() if body is not None else None
    req = urllib.request.Request("http://127.0.0.1:8080" + path, data=data, method=method)
    try:
        with urllib.request.urlopen(req, timeout=2) as r:
            return r.status, json.loads(r.read())
    except urllib.error.HTTPError as e:
        return e.code, json.loads(e.read())

print(call("GET", "/color"))              # (404, {'error': 'not found'})
print(call("PUT", "/color", "blue"))      # (201, ...) created
print(call("PUT", "/color", "blue"))      # (200, ...) retry: same state, idempotent
print(call("GET", "/color"))              # (200, {'key': 'color', 'value': 'blue'})
```

Now imagine running two copies of `server.py` behind a load balancer. `STORE` lives in
each process's memory, so a `PUT` to one and a `GET` to the other returns 404. That is
the stateful-server problem in miniature, and the fix is exactly what the previous
section described: move `STORE` into a shared database.

## How It Actually Works

**Why the handshakes cost round trips.** TCP must agree on initial sequence numbers in
both directions before it can guarantee ordered, reliable delivery, which needs one full
exchange. TLS then needs the client and server to agree on a shared secret via key
exchange and for the client to validate the server certificate against trusted
authorities. Neither can be skipped for a brand-new connection, which is why connection
reuse is such a large win and why connection pools exist in every serious HTTP client.

**How HTTP/2 and HTTP/3 change things.** HTTP/1.1 sends one request at a time per
connection, so browsers open several connections in parallel. HTTP/2 multiplexes many
request streams over one TCP connection, but a single lost TCP packet stalls all streams
(head-of-line blocking at the transport layer). HTTP/3 runs over QUIC, which is built on
UDP and handles loss per stream, and it merges the transport and TLS handshakes. For a
designer the takeaway is simple: fewer, reused connections are cheaper than many short
ones, and chatty APIs pay for every round trip.

**Where caching hooks in.** HTTP defines headers like `Cache-Control: max-age=3600`,
`ETag`, and `If-None-Match`. A client or proxy that holds a response with an `ETag` can
ask "has it changed?" and receive a tiny `304` instead of the full body. Every layer —
browser, CDN, reverse proxy — uses these same headers, so correct headers on your API
are free performance.

## Common mistakes

- **Using GET for actions with side effects.** Crawlers, prefetchers, and caches assume
  GET is safe and will happily trigger it.
- **Storing sessions in app-server memory** and then being surprised by random logouts
  after adding a second server or deploying.
- **Returning 200 with an error body.** Monitoring, retries, and caches key off status
  codes; lying in them breaks all three.
- **Designing chatty APIs.** Ten sequential calls from a mobile client on a slow network
  can cost seconds. Batch or aggregate on the server.
- **No timeouts on outbound calls.** A client without a timeout waits forever on a hung
  server and ties up a thread or connection while it does.

## Exercise

1. Run the server and client above. Add a `DELETE` handler and show that sending it twice
   leaves the store in the same state (what status should the second call return, and
   does that break idempotency? Argue your answer).
2. Add a `POST /counter` endpoint that increments a number. Call it twice with the same
   body and explain why it is not idempotent.
3. Estimate the time to first byte for a user 150 ms RTT away on a cold HTTPS connection
   with TLS 1.3, assuming 20 ms of server processing and a cached DNS entry. Then
   estimate it for the second request on the same connection.
