---
name: web-performance
description: Use when a site or web app is slow to load or laggy to interact with, when optimizing Core Web Vitals (LCP, INP, CLS), bundle size, images, fonts, or third-party scripts, when setting performance budgets, or when reviewing frontend changes for payload/runtime cost. Complements frontend-architecture (waterfalls, rendering strategy) and caching-strategy (HTTP caching).
---

# Web Performance

Rule one: **field data over lab data.** Lighthouse and DevTools runs are debugging tools; what users experience is the field distribution, and Google scores the **75th percentile of real visits** (CrUX / your own RUM) — a fast M4 MacBook on fiber tells you nothing about your p75. Rule two: performance is a budget, not a project — a one-off "perf sprint" decays in a quarter; a CI budget doesn't.

## The targets (p75, field)

| Metric | Good | What it measures |
|---|---|---|
| **LCP** | ≤ 2.5s | Largest content paint — loading |
| **INP** | ≤ 200ms | Interaction to next paint — runtime responsiveness |
| **CLS** | ≤ 0.1 | Layout shift — visual stability |

(Core Web Vitals — Google). Beyond SEO, these correlate with conversion/bounce — but measure *your* funnel before promising revenue.

## Decision 1: Measure before touching anything

- **RUM first:** collect the vitals from real users (`web-vitals` library → your analytics, or CrUX for public pages), segmented by page template, device class, and country. Optimize the failing *template*, not the homepage you happen to look at.
- Lab (Lighthouse CI, WebPageTest, DevTools traces) is for *diagnosis and regression-catching* — throttled, reproducible, wrong about absolute numbers.
- Every fix gets validated the same way it was found: in the field, at p75, on the same segment. (`observability-slo` logic applied to the browser.)

## Decision 2: LCP — the delivery chain

Work the chain in order; each link gates the next (Optimize LCP — web.dev):

1. **TTFB:** slow HTML caps everything — CDN at the edge, cache the HTML where possible (`caching-strategy` Decision 6), fix slow server rendering (`capacity-estimation` for where the time goes).
2. **Discovery:** the LCP resource must be findable in the initial HTML — an `<img>` in markup, not a CSS background or JS-inserted node discovered after hydration. If it can't be in markup, `<link rel="preload">` it.
3. **Priority:** `fetchpriority="high"` on the LCP image; never `loading="lazy"` on it (lazy-loading the hero is the classic self-inflicted LCP failure). Lazy-load *below-fold* images only (MDN: fix image LCP).
4. **Size:** modern formats (AVIF ~50% smaller than JPEG, WebP ~30%) + `srcset`/`sizes` so phones don't download desktop pixels. An image CDN/transform service makes this a URL parameter instead of a build pipeline.
5. **Render-blocking:** inline critical CSS if measurement says CSS is the gate; `defer` all scripts that don't need to run before paint.

Fonts: `font-display: swap` + preload the one or two critical WOFF2 files + subset them; self-host (third-party font CDNs add a connection to the critical path).

## Decision 3: INP — the main thread

INP is the expensive one — it's architectural, not asset-shaped (INP guidance — web.dev):

- **Long tasks are the enemy:** any main-thread task >50ms can trap an interaction behind it. Find them in the Performance panel / RUM long-task attribution — profile first, the slow code is never where you think (`capacity-estimation`: find the bottleneck).
- Break up work: yield to the event loop between chunks (`scheduler.yield()`/`setTimeout` batching); move pure computation to a **Web Worker**; debounce input-driven recomputation.
- Framework-level: avoid re-rendering the world on keystroke (state colocation and selector granularity — `frontend-architecture`); virtualize long lists; memoize expensive subtrees only where profiling shows re-render cost.
- **Hydration is INP debt:** shipping SSR HTML then rehydrating a huge bundle makes the page *look* ready while the main thread is pegged — part of the rendering-strategy decision, not an afterthought.
- Give instant visual feedback (optimistic UI, pending states) — INP measures paint-after-interaction, and users judge the same thing.

## Decision 4: CLS — layout discipline

The solved metric; it's rules, not investigation: explicit `width`/`height` (or `aspect-ratio`) on every image/video/embed; reserve space for ads, banners, and late content with min-height placeholders; never insert content above existing content except from user action; `font-display: swap` with metric-compatible fallback fonts (`size-adjust`) to stop swap-shift; animate `transform`/`opacity`, never layout properties.

## Decision 5: JavaScript payload

- **Code-split by route** as the default; lazy-load heavy widgets (editor, charts, maps) on interaction or visibility (reduce JS payloads — web.dev). Everything in the entry chunk is a tax on every page.
- Audit before optimizing: bundle analyzer, find the top-10 heaviest deps, then: remove (moment→date-fns/Intl, lodash→native), tree-shake (import specifics, check `sideEffects`), or defer. Dropbox cut 33% largely by deleting and deduplicating, not by clever config (Dropbox bundle-size writeup).
- **Third-party scripts are the usual worst offender:** each tag manager addition ships to every user forever. Load them `defer`/after-interaction, use facades for heavy embeds (YouTube, chat widgets), and audit quarterly — marketing tags outlive the campaigns that added them.

## Decision 6: Budgets in CI

A budget is a **failing check**, not a wiki page: entry-bundle size limit (e.g. ≤200KB gzip, then ratchet down), per-route chunk limits, Lighthouse CI assertions on lab vitals for key templates. Wire into PRs (`size-limit`/`bundlesize`/Lighthouse CI) so the regression conversation happens at review time, at the cost of one red check — not a quarter later as a rescue project. New dependency = its gzip cost stated in the PR description.

## Review checklist

- Is there RUM/CrUX data identifying which template and metric actually fail? (No data → that's the first task.)
- LCP resource discoverable in initial HTML, `fetchpriority="high"`, not lazy-loaded, AVIF/WebP + srcset?
- Dimensions/aspect-ratio on all media? Space reserved for late-loading content?
- New heavy work on the main thread — chunked, yielded, or workered?
- New dependency: bundle cost stated? New route in its own chunk?
- Third-party script additions: deferred, facaded, owner named?
- Budget checks still green — and were they tightened after wins?

## Anti-patterns

- Optimizing to a Lighthouse score on your own laptop; celebrating lab wins with no field validation
- `loading="lazy"` on the hero image; LCP image discovered via CSS/JS
- Hero shipped as 4000px JPEG to phones
- One bundle for the whole app; icon/date/utility libraries imported wholesale
- Tag-manager scripts accumulating with no owner or audit
- `useMemo`/`memo` sprinkled by superstition while a 400ms long task sits unprofiled
- Spinners without reserved space (CLS) or feedback (INP) — the twofer
- A "performance sprint" with no budget left behind
