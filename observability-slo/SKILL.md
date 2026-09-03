---
name: observability-slo
description: Use when adding monitoring, metrics, logging, tracing, dashboards, or alerts; when defining SLIs/SLOs or error budgets; when reviewing whether a new feature or service is production-ready; or when on-call pain (noisy alerts, undebuggable incidents) comes up. Covers golden signals / RED / USE, SLO and error-budget design, burn-rate alerting, structured logging vs metrics vs traces, and the instrument-before-ship checklist.
---

# Observability & SLOs

A feature without instrumentation isn't done — it's deployed, which is different. You will debug it at 3am with whatever you emitted at 3pm. The organizing question is never "what can we collect?" but "when this breaks, what will we wish we could see?"

## Decision 1: What to measure

- **Request-driven services → the four golden signals** ([Google SRE book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)): **latency** (successful and failed requests measured *separately* — fast 500s poison a blended average), **traffic** (demand), **errors** (rate of failed requests), **saturation** (how full; p99 latency over a small window is the early-warning proxy). The equivalent per-endpoint formulation is **RED** — Rate, Errors, Duration ([Tom Wilkie](https://grafana.com/blog/the-red-method-how-to-instrument-your-services/)).
- **Resources (hosts, pools, queues, DBs) → USE** — Utilization, Saturation, Errors ([Brendan Gregg](https://pagertree.com/learn/devops/what-is-observability/use-and-red-method)). RED for what the service does, USE for what it runs on; connection pools and queues are resources — instrument their saturation (see `distributed-reliability` on bounded queues).
- Every metric with a latency/duration dimension is recorded as a **histogram/percentiles — never a mean**. Mean latency is a number nobody experiences; p50/p95/p99 are; tail latency is where incidents live.
- Instrument at the *edge the user experiences* (load balancer / API gateway), not just inside the service — the difference is queueing and network you'd otherwise never see.

## Decision 2: SLOs and error budgets

- **SLI:** a measured indicator that tracks user experience — "fraction of requests served successfully in < 400ms at the LB" beats any CPU graph ([SRE book: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)).
- **SLO:** the target over a window (e.g. 99.9% over 28 days). **Not 100%** — 100% is wrong as a target because the marginal cost explodes and users can't tell past a point; the gap is the point.
- **Error budget = 1 − SLO.** 99.9% over 1M requests = 1,000 allowed failures. The budget converts reliability into a spendable resource: budget available → ship features and take risk; budget exhausted → reliability work takes priority over launches ([SRE Workbook: Error Budget Policy](https://sre.google/workbook/error-budget-policy/)). Write the policy *before* the first breach, or the first breach becomes a negotiation.
- Start with 2–3 SLOs per service (availability + latency, maybe freshness for pipelines). Ten SLOs = zero SLOs.

## Decision 3: Alerting

- **Symptoms, not causes.** Page on "users are experiencing errors/slowness" (SLI degradation), not "CPU is high" or "disk 80%" — causes without user impact are tickets, not pages ([SRE book](https://sre.google/sre-book/monitoring-distributed-systems/)).
- **Default mechanism: error-budget burn rate, multi-window.** Alert when the budget is being consumed abnormally fast — fast-burn (e.g. 1h window) pages, slow-burn (6h/3d) tickets. This is the SRE Workbook's recommended endpoint of the alerting maturity ladder ([Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)); it inherently trades off precision vs recall better than static thresholds.
- **Every page is actionable.** If the response is "acknowledge and go back to sleep," delete or demote the alert — alert fatigue is how the real incident gets ignored.
- Health checks and alerts report **own-service** symptoms; a dependency's outage should page the dependency's owner, and your service degrades gracefully instead of cascading (see `distributed-reliability` Decision 7).

## Decision 4: Logs vs metrics vs traces

| Signal | Answers | Cardinality/cost profile |
|---|---|---|
| **Metrics** | "Is it broken? How much? Since when?" — aggregates, alerting, dashboards | Cheap to store, **bounded label cardinality** (a user-ID label on a metric = cardinality explosion — that's what logs/traces are for) |
| **Structured logs / wide events** | "What exactly happened on *this* request?" | One wide, structured (JSON) event per request per service with high-cardinality fields: request ID, user, tenant, route, durations, error class, key business fields. Sample noisy successes; never sample errors |
| **Traces** | "Where did the time go across services?" | Cross-service latency attribution; propagated context. Adopt via OpenTelemetry so instrumentation is vendor-neutral |

Non-negotiables: a **correlation/request ID** minted at the edge and propagated through every service, queue message, and background job — an incident without one is archaeology; logs are structured from day one (grep-and-hope printf logs don't answer questions); never log secrets/tokens/PII (`security-boundaries`).

## Decision 5: Instrument-before-ship (definition of done)

A new service/feature ships with:
- [ ] RED metrics per endpoint (histograms), USE on its pools/queues
- [ ] Structured request logging with correlation IDs propagated end-to-end
- [ ] Error tracking wired (exceptions grouped, releases tagged)
- [ ] 1–3 SLOs with burn-rate alerts and a named owner
- [ ] Dashboard: golden signals + saturation of its critical resources
- [ ] For async/pipelines: queue depth, processing lag, DLQ size, per-message outcome logging
- [ ] Deploy markers visible against the metrics (the first incident question is "what changed?")

Business-level counters (orders placed, exports generated) sit beside system metrics — "requests are fine but orders dropped to zero" is a real and otherwise invisible failure class.

## Anti-patterns

- Mean latency on dashboards; latency without success/failure separation
- Paging on causes (CPU, disk, single errors) instead of symptoms; unactionable pages kept "for awareness"
- 100% as an implicit SLO (every blip a crisis) or no SLO (reliability by vibes)
- Unstructured string logs; no request IDs across service/queue hops
- User-ID labels on metrics (cardinality bomb) / no user IDs in logs (undebuggable)
- Health checks that ping dependencies (cascade restarts — `distributed-reliability`)
- Instrumenting only after the first production incident
- Ten dashboards nobody opens instead of one per service that answers the golden signals

## Sources

- [Monitoring Distributed Systems (golden signals) — Google SRE book](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Service Level Objectives — Google SRE book](https://sre.google/sre-book/service-level-objectives/) · [Implementing SLOs — SRE Workbook](https://sre.google/workbook/implementing-slos/)
- [Alerting on SLOs (multi-window burn rate) — SRE Workbook](https://sre.google/workbook/alerting-on-slos/) · [Error Budget Policy](https://sre.google/workbook/error-budget-policy/)
- [The RED Method — Tom Wilkie, Grafana Labs](https://grafana.com/blog/the-red-method-how-to-instrument-your-services/)
- [USE and RED methods overview (Brendan Gregg's USE)](https://pagertree.com/learn/devops/what-is-observability/use-and-red-method)
- [OpenTelemetry](https://opentelemetry.io/)
