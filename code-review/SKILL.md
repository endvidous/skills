---
name: code-review
description: Use when reviewing a diff, branch, or PR against a base commit — including requests to "review since X", "review this branch", "review for over-engineering", "what can we delete here", or "is this over-engineered".
metadata:
  type: engineering
---

# Code review

Three independent axes over one diff, each answering a different question:

- **Standards** — does the code follow this repo's documented conventions
  (plus a baseline set of universal code-smell heuristics)?
- **Spec** — does the code faithfully implement what the originating
  issue/spec asked for?
- **Complexity** — is there anything here that shouldn't exist: dead
  flexibility, reinvented stdlib, speculative abstraction?

Each axis is reported under its own heading. Never merge or rerank findings
across axes — a change can cleanly pass one axis and fail another, and
collapsing them into a single verdict hides that.

## 1. Pin the fixed point

User-supplied — a commit SHA, branch, tag, `main`, `HEAD~5`. If not
specified, ask.

Diff command: `git diff <fixed-point>...HEAD` (three-dot, so the comparison
is against the merge-base). Commit list: `git log <fixed-point>..HEAD
--oneline`.

Confirm the ref resolves (`git rev-parse <fixed-point>`) and the diff isn't
empty before continuing — a bad ref or empty diff should fail here, not
mid-review.

## 2. Find the spec

In order: issue references in the commit messages (`#123`, `Closes #45`,
etc. — fetch via whatever issue-tracker workflow the repo documents); a
path the user passed as an argument; a spec file under `docs/`, `specs/`,
or a scratch directory matching the branch or feature name. If nothing
turns up, ask where it is. If the user says there isn't one, the Spec axis
reports "no spec available" instead of running.

## 3. Find the standards

Anything the repo documents about how code should be written —
`CODING_STANDARDS.md`, `CONTRIBUTING.md`, a linter config worth noting.

On top of whatever the repo documents, the Standards axis always carries a
baseline of universal code-smell heuristics (adapted from Fowler's
*Refactoring*, ch. 3) — useful even in a repo with no written standards at
all. Two rules govern the baseline:

- A documented repo standard always wins — where the repo endorses
  something the baseline would otherwise flag, suppress the flag.
- Every baseline item is a judgment call, never a hard violation, and skip
  anything already enforced by tooling (linter/formatter/type-checker).

Baseline smells, each as *what it is → the fix*:

- **Mysterious name** — a function, variable, or type whose name doesn't
  reveal its purpose. → Rename it; if no honest name comes, the design
  itself is murky.
- **Duplicated code** — the same logic shape repeated across hunks or
  files in the change. → Extract the shared shape, call it from both
  sites.
- **Feature envy** — a method reaching into another object's data more
  than its own. → Move the method onto the data it envies.
- **Data clumps** — the same few fields or params traveling together
  everywhere (a type wanting to be born). → Bundle them into one type,
  pass that.
- **Primitive obsession** — a raw primitive or string standing in for a
  domain concept that deserves its own type. → Give the concept its own
  small type.
- **Repeated conditionals** — the same switch/if-cascade on the same type
  recurring across the change. → Replace with polymorphism, or one shared
  map both sites use.
- **Shotgun surgery** — one logical change forcing scattered edits across
  many files in the diff. → Gather what changes together into one module.
- **Divergent change** — one file or module edited for several unrelated
  reasons. → Split it so each module changes for one reason.
- **Speculative generality** — an abstraction, parameter, or hook added
  for a need the spec doesn't actually have. → Delete it; inline back
  until a real second need shows up.
- **Message chains** — a long `a.b().c().d()` navigation the caller
  shouldn't depend on. → Hide the walk behind one method on the first
  object.
- **Middle man** — a class or function that mostly just delegates onward.
  → Cut it, call the real target directly.
- **Refused bequest** — a subclass or implementer that ignores or
  overrides most of what it inherits. → Drop the inheritance, use
  composition instead.

## 4. Confirm before spawning sub-agents

Running the three axes as parallel sub-agents keeps their findings from
contaminating each other — a real advantage for this kind of review. But
this skill never spawns a sub-agent on its own judgment, however useful
it would be here — that decision belongs to the user, every time.

Before starting the review, ask the user: run Standards, Spec, and
Complexity as three parallel sub-agents (isolated context per axis,
matches this skill's design), or work through all three sequentially in
this thread instead? Wait for the answer.

If sequential: run step 5 for each axis one at a time, in the same order,
in this thread. Keep each axis's reasoning separate — don't let an early
finding on one axis color a later one; the point of the split survives
even without sub-agent isolation.

## 5. Run the three axes

Each axis works from the diff command, commit list, and its own source
material below. Nothing in one axis's report should reference another
axis's conclusions.

**Standards** — report, per file/hunk where relevant: (a) every place the
diff violates a documented standard, citing the file and the rule; (b) any
baseline smell spotted, named and quoted from the hunk. Mark
documented-standard breaches as hard violations; baseline smells stay
judgment calls always, and a documented repo standard overrides the
baseline. Skip anything tooling already enforces. Keep it under ~400
words.

**Spec** — report: (a) requirements the spec asked for that are missing or
partial; (b) behavior in the diff that wasn't asked for (scope creep); (c)
requirements that look implemented but look wrong. Quote the relevant spec
line for each finding. Under ~400 words. Skip entirely if no spec was
found — say so instead.

**Complexity** — one line per finding: `L<line>: <tag> <what>.
<replacement>.` (or `<file>:L<line>: ...` when the diff spans multiple
files). Tags:

- `delete:` dead code, unused flexibility, a speculative feature.
  Replacement: nothing.
- `stdlib:` hand-rolled logic the standard library already ships. Name the
  function.
- `native:` a dependency or custom code doing what the platform already
  does. Name the feature.
- `yagni:` an abstraction with one implementation, a config nobody sets, a
  layer with exactly one caller.
- `shrink:` same behavior, fewer lines. Show the shorter form.

End the Complexity axis with `net: -<N> lines possible`, or `Lean already.
Ship.` if there's nothing worth cutting. This axis never flags security,
correctness, or performance — route those to Standards or Spec instead —
and never flags a single smoke test or `assert`-based self-check as bloat;
that's the minimum, not the excess. It only lists what to cut, never
applies the fixes itself.

Example (Complexity axis, vague vs. terse):

- ❌ "This EmailValidator class might be more complex than necessary, have
  you considered whether all these validation rules are needed at this
  stage?"
- ✅ `L12-38: stdlib: 27-line validator class. "@" in email, 1 line, real
  validation is the confirmation mail.`
- ✅ `L4: native: moment.js imported for one format call.
  Intl.DateTimeFormat, 0 deps.`
- ✅ `repo.py:L88: yagni: AbstractRepository with one implementation.
  Inline it until a second one exists.`

## 6. Aggregate

Present all three reports under their own headings — `## Standards`,
`## Spec`, `## Complexity` — verbatim or lightly cleaned, never merged or
reranked against each other. Close with one line per axis: finding count,
and that axis's worst issue if it has one. Don't pick a single overall
winner across axes — a change that passes Spec while failing Complexity is
a real, separate signal, and collapsing it into one verdict is exactly
what the three-axis split exists to prevent.

## Why three separate axes

A change can pass any one axis while failing another:

- Follows every standard, implements the wrong thing → Standards pass,
  Spec fail.
- Does exactly what the issue asked, breaks repo conventions → Spec pass,
  Standards fail.
- Correct and on-spec, but drags in a needless abstraction or dependency
  → Standards and Spec pass, Complexity fails.

Reporting them separately stops any one axis from masking the other two.
