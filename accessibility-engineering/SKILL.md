---
name: accessibility-engineering
description: Use when building or reviewing any UI for accessibility — components, forms, modals, navigation, custom widgets, focus management, keyboard support, ARIA usage, color contrast, screen-reader behavior, or WCAG/EAA/ADA compliance questions. The engineering layer: semantic HTML, keyboard/focus mechanics, ARIA rules, and testing; visual-design skills handle aesthetics. Fires especially on custom interactive components (dropdowns, dialogs, tabs, comboboxes) and SPA route changes.
---

# Accessibility Engineering

The compliance floor is **WCAG 2.2 AA** — it's the reference for the European Accessibility Act (in force since June 2025), US ADA case law, and most procurement ([WCAG 2.2](https://www.w3.org/TR/WCAG22/), [EAA overview](https://www.levelaccess.com/compliance-overview/european-accessibility-act-eaa/)). But the engineering rule that prevents the most damage is simpler: **use the platform.** Semantic HTML elements ship with keyboard support, focus behavior, and screen-reader semantics already correct; almost every accessibility bug is a rebuilt platform feature missing half its contract.

## Decision 1: Semantic HTML before everything

- `<button>` for actions, `<a href>` for navigation, `<label>` wired to every input, `<select>`/`<input type=...>` before custom widgets, one `<h1>` and a heading hierarchy that doesn't skip levels, landmarks (`<nav>`, `<main>`, `<header>`) for orientation.
- A `<div onClick>` is not a button: it's missing keyboard activation (Enter/Space), focusability, the accessible role, and announced state — four bugs in one element.
- Reach for a custom widget only when a native element genuinely can't do the job — and then build it to the [ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/patterns/) pattern (roles + expected *keyboard interaction*, which is the part everyone skips) or, better, use a headless accessible library (Radix/React Aria/Headless UI-class) that has already shipped the keyboard contract.

## Decision 2: ARIA — less is more, verifiably

