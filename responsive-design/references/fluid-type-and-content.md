# Fluid Type, Spacing and Unruly Content

## Fluid type with clamp()

```css
:root {
  --step-0: clamp(1rem,     0.95rem + 0.25vw, 1.125rem);  /* body   16 → 18 */
  --step-1: clamp(1.25rem,  1.1rem  + 0.6vw,  1.5rem);    /* h3     20 → 24 */
  --step-2: clamp(1.5rem,   1.2rem  + 1.4vw,  2.25rem);   /* h2     24 → 36 */
  --step-3: clamp(2rem,     1.4rem  + 2.8vw,  3.5rem);    /* h1     32 → 56 */
}
```

Rules:
- **Always mix `rem` with `vw`** in the preferred value. Pure `vw` text ignores the user's font
  size setting and barely changes with browser zoom — a WCAG 1.4.4 failure.
- **Keep the max ≤ ~2.5× the min.** Bigger ranges make zoom ineffective at the top end.
- Body text rarely needs to be fluid; headings benefit most.
- In app UIs (editors, dashboards) prefer fixed `rem` sizes — density should be predictable.
  Fluid type is for content pages.
- Tailwind: `text-[clamp(2rem,1.4rem+2.8vw,3.5rem)]` or define theme tokens
  (`--text-display: clamp(…)` in `@theme` in v4).

Formula for a value going from `min` at viewport `vmin` to `max` at `vmax` (all px):
`slope = (max - min) / (vmax - vmin)`; `intercept = min - slope * vmin`;
`clamp(min/16 rem, intercept/16 rem + slope*100 vw, max/16 rem)`.

## Spacing

- Page gutters: `px-[clamp(1rem,4vw,3rem)]` — never less than 16px on phones.
- Section rhythm: `py-[clamp(3rem,8vw,8rem)]`.
- Component internals: fixed steps (`p-4`, `gap-3`) — fluid internals make components look
  inconsistent side by side.
- Use `gap` instead of margins between siblings; it never leaks outside the container and
  doesn't double up when items wrap.

## Line length

Cap reading text at ~60–75 characters: `max-w-prose` (65ch) or `max-w-[70ch]`. On wide screens,
add whitespace or a second column — don't stretch paragraphs.

## Content that breaks layouts

Real content is longer, wider and stranger than placeholder text. Plan for:

| Content | Defence |
|---|---|
| Long URLs, emails, hashes, German compounds | `overflow-wrap: anywhere` (Tailwind v4 `wrap-anywhere`; v3 `[overflow-wrap:anywhere]`) or `break-words` on the container; `hyphens: auto` with a `lang` attribute for prose |
| Single-line labels that must not wrap | `truncate` + `min-w-0` on the flex parent + `title`/tooltip with the full text |
| Multi-line clamp | `line-clamp-2` |
| Code blocks, `<pre>` | `overflow-x-auto` on the `<pre>`, `max-w-full`, and `min-w-0` on its flex/grid parent |
| Tables | Wrap in `overflow-x-auto` (see interaction-and-forms.md) |
| Numbers in columns | `tabular-nums` so widths don't jitter |
| Translations | Allow +30–40% text length (German, Finnish); never fix button widths or heights |
| User-generated names | Test with 1 char and 80 chars |
| Headings | `text-wrap: balance` (`text-balance`) for even lines; `text-pretty` for paragraphs |
| RTL languages | Use logical properties: `ms-*`/`me-*`/`ps-*`/`pe-*`, `start-*`/`end-*`, `text-start` |

## Units cheat sheet

- `rem` — type, spacing, breakpoints (so they scale with user font size).
- `em` — things that should scale with the local font size (icon next to text, padding in a button).
- `%`/`fr` — shares of a container.
- `ch` — line-length caps.
- `px` — borders, shadows, hairlines. Avoid for type.
- `dvh`/`svh`/`lvh` — viewport height (see mobile-viewport.md).
- `cqi` — size relative to a container (see container-queries.md).
