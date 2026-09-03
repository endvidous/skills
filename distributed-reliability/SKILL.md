---
name: distributed-reliability
description: Use when designing or reviewing anything that crosses a network or process boundary — service-to-service calls, queues and message consumers, background jobs, webhooks, cron jobs, external API calls, caches, distributed locks, or multi-step workflows. Also fires when debugging duplicates, lost updates, stuck jobs, retry storms, or race conditions.
---

# Distributed Reliability

The network will fail, messages will duplicate, and processes will die mid-step. Every rule here exists because someone got paged for its absence.

## The three questions — ask at every boundary crossing

1. What happens if this call never returns? (timeout)
2. What happens if it executes twice? (idempotency)
3. What happens if the process dies right here? (atomicity/recovery — walk every pair of consecutive writes)

A mutation across a boundary must survive all three. Note that a client timeout does **not** mean the operation failed — the server may have completed it. Timeout means the state is *unknown*, and only idempotency makes unknown safe to retry.

## Decision 1: Timeout values

**Options:** (a) library default — often infinite or minutes, effectively no timeout; (b) round number pulled from the air; (c) derived from downstream latency data.

**Default: (c).** AWS practice: pick an acceptable false-timeout rate (e.g. 0.1%) and set the timeout at the downstream service's corresponding latency percentile (p99.9). No latency data yet? Use (b) but log timeout hits and revisit — an explicit wrong number beats an implicit infinite one.

**Also:** budgets flow downhill — a caller's timeout must exceed (callee timeout × attempts) + slack, or callers abandon work the callee is still burning CPU on.

## Decision 2: Retry strategy

**Options:** (a) no retries — surface the error; (b) immediate retry; (c) exponential backoff; (d) exponential backoff + jitter, capped attempts, retry budget.

**Reasoning:** (b) hammers a struggling dependency at the worst moment. (c) synchronizes clients into waves — after an outage, every client retries on the same schedule. AWS's simulation work shows **full jitter** (random delay between 0 and the exponential cap) beats plain exponential backoff on both total work and time-to-success.

**Default: (d)** with:
- **Cap attempts** (3 is usually right) and cap total elapsed time.
- **Retry budget**: Google SRE uses a per-client budget — retry only while retries are <10% of request volume — plus a per-request cap of 3 attempts. AWS SDKs use a token bucket for the same purpose (built in since 2016).
- **Classify errors**: retry timeouts, 429, 5xx-transient; never retry 400/401/403/422 — retrying a validation error is a self-inflicted DoS.
- **Retry at ONE layer.** Stacked retries multiply: 3 retries at each of 3 layers = up to 4³ = 64 attempts hitting the database for one user action. Put retries at the layer that owns the idempotency key; layers below propagate failure.

**Choose (a)** when the operation is not idempotent and can't be made so — "APIs with side effects aren't safe to retry unless they provide idempotency" (AWS).

## Decision 3: Surviving duplicates (idempotency)

Exactly-once **delivery** is impossible in a distributed system (Two Generals); what exists is at-least-once delivery + idempotent processing = exactly-once *effect*. Broker "exactly-once" features narrow the window; they do not remove the need for idempotent consumers once side effects leave the broker.

