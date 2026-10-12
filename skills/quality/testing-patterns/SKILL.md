---
name: testing-patterns
description: Designing test strategy — the test pyramid, AAA structure, unit vs integration vs E2E layering, the Playwright page-object pattern, and keeping suites fast and non-flaky.
---

# Testing Patterns

A strategy for test suites that catch real regressions, stay fast, and do
not rot. Written language-neutral; examples use common xUnit/Playwright
idioms.

## The test pyramid

Structure the suite as a pyramid, not an ice-cream cone:

- **Unit tests (base, most numerous)** — a single class, function, or
  module in isolation; dependencies replaced with fakes/stubs/mocks.
  Fast (milliseconds), deterministic, run on every save.
- **Integration tests (middle)** — the seams: repository against a real
  database, HTTP client against a test server, message producer against a
  real broker. Fewer than unit tests, slower, run on every commit.
- **E2E tests (tip, fewest)** — full user journeys through the real stack
  in a browser or client. Expensive and slow; reserve for the critical
  paths (signup, checkout, core workflows). A suite that is mostly E2E is
  slow, brittle, and tells you *that* something broke without saying
  *where*.

Invert the pyramid and you get slow feedback and flaky signals; keep the
ratios honest by pushing each test down to the lowest layer that can
meaningfully cover it.

## AAA structure

Every test follows Arrange–Act–Assert, visibly separated:

1. **Arrange** — build the world: fixtures, fakes, seed data. Prefer
   explicit builders/factories over shared mutable fixtures; shared
   fixtures couple tests together and make failures cascade.
2. **Act** — invoke the behaviour under test, once. If the act needs
   three calls, you are testing three things.
3. **Assert** — verify the observable outcome: return values, state
   changes, and interactions with collaborators that have side effects.

One behaviour per test, one reason to fail. Name tests as specifications:
`rejects_expired_token_with_401` reads as a requirement; `test_auth_2`
does not. Tests are documentation that executes — future readers learn
the intended behaviour from them.

## What to assert at each layer

- **Unit:** logic, edge cases, error paths, and the contract with
  collaborators (verify interactions only where the side effect is the
  point — over-mocked tests assert implementation details and break on
  every refactor). Test behaviour, not methods: refactor the internals and
  the test should still pass.
- **Integration:** the seam actually works — SQL executes, migrations
  apply, serialisation round-trips, auth headers propagate. Use the real
  dependency (testcontainers or an ephemeral instance), never a mock of
  the thing you are integrating with.
- **E2E:** the user can complete the journey and see the right outcome;
  assert on what the user sees (visible text, URLs, persisted state via
  the UI), never on internal implementation markers.

Common gaps to check deliberately: empty/null/boundary inputs, concurrent
or repeated invocation, clock-dependent logic (inject the clock), and
error paths — the sad paths outnumber the happy path and carry the bugs.

## The page-object pattern (E2E)

Never scatter selectors and flows through test files. Encapsulate each
page or component in an object exposing intention-revealing actions:

```python
# login_page.py
class LoginPage:
    def __init__(self, page): self.page = page
    async def goto(self): await self.page.goto("/login")
    async def login_as(self, user, password):
        await self.page.get_by_label("Email").fill(user)
        await self.page.get_by_label("Password").fill(password)
        await self.page.get_by_role("button", name="Sign in").click()
```

Tests then read as user stories (`await login.login_as("a@b.c",
"secret")`) and a UI change touches one file. Prefer role/label-based
locators over CSS or XPath — they track accessibility and survive markup
churn. Assert with auto-retrying web-first assertions, not fixed sleeps;
explicit waits are the number-one source of flakiness.

## Flaky-test discipline

A flaky test is worse than no test: it trains everyone to ignore red.
Rules:

- Deterministic by construction: seed randomness, freeze time, isolate
  parallel tests (separate databases/schemas, unique resource names).
- No sleeps, no wall-clock dependence, no ordering dependence between
  tests, no shared mutable state.
- Quarantine on first flake: move it out of the main suite immediately,
  fix or delete within the sprint. A "retry until green" wrapper hides
  the rot — track flake rate per test and treat a rising rate as a bug.
- E2E suites get a strict budget (count and runtime); every new E2E test
  must justify why a lower layer cannot cover it.

## Suite hygiene

- Fast feedback loop: unit tests run in seconds on every save; the full
  suite (minus slow E2E) finishes in minutes on every push. If the suite
  is slow, developers stop running it — profile and parallelise before
  adding more tests.
- Coverage is a floor, not a goal: enforce a minimum (e.g. 80%) to catch
  untested code, but chase *meaningful* assertions over percentages —
  100% coverage of trivial getters proves nothing.
- Keep tests independent and hermetic: each test sets up its own data and
  cleans up after itself; no test may depend on another having run.
- Treat test code as production code: review it, refactor it, delete
  dead tests. Duplicated setup logic goes into builders/helpers; magic
  values get names.
- Run the suite in CI on every change, gate merges on green, and make
  failures fast to diagnose: clear assertion messages, captured logs, and
  (for E2E) screenshots/traces on failure.

## Input → Process → Output

**INPUT:** A CI suite with 400 unit tests, 60 integration tests, and 25
Playwright E2E tests; E2E fails on ~1 in 8 runs with "element not found" on
the checkout page.

**PROCESS:**
1. Classify the flake: rerun the failing spec 20× with retries disabled —
   3 failures, all "element not found" after navigation.
2. Bisect: failure correlates with a slow ad-network script on CI (never
   locally). The page-object waits for a structural CSS selector that races
   the network.
3. Fix at the right layer: switch the locator to a role-based selector
   (`getByRole("button", { name: "Pay now" })`), which resolves only when the
   element is actionable.
4. Quarantine: move the spec to a quarantine job tagged `quarantine:checkout`,
   track pass-rate over 7 days before promoting back.

**OUTPUT:** A fixed locator, a quarantine report (failures 3/20 → 0/50 after
fix), and a rule added to the page-object guide: role-based locators only.

## Proven pattern vs avoid pattern

- **Proven — quarantine with a pass-rate gate:** flaky specs move to a
  separate, non-blocking job and return only after N consecutive green runs
  (e.g. 50). The suite stays trustworthy while the flake is diagnosed.
- **Avoid — retry-until-green in the main pipeline:** wrapping the whole E2E
  stage in retries hides real regressions and trains everyone to ignore red.
  Retries belong in quarantine, never as a fix.

## Telemetry, testing & debugging

- **Instrument:** per-spec pass/fail history and duration trends (CI artifacts
  or a small flake dashboard); alert when a spec's 7-day pass rate drops below
  98%.
- **Test:** hermetic unit tests (no network, no wall clock — inject clocks);
  integration tests against real dependencies in containers; E2E only for
  user-visible journeys.
- **Debug:** bisect flakes by seed/environment with `--repeat-each` and
  `--shard`; capture Playwright traces (`trace.zip`) on failure and replay in
  the trace viewer instead of guessing from logs.
