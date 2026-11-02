# Layout Patterns

Proven structures for common surfaces. Each lists the intent, the code (Tailwind v4 syntax;
see `tailwind-notes.md` for v3), and the collapse strategy for narrow screens.

## Contents
1. App shell (header + workspace, independent scrolling)
2. Dual-pane builder (form/controls + live preview)
3. Sidebar + content (docs, settings, admin)
4. Three-pane editor (nav + canvas + inspector)
5. Dashboard grid
6. Auto-fitting card grid
7. Master–detail (inbox, file browser)
8. Document page (marketing, article)
9. Overlays: drawers, sheets, modals

---

## 1. App shell

The shell fills the viewport exactly; regions inside scroll on their own.

```html
<div class="flex h-dvh flex-col overflow-hidden bg-gray-50">
  <header class="flex h-14 shrink-0 items-center gap-3 border-b bg-white px-4">…</header>
  <main class="min-h-0 flex-1">  <!-- min-h-0 lets this shrink so children can scroll -->
    …regions with their own overflow-y-auto…
  </main>
</div>
```

Why each piece:
- `h-dvh` (not `h-screen`): matches the visible area on mobile as URL bars show/hide.
- `overflow-hidden` on the root: the document never scrolls, so there's one scroll per pane.
- `shrink-0` on the header: it never gets squeezed by the flexible area.
- `min-h-0` on `main`: a flex child's default `min-height:auto` would let it grow to fit its
  content, and then the panes inside never overflow and never scroll.

Keep fixed chrome slim on short viewports. On a 1280×600 laptop, a 64px header + 48px toolbar
+ 40px footer leaves ~450px of workspace. Consider collapsing secondary bars below a height:
`[@media(max-height:700px)]:h-10`.

## 2. Dual-pane builder (resume builders, slide/email editors, code playgrounds, form+preview)

```html
<!-- data-tab="edit" | "preview" is set by the switcher; it only matters below xl -->
<main data-tab="edit"
      class="group/ws grid min-h-0 flex-1 grid-cols-1
             xl:grid-cols-[minmax(340px,440px)_minmax(0,1fr)]">

  <!-- Controls -->
  <aside class="min-h-0 min-w-0 overflow-y-auto border-r bg-white [scrollbar-gutter:stable]
                max-xl:group-data-[tab=preview]/ws:hidden">
    <div class="mx-auto max-w-xl space-y-6 p-4 sm:p-6">…form…</div>
  </aside>

  <!-- Preview -->
  <section class="@container min-h-0 min-w-0 overflow-auto bg-gray-200 p-4 sm:p-8
                  max-xl:group-data-[tab=edit]/ws:hidden">
    …scaled page, see scaled-preview.md…
  </section>
</main>

<!-- Narrow-screen switcher (hidden at xl and up) -->
<nav class="grid shrink-0 grid-cols-2 border-t bg-white pb-[env(safe-area-inset-bottom)] xl:hidden">
  <button class="min-h-12" aria-pressed="…">Edit</button>
  <button class="min-h-12" aria-pressed="…">Preview</button>
</nav>
```

Decisions baked in:
- **`minmax(340px,440px)` sidebar**: controls stay readable and don't balloon on 4K.
- **`minmax(0,1fr)` preview**: the preview takes the rest *and is allowed to shrink*. A bare
  `1fr` is `minmax(auto,1fr)` and would refuse to go below the preview's content width.
- **Breakpoint at `xl` (1280)** is a starting point. Measure: if the preview at the narrowest
  side-by-side width is still legible (≥ ~45% scale for an A4 page), you can drop to `lg`
  (1024). Below that, tabs beat two cramped panes.
- **Tabs on narrow screens keep both panes mounted** (hidden with a class, not unmounted),
  so form state and preview render survive switching. A floating "Preview" button opening a
  full-screen sheet is an equally good alternative on phones.
- **Live preview on narrow screens:** consider a small sticky thumbnail of the preview at the
  top of the form instead of tabs if users need constant feedback.