The [first rule of ARIA](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA): don't use ARIA if native HTML can do it. This isn't style preference — pages *with* ARIA average significantly **more** detectable errors than pages without (59.1 vs 42 per page in the [2026 WebAIM Million](https://webaim.org/projects/million/)); bad ARIA actively lies to screen readers, which is worse than silence.

- ARIA changes *announcements*, never *behavior* — `role="button"` doesn't add keyboard handling; you still owe the JS.
- Legitimate uses: state on custom widgets (`aria-expanded`, `aria-selected`, `aria-current`), `aria-label` where visible text is absent (icon buttons), `aria-describedby` for error/help association, `aria-live` for async announcements.
- `aria-live="polite"` regions for toast/status updates; sparse — a chatty live region is unusable. Form-submit results, background-job completion, and route changes (Decision 3) are the standard cases.

## Decision 3: Keyboard and focus — where SPAs fail

Everything a mouse can do, the keyboard must: reachable (Tab), operable (Enter/Space/arrows per APG pattern), escapable (no traps), in a **visible focus indicator** (never `outline: none` without a replacement; WCAG 2.2 adds [Focus Not Obscured](https://www.w3.org/WAI/standards-guidelines/wcag/new-in-22/) — sticky headers/footers must not hide the focused element).

Focus is **managed state** in an SPA (`frontend-architecture`):
- **Route change:** nothing happens by default — move focus to the new page's heading/main and announce the navigation (live region or focus target); otherwise a screen-reader user "clicks" and hears nothing.
- **Modal open:** focus moves in, Tab is trapped inside, Escape closes, focus **returns to the trigger** on close. Native `<dialog>` handles most of this — use it.
- **Deletion/collapse:** focus moves to a sensible neighbor, never dropped to `<body>`.
- DOM order = tab order; positive `tabindex` is a finding. Only `0` (focusable) and `-1` (programmatic) are legitimate.

## Decision 4: Forms and errors

- Every input has a real `<label>` (placeholder is not a label — it vanishes on input and fails contrast); groups use `<fieldset>`/`<legend>`; correct `type=` and `autocomplete=` (mobile keyboards + WCAG 1.3.5).
- Errors: identified in *text* (not color alone), associated via `aria-describedby` + `aria-invalid`, and on failed submit focus moves to the first invalid field or an error summary that links to fields.
- WCAG 2.2 additions worth designing in: **Target Size** ≥24×24 CSS px for pointer targets, **Redundant Entry** (don't re-ask within a flow), **Accessible Authentication** (no cognitive tests — copy-paste allowed in password fields, no "type the 3rd character") ([new in 2.2](https://www.w3.org/WAI/standards-guidelines/wcag/new-in-22/)).

## Decision 5: The visual floor (engineering side)

Contrast ≥4.5:1 normal text, 3:1 large text and UI components — check tokens once in the design system so features inherit compliance (the design-system move beats per-feature auditing). Color never the sole signal (state, errors, chart series — pair with icon/text/pattern; `dataviz` handles charts). `prefers-reduced-motion` respected for any nontrivial animation. Zoom to 200% and 320px-wide reflow must not lose content or function.

## Decision 6: Testing — automation catches a minority

Automated tools (axe-core) catch roughly **30–57% of issues** ([axe-core](https://github.com/dequelabs/axe-core), [what automation misses](https://testparty.ai/blog/automated-accessibility-testing-guide)) — they verify attributes exist, not that the experience makes sense. Layer:

1. **CI:** axe in component/E2E tests (jest-axe, `@axe-core/playwright`) — zero-violation gate on new code; catches the mechanical 40%.
2. **Per-feature keyboard pass (the highest-yield manual habit):** unplug the mouse, complete the feature's journey — Tab reachability, visible focus, Escape, no traps. Five minutes, catches most of what axe can't.
3. **Screen-reader smoke on critical flows** (VoiceOver is already on the Mac; NVDA is free): does the announced experience make sense on signup, checkout, the core loop?
4. Periodic audit/real AT-user testing for compliance-critical products.

Definition of done for a UI feature includes 1 and 2 — retrofit costs a multiple of build-time correctness (`testing-strategy`'s economics apply).

## Review checklist (any UI diff)

- Native elements used, or APG-pattern/headless library for custom widgets (keyboard contract included)?
- Any `div`/`span` with click handlers? Any positive `tabindex`? Any `outline: none` without replacement?
- New ARIA: is it necessary, valid, and state-updating? (No ARIA beats bad ARIA.)
- Modals: trap, Escape, focus-return? Route changes: focus + announcement?
- Inputs labeled, errors text-based + associated + focused on submit?
- Icon-only buttons named? Images: meaningful `alt` or `alt=""` for decorative?
- Contrast from compliant tokens? Color-only signals? Reduced-motion respected?
- axe passing; keyboard-only pass done?

## Anti-patterns

- `<div onClick>` buttons; `href="#"` links doing button work
- ARIA sprinkled to "add accessibility" without the keyboard behavior (role-only widgets)
- `outline: none` globally "for the design"
- Placeholder-as-label; error states that are only red
- Modals you can Tab out of; focus dropped to `<body>` after actions; silent SPA route changes
- Positive `tabindex` fighting DOM order
- Accessibility as a pre-launch audit line-item instead of a per-PR check
- Overlay/widget products claiming to "make the site compliant" — they don't, and lawsuits agree
- Testing only with a mouse, on a 27" monitor, at 100% zoom

## Sources

- [WCAG 2.2 — W3C](https://www.w3.org/TR/WCAG22/) · [What's new in WCAG 2.2 — W3C WAI](https://www.w3.org/WAI/standards-guidelines/wcag/new-in-22/)
- [ARIA Authoring Practices Guide (APG) — W3C](https://www.w3.org/WAI/ARIA/apg/patterns/) · [ARIA — MDN](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA)
- [The WebAIM Million](https://webaim.org/projects/million/) (ARIA-correlates-with-errors data)
- [axe-core — Deque](https://github.com/dequelabs/axe-core) · [What automated testing catches and misses](https://testparty.ai/blog/automated-accessibility-testing-guide)
- [European Accessibility Act overview — Level Access](https://www.levelaccess.com/compliance-overview/european-accessibility-act-eaa/)
