---
name: migration-playbooks
description: Use when planning or reviewing any migration — replacing a legacy system, swapping a datastore/provider/library, moving traffic to a new implementation, large refactors of live code paths, or any change where old and new must coexist. Complements data-modeling's expand/contract for schema-level changes.
---

# Migration Playbooks

The law of live-system migration: **there is no big bang.** Every safe migration is a sequence of individually reversible steps through states where old and new coexist — the coexistence period *is* the design problem, not an unfortunate transition. If the plan contains a step described as "then we switch over," that step is where it fails.

## Decision 1: Which pattern fits the change

| Change | Pattern | Mechanics |
|---|---|---|
| Schema/API shape change | **Expand/contract** | Add new alongside old → migrate consumers → remove old. Fully specified in `data-modeling` (schemas) and `api-contracts` (breaking changes); root pattern: Fowler, ParallelChange |
| Replacing a system/service incrementally | **Strangler fig** | A façade/router in front of the legacy system sends each capability to old or new; functionality moves piece by piece; legacy keeps running throughout (Fowler, StranglerFigApplication) |
| Large in-codebase change without a long-lived branch | **Branch by abstraction** | Introduce an abstraction over the old implementation → build the new behind it → flip callers incrementally on trunk → delete old + (usually) the abstraction (Fowler, BranchByAbstraction) |
| Verifying a rewrite of a critical read path | **Shadow execution (Scientist pattern)** | Run old (control) and new (candidate) on real production traffic; *return the control result*; record mismatches. GitHub built Scientist for exactly this — multi-year permission-code rewrites verified on live traffic before any user saw the new path |

**Strangler-fig warnings** from the field: the interception point must exist or be buildable (a façade you can actually route through); old domain boundaries rarely map 1:1 to the new design — budget interface redesign; and the pattern's known failure mode is **never finishing** — "no pressure to finish" leaves you maintaining both systems forever. Every strangler plan needs a funded final step and a decommission date someone owns.

**Shadow-execution caveats:** only for *side-effect-free* comparisons (reads, pure computations) — never dual-fire writes, emails, or charges; expect benign mismatches (timestamps, ordering) and normalize before comparing; run until mismatch rate is ~zero on real traffic diversity, not three test cases.

## Decision 2: Data migration sequencing

Moving data between stores/schemas/providers while live — the sequence that never loses writes:

1. **New store ready** — schema in place, no traffic.
2. **Dual-write** — writes go to old (authoritative) and new. The write to new is best-effort-but-monitored: log/queue failures for repair; never fail the user request because the *new* store hiccuped. (True atomic both-writes needs an outbox — `distributed-reliability` Decision 4.)
3. **Backfill** history in batches, idempotently (re-runnable on failure), throttled (`data-modeling` migration rules).
4. **Verify** — counts, checksums, sampled row-by-row comparison, and ideally shadow reads (read both, compare, serve old). Verification is a step with an owner and exit criteria, not an assumption.
5. **Cut reads over** behind a flag, gradually (1% → 10% → 100%), watching error rates and data-mismatch metrics.
6. **Demote old** — stop writing old only after new has been authoritative through a full business cycle (month-end, billing run — the cycles are where migrations die).
7. **Decommission** old in its own deploy, after the bake period.

Skipping step 4 or rushing step 6 is how "successful" migrations surface corruption weeks later.

## Decision 3: Feature-flag discipline

Flags are the routing mechanism for most migration steps. Hodgson's taxonomy matters because each category has different lifetime and dynamism — managing them identically is how flag debt accrues:

| Category | Purpose | Lifetime |
|---|---|---|
| **Release** | Decouple deploy from release; migration cutovers | Days–weeks; **deleted after full rollout** |
| **Ops** | Kill switches, load shedding | Long-lived, few, runtime-togglable |
| **Experiment** | A/B measurement | Duration of the experiment |
| **Permission** | Entitlements/plans | Effectively permanent — product config, not a flag; manage as such |

Rules: every release flag has a removal ticket created at creation (a "temporary" flag without one is permanent); test both sides in CI while both are live; flag checks at **one decision point**, not sprinkled through the code (branch by abstraction gives you that point); document the safe rollback position per ops flag. Old, forgotten flags are how you get a Knight-Capital-shaped incident — dead code reactivated by a config change.

## Decision 4: Rollback design

- **Every step is reversible until the explicitly named point of no return** (typically the destructive contract step). The plan states the rollback action *per step* — "roll back" is not a plan; "set flag X to old, new store keeps receiving dual writes so no data loss" is.
- Rollback after users have written data through the *new* path is the hard case — that's why old stays write-current (dual-write) until well past cutover.
- Versions N and N−1 of every component must work against every intermediate state (the expand/contract invariant), because deploys and rollbacks will interleave with migration steps.
- Rehearse the rollback once while it doesn't matter. An untested rollback is a hypothesis.

## Review checklist (for any migration plan)

- Which pattern, and why that one? Any step that's secretly a big bang?
- Coexistence: both paths live for how long, and what keeps them consistent?
- Verification step with explicit exit criteria (mismatch rate, checksum parity)?
- Gradual cutover with metrics watched, and a per-step rollback action written down?
- Full business cycle baked before old is demoted? Decommission step funded, dated, owned?
- Flags: category identified, removal ticket for release flags, both sides tested?
- Point of no return named in the doc (`arch-design-doc` rollback section)?

## Anti-patterns

- Big-bang rewrite/cutover ("we'll switch over the weekend")
- Long-lived migration branches (use branch by abstraction on trunk)
- Strangler fig with no decommission plan — two systems forever
- Dual-write with no reconciliation/repair story for missed writes
- Verifying against test fixtures instead of shadowed production traffic
- Cutting reads over before backfill verification passes
- Destructive cleanup in the same deploy as the cutover
- Flag graveyard: release flags aging into de-facto (untested) architecture
- Declaring victory at 100% traffic — the migration ends at *decommission*, not cutover
