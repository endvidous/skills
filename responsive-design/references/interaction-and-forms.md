# Interaction, Forms and Tables Across Devices

Width tells you little about the input device. A 1366px Surface may be touch-only; a 768px
window may be a desktop with a mouse. Query capabilities directly.

## Pointer and hover

```css
@media (hover: hover) and (pointer: fine) { /* mouse/trackpad: hover reveals, denser UI */ }
@media (pointer: coarse) { /* touch: bigger targets, no hover dependence */ }
```

- Tailwind v4's `hover:` variant already only applies on devices that can hover. In v3 add
  `future: { hoverOnlyWhenSupported: true }`.
- Tailwind v4: `pointer-coarse:` / `pointer-fine:` variants exist; otherwise use
  `[@media(pointer:coarse)]:`.
- **Never hide essential actions behind hover.** Row actions revealed on hover must also be
  reachable on touch (always visible on coarse pointers, or via a "⋯" menu) and on keyboard
  (`focus-within:` reveals them too).
- Tooltips are hover-only by nature; don't put information users need only in a tooltip.

## Target sizes

| Standard | Minimum |
|---|---|
| WCAG 2.2 AA (2.5.8) | 24×24 CSS px (or enough spacing) |
| WCAG AAA / Apple HIG | 44×44 |
| Material | 48×48 dp |

Practical rule: 44px on coarse pointers, 32–36px acceptable for dense desktop tool UIs.
Expand hit areas without growing visuals: padding, or a pseudo-element
(`relative after:absolute after:-inset-2`). Keep ≥ 8px between adjacent targets on touch.

## Forms

- Single column on narrow screens; multi-column only for related short fields (city/state/zip)
  and only when each field keeps a sensible width: `grid gap-4 sm:grid-cols-2`.
- Labels above inputs on narrow screens; side labels (`grid-cols-[12rem_minmax(0,1fr)]`) only
  from `md` up.
- Inputs `w-full` inside a capped container (`max-w-xl`) — full-width inputs on a 2560px screen
  are hard to scan.
- `text-base` (16px) on touch inputs (iOS zoom; see mobile-viewport.md), and ≥ 44px tall:
  `min-h-11 text-base pointer-fine:min-h-9 pointer-fine:text-sm`. Width-based variants (`sm:`)
  get this wrong on tablets.
- Correct `type` and `inputmode` (`email`, `tel`, `numeric`, `decimal`, `url`) and
  `autocomplete` tokens — they choose the right mobile keyboard and enable autofill.
- Sticky action bars on long forms: `sticky bottom-0` inside the scroll container, with
  safe-area padding on mobile.
- Auto-growing textareas: `field-sizing: content` (Tailwind v4 `field-sizing-content`) where
  supported; it's a progressive enhancement — give a `min-h` and `max-h` with `overflow-y-auto`
  so unsupported browsers still work.
- Show validation messages inline below the field, not in tooltips that disappear on touch.

## Navigation

- Wide: inline links. Narrow: a menu button opening a drawer or full-screen sheet. Pick the
  breakpoint by measuring when links start wrapping, not by device.
- For ≤5 primary destinations on mobile apps, a bottom tab bar beats a hamburger (thumb reach,
  visibility).
- Priority+ pattern for toolbars: show as many items as fit, overflow the rest into a "More"
  menu (container queries or a ResizeObserver).
- Keyboard and screen-reader parity: the menu button has `aria-expanded`/`aria-controls`;
  hidden nav uses `hidden`/`inert`, not just `opacity-0`, so it isn't tabbable.

## Data tables

Pick a strategy per table:

1. **Scroll** (default for data-dense tables): `<div class="overflow-x-auto" role="region"
   aria-label="Orders" tabindex="0"><table class="min-w-[48rem] …">`. Make the first column
   sticky (`sticky left-0 bg-white`) so rows keep their label. The region needs `tabindex="0"`
   so keyboard users can scroll it.
2. **Prioritise**: hide low-value columns below breakpoints (`hidden md:table-cell`) and show
   them in an expandable row or detail view.
3. **Reflow to cards** below `md` for short records (≤ ~6 fields): each row becomes a card of
   label/value pairs. Keep the semantic `<table>` on wide screens.

Never let a table set the page width — always wrap it.

## Drag, resize and gestures

- Pointer Events (`pointerdown/move/up`) cover mouse, pen and touch in one API.
- Add `touch-action: none` only on the drag handle, not the whole panel, or you kill scrolling.
- Provide a non-drag alternative (buttons, keyboard arrows) for reorder and resize.
