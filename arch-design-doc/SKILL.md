---
name: arch-design-doc
description: Use when writing, structuring, or reviewing a design document, RFC, ADR, architecture proposal, or technical plan — or when the user asks "how should we build X" for anything non-trivial. Forces problem-first structure, explicit constraints, real alternatives with trade-off reasoning, failure modes, and rollback plans. Also use when critiquing an existing design or deciding whether a design doc is needed at all.
---

# Architecture Design Docs

A design doc exists to surface a decision and its trade-offs, not to describe a system. If a reviewer can't find what is being decided, what it costs, and how to undo it, the doc has failed regardless of how complete the diagrams are. Google's design-doc culture frames the doc's core value the same way: it documents "the high level implementation strategy and key design decisions with emphasis on the trade-offs that were considered" ([Design Docs at Google](https://www.industrialempathy.com/posts/design-docs-at-google/)).

## Decision 1: Do you need a doc at all? (size the artifact first)

Classify the decision using Bezos's Type 1 / Type 2 framework ([2015 Amazon shareholder letter](https://www.aboutamazon.com/news/company-news/2016-letter-to-shareholders)): **Type 2 (two-way door)** decisions are reversible — make them quickly with ~70% of the information; **Type 1 (one-way door)** decisions are irreversible or nearly so — these deserve slow, deliberate process. Using Type 1 process on Type 2 decisions produces "slowness, unthoughtful risk aversion, failure to experiment."

| Situation | Artifact | Why |
|---|---|---|
| Reversible in a day, blast radius one service | **No doc.** ADR only if the decision will be questioned later | Process cost exceeds decision cost |
| Reversible with effort, or touches 2+ teams/services | **ADR or 1-pager**: problem, constraints, decision, alternatives, rollback | Enough structure to force the trade-off reasoning |
| One-way door: schema, public API contract, data migration, vendor/build-vs-buy, protocol, multi-tenancy model | **Full design doc, reviewed before code** | Cost of being wrong is a migration, not a revert |

The common one-way doors in application work are: data models, public API shapes, ID schemes, multi-tenancy isolation, and event contracts. These look small in the diff and are the most expensive to reverse — default them to "full doc" even when the code change is trivial.

## Decision 2: Doc structure

