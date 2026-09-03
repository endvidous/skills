---
name: caching-strategy
description: Use when adding, reviewing, or debugging any cache — Redis/Memcached application caches, in-process memory caches, HTTP/CDN caching headers, or database query caches; when choosing TTLs, invalidation strategy, or read/write patterns; or when diagnosing stale data, cache stampedes, or hot keys. Complements llm-cost-latency (prompt caching) and capacity-estimation (cache sizing). Not about CPU caches.
---

# Caching Strategy

"There are only two hard things in Computer Science: cache invalidation and naming things" (Phil Karlton). A cache is a **deliberate consistency bug** you accept in exchange for latency or load relief — so every caching decision starts from "how stale can this data be before something breaks?" and works backward. And caching *hides* load rather than removing it: at a 99% hit rate the database is sized for 1% of traffic, which means a cache flush or cold restart is a self-inflicted outage (the 99%→90% hit-rate drop is a 10× DB load increase — `capacity-estimation`).

## Decision 1: Should this be cached at all?

Before caching, try the fixes that don't add a consistency problem: kill the N+1, add the index, denormalize the read model, paginate. Cache when a *measured* latency or load problem survives those. What a cache costs you: staleness bugs, invalidation code (the hard thing), a cold-start dependency, and one more system in every incident. Data that's cheap to compute, rarely read, or must always be fresh (balances, inventory at checkout, authorization decisions) doesn't get cached. Authorization results in particular: a cached "allowed" that outlives a revocation is a security bug (`security-boundaries`).

## Decision 2: Which layer

Cache as close to the consumer as correctness allows; each layer down costs a network hop but widens sharing:

| Layer | Use for | Watch out |
|---|---|---|
| **Browser / HTTP** | Static assets, public API GETs | Can't invalidate a browser remotely — TTL is a promise (Decision 6) |
| **CDN** | Assets, public pages, public API | `s-maxage` + purge API; per-user content must never be CDN-cached (`private`) |
| **In-process memory** | Hot, small, slow-changing (config, feature flags) | Per-node inconsistency; short TTL is the coherence mechanism; dies with deploys |
| **Distributed (Redis/Memcached)** | Shared server state: sessions, computed aggregates, DB-query relief | Network hop (~0.5ms); a shared dependency — its outage must degrade, not crash (every call error-handled, fallback to source) |
| **Database-internal** | Buffer pool, materialized views | Already there — often the reason you don't need Redis |

