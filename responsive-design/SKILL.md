---
name: responsive-design
description: Rulebook for building web UIs that hold up on every screen — phones, tablets, laptops at 125%/150% OS scaling, browser zoom up to 400%, short landscape viewports and ultrawide monitors. Use it whenever you write or fix HTML/CSS/Tailwind/React layout code (pages, app shells, dashboards, sidebars, split-pane editors, builders with live preview, card grids, tables, forms, modals, navbars, landing pages), and whenever someone says something "breaks", "overflows", "gets squished", "looks wrong on mobile/laptop", "has horizontal scroll", or asks to make a UI responsive, mobile-friendly or desktop-first — even if they never say the word "responsive".
---

# Responsive Design Rulebook

Responsive design is not "add a mobile breakpoint at the end". It is building layouts whose
pieces know their own limits — how small they can get before they stop working, how large
before they stop looking intentional — and then letting the browser fit them into whatever
space exists. Get the intrinsic behaviour right and breakpoints become a few deliberate
switches instead of a pile of patches.

This file is the core rulebook. Detailed patterns live in `references/` — read the one that
matches the task rather than all of them.

| Task involves… | Read |
|---|---|
| App shells, sidebars, split panes, dashboards, card grids, master-detail | `references/layout-patterns.md` |
| A document/canvas/slide preview that must keep exact proportions | `references/scaled-preview.md` |
| Components that live in containers of varying width | `references/container-queries.md` |
| Font sizes, spacing, `clamp()`, long words/URLs | `references/fluid-type-and-content.md` |
| Mobile viewport height, keyboards, notches, iOS quirks | `references/mobile-viewport.md` |
| Touch vs mouse, tap targets, forms, hover menus, tables | `references/interaction-and-forms.md` |
| Images, video, iframes, embeds | `references/media.md` |
| Tailwind v3 vs v4 syntax differences | `references/tailwind-notes.md` |

---

## 1. Know the real screen sizes (they're smaller than you think)

What matters is the **CSS pixel** viewport, not the monitor's resolution. OS scaling and
browser zoom both shrink it:

| Physical setup | CSS viewport (approx.) |
|---|---|
| 1920×1080 at 100% | 1920×~950 usable |
| 1920×1080 laptop at **125%** (Windows default on many laptops) | **1536×~730** |
| 1920×1080 at **150%** | **1280×~600** |
| 1366×768 at 100% (cheap laptops, still common) | 1366×~650 |
| MacBook Air/Pro default | 1440–1512 × ~800–880 |
| 1440 wide at 200% browser zoom | 720 wide |
| 1280 wide at **400% zoom** (WCAG reflow test) | **320 wide** |
| Phones | 360–430 wide |

Two consequences people miss:

- **Height is the scarcer dimension on laptops.** ~600–730px of usable height is normal once
  you subtract browser chrome. Fixed headers + toolbars + footers can eat a third of it. Design
  for short viewports, not just narrow ones.
- **Zoom turns desktops into tablets and phones.** A user at 200% zoom on a 1440 laptop gets
  your 720px layout. That's why every desktop-first app still needs a working narrow layout.

## 2. The ten rules

Each rule exists because of a specific, common failure. Apply them by default.

**1. Let content decide sizes; constrain with `min`/`max`, not fixed values.**
Prefer `max-w-*`, `min-w-*`, `minmax()`, `clamp()` over `w-[600px]`/`h-[400px]`. A fixed width
is a promise the viewport can't always keep; a range is one it can.

**2. Every flex/grid child that should shrink needs `min-w-0` (or `min-h-0` in columns).**
Flex and grid items default to `min-width: auto`, meaning "never smaller than my content".
One long URL, `<pre>`, table or wide input then pushes its siblings off-screen. In grid
templates write `minmax(0,1fr)` instead of bare `1fr` for the same reason.

**3. Never use `100vw` / `w-screen` for page width.** `100vw` includes the vertical
scrollbar on Windows/Linux, so it creates a horizontal scrollbar. Use `w-full`. Likewise, avoid
`h-screen` (100vh) on anything that must fit on mobile — use `h-dvh` / `min-h-dvh`
(see `references/mobile-viewport.md`).

**4. Choose the scroll model on purpose.**
*Document pages* (marketing, articles, settings): let the document scroll; it's what browsers,
screen readers, find-in-page and mobile URL bars are optimised for.
*App shells* (editors, dashboards, chat, builders): lock the shell to `h-dvh`, give each pane
its own `overflow-y-auto`, and put `min-h-0` on every flex/grid ancestor between the shell and
the scrolling pane — without it, the pane grows instead of scrolling.

**5. Use Grid for page structure, Flexbox for alignment within a row/column.**
Grid lets you state the rule directly: `grid-cols-[minmax(300px,420px)_minmax(0,1fr)]` means
"sidebar between 300 and 420, preview takes the rest and may shrink". Percentage splits
(`w-1/2`, `w-[40%]`) give a form panel 512px on a 1024 screen and 1280px on a 2560 one —
neither is what anyone wanted.

**6. Use auto-fitting grids before breakpoints.**
`grid-cols-[repeat(auto-fill,minmax(min(100%,16rem),1fr))]` gives 1→N columns with zero
media queries. The inner `min(100%, …)` stops a single card from overflowing very narrow
containers.