**Options, in order of preference:**
1. **Natural idempotency** — upsert keyed on a caller-supplied ID, PUT semantics, state-machine transitions that no-op when already done, conditional writes (`WHERE version = ?`). Zero extra infrastructure; prefer this.
2. **Explicit idempotency key + result store** — client sends a unique key; server stores the outcome and replays the stored response on retry (Stripe's model). Needed when the operation isn't naturally idempotent (charges, sends, side effects).
3. **Accept duplicates** — valid only when duplicates are harmless (e.g., cache warms). Say so explicitly in the design.

**Hard rule:** every consumer of every queue is idempotent — dedupe by message/event ID or by natural key. No exceptions; redelivery is a *when*, not an *if*.

## Decision 4: Writing to two systems (the dual-write problem)

Writing DB + queue, DB + search index, or DB + another service in one logical operation: one write will eventually succeed while the other fails.

**Options:**
- **Just do both writes ("failures are rare")** — rare × volume = daily. Never.
- **2PC / XA distributed transaction** — most modern brokers (e.g. Kafka) don't support XA; couples availability of both systems; avoid unless inside a system built for it.
- **Transactional outbox (default)** — write the event to an outbox table in the *same DB transaction* as the state change; a relay publishes from the outbox. The relay delivers at-least-once, so consumers dedupe (Decision 3). Costs: an extra table, a relay process, eventual consistency — accepted deliberately.
- **CDC (Debezium etc.)** — outbox with the database's replication log as the relay. Same guarantees, less app code, more infra. Choose when you already run CDC.

**Trigger for this decision:** any transaction followed by a publish/index/notify call, or any `await db.write(); await queue.send();` pair in review.

## Decision 5: Multi-step workflows (sagas)

A workflow spanning services can't be atomic. Each step needs idempotent execution plus a **compensating action** for rollback.

**Choreography vs orchestration:**
- **Choreography** (services react to events, no coordinator): loose coupling, no single point of failure — but the workflow exists nowhere; with more participants "it becomes harder to track the dependencies." Fine to ~3 steps.
- **Orchestration** (explicit coordinator: state machine, workflow engine, or a status column + poller): the workflow is visible, debuggable, and owns compensation. **Default beyond 3 steps or whenever anyone asks "where is this workflow stuck?"** — implicit workflows scattered across consumers are undebuggable at 3am.

Persist workflow state *before* executing each step, not after — a crashed orchestrator must know where it was.

## Decision 6: Distributed locks

First ask: is the lock for **efficiency** (avoid duplicate work; a rare double execution is harmless) or **correctness** (a double execution corrupts data)? The distinction is Kleppmann's.

- **Efficiency → single-node Redis lock is fine**: `SET key <random-value> NX PX <ttl>`, release via compare-and-delete script (plain `DEL` can release someone else's lock). TTL is mandatory — holders die.
- **Correctness → a TTL lock alone is NOT sufficient.** A paused (GC, network) client can act after its lock expired while another holds it. You need a **fencing token** — a monotonically increasing number issued with the lock, checked by the protected resource, rejecting stale tokens — or make the protected operation itself conditional (compare-and-swap, unique constraint, `WHERE version = ?`). Kleppmann's analysis shows Redlock cannot generate fencing tokens and adds cost without safety; for correctness-critical locks use a consensus system (ZooKeeper/etcd) or, better, restructure so a conditional DB write is the arbiter.
- Never hold any lock across a call you don't control (network, user think-time).
- Multi-node cron: every scheduled job takes a distributed lock or is idempotent enough to run twice — concurrent runs on two nodes is the default failure, not the edge case.

## Decision 7: Overload behavior

**Options when demand exceeds capacity:** queue it, shed it, or degrade.

- **Unbounded queueing is not an option** — it converts overload into OOM plus minutes-stale work. Every queue and buffer is bounded.
- **Shed early and cheaply (default)**: reject at the edge with 429/503 + Retry-After when load exceeds capacity — "drop a fraction of traffic (including retries!) upstream" (Google SRE).
- **Circuit breakers** on outbound dependencies: fail fast when a dependency is down; without one, a hung dependency consumes your whole connection/thread pool (add bulkheads — per-dependency pool limits).
- **Health checks report own-process health only.** A health check that pings dependencies converts one dependency outage into a fleet-wide restart cascade.

## Time and ordering

- Never use wall-clock time for ordering, locks, or uniqueness across machines — clocks skew and jump. Order with sequence numbers, version counters, or single-writer designs; use monotonic clocks for in-process durations.
- Consumers tolerate out-of-order and ancient messages: check the message's version/timestamp against current state; stale update → no-op.

## Review checklist (per network boundary in the design/diff)

- [ ] Explicit timeout, derived from downstream latency where data exists?
- [ ] Retries: backoff + full jitter, capped, budgeted, retryable-errors-only, at exactly one layer?
- [ ] Duplicate-safe (natural idempotency / key store / explicitly accepted)?
- [ ] Process dies between write A and write B — recoverable? (walk each consecutive pair)
- [ ] Dual-write anywhere? → outbox or CDC
- [ ] Queue bounded? Consumer idempotent? Poison messages → DLQ + alert?
- [ ] Lock: efficiency or correctness? If correctness — fencing token or conditional write?
- [ ] Overload: shed or degrade — decided, not defaulted to unbounded queueing?

## Anti-patterns

- `sleep()` as synchronization
- Retry without idempotency; retry on 4xx; retries stacked at every layer
- Dual-write justified by "failures are rare"
- "The broker gives us exactly-once" as a consumer-design excuse
- Cron on multiple nodes without a lock or idempotent design
- Unbounded in-memory queue/buffer; health check that pings dependencies
- Catch-and-continue around a partial multi-step write
- TTL lock guarding a correctness-critical section with no fencing/conditional write
