---
name: data-modeling
description: Use when designing or modifying database schemas, tables, columns, indexes, constraints, or writing/reviewing migrations — raw SQL, Prisma, or any ORM. Also fires for slow-query fixes and lock-related migration incidents.
---

# Data Modeling & Migrations

The schema outlives the code that created it. Every shortcut here is permanent in a way application code never is — and every migration runs against a live system where the lock queue, not the data volume, is what takes you down.

## Decision 1: Primary keys

**Options and trade-offs:**
- **bigint identity** — compact (8 bytes), append-ordered (B-tree friendly), human-readable. Costs: generated only at the DB (no client-side ID creation), enumerable — leaks volume and enables IDOR-style scraping if exposed.
- **UUIDv4** — generate anywhere, non-enumerable. Cost: fully random values insert into arbitrary B-tree positions, causing page splits, index bloat, write amplification, and cold caches. Benchmarks: a 50M-row bulk insert took ~20 min with v4 vs ~1.8 min with v7, with ~25% larger indexes.
- **UUIDv7 (default for new distributed-ish systems)** — 48-bit timestamp prefix makes inserts append-ordered like a bigint while keeping 128-bit global uniqueness and client-side generation; native `uuidv7()` in PostgreSQL 18. Residual cost: 16 bytes; timestamp prefix leaks creation time (rarely matters).
- **Natural keys (email, username, SKU) as PK — never.** Anything a human chose can change; changing a PK cascades through every FK. Natural keys get `UNIQUE` constraints on a normal column instead.

**Default:** bigint for purely-internal tables in a single-writer system; UUIDv7 when clients generate IDs, rows merge across systems, or IDs are exposed. If you expose bigint IDs publicly, pair them with a separate non-enumerable public identifier.

## Decision 2: Normalize or denormalize

Start at 3NF. Denormalize only for a **measured** read-path problem, and when you do, write down where the duplicate lives and what reconciles it — an unreconciled duplicate always diverges. The middle options beat copying data: a covering index, a materialized view, or a cache with explicit invalidation. Every table gets `created_at`/`updated_at` from day one — they cost nothing now and are unrecoverable later.

## Decision 3: Delete strategy

- **Hard delete (default)** — honest, keeps queries and unique constraints simple. Pair with an audit log if you need history.
- **Soft delete (`deleted_at timestamptz`, not a boolean)** — choose only for genuine undo/restore or referential-integrity needs. Costs you accept: every read path must filter `deleted_at IS NULL`, every unique index becomes a partial index (`UNIQUE ... WHERE deleted_at IS NULL` — otherwise deleted rows block re-registration), and tables grow forever.
- **Archive table** — when deleted rows are kept for compliance but never queried in the hot path; keeps the live table lean.

The timestamp-over-boolean rule generalizes: model state as a **state machine** — a `status` enum plus a timestamp per interesting transition (`shipped_at`, `cancelled_at`) — never a pile of booleans (`is_shipped`, `is_cancelled`) whose 2ⁿ combinations are mostly invalid.

## Decision 4: JSONB vs columns

JSONB for genuinely schemaless payloads: external API responses, user-defined fields, event blobs. The moment a JSON field is filtered, joined, or updated *independently*, extract it to a column — JSON is where constraints, FKs, and types go to die. Rule of thumb: if it appears in a `WHERE` clause more than incidentally, it's a column.

## Decision 5: Where invariants live

The app validates for UX; the database validates for truth — it's the last line of defense against every future code path, backfill script, and console session. Declare every invariant the DB can express: `NOT NULL`, FKs, `UNIQUE`, `CHECK`.

- **Uniqueness is only real as a constraint.** SELECT-then-INSERT is a race under any concurrency; use the unique index and handle the violation, or `INSERT ... ON CONFLICT`.
- **FKs are real constraints** unless a measured write-throughput reason says otherwise. Choose `ON DELETE` per relationship deliberately: `RESTRICT` default; `CASCADE` only when the child is meaningless without the parent — and mind the cascade blast radius.
- **Postgres does not auto-index FK columns.** Every FK you join or delete through gets an explicit index, or parent deletes take sequential scans on the child.

## Decision 6: Concurrent writes to the same rows

