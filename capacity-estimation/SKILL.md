---
name: capacity-estimation
description: Use when sizing a system or component — estimating QPS, storage growth, bandwidth, connection/thread pools, cache sizes, queue throughput, or infrastructure cost — when a design doc needs its "numbers" section, or when someone asks "will this scale?" / "how big should the pool be?".
---

# Capacity Estimation

A design without numbers can't be evaluated (see `arch-design-doc` — the Numbers section is mandatory). The goal is never precision; it's **order of magnitude**: is this 10 QPS or 10,000? Does the data fit in RAM, on one node, or neither? A crude estimate stated with its assumptions beats both no estimate and precision theater.

## Method rules

1. **Round aggressively to powers of ten.** 86,400 seconds/day ≈ 100K. Estimate, don't calculate.
2. **Write assumptions next to results** — "~100 rps, assuming 10% of 1M DAU do this once" is checkable and correctable; "~100 rps" alone is neither.
3. **Peak ≠ average.** Provision for peak: typical web traffic peaks at 2–5× the daily average; spiky domains (lunch rush, campaign sends, cron-aligned batch jobs) 10×+. State which multiplier you used.
4. **Design for ~10× current estimates, not 100×.** 10× headroom is prudent; 100× is speculative architecture the constraints don't justify (see `arch-design-doc` anti-patterns).
5. **Find the first bottleneck.** The output of the estimate is not "it fits" but *which resource saturates first* — that's what the design must address, and the only place optimization effort belongs yet.

## The reference numbers

The canonical table ("latency numbers every programmer should know," Jeff Dean / Peter Norvig); the original figures are dated, so treat as ratios and check napkin-math (Simon Eskildsen) for maintained numbers:

| Operation | Order of magnitude |
|---|---|
| L1 cache reference | ~1 ns |
| Main memory reference | ~100 ns |
| Compress 1KB | ~µs |
| SSD random read | ~100 µs |
| Read 1MB sequentially from memory | ~0.25 ms (≈ GB/s: tens) |
| Round trip within a datacenter | ~0.5 ms |
| Read 1MB sequentially from SSD | ~1 ms |
| HDD seek | ~10 ms |
| Round trip same continent (WAN) | ~10–50 ms |
| Round trip cross-continent | ~150 ms |

The ratios that decide designs: memory is ~1000× faster than SSD random access; a WAN round trip costs ~100× a datacenter round trip (chatty cross-region protocols are dead on arrival); sequential beats random by orders of magnitude on every medium (why append-only logs and batch reads win — see `data-modeling` on B-tree behavior).

## Little's law — the one equation to know

**L = λ × W**: concurrency = arrival rate × time in system (Little's law). It holds regardless of arrival or service distributions, which is why it's safe for napkin use.

- **Sizing pools:** 500 RPS at 200ms average latency → 500 × 0.2 = **100 requests in flight**. Thread pools, DB connection pools, and in-flight budgets need headroom above that (say 2×) or any latency blip queues into an outage.
- **Throughput ceilings:** max sustainable RPS = concurrency limit ÷ latency. 200 concurrency slots at p99 0.4s ≈ 500 RPS ceiling — more throughput requires more slots or *less latency*, not more CPU by itself.
- **Layer by layer:** gateway, app servers, DB pool each obey it separately; the system ceiling is the **minimum across layers**, and the layer at its minimum is your bottleneck (backpressure per `distributed-reliability`).
- Corollary: latency reduction *is* capacity. Halving W doubles what the same concurrency serves.

## Estimation templates

- **QPS from users:** `DAU × actions/user/day ÷ ~100K s` = average; × peak multiplier. (1M DAU × 10 actions ÷ 100K ≈ 100 QPS avg → ~500 peak.)
- **Read/write split:** estimate separately; most systems are 10–100× read-heavy, which decides the caching and replication story.
- **Storage growth:** `rows/day × bytes/row × retention days` (indexes and replication typically ~2–3× the raw figure). 1M rows/day × 1KB ≈ 1 GB/day ≈ 365 GB/yr — fine; 100× that needs a partitioning/retention plan at design time (`data-modeling` Growth).
- **Bandwidth:** QPS × payload size, both directions; egress is the one that costs money.
- **Cache sizing:** working set = hot fraction × object size × object count — the question is "does the *hot* set fit in memory," not the whole corpus. Estimate hit rate consequences: 99% vs 90% hit rate is a 10× difference in DB load.
- **Queue/worker sizing:** workers needed = arrival rate × processing time (Little again); then check the drain story — at what backlog does lag breach the SLO (`observability-slo`)?
- **Cost sanity check:** nodes × unit price + storage + egress, to one significant figure — catches "this design costs 100× the revenue" before the review does.

## Worked micro-example

Export feature: 50K exports/day, ~2 MB each, kept 90 days. QPS: 50K ÷ 100K ≈ 0.5 avg, ~5 peak — trivial, no queue needed for rate; but generation takes 30s, so concurrency = 5 × 30 = **150 concurrent generations** at peak — that's the real problem (worker pool + queue, not web threads). Storage: 50K × 2MB ≈ 100 GB/day ≈ 9 TB retained → object storage with lifecycle expiry, not the DB. Two design decisions fell out of four multiplications.

## Review checklist

- Numbers present, with assumptions, rounded to magnitude?
- Peak multiplier stated? Read/write split?
- Concurrency computed (L = λW) for every pool the request path touches, with headroom?
- First bottleneck identified — and is the design effort pointed at *it*?
- Growth: storage/table growth per `data-modeling` retention rules?
- Any cross-region chatty path (100× RTT penalty) hiding in the design?
- Cost within an order of magnitude of acceptable?

## Anti-patterns

- No numbers ("should be fine") or false precision (four significant figures on a guess)
- Provisioning for average, discovering peak in production
- Pool sizes copied from framework defaults instead of computed from λ × W
- Designing for 100× imagined scale; ignoring 10× real growth
- Optimizing a component that isn't the bottleneck
- Treating latency and throughput as independent knobs (Little's law says otherwise)