**Options considered:**
- **Free-form prose** — fast, but lets the author skip the sections that would have exposed the problem (alternatives, failure modes). Google explicitly keeps docs informal but still converges on a standard skeleton because those sections are where review value concentrates.
- **Heavy template (30+ sections)** — forces completeness but produces fill-in-the-blanks compliance writing nobody reads; violates the "docs must stay small to stay updated" observation from [Nygard's ADR essay](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions).
- **Minimal forced skeleton (default)** — the sections below, sized to the decision. This matches the common structure at Google: context and scope, goals and non-goals, the design, alternatives considered, cross-cutting concerns ([Design Docs at Google](https://www.industrialempathy.com/posts/design-docs-at-google/)).

**Mandatory sections, in order — no solution content before section 4:**

1. **Problem / context** — observable pain in the user's or system's terms. If the problem statement names a technology, it's a solution in disguise; rewrite it.
2. **Constraints** — numbered and explicit: latency budget, data volume, deadline, team size, compliance, systems it must not break. A design cannot be evaluated without these; most bad reviews are two people assuming different constraints.
3. **Goals and non-goals** — non-goals are where scope creep dies. Every "we could also…" in review gets pointed here. A non-goal is not "we won't ever" — it's "not this doc."
4. **Numbers** — expected QPS, data size, growth rate, read/write ratio, latency budget. Crude estimates beat omissions: write "~100 rps, guess" rather than nothing. If the numbers are small, say so plainly — it usually deletes half the design.
5. **Proposed design** — the smallest design that meets the constraints. Include every boundary: external calls, queues, stores, and what crosses them.
6. **Alternatives considered** — minimum two real ones. Each gets: what it is, why rejected, and what would make it the right choice later. "Rejected: too complex" is not a reason; "rejected: requires eventing infra we don't run; revisit if event volume exceeds X" is. A strawman alternative is worse than none — it signals the decision was made before the analysis.
7. **Failure modes** — for every dependency: what happens when it is down, slow, or returning garbage? What is the blast radius? What does the user see? (Pair with the `distributed-reliability` skill's checklist.)
8. **Rollback and migration** — how to turn this off after it has been live and has written data. "Feature flag" is not a rollback plan once state exists. If rollback becomes impossible past a point, name that point explicitly.
9. **Open questions** — real unknowns with owners. A doc with zero open questions was not thought about hard enough.

## Decision 3: ADR vs design doc vs neither

Use an **ADR** when the decision is one choice among alternatives (library, pattern, convention) rather than a system design. Nygard's format ([Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions), templates at [adr.github.io](https://adr.github.io/)):

```markdown
# ADR-NNN: <short noun phrase>
Status: proposed | accepted | superseded by ADR-MMM
Context: <the forces — technical, political, project-local. Neutral tone.>
Decision: <active voice: "We will …">
Consequences: <ALL of them — positive, negative, neutral.>
```

Hard rules for ADRs: one decision per record; append-only (supersede, never edit history); one or two pages max — "large documents are never kept up to date; small modular documents have at least a chance" (Nygard). A Consequences section with no downsides is propaganda, not analysis — every real decision costs something.

## Decision 4: Review process

- **Heavyweight review meeting** — reserve for Type 1 decisions crossing team boundaries.
- **Async comment thread on the doc (default)** — Google's dominant mode: share the doc, discussion happens in comments ([Design Docs at Google](https://www.industrialempathy.com/posts/design-docs-at-google/)).
- **No review** — acceptable only for Type 2 decisions inside one team's blast radius.

Review effort scales with reversibility cost, not diff size.

## Review checklist (when critiquing any design)

- Can the decision be stated in one sentence? If not, the doc is a description, not a design.
- Does every component earn its place from a numbered constraint? Delete any component no constraint demands.
- Are the alternatives real or strawmen? Would a proponent of the alternative recognize the description?
- Is there a number for scale, and does the design match the number rather than 100× the number?
- What breaks first at 10× load? Is that acceptable, and is it written down?
- For each dependency: down / slow / wrong — all three answered?
- Can this be rolled back after a week of production writes?
- Does anything serve a hypothetical future requirement? Move it to non-goals.
- Is any one-way door being walked through casually (ID format, tenancy model, event schema, public field names — see Hyrum's Law in the `api-contracts` skill)?

## Anti-patterns

- **Solution-first doc** — problem section written last to justify a chosen design.
- **Strawman alternatives** — alternatives listed only to be knocked down.
- **Designing for imagined scale** — no numbers, or a design sized for numbers nobody wrote down.
- **"We can add that later"** without verifying later is possible — multi-tenancy, audit trails, and data-model changes usually can't be bolted on.
- **Consensus by vagueness** — a doc so abstract everyone agrees because nothing was decided.
- **Diagram as failure analysis** — a boxes-and-arrows picture answers none of the down/slow/wrong questions.
- **Zero-consequence decisions** — a Consequences/trade-offs section listing only upsides.

## Sources

- [Design Docs at Google — Malte Ubl, industrialempathy.com](https://www.industrialempathy.com/posts/design-docs-at-google/)
- [Documenting Architecture Decisions — Michael Nygard, Cognitect](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
- [Architecture Decision Records — martinfowler.com bliki](https://martinfowler.com/bliki/ArchitectureDecisionRecord.html) · [adr.github.io](https://adr.github.io/)
- [Jeff Bezos 2015/2016 shareholder letters — Type 1 / Type 2 decisions](https://www.aboutamazon.com/news/company-news/2016-letter-to-shareholders)
- [Software Engineering at Google — Documentation chapter](https://abseil.io/resources/swe-book/html/ch10.html)
- [RFC and design doc examples across companies — Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/software-engineering-rfc-and-design)