Postgres's default `READ COMMITTED` lets two transactions read-then-write past each other (lost updates, write skew). Options, cheapest first:
1. **Atomic single-statement writes** — `UPDATE ... SET n = n + 1` (never read-modify-write for counters).
2. **Optimistic version column** — `UPDATE ... WHERE id = ? AND version = ?`, check rowcount, retry/409 on miss. Default for user-facing edits (no held locks, plays well with idempotency).
3. **`SELECT ... FOR UPDATE`** — when a multi-row invariant must hold (balance can't go negative across reads). Keep the transaction short.
4. **SERIALIZABLE** — correct by construction, but requires app-wide retry-on-serialization-failure discipline; reserve for genuinely tangled invariants.

Transactions: short, and **never held across a network call, queue publish, or user think-time** — DB-plus-external-system in one logical operation is the dual-write problem; use the outbox (see `distributed-reliability` skill, Decision 4).

## Decision 7: Indexing

- Index what queries actually filter, join, and sort on — not speculatively. Every index taxes every write; audit and drop unused ones (`pg_stat_user_indexes`).
- **Composite order: equality columns first, then the range/sort column.** An index on `(a, b)` serves `a` and `a,b` lookups — not `b` alone.
- Partial indexes for hot subsets (`WHERE status = 'pending'`); `INCLUDE` (covering) for hot read paths.
- An unverified index is not an optimization: `EXPLAIN ANALYZE` before and after.
- **Pagination on growing tables: keyset, not OFFSET.** `WHERE (sort_key, id) > (?, ?) ORDER BY sort_key, id LIMIT n` is constant-cost at any depth and stable under concurrent inserts, while OFFSET reads-and-discards all preceding rows and drifts. This is the query layer beneath cursor pagination in the `api-contracts` skill.

## Decision 8: Multi-tenancy (a one-way door — decide before table #2)

- **Row-level (`tenant_id` everywhere) — default**: cheapest to operate; requires discipline — `tenant_id` on every tenant-owned table, composite indexes leading with `tenant_id`, and ideally RLS or a query-layer guard.
- **Schema-per-tenant**: stronger isolation, but migrations × tenants and connection-pool pressure.
- **DB-per-tenant**: for regulated/enterprise isolation; highest operational cost.

Retrofitting tenancy is among the most painful migrations that exist — this decision gets a design doc (see `arch-design-doc`).

## Migrations — expand/contract, always

The governing pattern is Fowler's **ParallelChange (expand/contract)**: break every breaking change into expand → migrate → contract so no consumer must switch at the same instant. For schemas, app versions N and N−1 run against every intermediate state:

1. **Expand** — add new nullable column/table/index. No drops, no renames.
2. **Dual-write** old + new from the app.
3. **Backfill in batches** with throttling — one giant `UPDATE` bloats the table, holds locks, and lags replicas.
4. **Switch reads** to the new shape (flag if risky).
5. **Stop writing** old.
6. **Contract** — drop old, one bake period later, in its own deploy.

**Lock mechanics (Postgres) — the part that causes outages:**
- `ALTER TABLE` takes `ACCESS EXCLUSIVE`. The danger is the *queue*, not the duration: your ALTER waits behind one long-running query, and every subsequent query queues behind your ALTER until the pool exhausts.
- Therefore: **set `lock_timeout` (e.g. 1–2s) on every DDL statement and retry on failure** — aborting a deploy beats taking the app down.
- Indexes on live tables: `CREATE INDEX CONCURRENTLY` — no write-blocking, but it can't run inside a transaction and a failed run leaves an `INVALID` index to drop and rebuild.
- Constraints on existing tables: add as `NOT VALID` (brief lock), then `VALIDATE CONSTRAINT` separately (weaker `SHARE UPDATE EXCLUSIVE` lock, scans without blocking writes). Same staging for `NOT NULL`: nullable → backfill → constraint.
- Never rename or change a column type in place on a hot table — both are expand/contract sequences.
- The destructive step never ships in the same deploy as the code that stops needing it; every migration is reversible or its PR explicitly marks the point of no return.

## Growth

Any table that only grows (events, logs, audit, notifications) gets a retention/archival decision at creation — "keep forever, it's small" is acceptable, but written down. Hot counts are maintained counters or estimates, never `COUNT(*)` at read time on big tables. Timestamps are `timestamptz`, UTC stored, converted at the edge.

## Review checklist (schema/migration diffs)

- [ ] Invariants as DB constraints? Uniqueness race-proof (constraint, not check-then-insert)?
- [ ] FK columns indexed? `ON DELETE` chosen deliberately?
- [ ] State machine (enum + timestamps), not boolean pile? Soft delete really needed — and partial unique indexes if so?
- [ ] PK choice matches generation/exposure needs (Decision 1)?
- [ ] `tenant_id` + leading composite index on every tenant-owned table?
- [ ] Migration expand/contract-safe with app version N−1? `lock_timeout` set? Backfill batched? `CONCURRENTLY`/`NOT VALID` where applicable?
- [ ] Destructive step in its own later deploy?
- [ ] Append-only table → retention plan written down?

## Anti-patterns

- Natural primary keys; UUIDv4 PKs on large hot tables
- Boolean status piles; `is_deleted` booleans; soft delete "just in case"
- EAV tables; polymorphic FKs with no constraint (prefer one nullable FK per target + `CHECK` exactly-one)
- SELECT-then-INSERT uniqueness; read-modify-write counters
- JSON columns queried in every `WHERE`
- One-shot full-table `UPDATE` backfills; renames in place; DDL without `lock_timeout`
- `COUNT(*)` on unbounded tables in the request path; OFFSET pagination on growing data