**7. Components respond to their container; pages respond to the viewport.**
A card in a 300px sidebar and the same card in a 900px main area need different layouts even
though the viewport is identical. Use container queries (`@container` + `@md:` in Tailwind v4)
for components, media queries for page-level structure. See `references/container-queries.md`.

**8. Switch layout modes where the content breaks, not at device names.**
Find the width at which the design stops working (labels truncate, preview becomes too small to
read) and put the breakpoint there. Common landing points: ~640 (phone → wider), ~768,
~1024 (sidebars collapse to drawers), ~1280 (multi-pane apps go side by side). Treat these as
starting guesses to test, not truths.

**9. Fluid where it should be, fixed where it must be.**
Type and spacing scale smoothly with `clamp()` (always containing a `rem` term so user font
settings and zoom still work). Things with physical meaning — a printed A4 page, a slide, a
video, a canvas — keep a fixed internal size and are *scaled as a whole*
(see `references/scaled-preview.md`).

**10. Design for input type and accessibility, not just width.**
Hover-only menus fail on touch; 16px tap targets fail on phones; text in `px` that ignores user
font settings fails low-vision users; content that needs 2D scrolling at 320px fails WCAG
1.4.10 Reflow. Use `@media (hover:hover) and (pointer:fine)` for hover enhancements, ≥44px
targets on touch (24px absolute minimum per WCAG 2.2), `rem` units, and test at 320px.

## 3. Workflow

When building or fixing a layout:

1. **Classify the surface.** Document page or app shell? This decides the scroll model (rule 4)
   and roughly which pattern in `references/layout-patterns.md` applies.
2. **Write the smallest working layout first, mobile or narrow-first in CSS** (base styles =
   single column, add `md:`/`lg:`/`xl:` to enhance) — even for "desktop-first" products. It's
   about which way the CSS cascades, not which device you care about most: narrow-first CSS
   degrades gracefully when zoomed, desktop-first CSS has to be un-done at every breakpoint.
3. **Give every region its limits:** min/max widths, which part flexes, which part scrolls,
   what happens to overflow (wrap, truncate with a tooltip, scroll horizontally, or reflow).
4. **Decide the collapse strategy** for each secondary region below its breakpoint:
   stack, drawer/sheet, tabs, accordion, or hide behind a toggle. Never just let it shrink
   to an unusable width.
5. **Check against the real sizes in section 1** — especially a short laptop viewport
   (1280×600), 200% zoom, and 320px wide.

When explaining a fix to the user, name the root cause ("the flex child had `min-width:auto`,
so the long URL forced it wider than the column") — it's what stops the bug coming back.

## 4. Quick diagnosis table

| Symptom | Usual cause | Fix |
|---|---|---|
| Horizontal scrollbar on the page | `100vw`, fixed width, unbreakable text, negative margins, an element wider than its parent | `w-full`; `max-w-full`; `min-w-0` on flex/grid parents; `break-words`/`overflow-wrap:anywhere` |
| One pane crushes the other | Flex/grid child with `min-width:auto` containing wide content | `min-w-0`; `minmax(0,1fr)` |
| Inner pane won't scroll, whole page scrolls instead | Missing `min-h-0` on a flex/grid ancestor, or missing height on the shell | `h-dvh` on shell, `min-h-0` down the chain, `overflow-y-auto` on the pane |
| Bottom bar/button hidden on mobile | `100vh` (= the large viewport, under the URL bar) | `h-dvh` / `min-h-svh`; `env(safe-area-inset-bottom)` padding |
| Layout fine at 1920, broken on a laptop | Designed at 100% scaling; real users at 125–150% | Test at 1536×730 and 1280×600; add height-aware rules |
| Text overflows buttons/cards when translated or zoomed | Fixed heights/widths around text | Remove fixed heights; use `min-h`; let text wrap |
| Scaled preview leaves a huge empty gap or overlaps | `transform: scale()` doesn't change layout size | Size the wrapper to the scaled dimensions (see `scaled-preview.md`) |
| iOS zooms into a field on focus | Input font-size below 16px | `text-base pointer-fine:text-sm` on inputs (key to pointer, not width) |
| Sticky header covers anchor targets | No scroll offset | `scroll-mt-*` on targets or `scroll-padding-top` on `html` |

## 5. Before you call it done

- [ ] No horizontal page scroll from 320px to 2560px wide
- [ ] Works at 1280×600 (laptop at 150%) and at 200% browser zoom
- [ ] Every flex/grid child that holds variable content can shrink (`min-w-0` / `minmax(0,…)`)
- [ ] Exactly one deliberate scroll model; no nested scroll traps on mobile
- [ ] Secondary regions have a named collapse strategy below their breakpoint
- [ ] Images/media have intrinsic ratios reserved (no layout shift) and `max-w-full`
- [ ] Touch targets ≥ 44px on coarse pointers; nothing is hover-only
- [ ] Type uses `rem`/`clamp(rem…)`; inputs ≥ 16px on mobile
- [ ] Long words, URLs, numbers and translated strings don't break the layout
- [ ] Mobile: `h-dvh` not `h-screen`, safe-area insets on fixed bars, `<meta name="viewport" content="width=device-width, initial-scale=1">` present and zoom not disabled
