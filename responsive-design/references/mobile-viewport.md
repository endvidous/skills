# Mobile Viewport, Keyboards and Safe Areas

## The viewport meta tag

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

- Never add `maximum-scale=1` or `user-scalable=no`. It blocks pinch-zoom (an accessibility
  failure) and is usually added to hide an iOS input-zoom bug that has a proper fix (below).
- `viewport-fit=cover` lets content draw under the notch/home indicator; only add it if you
  then pad with safe-area insets.

## Viewport height units

Mobile browsers show and hide their URL/tool bars, so "the viewport height" has three values:

| Unit | Means | Use for |
|---|---|---|
| `svh` | small — bars visible (smallest) | Content that must be fully visible at all times (hero CTA) |
| `lvh` | large — bars hidden (largest) | Backgrounds that shouldn't jump |
| `dvh` | dynamic — the current one | App shells (`h-dvh`) that should always fill the visible area |
| `vh` | = `lvh` on mobile | Avoid for anything that must fit |

Tailwind: `h-dvh`, `min-h-svh`, `h-lvh` etc. (v3.4+ and v4).

`h-screen` / `100vh` on a mobile app shell hides its bottom ~60–100px under the browser bar —
the classic "my bottom nav/button is cut off" bug.

## Safe areas (notches, rounded corners, home indicator)

```html
<nav class="fixed inset-x-0 bottom-0 pb-[env(safe-area-inset-bottom)]">…</nav>
<header class="sticky top-0 pt-[env(safe-area-inset-top)]">…</header>
```

Landscape phones also have left/right insets: `px-[max(1rem,env(safe-area-inset-left))]`.
`env()` falls back to 0 where not applicable, so it's safe everywhere.

## The on-screen keyboard

- When the keyboard opens, the *visual* viewport shrinks; on iOS the layout viewport doesn't,
  so `position:fixed; bottom:0` bars can end up behind the keyboard or float mid-screen.
- For chat inputs and bottom-anchored forms, either avoid fixed positioning (put the composer
  at the end of a `flex flex-col h-dvh` shell) or follow `window.visualViewport` resize events.
- Chromium supports `<meta name="viewport" content="…, interactive-widget=resizes-content">`
  to make the keyboard resize the layout viewport, so `dvh` layouts shrink above it.
- Make sure the focused field scrolls into view: avoid `overflow:hidden` on the document while
  a form is on screen.

## iOS input zoom

iOS Safari zooms the page when focusing an input whose font-size is < 16px. Fix: inputs at
`text-base` (16px) on touch devices — e.g. `text-base pointer-fine:text-sm` — instead of disabling
zoom. Key it to the pointer, not the width: an iPad at 820px is above `sm` but still zooms.

## Scrolling behaviour

- `overscroll-behavior: contain` (`overscroll-contain`) on drawers, modals and inner scroll panes
  so reaching the end doesn't scroll the page behind (or trigger pull-to-refresh).
- Avoid nested vertical scrollers on phones — users can't tell which one they're moving. On
  narrow layouts, prefer one document scroll; reserve per-pane scrolling for wide layouts.
- Horizontal carousels: `overflow-x-auto snap-x snap-mandatory` with `snap-start` children;
  show a peek of the next item so the affordance is visible.
- `scroll-padding-top` on `html` equal to the sticky header height fixes anchor links.

## Orientation and short screens

Landscape phones are ~360–430px tall. Fixed headers + bottom bars can leave almost nothing.
Use height queries to slim chrome:

```css
@media (max-height: 30rem) { .app-header { height: 2.5rem } .bottom-nav { display:none } }
```

Tailwind v4: `[@media(max-height:30rem)]:h-10` or a custom variant
(`@custom-variant short (@media (max-height: 30rem));` → `short:h-10`).

## PWA / standalone

`display-mode: standalone` removes browser chrome; check `@media (display-mode: standalone)` if
you show "install" prompts or need extra top padding for the status bar.
