# Critique, audit, and refinement

Three related but distinct passes over an existing design: **critique**
(is this good design?), **audit** (is this technically sound?), and
**refinement** (bolder/quieter/distill — deliberately push a settled design
in one direction). Use critique and audit to find problems; use refinement
once the direction is right and only the intensity needs adjusting.

## Critique: is this good design?

A design critique scores holistic quality — not code correctness. Ground
every finding in something you can point to: a screen, a state, a specific
element. A finding you can't locate isn't a finding yet.

### Nielsen's 10 usability heuristics (score each 0–4)

1. **Visibility of system status** — does the interface keep the user
   informed with reasonable, timely feedback?
2. **Match between system and the real world** — does it speak the user's
   language, with familiar concepts, rather than system-internal jargon?
3. **User control and freedom** — can users undo, cancel, or leave a
   mistaken path easily?
4. **Consistency and standards** — do the same words, situations, and
   actions mean the same thing everywhere in the product?
5. **Error prevention** — does the design prevent problems before they
   happen, not just report them after?
6. **Recognition rather than recall** — are objects, actions, and options
   visible, not something the user has to remember from a previous screen?
7. **Flexibility and efficiency of use** — are there accelerators for
   experienced users that don't get in a novice's way?
8. **Aesthetic and minimalist design** — does every unit of information
   compete for attention only when it's actually relevant?
9. **Help users recognize, diagnose, and recover from errors** — are
   errors expressed in plain language, precisely, with a constructive fix?
10. **Help and documentation** — if help is needed, is it easy to search,
    focused on the user's task, and concrete?

Score band (sum out of 40, or renormalize to a percentage if some
heuristics are marked not-applicable for this surface type):
36–40 Excellent · 28–35 Good · 20–27 Acceptable · 12–19 Poor · 0–11 Critical.

### Cognitive load assessment

Three load types: **intrinsic** (inherent task complexity — can't be
designed away, only clarified), **extraneous** (load added by bad design —
this is what you're actually fixing), **germane** (effort that builds real
understanding — worth preserving even if it "feels harder").

Checklist (score 0–1 low, 2–3 moderate, 4+ high per item; total load rises
fast past 4 simultaneous high-scoring items — Miller's Law / Cowan 2001 puts
working memory at roughly 4 chunks):
- Number of choices visible at once
- Number of steps to complete the primary task
- Amount of text the user must read before acting
- Number of different UI patterns used for similar actions
- Reliance on memory across steps (vs. recognition in place)
- Visual noise competing with the primary action
- Ambiguity in what an element does before interacting with it
- Depth of navigation required to reach a common task

Common violations and their fix: too many primary actions on one screen →
pick one primary, demote the rest; jargon in an error message → plain
language naming the actual problem; a multi-step flow with no progress
indicator → show where the user is; inconsistent terminology for the same
concept across screens → one glossary, enforced.

### Persona-based testing

Pick 2–3 personas relevant to the surface, from:
- **Alex, impatient power user** — wants speed, keyboard shortcuts, no
  hand-holding. Red flag: forced tutorials, no way to skip onboarding.
- **Jordan, confused first-timer** — needs orientation, clear next steps.
  Red flag: jargon, no empty-state guidance, unclear primary action.
- **Sam, accessibility-dependent user** — screen reader or keyboard-only.
  Red flag: missing labels, broken tab order, color-only state.
- **Riley, deliberate stress tester** — tries to break things, edge-case
  inputs, rapid clicking. Red flag: no loading-state guard, no validation.
- **Casey, distracted mobile user** — one thumb, small screen, low
  attention. Red flag: tiny touch targets, dense text blocks, no thumb-
  reachable primary action.

For each persona: what do they do first, what confuses them, where do they
get stuck, what would make them contact support?

### Issue severity (P0–P3)

- **P0 Blocking** — prevents task completion. Fix immediately.
- **P1 Major** — significant difficulty or a WCAG AA violation. Fix before
  release. The test: would a real user contact support over this?
- **P2 Minor** — an annoyance with a workaround. Fix in the next pass.
- **P3 Polish** — nice-to-fix, no real user impact.

### Report shape

Design Health Score table (heuristics 0–4 each) → Overall Impression →
What's Working → Priority Issues (P0–P3, each with what/why/fix) → Persona
Red Flags → Minor Observations → Questions worth asking the user when ≥3
Priority Issues surfaced.

## Audit: is this technically sound?

A code-level check, not a design judgment — score what's measurable in the
implementation across 5 dimensions, 0–4 each (total /20):

1. **Accessibility** — contrast ratios below 4.5:1, missing
   `prefers-reduced-motion` handling, missing ARIA roles/labels/states,
   broken keyboard nav or focus indicators, div-as-button instead of
   semantic elements, missing/poor alt text, unlabeled form inputs.
