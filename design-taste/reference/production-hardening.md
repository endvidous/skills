# Production hardening

A design that only works with perfect data isn't production-ready. Harden
against the inputs, errors, languages, and network conditions real users
actually produce, then optimize what's actually measured as slow.

## Hardening: test with extreme inputs before shipping

- Very long text (names, titles, descriptions) and very short text (empty,
  single character).
- Special characters: emoji, RTL text, accented characters.
- Large numbers (millions/billions) and large datasets (1000+ list items,
  50+ select options).
- No data at all — the empty state.

## Hardening: error scenarios to design for

Network failure (offline, slow, timeout), API errors per status code
(400 validation, 401 → redirect to login, 403 permission error, 404 not
found, 429 rate limit, 500 generic error with support path), form
validation errors, permission errors, concurrent operations (double-submit,
race conditions).

## Hardening: internationalization

- Text expansion budget: 30–40% extra space for translated strings (German
  is usually the longest common case). Never use fixed-width containers for
  translatable text.
- RTL support via logical CSS properties (`margin-inline-start`,
  `padding-inline`, `border-inline-end`) rather than physical left/right
  properties, so RTL flips automatically.
- UTF-8 everywhere; test with CJK characters and emoji (which can be 2–4
  bytes and break naive length-based truncation).
- Use `Intl.DateTimeFormat` and `Intl.NumberFormat` rather than hand-rolled
  date/currency formatting — locale-correct formatting is not worth
  reimplementing.
- Use a real i18n library's pluralization rules rather than an
  English-only `count !== 1 ? 's' : ''` check — many languages have more
  than two plural forms.

## Hardening: dimensions to systematically cover

**Text overflow** — single-line truncation with ellipsis, multi-line clamp
(`-webkit-line-clamp`), or explicit wrapping (`overflow-wrap: break-word`,
`hyphens: auto`) chosen deliberately per element, not left to accident.
`min-width: 0` on flex/grid children so they can shrink below their content
size instead of overflowing their container.

**Error handling** — clear messages with a retry affordance and an
explanation of what happened, not a bare "Error occurred." Preserve user
input on validation failure rather than clearing the form. Progressive
enhancement: core functionality should degrade gracefully, not hard-fail,
when a non-critical feature is unavailable.

**Edge cases** — empty states with a clear next action (not just "No
items"); loading states that name what's loading and give a time estimate
for genuinely long operations; large datasets via pagination or virtual
scrolling rather than rendering 10,000 items at once; concurrent-action
guards (disable a submit button while its request is in flight); explicit
read-only/permission states with a reason, not a silently disabled control.

**Input validation** — client-side validation for immediate feedback,
always paired with server-side validation (never trust the client alone);
explicit constraints communicated before submission, not just enforced
after.

**Accessibility resilience** — full keyboard operability with a logical
tab order and managed focus in modals; live-region announcements for
dynamic content changes; don't rely on color alone in Windows high-contrast
mode.

**Performance resilience under bad conditions** — skeleton screens and
progressive image loading for slow connections; cleaned-up event
listeners, cancelled subscriptions, cleared timers, and aborted in-flight
requests on unmount to prevent memory leaks; debounce search input (~300ms)
and throttle scroll handlers (~100ms).

## Optimization: measure before touching anything

Identify the actual bottleneck — initial load, an interaction, or an
animation — and what's causing it, before making any change. Premature
optimization on a part that isn't actually slow wastes the pass. Measure
again after.

### Loading performance
- Modern image formats (WebP/AVIF), correctly sized (don't ship a 3000px
  image for a 300px slot), lazy-loaded below the fold, served via
  `srcset`/`sizes` for responsive delivery, compressed to 80–85% quality
  (usually imperceptible).
- Code-split by route and by heavy component; tree-shake; remove unused
  dependencies; dynamic `import()` for anything non-critical to first
  paint.
- Critical CSS inlined, the rest loaded async; unused CSS removed.
- `font-display: swap` (or `optional`), subset to the characters actually
  used, preload the critical font file, limit the number of weights
  loaded.
- Preload critical assets, prefetch likely next navigations, async/defer
  non-critical scripts.

### Rendering performance
- Batch DOM reads, then batch DOM writes — never alternate them in a loop
  (each read-after-write forces a synchronous layout recalculation).
- `content-visibility: auto` for long off-screen sections; virtual
  scrolling (react-window, TanStack Virtual) for genuinely long lists;
  flatter DOM trees render faster than deeply nested ones.
- Prefer `transform`/`opacity` for movement; reserve blur/filter/shadow
  effects for small, bounded, isolated areas since they're expensive to
  composite; use `will-change` only for a specific known-expensive
  animation, never as a blanket default.

### Animation performance
Target 16ms/frame (60fps). GPU-accelerated properties
(`transform`/`opacity`) over layout-triggering ones (`left`/`top`/`width`).
`IntersectionObserver` to trigger animation only when an element is
actually visible, rather than computing it continuously off a raw scroll
listener.

### Framework-level (React or equivalent)
`memo()` for expensive components, `useMemo`/`useCallback` for expensive
computations, virtualized long lists, route-level code splitting, no
inline function/object creation inside a render path that re-triggers
child re-renders.

### Network
Combine small requests, use an SVG sprite for icon sets, remove unused
third-party scripts, paginate API responses rather than returning
everything, use compression (gzip/brotli) and HTTP caching headers, serve
static assets from a CDN.

### Core Web Vitals targets
| Metric | Target | Primary levers |
|---|---|---|
| LCP (Largest Contentful Paint) | < 2.5s | Optimize hero image, inline critical CSS, preload key resources, SSR |
| INP (Interaction to Next Paint) | < 200ms | Break up long JS tasks, defer non-critical JS, web workers for heavy compute |
| CLS (Cumulative Layout Shift) | < 0.1 | Explicit `width`/`height` or `aspect-ratio` on images/video/embeds, never inject content above existing content |

## Code quality (pairs with a redesign/hardening pass)

Remove dead CSS, unused components, and orphaned files as part of the same
pass that touches them — don't leave cleanup for later. Replace one-off
custom implementations with the shared component the system already owns.
Promote genuinely reusable values to tokens; resist creating a system
abstraction for a single local exception.

## Strategic omissions

The highest-value hardening work is often not visible in a screenshot:
missing empty states, missing loading states, missing error states, and
missing permission states are frequently the actual gap in an otherwise
polished-looking interface. Auditing for what's *absent* — not just what's
under-styled — is where a hardening pass earns the most.

## Never

- Assume perfect input; validate everything.
- Skip internationalization; design for global by default.
- Leave a generic "Error occurred" message.
- Forget offline/slow-connection scenarios.
- Trust client-side validation alone.
- Use fixed widths for translatable text.
- Optimize a micro-optimization while a bigger bottleneck sits unaddressed.
- Sacrifice accessibility for a performance win, or functionality for an
  optimization.
- Measure only on a fast desktop machine with a fast connection — test on
  a low-end device and a throttled (3G) connection.

## Verify

- Long text (100+ characters), emoji, RTL, and CJK all render without
  breaking layout.
- Network failure, slow connection, and all relevant API error codes have
  a real, tested state.
- 1000+ item datasets and rapid repeated submission don't break the UI.
- Before/after performance metrics were actually compared, not assumed.
- Tested on a real mid-range device and a throttled connection, not just a
  fast desktop.
