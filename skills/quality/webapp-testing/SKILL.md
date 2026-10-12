---
name: webapp-testing
description: Testing strategy for web applications across unit, integration, and E2E layers. Use when deciding what to test and how, writing Playwright end-to-end tests, adding visual regression or accessibility checks, debugging UI behavior against a live app, or choosing selectors that survive refactors.
---

# Web App Testing — Layered Strategy

Test at the cheapest layer that can catch the bug. Most defects belong in unit
tests; a few critical user journeys belong in the browser; everything in between
is integration.

## The layers and what to assert at each

- **Unit** — fastest, most numerous. Pure functions, validation rules,
  formatting, state transitions.
  - Assert behavior on edge cases: empty input, boundaries, invalid data.
  - No browser, no network, no mocks of things you own.
- **Integration** — the seams. API handlers against a test database,
  repositories against real queries, service-to-service calls.
  - Assert that the layers agree: request shapes, status codes,
    persistence round-trips, error mapping.
- **E2E (browser)** — few, high-value. Only full user journeys that cross
  the whole stack: sign up, checkout, create-and-edit a record.
  - Assert what the user sees *and* what the system did
    (UI state + resulting data), not implementation details.

A healthy suite is a pyramid: many unit tests, fewer integration tests,
a handful of E2E. If E2E is your main safety net, the suite is slow,
flaky, and expensive to maintain.

## Playwright E2E patterns

- Drive Chromium headless via the sync API; always close the browser when done.
- **Reconnaissance before action** on dynamic apps:
  1. navigate and wait for `networkidle`,
  2. screenshot or inspect the DOM to discover selectors,
  3. then execute actions with the discovered selectors.
  - Don't guess selectors from source before the JS has run.
- Prefer resilient selectors, in this order:
  1. accessible role/name (`role=button[name="Save"]`),
  2. visible text,
  3. test IDs,
  4. CSS as a last resort.
  - Avoid deep structural selectors that break on layout tweaks.
- Add explicit waits (`wait_for_selector`, assertions that retry) instead of
  fixed sleeps — fixed sleeps are the primary source of flakiness.
- Keep scripts focused: server management, navigation, and assertions are
  separate concerns.
- For local apps, manage the dev server lifecycle explicitly:
  start, wait for the port, run tests, tear down.
- Capture console logs during runs — browser console errors are often the
  fastest pointer to the real failure.
- Static HTML vs dynamic apps: for static pages, selectors can be read
  straight from the markup; for JS-driven apps, always inspect rendered
  state first.

## Visual regression

- Screenshot full pages (and key components) at fixed viewports;
  compare against checked-in baselines in CI.
- Keep the set small and deliberate — hero, critical flows, dense components.
  Every baseline is maintenance.
- Stabilize renders before capture:
  - wait for network idle,
  - mock or freeze dates, animations, and random data,
  - use a deterministic viewport and device scale.
- Review diffs as images, not text. A passing pixel-diff on a broken layout
  is still broken — spot-check the actual screenshots when UI changes
  intentionally.

## Accessibility checks

- Every E2E run of a key page should include automated a11y checks:
  - keyboard operability (tab through, activate with Enter/Space),
  - visible focus indicators,
  - semantic roles and labels on interactive elements,
  - color contrast,
  - reduced-motion handling.
- Assert behavior, not just attribute presence: a labeled button that can't
  be reached by keyboard still fails.
- Automated checks catch roughly a third of issues; pair them with a
  keyboard-only pass on critical flows before release.

## What makes tests trustworthy

- **Deterministic** — same code, same result. Seed randomness, control time,
  isolate the network.
- **Independent** — tests don't share state or depend on execution order;
  parallel-safe.
- **Fast feedback** — unit suite in seconds, integration in minutes,
  E2E reserved for flows that justify the cost.
- **Failure clarity** — a failing test names the broken contract, not just
  "expected X got Y". Structure assertions around user-visible outcomes.
- **No testing the framework** — don't assert library internals;
  assert your logic at the boundary.

## Debugging UI behavior against a live app

When something looks wrong:

1. load the page and wait for idle,
2. screenshot and dump the DOM,
3. list interactive elements,
4. capture console output.

Reproduce with the smallest script that shows the failure before touching
code. Fix the code, re-run the same script — the reproduction script becomes
the regression test.

## Input → Process → Output

**INPUT:** "Test the new invoice-creation flow end to end before release."

**PROCESS:**
1. Map the journey: login → dashboard → new invoice → add line items →
   submit → invoice appears in list and in the database.
2. Unit layer: line-item math (tax, rounding) as pure-function tests with
   edge cases (zero quantity, 3-decimal prices).
3. Integration layer: POST the create-invoice handler against a test database;
   assert the row, the audit entry, and the 422 on a bad tax code.
4. E2E layer (Playwright): one spec for the happy path only. Reconnaissance
   first — `waitForLoadState("networkidle")`, discover selectors by role,
   assert user-visible outcomes (toast "Invoice #INV-1042 created") plus the
   resulting database row.
5. Visual regression: screenshot the invoice form at two breakpoints,
   stabilized (fonts loaded, animations off), small deliberate set.
6. Accessibility: keyboard-only pass on the form; focus visible on every
   field; contrast on the error text.

**OUTPUT:** A layered suite — fast unit/integration tests owning the logic,
one E2E spec owning the journey, screenshots and an a11y pass — with each
layer asserting only what its level can see.

## Proven pattern vs avoid pattern

- **Proven — reconnaissance before action in E2E:** wait for network idle,
  then locate by role/name, then act. Role-based locators resolve only when
  the element is actionable, which eliminates the largest class of flakes.
- **Avoid — asserting implementation details in E2E:** checking Redux state,
  localStorage keys, or CSS classes from a Playwright spec. It couples the
  spec to refactors and tells you nothing the user would notice. Assert
  user-visible outcomes and resulting data.

## Telemetry, testing & debugging

- **Instrument:** E2E spec duration and pass-rate trends; console-error
  counts per spec run (fail on unexpected console errors); visual-diff
  pixel counts to catch unstable screenshots.
- **Test:** keep E2E count small (journeys, not branches); run the full
  pyramid in CI with unit → integration → E2E ordering so cheap failures
  surface first.
- **Debug:** on failure, open the Playwright HTML report with the trace —
  replay the exact action timeline and DOM snapshots instead of rerunning
  blind; use `page.pause()` locally for interactive stepping.
