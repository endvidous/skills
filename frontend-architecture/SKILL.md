---
name: frontend-architecture
description: Use when structuring a frontend app or reviewing frontend code for architecture (not visuals) — state management choices (React Query/SWR vs Redux/Zustand vs context vs URL), data fetching and caching, network waterfalls, rendering strategy (CSR/SSR/SSG/ISR/islands), component structure, forms, and error/loading states. For visual design, spacing, typography, and polish, defer to the design skills (impeccable, emil-design-eng, etc.) — this skill is the engineering layer under them.
---

# Frontend Architecture

The organizing insight: **server state and client state are different problems**, and most frontend complexity comes from treating one as the other. Data fetched from an API is a *cached snapshot you don't own* — it can be stale the moment it arrives, and the correct operations on it are fetch, cache, invalidate, refetch. Client state (an open modal, a draft, a selection) is *owned* — the correct operations are read and write. A library built for one is the wrong tool for the other ([TkDodo, React Query as a State Manager](https://tkdodo.eu/blog/react-query-as-a-state-manager)).

## Decision 1: Where each piece of state lives

Work down this ladder; state goes at the *first* rung that fits, and the burden of proof rises as you descend:

1. **URL** — filters, pagination, active tab, selected item, search text. Shareable, bookmarkable, survives refresh, back-button works. The most under-used state location; if a user could plausibly share or bookmark the view, the state belongs here.
2. **Local component state** (`useState`) — anything one component or a tight parent-child pair owns. Default for UI state; lift only when a real second consumer appears.
3. **Server-cache layer** (React Query / SWR / RTK Query / Apollo) — *all* data that originates from an API. Owns caching, deduplication, background refetch, invalidation-after-mutation.
4. **Global client store** (Zustand / Redux / Jotai) — cross-component *client-owned* state with no server representation: editor state, multi-step wizard drafts, UI preferences. After server state moves to rung 3, what's left is "usually very tiny" ([TanStack: does this replace client state?](https://tanstack.com/query/v3/docs/framework/react/guides/does-this-replace-client-state)) — a big global store is usually server state in the wrong place.

The one hard rule: **never copy server-cache data into a client store.** The copy immediately stops being invalidated, refetched, or deduped — you've built a second, broken cache. If it needs invalidating after a mutation, it's server state, full stop. (Exception that proves it: genuinely client-heavy apps — visual editors, DAWs — where the document *is* client state between explicit saves.)

## Decision 2: Data fetching

