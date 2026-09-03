---
name: testing-strategy
description: Use when designing a test suite, deciding what kind of test to write (unit vs integration vs E2E vs contract), reviewing test code, setting coverage or CI policy, dealing with flaky tests, or choosing a mocking approach. Complements api-contracts (what a contract is) and migration-playbooks (testing both flag sides).
---

# Testing Strategy

A test suite is judged by two things: **the bugs it catches before users do, and the refactors it survives.** Coverage, count, and speed are proxies. The single best heuristic for writing any test: "the more your tests resemble the way your software is used, the more confidence they can give you" (Kent C. Dodds) — a test coupled to *implementation* rather than *behavior* has negative value: it fails when you refactor and passes when you break the user.

## Decision 1: The shape of the suite

Three named models, one underlying trade-off:

- **Test pyramid** (Cohn, Fowler): many unit, fewer integration, few E2E — optimizes for speed and pinpointing.
- **Testing trophy** (Dodds): static analysis at the base, *integration as the bulk*, some unit, few E2E — optimizes for confidence per test, arguing most real bugs live in the wiring between units.
- **Google's small/medium/large** (Test Sizes, SWE at Google ch.11): classify by *resource boundary* — small = one process (no I/O, no sleep), medium = one machine (localhost DB fine), large = anything (real network) — because what you actually care about is **speed and determinism**, and those degrade at process/machine/network boundaries, not at "unit vs integration" labels.

These agree more than they disagree: the axis is **determinism+speed vs fidelity**. Default shape: **the bulk of tests at the module/API boundary with real-ish infrastructure** (handler → service → containerized or in-memory DB — medium tests), unit tests for pure logic with real branching (parsers, pricing, date math), typechecker+linter doing the trivia, and a *handful* of E2E smoke journeys. When to deviate: a library/algorithm codebase inverts toward unit; a thin CRUD app may be nearly all integration.

## Decision 2: Which level for which behavior

| Behavior | Level | Why |
|---|---|---|
| Pure logic, many branches (money, dates, parsing) | Unit | Cheap to enumerate cases; no wiring to get wrong |
| An endpoint's behavior: auth, validation, persistence, response shape | Integration (the default home) | The bug is usually in the wiring; testing through the API tests what users get and survives refactors |
| Two services agreeing on a payload | **Contract test**, not E2E | Consumer-driven contracts (Pact) verify each side against the *recorded expectations of real consumers* — provider changes that no consumer uses stay free (Fowler, CDC; Pact docs); spinning up both services to check JSON shapes is the slow, flaky way to do this |
| Critical user journeys (signup, login, checkout, export) | E2E, few | The only level that catches "everything works but nothing is wired together"; Google's rule of thumb: ~70/20/10 unit/integration/E2E, and "just say no to *more* E2E" — each E2E must earn its place |
| Types, nullability, unused vars, exhaustiveness | Static analysis | Free tests; strict compiler flags are suite design too |

Test **behavior at a boundary you're committed to** (public function, API endpoint, UI interaction). If a test would break under a pure refactor (rename a private, split a function), it's testing implementation — delete or rewrite it against the boundary.

## Decision 3: Test doubles

- **Mock at the boundaries you don't own** — payment providers, email, third-party APIs, clocks. Never mock *your own* repository/DB in the integration layer: a mocked-DB "integration" test verifies your assumptions, not your queries — the real DB (testcontainers, or in-memory Postgres-compatible where honest) is the point.
- **Don't mock what you don't own** directly — wrap the third-party client in your own thin interface and mock *that*; otherwise your mocks encode your (possibly wrong) beliefs about their API, and their SDK update breaks you invisibly.
- **Prefer fakes to mocks** for anything used widely: a small in-memory implementation (fake S3, fake mailer) beats per-test mock setups — one place to maintain, and verifiable against the real thing with a shared contract-test suite run against both.
- Assert on **outcomes, not interactions** (state, responses, records written) — `verify(x).calledWith(...)` couples the test to call structure; reserve interaction assertions for cases where the call *is* the outcome (the email was sent).
- Time, randomness, and IDs are injected dependencies from day one — `Date.now()` sprinkled through code is a future flaky test (Decision 4) and an untestable branch.

## Decision 4: Flakiness discipline

At Google, ~1.5% of all test runs report flaky results, almost 16% of tests exhibit some flakiness, and flakes are the #1 reported cause of losing trust in the suite (Flaky Tests at Google); flakiness rises with test size — larger tests, using more real machines/network, flake more (Where do our flaky tests come from?).

- **Policy first:** a flaky test is **quarantined immediately** (removed from the blocking suite, ticketed) and then fixed or deleted. The corrosive option is the default one — retry-until-green normalizes ignoring red, and then the real failure ships.
- Root causes and their fixes: *sleeps* → wait on conditions with timeout, never durations; *shared state* → fresh fixtures/schema (or transaction-rollback isolation) per test, tests runnable in any order and in parallel; *real time* → injected clock; *real concurrency* → deterministic seams or move the assertion; *external services in tests* → fakes (Decision 3).
- Auto-retry is diagnostic tooling (detect and flag flakes), never a green-making mechanism.

## Decision 5: Test data

- **Builders/factories with explicit overrides** — `makeUser({ plan: 'pro' })` — where each test states only what it cares about; defaults are valid and boring.
- No shared mutable seed corpus that every test implicitly depends on: it couples all tests to all tests, and "fix one fixture, break forty tests" is how suites die.
- The test should read as its own spec: setup shows what matters, assertion shows what's expected, and a reader needs nothing off-screen.

## Decision 6: Coverage and CI economics

- **Coverage is a gap-finder, not a target.** Use the report to find untested *branches that matter*; a mandated percentage gets gamed with assertion-free tests (Goodhart's law applied to `--coverage`). If you gate at all, gate on *new-code* coverage (ratchet), never retroactively.
- **The blocking suite must be fast enough to run always** — set a wall-clock budget (e.g. <5 min PR gate), parallelize, and push slow large tests to post-merge/nightly with an owner watching them. A suite developers skip is a suite that doesn't exist.
- Tests are the safety mechanism for everything else — migrations (`migration-playbooks`: both flag sides tested while both are live), refactors, upgrades. An untested path is a path you can't change.

## Review checklist

- Does each new test assert behavior at a stable boundary (would it survive a refactor)?
- New endpoint: integration test through the API with real DB — auth, validation, success, and failure paths?
- Own code mocked anywhere? Third-party mocked directly instead of via a wrapper?
- Any `sleep`? Any test-order dependence? Clock/randomness injected?
- Fixtures: builders with overrides, no shared magic corpus?
- E2E added — does it cover a journey no cheaper level can, and who maintains it?
- Suite still within its time budget?

## Anti-patterns

- Tests that break on rename/refactor but not on behavior change
- Mocking your own database, then calling it an integration test
- `sleep(3000)` as synchronization; retry-until-green as policy
- 100% coverage mandates; coverage-farming tests with no assertions
- One giant seed fixture every test leans on
- E2E as the default test type; testing every validation branch through the browser
- Testing third-party libraries' behavior instead of your use of them
- Interaction-verifying (`calledWith`) when a state assertion exists
- Skipped/quarantined tests with no ticket — a graveyard, not a quarantine
