---
description: "How CDNs work: edge caching, cache keys, Cache-Control headers, TTLs vs purges, origin shielding, and what a CDN can and cannot accelerate."
---

# 08 · CDNs & Edge Caching

A content delivery network (CDN) is a large, shared set of caching servers placed in
many locations ("points of presence", or PoPs) close to users. When a user requests a
file, a nearby edge server answers from its cache if it can; otherwise it fetches from
your **origin** and keeps a copy for the next user in that area.

The payoff is threefold: lower latency (short round trips to a nearby PoP), much less
bandwidth and load on your origin, and absorption of traffic spikes. For media-heavy
products — images, video, downloads, JavaScript bundles — a CDN is usually the single
most effective scaling component.

## What a CDN accelerates

| Content | CDN benefit |
|---|---|
| Static assets (JS, CSS, fonts, images) | Very high — cache for a long time with versioned filenames |
| Video segments | Very high — the same segments are watched by many users |
| Public API responses that are identical for everyone | High, with short TTLs |
| Personalized pages / per-user API responses | Little caching; still gains connection setup near the user |
| Writes (POST, uploads) | No caching; some benefit from terminating TLS nearby and faster routing |

Even for uncacheable requests, terminating TCP and TLS at a nearby edge helps: the
several handshake round trips (lesson 2) happen over a short distance, and the edge
reuses warm, long-lived connections to the origin.

## Controlling what gets cached

CDNs follow HTTP caching headers from your origin (plus any rules you configure):

```http
Cache-Control: public, max-age=31536000, immutable      # versioned static asset
Cache-Control: public, max-age=60, stale-while-revalidate=30   # public, changes often
Cache-Control: private, no-store                        # per-user or sensitive
```

- `max-age` is how long a shared or browser cache may reuse the response.
- `s-maxage` overrides `max-age` for shared caches (CDNs) only.
- `stale-while-revalidate` lets a cache serve a slightly stale copy while it fetches a
  fresh one in the background, hiding origin latency.
- `private` forbids shared caches from storing it; `no-store` forbids storing at all.

### Cache keys

The **cache key** decides which requests share a cached object. By default it is roughly
the host plus path plus query string. Two traps:

- Irrelevant query parameters (tracking tags like `?utm_source=...`) split one object
  into thousands of cache entries and destroy the hit ratio. Strip or ignore them.
- Varying on headers (`Vary: Cookie`, `Vary: User-Agent`) can multiply cache entries or
  effectively disable caching. Vary only on what really changes the response.

## Invalidation: versioning beats purging

To change a cached file you can wait for the TTL, purge it, or change its URL.

- **Versioned URLs** (`app.3f9a2c.js`, `/img/logo.png?v=12`): the new version is a new
  object; the old one is never requested again. Allows year-long TTLs with instant
  updates. This is the standard approach for build artifacts.
- **Purge APIs**: tell the CDN to drop an object everywhere. Useful for mistakes and
  mutable URLs, but propagation to every PoP takes time, and purging huge numbers of
  objects at once can send a burst of misses to your origin.
- **Short TTLs**: simple, but every expiry costs an origin request per PoP.

## Worked example: serving a news site's front page

A news site gets a surge of traffic when a big story breaks. The front page is the same
for all anonymous users, updated every few minutes. Logged-in users see a small
personalized header.

Design:

1. Serve the anonymous front page HTML with `Cache-Control: public, s-maxage=30,
   stale-while-revalidate=60`. Each PoP asks the origin at most about once per 30 seconds.
2. Load the personalized header separately via a small, uncached API call from the page's
   JavaScript, so the heavy HTML stays shareable.
3. Article images and scripts use versioned URLs with a one-year TTL.
4. Enable **origin shielding**: PoPs fetch from a designated mid-tier cache instead of
   directly from the origin, so a miss in 100 PoPs becomes one origin request.

Rough effect, with assumptions stated: if 200 PoPs each refresh the page every 30 s, the
origin sees under 7 requests per second for the front page regardless of whether 1,000
or 1,000,000 people are reading it. With shielding it sees about one every 30 s.

## How It Actually Works

**Getting the user to the nearest PoP.** CDNs use two main techniques. With **DNS-based
routing**, the CDN's DNS servers answer the lookup for `cdn.example.com` with the IP of a
PoP chosen based on the resolver's location and PoP health. With **anycast**, the same IP
address is announced from many PoPs, and internet routing (BGP) naturally delivers
packets to a topologically close one. Many CDNs combine both.

**Inside a PoP.** A PoP holds many cache servers. Requests for a given object are
usually hashed to a particular server within the PoP (so the PoP stores one copy, not
one per server), and hot objects may be kept in memory with the rest on SSD. On a miss,
the edge fetches from a shield/parent cache or the origin, often **collapsing** many
simultaneous requests for the same missing object into a single upstream fetch.

**Tiered caching.** Edge → regional shield → origin forms a hierarchy. Each tier sees
only the misses of the tier below, so the origin's load depends on the number of unique
objects and their TTLs, not on the number of viewers.

## Common mistakes

- **Caching personalized responses publicly** — the classic incident where users see
  each other's account pages. Mark per-user responses `private` or `no-store`.
- **Cookies on static assets** causing CDNs to bypass the cache.
- **Relying on purges for routine deploys** instead of versioned filenames.
- **Assuming the CDN protects an uncached origin** — cache-busting query strings or
  uncacheable paths go straight through to your servers.
- **No origin capacity plan for a full purge or CDN failover.**

## Exercise

1. Pick any public website and open your browser's developer tools (Network tab). For
   five different resources, record the `Cache-Control`, `Age`, and any CDN-specific
   cache-status header. Explain what each tells you.
2. Write the caching headers you would use for: a user's profile photo (public, changes
   rarely), a stock price API (public, changes every second), and a checkout page.
3. A marketing team appends `?campaign=...` to every link. Estimate the effect on hit
   ratio and propose a fix.