- **Use a server-cache library, not hand-rolled `useEffect` + `fetch`.** The hand-rolled version silently lacks: request deduplication, caching, race-condition handling on unmount/param-change, retry, and invalidation. Those aren't nice-to-haves; their absence is the bug class.
- **Kill waterfalls.** Fetch-on-render (component renders → fetches → child renders → fetches…) serializes network round trips down the component tree; each level adds full latency ([Sentry on fetch waterfalls](https://blog.sentry.io/fetch-waterfall-in-react/), [TanStack: Request Waterfalls](https://tanstack.com/query/latest/docs/framework/react/guides/request-waterfalls)). Fixes, in order: parallelize sibling queries (`useQueries`/`Promise.all`); hoist data requirements to the route level (route loaders, `prefetchQuery` on navigation intent) — render-as-you-fetch; restructure the API so one request serves the screen instead of N chained ones (`api-contracts`).
- **Mutations update the cache through invalidation**, not by hand-patching copies: mutate → `invalidateQueries` → refetch. Optimistic updates are the optimization, applied per-interaction where latency is felt, always with rollback on error.
- **Query keys are an API — centralize them.** Scattered inline key arrays make invalidation a guessing game; a single keys module (factories per domain) makes "what does this mutation invalidate?" answerable.
- Never call the client store from inside a fetcher; state updates that react to mutations belong in `onSuccess` at the call site — keeps fetchers pure and cache-replayable.

## Decision 3: Rendering strategy

Decide per app — and increasingly per *page* — using two questions: **is the content public and crawlable?** and **does time-to-first-paint materially affect conversion?**

| Strategy | Choose when | Cost |
|---|---|---|
| **CSR (SPA)** — default for authenticated apps | Dashboards, tools, anything behind login: no SEO need, users tolerate one shell load | Blank-shell first paint; bundle discipline required |
| **SSG** | Content known at build time: marketing, docs, blog | Rebuild to change; per-user content impossible |
| **ISR / SSG+revalidate** | Mostly-static content that changes on its own schedule | Staleness window; framework-specific semantics |
| **SSR** | Public + dynamic + first-paint-critical: e-commerce listings, feeds, landing pages with user data | A server in the request path — its latency, scaling, and caching are now your problem |
| **Islands / partial hydration** | Content-heavy pages with small interactive pockets (Astro-style) | Ecosystem/framework constraint |

Default reasoning: lean static (SSG/ISR) as far as content allows, add SSR only where freshness-at-first-paint is proven to matter, keep authenticated app surfaces CSR ([Vercel: choosing a rendering strategy](https://vercel.com/blog/how-to-choose-the-best-rendering-strategy-for-your-app), [patterns.dev](https://www.patterns.dev/), [web.dev: Rendering on the Web](https://web.dev/articles/rendering-on-the-web)). Adopting a full-stack meta-framework for a login-walled dashboard buys the SSR operational burden and returns almost nothing — SEO doesn't exist behind auth.

## Decision 4: Component structure

- **Colocation:** code lives as close to where it's used as possible — component + its styles + its query + its test in one place; promote to `shared/` only on the second real consumer (the frontend Rule of Three).
- **Composition before context, context before store:** prop drilling pain is usually solved by composition (pass the composed child, not its props through three layers). React context is for low-frequency, app-wide values (theme, auth identity, locale) — every consumer re-renders on change, so fast-changing data in context is a performance bug; that's what store selectors are for.
- **Feature folders over type folders** at app scale: `features/billing/{components,queries,hooks}` beats global `components/`, `hooks/` buckets, because change is feature-shaped — same bounded-context logic as `service-boundaries`, one level down.

## Decision 5: Forms

- Local/uncontrolled state while editing (form libraries: react-hook-form et al. for anything beyond 2–3 fields — re-render isolation and wiring for errors/dirty/submit); draft state never goes in a global store unless it must survive navigation.
- **One schema, both sides:** define validation as a schema (zod/valibot), infer the TS types from it, run it in the form *and* in the API layer. Client-side validation is UX; the server-side run is the actual trust boundary (`security-boundaries` Decision 5).
- Submit = mutation (Decision 2 rules apply: pending state on the button, server errors mapped back to fields, invalidate affected queries on success).

## Decision 6: Error and loading states are architecture

- Every async boundary has **four states — loading, error, empty, success** — and "empty" and "error" are designed, not discovered in production. A review that only checks the success path is half a review.
- **Error boundaries per route segment**, so a crashed widget takes out its panel, not the app; pair with the server-cache library's error propagation (`throwOnError`) so query failures hit the boundary instead of rendering `undefined`.
- Skeletons follow layout (prevent shift); spinners for genuinely unknown durations; delay showing either ~150–300ms so fast responses never flash.
- Client errors get reported (Sentry-class) with release tags — the frontend is the least-observable tier by default (`observability-slo` applies to the browser too).

## Review checklist

- Any server data copied into a client store, or fetched via raw `useEffect`? (Both are findings.)
- State at the highest fitting rung — URL for shareable view state?
- Waterfalls: sibling queries parallel? Route-level prefetch for known-ahead data?
- Mutations invalidate via centralized query keys? Optimistic updates roll back?
- Rendering strategy argued from public/crawlable + first-paint, not framework default?
- Forms: schema shared with the server? Submit handles pending/error/field mapping?
- All four async states present? Error boundaries at route segments?
- New shared abstraction justified by a second real consumer?

## Anti-patterns

- Query results mirrored into Redux/Zustand "so everything's in one place" — a second cache with no invalidation
- `useEffect` fetch chains; loading flags hand-managed per request
- Global store as the first resort; context carrying fast-changing data
- Inline ad-hoc query keys — invalidation by string-guessing
- SSR meta-framework for an auth-only dashboard; CSR for a public conversion page
- Sequential awaits for independent data (self-inflicted waterfall)
- Forms writing every keystroke to a global store
- Success-path-only components (no empty, no error, no pending-disable)
- `components/`, `hooks/`, `utils/` type-buckets as the only structure at scale

## Sources

- [React Query as a State Manager — TkDodo](https://tkdodo.eu/blog/react-query-as-a-state-manager) · [Does React Query replace client state managers? — TanStack](https://tanstack.com/query/v3/docs/framework/react/guides/does-this-replace-client-state)
- [Request Waterfalls — TanStack Query docs](https://tanstack.com/query/latest/docs/framework/react/guides/request-waterfalls) · [Fetch Waterfall in React — Sentry](https://blog.sentry.io/fetch-waterfall-in-react/) · [How to fetch data in React with performance in mind — Developer Way](https://www.developerway.com/posts/how-to-fetch-data-in-react)
- [Rendering on the Web — web.dev](https://web.dev/articles/rendering-on-the-web) · [Rendering patterns — patterns.dev](https://www.patterns.dev/) · [How to choose the best rendering strategy — Vercel](https://vercel.com/blog/how-to-choose-the-best-rendering-strategy-for-your-app)
