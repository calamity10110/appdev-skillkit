---
name: react-best-practices
description: Write and review React/Next.js code for performance and correctness — kill request waterfalls, keep bundles lean, separate server from client work, and control re-renders. Triggers when building, refactoring, or reviewing React components, Next.js pages, data fetching, or bundle optimization.
---

# React / Next.js Performance Practices

A condensed field guide for fast, correct React. Ordered by impact: fix waterfalls and
bundle bloat first; micro-optimizations last.

## 1. Eliminate async waterfalls

Sequential `await`s are the #1 performance killer. Instead:

- **Start early, await late.** Kick off independent promises immediately; only `await`
  them in the branch that needs the value.
- **Parallelize independent work** with `Promise.all()`. If fetches have partial
  dependencies, group them so each group starts as soon as its inputs exist.
- **Check cheap sync conditions before awaiting.** Don't await a flag or remote value
  before a trivial guard that would skip the work anyway.
- **Stream with Suspense boundaries.** Wrap slow sections in `<Suspense>` so the rest
  of the page renders and streams in; don't make the whole page wait on one fetch.

## 2. Keep the bundle lean

- **Import directly, not through barrel files** (`import { x } from 'lib/x'`, not from
  `'lib'`), so tree-shaking can drop the rest.
- **Code-split heavy components** with `next/dynamic` — charts, editors, modals —
  and load them only when the feature activates.
- **Defer third-party scripts** (analytics, logging) until after hydration.
- **Prefer statically analyzable paths.** Fully static import paths let the bundler
  trace exactly what's needed; dynamic lookups widen the bundle silently.
- **Preload on intent:** `prefetch` links and preconnect on hover/focus for perceived speed.

## 3. Server vs client: put work in the right place

- **Default to Server Components.** Fetch, filter, and transform data on the server;
  send the client only what's rendered.
- **Minimize serialized props.** Everything crossing the server→client boundary is
  serialized — trim it to what the client component actually reads.
- **Deduplicate per request** with `React.cache()` for functions called from multiple
  components in the same render.
- **No module-level mutable state in RSC/SSR.** Module state is shared across
  requests; request-scoped state belongs in context or per-request stores.
- **Hoist static I/O** (fonts, logos, config reads) to module scope so it runs once.
- **Authenticate server actions like API routes** — never trust the client call.

## 4. Control re-renders

Memoization is a scalpel, not a habit. Rules:

- **Don't subscribe to state you only use in callbacks.** Read it via a ref or a
  functional update instead of triggering renders.
- **Derive during render, not in effects.** If a value is computable from props/state,
  compute it inline — `useEffect` + `setState` derivations cause an extra render.
- **Prefer primitive deps in effects** and derived booleans over raw objects, so
  referentially-unstable values don't re-fire effects.
- **Memoize expensive subtrees**, not everything: `useMemo`/`memo` pay off when the
  computation or render is genuinely costly; memoizing simple primitives adds noise.
- **Split hooks with independent dependencies** so unrelated state changes don't
  re-run unrelated logic.
- **Use `useState(() => expensiveInit())`** lazy initialization; use functional
  `setState` updates to keep callbacks stable.
- **Never define components inside components** — they remount on every render.
- **Use transitions for non-urgent updates** (`startTransition`, `useDeferredValue`)
  to keep input responsive during expensive renders.
- **Put interaction logic in event handlers**, not effects that watch state.

## 5. Client-side data fetching

- **Deduplicate requests** with a client cache (SWR or equivalent): identical mounts
  should share one in-flight request.
- **Deduplicate global event listeners** and use passive listeners for scroll/touch
  to avoid blocking the main thread.
- **Version localStorage data** and keep stored payloads small — schema drift breaks
  silently.

## 6. Rendering details

- **Hoist static JSX** out of components; don't rebuild constant trees per render.
- Use **`content-visibility`** for very long lists to skip off-screen layout work.
- Prefer inline critical script for client-only initial data over hydration flicker.
- Trim SVG coordinate precision; animate wrapper `<div>`s rather than SVG elements
  for smoother transforms.

## 7. Common traps

- **Hydration mismatches.** Client-only values (timestamps, random IDs, `window`
  size) must not differ between server render and hydration — gate them behind
  mount checks or `suppressHydrationWarning` where truly unavoidable.
- **Non-blocking side effects.** Fire-and-forget work after a response (logging,
  cache warm-up) belongs in `after()` / background tasks, not in the request path.
- **Effect-as-event-handler.** If an effect fires in response to user interaction,
  it should be an event handler instead — effects are for synchronization with
  external systems.
- **Stale closures.** When a callback needs the latest value without re-subscribing,
  use a ref mirror or the latest-value pattern rather than widening deps.

## 8. Review checklist

Before shipping React/Next code, ask: any sequential awaits that could be parallel?
Any client component that could be a server component? Any barrel imports or heavy
code pulled into the initial bundle? Any memo that memoizes nothing? Any state read
only in callbacks? Any component defined inside another? Any request fired twice?

## Input → Process → Output

**INPUT:** A Next.js dashboard page takes 4.2s to interactive. Symptoms: three
sequential `await fetch()` calls in a server component, a 900KB chart library
imported in the initial bundle, and a filter input that re-renders the whole
table on every keystroke.

**PROCESS:**
1. Kill the waterfall: start all three fetches immediately, `await` them together
   with `Promise.all()`; wrap the slowest section in `<Suspense>` so the rest
   streams in.
2. Shrink the bundle: move the chart library behind `next/dynamic` so it loads
   only when its tab activates; replace the barrel import
   (`from 'lib/charts'`) with a direct import.
3. Move work server-side: the data filtering already happens on the server —
   keep it there; trim serialized props to what the client table reads.
4. Fix the re-render: derive the filtered rows during render instead of in
   `useEffect` + `setState`; wrap the keystroke update in `startTransition`
   so typing stays responsive.
5. Verify: re-run the profiler and the bundle analyzer; confirm one network
   round per request via `React.cache()`.

**OUTPUT:** Time-to-interactive ~1.1s, initial JS down ~500KB, filter input at
60fps while the table updates — plus a note of which fixes had which impact,
so the next slowdown gets the same triage order.

## Proven pattern vs avoid pattern

- **Proven:** Start early, await late + default to Server Components. Parallel
  fetches and server-side data work remove whole seconds; everything else is
  fractions. Measure the waterfall first — it's the #1 killer for a reason.
- **Avoid:** Memoizing everything. Wrapping every component in `memo` and every
  value in `useMemo` adds referential bookkeeping without fixing the actual
  costs (network, bundle, misplaced work) — and stale-closure bugs as a bonus.
  Memoize only proven-expensive subtrees.

## Telemetry, testing & debugging

- **Measure:** Core Web Vitals (LCP, INP, CLS) via the `web-vitals` library or
  Vercel Analytics; bundle size per route with `@next/bundle-analyzer`; render
  counts and commit durations with the React DevTools Profiler.
- **Test:** Lighthouse CI budgets on PRs (fail on LCP/INP regressions); assert
  no duplicate network requests in E2E (Playwright route interception);
  hydration-mismatch tests for client-only values.
- **Debug:** Profiler flame graph → which component re-renders and why
  ("why did this render?" via the DevTools highlight updates); Network panel
  waterfall view for sequential fetches; `next build` output for bundle
  composition. Reproduce slowness with CPU throttling (6x) before optimizing.
