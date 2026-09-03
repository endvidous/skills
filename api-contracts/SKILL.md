---
name: api-contracts
description: Use when designing, adding, or modifying any API surface — REST/HTTP endpoints, webhooks, GraphQL schemas, gRPC services, DTOs, or public SDK interfaces. Enforces error envelopes, pagination, versioning, idempotency keys, field conventions (IDs, money, timestamps, enums), and a breaking-change checklist. Also fires when reviewing API diffs, naming endpoints or fields, or designing webhook delivery.
---

# API Contracts

An API is a promise. The governing law is Hyrum's: "with a sufficient number of users of an API, it does not matter what you promise in the contract: all observable behaviors of your system will be depended on by somebody" ([hyrumslaw.com](https://www.hyrumslaw.com/)). Assume every shipped field, error string, default sort, and latency characteristic is load-bearing for someone. Design accordingly: decide conventions once, before endpoint #1 — each of these is a one-way door whose later change is a client migration.

## Decide once, before the first endpoint

1. Error envelope · 2. Pagination scheme · 3. Versioning + breaking-change policy · 4. Field conventions (casing, timestamps, IDs, money, enums) · 5. Auth scheme. Everything below gives the decision and its reasoning.

## Decision 1: Error format

**Options:**
- **Ad-hoc per endpoint** — no. Clients end up with per-endpoint error parsers; inconsistency is itself a bug (Hyrum's law: they'll parse your strings).
- **RFC 9457 `application/problem+json`** — the IETF standard: `type` (URI), `title`, `status`, `detail`, `instance`, plus extensions clients must ignore if unrecognized ([RFC 9457](https://datatracker.ietf.org/doc/html/rfc9457)). Choose when interoperability with third-party tooling matters or you want to skip bikeshedding.
- **Stripe-style code envelope** — `{ code, message, request_id, details? }` where `code` is a stable machine-readable string (`resource_not_found`). Choose for product APIs where you control the SDKs.

**Default:** either of the last two — the invariant that matters is shared: **a stable, machine-readable `code`/`type` is the contract; the human `message` may change freely and must never be parsed.** One envelope everywhere, including validation errors (a `details`/`errors` array with field paths so clients can render field-level messages). Never leak stack traces, SQL, or internal service names.

**Status-code discipline:** status codes are coarse routing; the code field is precise. 400 malformed vs 422 semantically-invalid (pick one convention and hold it); 401 unauthenticated vs 403 unauthorized; 409 for version conflicts; 429 with `Retry-After`. **403 vs 404 on foreign resources is a deliberate choice:** 404 hides resource existence (prevents enumeration/leaks), 403 is more honest for debugging — decide per resource sensitivity, then be consistent. Never 200-with-error-body.

## Decision 2: Idempotency

- GET/PUT/DELETE are idempotent by HTTP definition — keep them so.
- **POST that creates or charges:** accept an `Idempotency-Key` header; store the first outcome; replays return the *original response* (same status, same body), not a duplicate and not an error. This is the Stripe model ([Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency); Postgres implementation walkthrough at [brandur.org/idempotency-keys](https://brandur.org/idempotency-keys)). Keys expire (Stripe: 24h) — document the window.
- The alternative — telling clients "don't retry" — fails because timeouts leave the client not knowing whether the operation happened; retry-with-key is the only safe answer (see the `distributed-reliability` skill, Decision 3).

## Decision 3: Pagination

**Options** (Slack's API walked this exact path: none → offset → cursor — [Evolving API Pagination at Slack](https://slack.engineering/evolving-api-pagination-at-slack/)):
- **None / unbounded list** — never. Every list endpoint is paginated with an enforced max page size from day one; retrofitting pagination is a breaking change.
- **Offset (`?page=3`)** — simple, allows jump-to-page, but the DB reads and discards `offset` rows (O(n) deep pages) and concurrent inserts shift the window, skipping or duplicating items. Acceptable only for small, mostly-static, admin-style data.
- **Cursor (default)** — return `{ data, next_cursor, has_more }`; the cursor is opaque (encoded, versioned server-side) and points at a record, not a position, so it's stable under writes and constant-cost at any depth. Back it with a keyset query on an indexed sort key with an ID tiebreak (see the `data-modeling` skill).

Slack's conclusion after supporting all three: cursor pagination that "does one thing well — forward pagination through an entire dataset in a backwards-compatible way." Opaque cursors also preserve your freedom to change the underlying query — an offset number is Hyrum-law surface area.

## Decision 4: Versioning and breaking changes

**Additive is free, and defined precisely:** new endpoints, new *optional* request params, new response properties, new enum values *provided clients are told to tolerate unknown values* — this is Stripe's exact backwards-compatibility contract ([Stripe API versioning docs](https://docs.stripe.com/api/versioning)).

**Breaking** = removing/renaming a field or endpoint, changing a type or the *semantics* of a value, tightening validation, changing an error `code`, changing defaults (sort, page size), adding a new required param, adding auth requirements. Any yes → it does not ship as a change; it ships as a new version or a new field alongside the old (expand/contract — see `data-modeling` for the same pattern on schemas).

**Version mechanism options:**
- **URL versioning (`/v2/`)** — coarse, forces big-bang migrations of entire integrations; simple to route. Fine for APIs that will realistically never version.
- **Date-based rolling versions (Stripe)** — each account pins to a date; each change ships with request/response transforms between adjacent versions, so compatibility is maintained indefinitely and migrations are per-change, unit-testable objects ([APIs as infrastructure — Stripe](https://stripe.com/blog/api-versioning)). The gold standard, but real machinery to build.
- **Additive-only + sunset headers (default for internal/small-surface APIs)** — never break; when you must, add the new shape alongside, emit `Deprecation`/`Sunset` headers, migrate consumers, remove after a bake window.

Pick based on consumer count you can't coordinate with: can coordinate → additive-only; can't → date-based or URL versions with a published sunset policy.

## Decision 5: Field conventions (each is a mini-decision — defaults with reasoning)

- **Timestamps:** ISO 8601 UTC with `Z`, field names suffixed `_at`. Rejected: epoch integers (ambiguous seconds vs millis — a classic integration bug).
- **IDs:** opaque strings in the contract even if bigints internally — preserves freedom to change the generator, and prevents clients doing arithmetic on them. Sequential IDs leak volume and enable enumeration (IDOR); expose non-enumerable IDs where that matters (see `data-modeling` Decision 1).
- **Money:** integer minor units + ISO 4217 currency code (Stripe's convention). Rejected: floats (rounding), decimal strings (locale parsing hazards).
- **Enums:** strings, not ints; document that clients must tolerate unknown values — that tolerance is precisely what makes adding values non-breaking (Stripe's compat contract).
- **Booleans that will grow:** a `status` enum beats `is_x` booleans; a `deleted_at`/`shipped_at` timestamp beats `is_deleted`/`is_shipped` — it carries *when* for free and can't contradict siblings.
- **Nullable vs absent:** define once, especially for PATCH — JSON Merge Patch semantics (absent = unchanged, `null` = clear) is the least-surprise default.
- **One casing everywhere** (pick snake_case or camelCase; Zalando's guidelines mandate snake_case JSON — the value is the mandate, not the choice: [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/)).

## Decision 6: Webhooks

- **Sign every payload:** HMAC-SHA256 over `timestamp.raw_body` with a per-endpoint secret, timestamp in the signed material to kill replays (verify within a ~5-minute tolerance) — the Stripe scheme ([Stripe webhooks](https://docs.stripe.com/webhooks)). Verify against the exact raw bytes, never a re-serialized body.
- **Delivery is at-least-once and unordered** — say so. Include `event_id`, `type`, `created`; consumers dedupe by `event_id` and tolerate out-of-order ([Svix: best practices for receiving webhooks](https://www.svix.com/resources/webhook-best-practices/receiving/)).
- **Thin vs fat payloads:** thin (ID + type; consumer fetches current state) self-heals staleness and ordering at the cost of a read-back call; fat payloads save the call but deliver stale snapshots out of order. Default thin for state-changing events; fat only for immutable events (logs, analytics).
- Retry failing endpoints with backoff on a published schedule; after exhaustion, disable + notify the owner.
- Consumers: verify signature → enqueue → ack fast; process async. Slow webhook handlers get you dropped by the sender's timeout.

## Design shape rules

- Model **resources and state transitions**, not RPC verbs: `POST /orders/{id}/cancel` (transition) is fine; `POST /doStuff` is not.
- No chatty contracts: if rendering one screen takes N+1 calls, add expansion params or a compound endpoint.
- Response shape must not vary by caller identity (breaks caching and client code) — vary via explicit query params/expansions.
- The DTO layer *is* the contract: never serialize DB rows raw — that couples schema migrations to client breakage.
- Rate limits: communicate in headers; reject with 429 + `Retry-After`.

## Breaking-change checklist (run on every API diff)

Field removed/renamed? Type or nullability changed? Semantics of an existing value changed? Validation tightened? Error `code` changed? Default sort/page-size/filter changed? Enum value removed? New required param? Auth added? — any **yes** = breaking = version it or add-alongside instead.

## Anti-patterns

- 200-with-error, `success: true` flags, envelope-less errors, parseable error messages
- Unbounded lists; offset pagination on growing data
- Timestamps in three formats; floats for money; ints for enums
- POST-only RPC tunneling with verbs in the body
- Raw DB rows as responses
- Webhooks without signatures, event IDs, or a documented retry policy
- "We'll version it later" — you can't un-ship a field (Hyrum's law)

## Sources

- [Hyrum's Law](https://www.hyrumslaw.com/)
- [RFC 9457 — Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc9457)
- [Designing robust and predictable APIs with idempotency — Stripe](https://stripe.com/blog/idempotency) · [brandur.org/idempotency-keys](https://brandur.org/idempotency-keys)
- [APIs as infrastructure: future-proofing Stripe with versioning](https://stripe.com/blog/api-versioning) · [Stripe versioning reference](https://docs.stripe.com/api/versioning)
- [Evolving API Pagination at Slack — slack.engineering](https://slack.engineering/evolving-api-pagination-at-slack/)
- [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/)
- [Stripe webhooks documentation](https://docs.stripe.com/webhooks) · [Svix: receiving webhooks best practices](https://www.svix.com/resources/webhook-best-practices/receiving/)
