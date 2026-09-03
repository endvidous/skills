---
name: service-boundaries
description: Use when deciding whether to split a service, extract a microservice, restructure a monolith, define module/domain boundaries, or review a proposal that adds a new service, queue-connected worker, or repo. Covers monolith-first reasoning, bounded contexts, Conway's law, the legitimate split triggers, modular-monolith discipline, and inter-service coupling rules. Also fires when someone says "we should move to microservices."
---

# Service Boundaries

The evidence is one-directional: "almost all the successful microservice stories have started with a monolith that got too big and was broken up; almost all the cases where a system was built as a microservice system from scratch have ended up in serious trouble" ([Fowler: MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)). Services are a tool for scaling *organizations*, paid for with a distributed-systems tax on every call. Default to not splitting.

## Decision 1: Monolith vs services

- **Modular monolith (default)** — one deployable, strictly separated internal modules. You keep transactions, in-process calls, one deploy, one debugger — and you keep the *option* to split later along boundaries that reality has validated.
- **Microservices** — pay the "microservice premium" (distributed failure modes, eventual consistency, deploy orchestration, observability overhead) only when system + team complexity genuinely demands it ([Fowler: MicroservicePremium](https://martinfowler.com/bliki/MicroservicePremium.html)).

The cautionary cases are well documented: Segment collapsed 140+ destination services back into a monolith after operational overhead scaled linearly with services and three full-time engineers were consumed just keeping the system alive ([Goodbye Microservices — Twilio Segment](https://www.twilio.com/en-us/blog/developers/best-practices/goodbye-microservices)); Amazon Prime Video's monitoring team merged its serverless microservices into a monolith and cut infrastructure cost by over 90% ([The New Stack coverage](https://thenewstack.io/return-of-the-monolith-amazon-dumps-microservices-for-video-monitoring/)). Neither means "monoliths always win" — both mean the split must be justified by measurements, not fashion.

Why splitting early fails specifically: "even experienced architects working in familiar domains have great difficulty getting boundaries right at the beginning" (Fowler). Refactoring a boundary between modules is an afternoon; between services it's a distributed migration.

## Decision 2: Where boundaries go

- **Bounded contexts, not technical layers.** A boundary encloses a business capability with its own consistent language and model — orders, billing, identity — never "the API layer" / "the data layer" ([Fowler: BoundedContext](https://martinfowler.com/bliki/BoundedContext.html)). Horizontal (layer) services guarantee every feature touches every service.
- **Test:** if a routine feature regularly requires coordinated changes on both sides of a boundary, the boundary is wrong — cohesion belongs inside, coupling across is the failure signal.
- **Conway's law is load-bearing:** system structure mirrors communication structure ([Fowler: ConwaysLaw](https://martinfowler.com/bliki/ConwaysLaw.html)). A service without a clear owning team becomes an orphan; two teams inside one service fight over it. Align boundaries with team ownership deliberately (the "inverse Conway maneuver") — or accept that the org chart will redraw your architecture for you.
- **Data draws the real boundary:** each context owns its data exclusively. If two candidate services need to transact over the same rows, they're one service.

## Decision 3: When a split is actually justified

Legitimate triggers — you should be able to name which one, with a number attached:
1. **Independent scaling, measured** — one component's load profile differs by ~10× and co-scaling wastes real money (not "might need to scale someday").
2. **Team contention** — multiple teams queueing on one deploy pipeline, stepping on each other's releases, needing different release cadences. This is the most common *valid* reason: services are an organizational scaling tool.
3. **Isolation requirements** — a hard blast-radius, compliance, or security boundary (payment processing, untrusted-code execution).
4. **Genuinely divergent technical needs** — a component that needs a different runtime/language/hardware (ML inference, video processing).

Not triggers: "microservices are best practice," conference talks, resume-driven design, imagined future scale (see `arch-design-doc` — numbers or it didn't happen). And a split proposal must budget for the checklist it inherits: per-service observability, deploy pipeline, on-call story, and the `distributed-reliability` skill applied to every new network edge.

## Decision 4: Modular-monolith discipline (what keeps the option open)

The monolith-first strategy only works if the monolith stays modular:
- **Modules communicate through explicit public interfaces** (exported module APIs); reaching into another module's internals or tables is a review-blocking violation. Enforce mechanically where the platform allows (visibility rules, lint boundaries, separate packages).
- **No cross-module joins for module-owned data**; module schemas kept separable. Cross-module reads go through the owning module's interface. This is precisely what makes a later extraction a lift instead of a rewrite.
- **Domain events for cross-module side effects** ("order placed" → notifications module reacts) rather than direct orchestration of other modules' internals — the in-process rehearsal for a future queue.
- One module = one clear owner (Conway again).

## Decision 5: Coupling rules once services exist

- **Synchronous call chains are the distributed monolith's tell.** Each sync hop multiplies latency and compounds failure probability; a request fanning through 4 services at 99.5% each is ~98% — and every edge needs the full `distributed-reliability` treatment (timeouts, retries, breakers). Prefer: async events for non-blocking effects, data replication/caching for read paths, and keeping chatty collaborators in one service.
- **Never share a database between services.** A shared schema is an unversioned, ungoverned API — the worst coupling of both worlds. Integration is APIs and events only.
- **Deploy-together = one service.** If releases must be coordinated ("deploy A and B in lockstep"), the boundary is fake; merge them or fix the contract (see `api-contracts` on additive change).
- **Version contracts between services** like public APIs — internal consumers obey Hyrum's law too.

## Review checklist (for any "new service" proposal)

- Which of the four triggers, with what measurement?
- Is it a bounded context with a single owning team, or a technical layer?
- Does it own its data exclusively? Any transaction spanning it and another service?
- What's the sync-call fan-out per user request after the split?
- Deploy, observability, on-call story budgeted?
- Could a module boundary inside the monolith deliver the same benefit? (Usually yes — say why not.)

## Anti-patterns

- Greenfield microservices before any boundary has survived contact with users
- Layer services (API service → business-logic service → data service)
- Nanoservices: services so small the overhead exceeds the logic (Segment's 140-repo failure mode)
- Shared database integration; cross-service joins
- Entity services ("UserService" that every feature must call synchronously)
- Distributed monolith: N repos, one lockstep release train
- Splitting to "keep the code clean" — discipline problems follow you across the network, with worse tooling

## Sources

- [MonolithFirst — Martin Fowler](https://martinfowler.com/bliki/MonolithFirst.html) · [MicroservicePremium](https://martinfowler.com/bliki/MicroservicePremium.html)
- [BoundedContext — Martin Fowler](https://martinfowler.com/bliki/BoundedContext.html) · [ConwaysLaw](https://martinfowler.com/bliki/ConwaysLaw.html)
- [Goodbye Microservices — Twilio Segment](https://www.twilio.com/en-us/blog/developers/best-practices/goodbye-microservices) · [InfoQ analysis](https://www.infoq.com/news/2020/04/microservices-back-again/)
- [Prime Video monitoring: microservices → monolith, −90% cost — The New Stack](https://thenewstack.io/return-of-the-monolith-amazon-dumps-microservices-for-video-monitoring/)
- [Domain analysis for service boundaries — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/domain-analysis)