2. **Performance** — layout thrashing (alternating reads/writes), casual
   layout-property animation, images without lazy loading, `will-change`
   left on at rest or applied broadly, unused dependencies bloating the
   bundle, unnecessary re-renders.
3. **Theming** — hard-coded colors bypassing tokens, missing or broken
   dark-mode variants, inconsistent token usage, values that don't update
   on theme switch.
4. **Responsive design** — fixed widths that break on mobile, touch
   targets under 44×44px, horizontal overflow on narrow viewports, layouts
   that break under text-size increase, missing breakpoints.
5. **Implementation integrity** — repeated shortcuts, design-system drift,
   decorative content masquerading as functional, structure generic enough
   to belong to any product.

Score bands: 18–20 Excellent · 14–17 Good · 10–13 Acceptable · 6–9 Poor ·
0–5 Critical. Tag every finding P0–P3 as above, with location, category,
impact, the WCAG standard if applicable, and a concrete fix.

## Existing-project design audit (redesign context)

When auditing an existing codebase rather than a fresh design, work through
these dimensions and pair each problem with its fix:

- **Typography** — default system font stack with no consideration →
  choose a deliberate face; inconsistent sizing → establish a scale.
- **Color and surfaces** — pure black/white with no depth → introduce
  tonal surfaces; hard-coded hex scattered through the codebase → extract
  tokens.
- **Layout** — cramped or inconsistent spacing → apply a documented scale;
  everything center-aligned by default → make alignment a decision.
- **Interactivity and states** — missing hover/focus/active/disabled states
  → add the full state set per interactive element.
- **Content** — generic placeholder copy left in production → replace with
  real, specific copy.
- **Component patterns** — one-off implementations of things a shared
  component should handle → consolidate.
- **Iconography** — mixed icon libraries/weights → standardize on one.
- **Code quality** — dead CSS, unused components, orphaned files →
  remove them as part of the same pass.
- **Strategic omissions** — missing empty/loading/error states entirely,
  not just under-styled ones → add them; this is often more valuable than
  polishing what's visible.

### Fix priority order
1. Font swap (fastest, highest-visibility win)
2. Color cleanup (tokens, contrast, dark mode)
3. Hover/active states on all interactive elements
4. Layout and spacing consistency
5. Replace generic components with considered ones
6. Loading/empty/error states
7. Typography polish (final pass, once structure is settled)

### Redesign rules
Work with the existing stack — don't introduce a new framework to redesign
a page. Don't break functionality while restyling. Check what's already a
dependency before importing something new. Ship small, reviewable changes
rather than one giant diff.

## Refinement passes

Use these once a design's direction is settled and only its intensity
needs adjusting — not for finding new problems.

### Distill (simplify to essence)
Ask: what's the ONE primary goal of this surface? What's actually necessary
vs. nice-to-have? What can be hidden until needed (progressive disclosure)
or combined? Then cut: reduce the color palette to 1–2 plus neutrals, cap
typography at one family/3–4 sizes/2–3 weights, remove decorative borders/
shadows/backgrounds that don't serve hierarchy, flatten nested containers,
cut copy in half and then in half again. Never remove necessary
functionality or accessibility labels in the name of simplicity — mystery
isn't minimalism.

### Bolder (amplify one section, not the whole page)
"Bolder" almost always scopes to something that already exists — the
surrounding system, page, and conventions are the given. Touch only the
named target; don't add new colors, fonts, or system primitives the rest of
the page doesn't already own. A section usually reads flat because it opts
out of a move the rest of the page already commits to — bring it up to the
same expressive level in the system's own vocabulary, not a new one. Strip
the copy and check the skeleton alone still says what the section is; if it
only works once the words return, the boldness lived in the text size, not
the design. Commit to one decisive move completely rather than making
everything slightly louder — turning every element up simultaneously reads
as flatter, not bolder.

### Quieter (reduce intensity without going generic)
Subtlety needs precision — quiet without intent collapses into generic, not
sophisticated. Desaturate toward 70–85% rather than fully gray; let neutrals
do more work with color reserved as a true accent (~10% rule); reduce
font-weight extremes (900→600, 700→500) rather than removing weight
contrast entirely; increase whitespace; reduce shadow/blur/glow layering;
shorten animation distances (40px → 10–20px) and switch to ease-out-quart,
never bounce/elastic. Preserve at least one strong anchor — making
everything uniformly small and light removes hierarchy, not intensity.

## Verify

- Every Priority Issue traces to a concrete location, not a vague
  impression.
- Findings are prioritized — not everything can be P0.
- Positive findings are noted, not just problems.
- A refinement pass (bolder/quieter/distill) stayed inside its stated
  scope and didn't silently redesign anything outside the named target.
