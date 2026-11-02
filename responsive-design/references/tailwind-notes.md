# Tailwind Notes (v4 default, v3 differences)

Check the project's version first: v4 projects have `@import "tailwindcss";` in CSS and usually
no `tailwind.config.js`; v3 projects have `@tailwind base; …` and a config file.

## Breakpoints (min-width, mobile-first)

| Prefix | v4 | v3 |
|---|---|---|
| `sm` | 40rem (640px) | 640px |
| `md` | 48rem (768px) | 768px |
| `lg` | 64rem (1024px) | 1024px |
| `xl` | 80rem (1280px) | 1280px |
| `2xl` | 96rem (1536px) | 1536px |

Unprefixed classes apply at all sizes; `md:x` means "from 768 up". To target *only below* a
breakpoint use `max-md:` (v3.2+ and v4). Ranges: `md:max-xl:hidden`.

v4 custom breakpoints in CSS:

```css
@theme { --breakpoint-3xl: 120rem; --breakpoint-xs: 30rem; }
```

v3: `theme.extend.screens` in the config.

Arbitrary: `min-[900px]:grid-cols-3`, `max-[600px]:hidden`.

## Container queries

v4: built in (`@container`, `@md:`, `@max-md:`, `@container/name`, `@md/name:`).
v3: install `@tailwindcss/container-queries`.

## Height, pointer and other media

| Need | v4 | v3 |
|---|---|---|
| Short screens | `[@media(max-height:700px)]:…` or `@custom-variant short (@media (max-height:700px));` | arbitrary variant or `screens: { short: { raw: '(max-height: 700px)' } }` |
| Touch | `pointer-coarse:` | `[@media(pointer:coarse)]:` |
| Hover-capable only | `hover:` does this by default | `future.hoverOnlyWhenSupported` |
| Reduced motion | `motion-reduce:` / `motion-safe:` | same |
| Orientation | `portrait:` / `landscape:` | same |
| Print | `print:` | same |

## Viewport units

`h-dvh`, `h-svh`, `h-lvh`, `min-h-dvh`, `max-h-dvh` — v3.4+ and v4.
`size-*` (width+height) — v3.4+ and v4.

## Useful utilities for responsive work

- `min-w-0`, `min-h-0` — let flex/grid children shrink.
- `shrink-0` — protect fixed chrome.
- `grid-cols-[minmax(300px,420px)_minmax(0,1fr)]` — arbitrary track lists (underscores = spaces).
- `truncate`, `line-clamp-*`, `break-words`, `text-balance`, `text-pretty`.
- `aspect-video`, `aspect-[4/3]`, `object-cover`.
- `overflow-x-auto`, `overscroll-contain`, `snap-x snap-mandatory`, `scroll-mt-*`.
- `[scrollbar-gutter:stable]` — reserve scrollbar space to avoid layout shift.
- `group-data-[state=open]:…`, `data-[active=true]:…` — drive layout modes from state without
  class juggling.
- `inert` attribute (native) on hidden off-canvas panels.

## Common mistakes

- Using `sm:` to mean "small screens". `sm:` means *640 and up*. Small-screen styles are the
  unprefixed ones.
- Stacking `md:`, `lg:`, `xl:` overrides for every property instead of using intrinsic layouts
  (auto-fit grids, `minmax`, `clamp`) that need none.
- `w-screen` for full width → horizontal scroll on Windows (use `w-full`).
- `h-screen` app shells on mobile → bottom cut off (use `h-dvh`).
- Dynamic class names (`` `md:grid-cols-${n}` ``) — Tailwind can't see them; map to full
  literal class strings.