Optional resizable divider: store the sidebar width in a CSS variable and clamp it,
`grid-cols-[clamp(300px,var(--side,400px),50vw)_minmax(0,1fr)]`; update `--side` from a
pointer-drag handler. Always keep the clamp so a dragged value can't break the layout on a
smaller screen later.

## 3. Sidebar + content

```html
<div class="grid min-h-dvh lg:grid-cols-[16rem_minmax(0,1fr)]">
  <nav class="hidden border-r lg:block">
    <div class="sticky top-0 max-h-dvh overflow-y-auto p-4">…</div>
  </nav>
  <main class="min-w-0 px-4 py-6 sm:px-8">
    <div class="mx-auto max-w-3xl">…</div>
  </main>
</div>
```

Below `lg` the nav becomes a drawer opened from a header button. Keep the content column
capped (`max-w-3xl`/~70ch) — wide screens should add margin, not line length.

## 4. Three-pane editor (nav + canvas + inspector, Figma-style)

```html
<main class="grid min-h-0 flex-1
             grid-cols-[minmax(0,1fr)]
             lg:grid-cols-[15rem_minmax(0,1fr)]
             2xl:grid-cols-[15rem_minmax(0,1fr)_18rem]">
```

Collapse order matters: drop the panel used *least often while working* first. Typically the
inspector becomes a drawer below 2xl, the nav becomes a drawer below lg, the canvas never
collapses. Give each collapsed panel a toggle in the toolbar, and remember its open/closed state.

## 5. Dashboard grid

```html
<div class="grid grid-cols-1 gap-4 md:grid-cols-6 xl:grid-cols-12">
  <section class="md:col-span-3 xl:col-span-4">…KPI…</section>
  <section class="md:col-span-6 xl:col-span-8">…chart…</section>
</div>
```

Charts must resize with their cell: give the chart wrapper `min-w-0` and either an
`aspect-[16/9]` or a fixed height range (`h-64 lg:h-80`), and use the chart library's
responsive mode (or a ResizeObserver). Tables in cells get `overflow-x-auto`.

## 6. Auto-fitting card grid

```html
<ul class="grid grid-cols-[repeat(auto-fill,minmax(min(100%,16rem),1fr))] gap-4">
```

- `auto-fill` keeps empty tracks so a lone card doesn't stretch across the row;
  `auto-fit` collapses them so few cards fill the row. Pick deliberately.
- Equal-height cards: grid rows already stretch; inside the card use `flex flex-col` and
  `mt-auto` on the footer so buttons align.

## 7. Master–detail

Wide: list and detail side by side (`grid-cols-[22rem_minmax(0,1fr)]`, each pane scrolling).
Narrow: one at a time — the list, then the detail as its own route/view with a back button.
Make the selection part of the URL so back/forward and refresh behave and the narrow layout
can deep-link to a detail.

## 8. Document page

Let the document scroll. Use a centered column with fluid side padding:

```html
<main class="mx-auto w-full max-w-6xl px-[clamp(1rem,4vw,3rem)]">
```

Full-bleed sections inside a constrained column: wrap them outside the column rather than using
`w-screen`/negative-margin tricks that cause horizontal scroll.

## 9. Overlays

- **Drawer/sheet**: `fixed inset-y-0 left-0 w-[min(20rem,85vw)]`; trap focus; close on Escape
  and backdrop click; restore focus to the trigger.
- **Modal**: `max-h-[calc(100dvh-2rem)] w-[min(32rem,calc(100%-2rem))]` with an internal
  scroll area, so tall content on short screens scrolls inside instead of falling off-screen.
  On phones a bottom sheet (`inset-x-0 bottom-0 rounded-t-2xl`) is often better.
- The native `<dialog>` element with `showModal()` handles focus trapping and the top layer.
- Lock background scroll while open (`overflow:hidden` on `html`), and add
  `overscroll-behavior: contain` on the overlay's scroller to stop scroll chaining.
