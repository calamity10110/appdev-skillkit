---
name: tdd-workflow
description: Use before writing any implementation code for a feature, bugfix, or behavior change — test-first red-green-refactor discipline.
---

# TDD Workflow

## The Rule

No production code exists without a failing test written first. If code
gets written before its test, delete the code and start from the test.
A test you never watched fail proves nothing about itself.

## When to Use

**Always:** new features, bug fixes, refactors, behavior changes.

**Exceptions (need the human's explicit ok):** throwaway spikes, generated
code, pure configuration files.

**Rationalization alarm.** Any thought of skipping "just this once" is the
thought that produces the bugs TDD would have caught. The 30-second test is
cheaper than the production debug session.

## The Cycle

### 1. RED — Write one failing test

- Cover exactly one behavior. If the test name contains "and", split it.
- Name the test after the desired behavior in plain words.
- Exercise the real code path. Mock only what cannot run in a test.
- Keep it minimal: the smallest assertion that pins the behavior down.

### 2. Verify RED — Watch it fail

Run it. This step is mandatory, not ceremonial.

- Confirm it **fails**, not errors. An error means the test itself is
  broken — fix it and re-run.
- Confirm it fails for the **right reason**: the feature is missing, not a
  typo in the test.
- If it passes immediately, the test is checking existing behavior. Fix
  the test, not the code.

### 3. GREEN — Minimal implementation

Write the simplest code that makes the test pass. Nothing more:

- No extra features, no "while I'm here" refactors, no speculative
  generality.
- A test that is hard to write is feedback about the design: simplify the
  interface until the test is easy.

### 4. Verify GREEN — Prove the suite is whole

- The new test passes.
- Run the **project's full test command**, not just the file you touched.
  A green run of one file is not a green suite.
- Output is clean: no errors, no warnings.
- Report every failure by name — including ones you didn't cause. A red
  test you watched and stayed silent about falsifies the report.

### 5. REFACTOR — Clean up, green only

After green: remove duplication, improve names, extract helpers. Keep the
suite green throughout. Never add behavior during refactor.

Repeat the cycle for the next behavior.

## Bug Fixes

A bug fix always starts as a reproduction test: write the test that shows
the bug, watch it fail, then fix. The test stays behind as the regression
guard. Never fix a bug without one.

## Test Quality Checklist

- [ ] One behavior per test, name states it plainly
- [ ] Asserts on real outcomes, not on mock behavior
- [ ] Edge cases and error paths covered, not just the happy path
- [ ] Test-only helpers live in test utilities, not production classes
- [ ] Before writing it, you can name the production change that would
      make it fail

## Verification Before Done

- [ ] Every new behavior has a test
- [ ] Watched each test fail first, for the expected reason
- [ ] Implementation is minimal — nothing the tests don't demand
- [ ] Full suite green, output pristine

Any unchecked box means TDD was skipped. Start that piece over.

## Input → Process → Output

**INPUT** — a feature request: "The HTTP client should retry failed requests
with exponential backoff, up to 3 attempts."

**PROCESS** (the skill applied):

1. Write the failing test first — `retries_up_to_3_times_then_errors`:
   a mock server that fails twice then succeeds; assert the client makes
   exactly 3 requests and returns the success.
2. Run it. Watch it fail because no retry logic exists (compilation failure
   counts — but prefer a behavioral failure: keep the client compiling and
   let the assertion fail).
3. Implement the minimum: a retry loop with `backoff = base * 2^n`, capped at
   3 attempts.
4. Run the new test — green. Run the full suite — green.
5. Add the edge-case tests the request implies: `gives_up_after_3_failures`
   (asserts exactly 3 attempts, then the error surfaces), `does_not_retry_on_4xx`
   (client errors are not retried — only network/5xx).
6. Refactor: extract the backoff computation into a pure function and unit
   test it directly.

**OUTPUT** — a test file with 4 tests, all green, plus the smallest client
change that satisfies them. The tests are the specification: anyone reading
them later knows exactly what "retry with backoff" means in this codebase.

## Proven pattern vs avoid pattern

- **Proven:** reproduction tests for bugs. When a bug report arrives, the
  first artifact is a test that fails for the reported reason — not a fix.
  It pins the bug, proves the fix, and guards the regression forever.
  Example: `checkout_fails_with_10_items` failing with the reported 500
  before any code changes.
- **Avoid:** writing the implementation first and "backfilling" tests after.
  It fails because backfilled tests describe what the code *does*, not what
  it *should do* — they pass trivially and catch nothing. If you catch
  yourself doing this, delete the implementation and start from the test.

## Telemetry, testing & debugging

- **Record:** number of new tests per change; for each, whether you observed
  the red phase and what the failure was (assertion vs compile); full-suite
  result after each green. A test whose red phase you never saw is suspect.
- **Test the skill's output:** a good TDD session leaves a test that fails
  if the implementation is deleted (`git stash` the src, keep tests, run —
  they must fail). If tests pass without the implementation, they're testing
  nothing.
- **Debug when it goes wrong:** if a test fails in a way you can't explain,
  you skipped a step — most often the red phase (the test was already
  passing, so it can't detect change) or the minimal-green rule (the fix
  changed behavior the test doesn't cover). Re-run the cycle for that piece:
  revert, write the test, watch it fail, then implement.