Default: HTTP/CDN first for anything public (it's free, standardized infrastructure), Redis for shared dynamic state, in-process only for config-grade data with a short TTL.

## Decision 3: Read/write pattern

- **Cache-aside (lazy loading) — default.** App checks cache → miss → read source → populate. Simple, transparent, only caches what's read; cost: first-read latency and a stale window after writes.
- **Read-through / write-through** — the cache layer itself loads/stores. Write-through keeps reads fresh at the cost of write latency (every write pays both systems) and cache pollution with never-read data; still has the dual-write inconsistency problem.
- **Write-behind (write-back)** — write to cache, flush to source asynchronously. Fast writes, but the cache holds the only copy during the window: crash = data loss. Only for data you can afford to lose (metrics, counters) — never for records (`data-modeling`).

## Decision 4: Invalidation

- **TTL on everything, always** — even entries with event-driven invalidation get a TTL. The TTL is the backstop that bounds how long any invalidation bug can persist. Pick it from the business staleness tolerance ("how old may a profile page be? 60s") — not from a habit of `3600`.
- **Event-driven: invalidate by DELETE, not by SET.** On write, delete the cache key and let the next read repopulate. Writing the new value from the writer races with concurrent readers repopulating from a stale read — the interleaving leaves stale data in the cache *forever* (absent TTL). This stale-set race is real enough that Facebook built **leases** into memcache — a per-key token that rejects sets from readers that lost the race — which also cut stampede-driven DB load from 17K to 1.3K queries/s (Scaling Memcache at Facebook, NSDI '13).
- **Versioned keys** — put a version/updated-at in the key (`user:42:v7` or hash of inputs); writers bump the version, old entries age out via TTL and eviction. No delete fan-out, immune to the stale-set race; cost: you need a cheap way to know the current version, and dead entries occupy memory until evicted. Best when one write invalidates many derived entries.
- Deletes/invalidations that must span systems (cache + DB) have the same dual-write problem as any distributed write — if it must be reliable, it goes through the outbox/CDC path (`distributed-reliability` Decision 4), not a best-effort `DEL` after commit.

## Decision 5: Stampede protection (hot keys)

When a popular key expires, every concurrent request misses and hits the source at once — the cache stampede / thundering herd. Defenses, cheapest first:

1. **Jitter TTLs** (`ttl + rand(0, 10%)`) — one line; prevents synchronized expiry of keys populated together (deploys, warmups, midnight crons).
2. **Request coalescing / singleflight** — at most one in-flight recompute per key per node (Go's `singleflight`, a per-key promise/mutex in Node); distributed variant: a short-TTL lock key (`SET lock:key NX PX`), losers serve stale or wait.
3. **Serve-stale (soft TTL)** — store a logical expiry inside the value; after it passes, serve the stale copy and refresh in the background. Smooths the cliff entirely; HTTP's version is `stale-while-revalidate` (Decision 6).
4. **Probabilistic early expiration** — each request refreshes early with probability rising near expiry (XFetch; Vattani et al., "Optimal Probabilistic Cache Stampede Prevention") — for the few ultra-hot keys where coalescing alone still leaves a refresh spike.

Default: jitter always; coalescing + serve-stale for known-hot keys. Also protect the *source*: the recompute path needs a concurrency cap so even a defeated defense degrades instead of cascading (`distributed-reliability` Decision 7).

## Decision 6: HTTP and CDN caching

- **Immutable assets** (content-hashed filenames: `app.3f9a1c.js`): `Cache-Control: public, max-age=31536000, immutable` — the hash in the name *is* the invalidation, `immutable` skips even revalidation.
- **HTML / API responses**: default `no-store` for personalized/authenticated responses; `private, no-cache` + `ETag` for per-user content that may revalidate; `public, max-age=<short>, s-maxage=<longer>` for public content — browsers stay fresh, the CDN (which you *can* purge) absorbs the load.
- **`stale-while-revalidate=N` / `stale-if-error=N`** (RFC 5861): serve stale during background revalidation / origin failure — the standardized serve-stale pattern, and cheap resilience.
- `ETag`/`Last-Modified` make expired entries cheap (304, no body) — set them wherever responses are cacheable.
- Remember the asymmetry: you can purge your CDN; you cannot purge browsers. Long browser TTLs only on content whose *URL changes* when content changes.

## Decision 7: Operating the cache

- **Key naming:** namespaced and versioned — `app:entity:id[:vN]`; a global key rename is your schema migration tool for cached shapes (bump the namespace, old entries age out).
- **Eviction:** set a memory bound and policy explicitly (Redis `maxmemory` + `allkeys-lru` for pure cache; `volatile-*` only when the instance mixes cache and durable-ish data — better: don't mix; cache Redis and queue/session Redis have different loss tolerances).
- **Negative caching:** cache "not found" with a *short* TTL to stop miss-storms on nonexistent keys — but never negative-cache auth failures or transient errors.
- **Instrument it:** hit rate, per-key-pattern if possible, plus source-load-on-miss (`observability-slo` USE for the pool). A silently falling hit rate is capacity erosion; alert on it before the DB discovers it.
- **Serialize small:** cache the fields the read path needs, not the ORM entity graph; big values amplify network cost and eviction pressure.

## Review checklist

- What's the measured problem this cache solves, and was a non-cache fix considered?
- Staleness tolerance stated by the business, TTL derived from it, jitter applied?
- Invalidation: delete-on-write (not set-on-write) or versioned keys? TTL backstop present?
- Stampede story for hot keys? Source protected by a concurrency cap regardless?
- Cache down / cold: does the system degrade (slower, shed load) rather than crash? Is the DB survivable at 0% hit rate, and if not, is that written down as a risk?
- Anything personalized reachable by a shared cache (CDN, unscoped key)? Keys include tenant/user scope (`security-boundaries`)?
- Hit rate and miss-cost instrumented and alerted?

## Anti-patterns

- Caching as the first fix for a slow query nobody profiled
- Writer SETs the new value into the cache (stale-set race) instead of deleting
- No TTL because "we invalidate on write" — until the one code path that doesn't
- Uniform TTLs on co-populated keys (synchronized expiry stampede)
- Caching authorization/entitlement results past their revocation window
- Per-user data under a key without the user/tenant ID — cache-layer IDOR
- Redis treated as durable storage because it's already there (write-behind for records)
- `Cache-Control: public` on authenticated responses; long browser TTLs on unversioned URLs
- One Redis for cache + sessions + queues — one eviction policy can't serve all three
