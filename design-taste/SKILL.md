---
name: design-taste
description: Use for any frontend visual/UI design work — building new pages or components, redesigning existing ones, or critiquing a design for quality.
metadata:
  type: design
---

# Design taste

Taste is trained, not innate. The unseen details compound — the ones a user
can't name but can feel. Beauty is leverage: it's the cheapest lever with the
biggest payoff on how a product is perceived. This skill exists to make that
judgment repeatable instead of vibes.

## The brief wins

Every rule in the reference files below is a default, not a law. A default
exists because most briefs don't say otherwise — the moment a brief names a
constraint, a reference, a brand commitment, or a genre (neobrutalist,
editorial, glass), that instruction overrides the default. Read the brief
first. Never silently override an explicit brand, content, or scope decision
under the banner of "good design."

The one default that is not negotiable: no brief earns back a kicker/eyebrow
label above a heading, and the em dash is never used anywhere in shipped
copy. See [anti-slop-tells.md](reference/anti-slop-tells.md).

## Anti-default discipline

Before writing anything, name in one line: what kind of page this is, the
vibe words the brief implies, any reference signals given, who the audience
is, what brand assets already exist, and any quiet constraints (must match
an existing app shell, must ship without a build step, etc). If a page could
be dropped into a stock-template gallery from any of the last five years,
that's the tell you defaulted instead of deciding. Rewrite the read until it
names something no generic instance of this page-type would say.

## Three dials

Every design decision can be pulled toward or away from three axes. State
them explicitly (numbers 1–10) before building, inferred from the brief's
vibe words, reference signals, and page type:

- **Design variance** — how far from the category default the visual
  language sits. Low = safe, conventional, boring-on-purpose (banking,
  government, internal tools). High = distinctive, opinionated, willing to
  surprise (portfolios, launches, editorial).
- **Motion intensity** — how much the interface moves. Low = static or
  purely functional transitions. High = choreographed, scroll-driven,
  physical.
- **Visual density** — how much is on screen at once. Low = spacious,
  editorial, single-focus. High = data-dense, utility, dashboard-adjacent.

Baseline for an unscoped "make it look good" brief: variance 8, motion 6,
density 4 — lean toward distinctive and alive, moderate information load.
Pull dials down hard for Operate/Read surfaces (see below); pull them up for
Persuade/Experience surfaces with room to perform.

## Four visitor modes

Borrowed from impeccable's mode framework — useful for calibrating every
other reference file, not just this section:

- **Persuade** — a visitor must be convinced and act (landing pages,
  marketing, pricing). Voice and composition can lead.
- **Operate** — a user is doing a task repeatedly (dashboards, tools,
  internal apps). Stability and predictability are the affordance; low
  variance, low motion, whatever density the task actually needs.
- **Read** — a visitor is here for the content (docs, articles, editorial).
  Comprehension and wayfinding outrank decoration.
- **Experience** — the interface itself is the product (portfolios, art,
  playful tools). Full permission to commit to a bold world.

Name the mode before pulling the dials — it's usually the fastest way to
answer "how far can this go."

## Reference files

| File                                                               | Use for                                                                                                                                 |
| ------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| [typography-color-layout.md](reference/typography-color-layout.md) | Choosing type, color, and structural layout; materiality (shadows/cards/depth)                                                          |
| [motion.md](reference/motion.md)                                   | Animation decisions — what to animate, easing, duration, choreography, interruptibility                                                 |
| [anti-slop-tells.md](reference/anti-slop-tells.md)                 | The banned-pattern checklist — what makes a design read as generic/AI-generated                                                         |
| [critique-and-audit.md](reference/critique-and-audit.md)           | Scoring a design's quality (heuristics, cognitive load, personas), auditing existing UI, and refinement passes (bolder/quieter/distill) |
| [production-hardening.md](reference/production-hardening.md)       | Edge cases, i18n, error/empty/loading states, performance, accessibility resilience                                                     |
| [copy-clarity.md](reference/copy-clarity.md)                       | Interface copy — labels, errors, empty states, content density                                                                          |
| [design-systems.md](reference/design-systems.md)                   | Which official design system or aesthetic direction fits a given brief, with install references                                         |

## Verify before shipping

Before calling any design work done, run the checklist that matches the
work: [anti-slop-tells.md](reference/anti-slop-tells.md) for pattern bans,
the "Verify" section of [typography-color-layout.md](reference/typography-color-layout.md)
and [motion.md](reference/motion.md) for the mechanics, and
[production-hardening.md](reference/production-hardening.md) for edge cases.
A clean pass on all of them is a floor, not proof of good design — it never
substitutes for actually looking at the rendered result.
