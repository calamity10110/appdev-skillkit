---
name: systematic-debugging
description: Use on any bug, test failure, or unexpected behavior before proposing any fix — root-cause-first debugging in four phases.
---

# Systematic Debugging

## The Rule

Find the root cause before attempting any fix. Treating a symptom is not
debugging — it is deferring the bug. Under time pressure this discipline
is *more* valuable, not less: guessing is what makes emergencies long.

## Phase 1 — Root Cause Investigation

Complete all of this before proposing a fix:

1. **Read the errors fully.** Stack traces, warnings, line numbers, error
   codes — they often contain the answer. Skimming is guessing.
2. **Reproduce reliably.** Pin down the exact steps. If it doesn't
   reproduce consistently, gather more data — never guess from a flaky
   symptom.
3. **Check what changed.** Recent commits, diffs, new dependencies,
   config or environment differences. Most bugs were introduced, not born.
4. **Instrument across boundaries.** In multi-component systems (API →
   service → database, CI → build → signing), log what enters and exits
   each boundary and run once. The evidence shows which layer breaks —
   investigate that layer, not the one that complained loudest.
5. **Trace backward to the source.** From the bad value, ask what produced
   it, then what produced *that*, until you reach the origin. Fix at the
   origin, not at the symptom.

You are done with Phase 1 when you can state, in one sentence, what is
wrong and why.

## Phase 2 — Pattern Analysis

1. Find similar working code in the same codebase.
2. Compare: list every difference between working and broken, however
   small. Do not dismiss a difference as "can't matter" — verify it.
3. Map dependencies: settings, config, environment, and assumptions the
   broken path needs but the working one doesn't.

## Phase 3 — Hypothesis

1. State a single hypothesis: "I think X is the root cause because Y."
   Be specific. Write it down.
2. Test it with the smallest possible change. One variable at a time.
3. Hypothesis confirmed → proceed. Refuted → form a new one. Never stack
   a second fix on top of a failed one.
4. If you don't understand something, say so and research or ask. Don't
   pretend and don't proceed on hope.

## Phase 4 — Implement and Verify

1. **Write the reproduction test first** — simplest automated test that
   shows the bug. (Use the TDD workflow skill.)
2. **Apply one fix**, addressing the root cause only. No bundled
   refactors, no drive-by improvements.
3. **Verify the fix:** reproduction test passes, full suite still green,
   the original symptom is actually gone. Evidence, not confidence.
4. **Count your fixes.** If two fixes have failed, stop and return to
   Phase 1 with the new information.

### Three failed fixes → question the architecture

If three separate fixes each reveal a new problem in a different place,
or each fix demands major restructuring, the pattern is wrong, not the
patches. Stop fixing. Ask whether the architecture should change — and
discuss that with your human partner before writing more code.

## Environmental / Intermittent Issues

If the investigation genuinely concludes the cause is environmental,
timing-dependent, or external: document what you checked, add handling
(retry, timeout, clear error) plus logging for next time. But treat "no
root cause" as a strong signal the investigation was incomplete — most
are.

## Red-Flag Thoughts (stop and return to Phase 1)

- "Quick fix for now, investigate later"
- "Just try changing X and see"
- "It's probably X, let me patch it"
- "One more fix attempt" (after two failures)
- Listing solutions before tracing the data flow
- "Skip the test, I'll verify manually"

## Quick Checklist

- [ ] Reproduced reliably with exact steps
- [ ] Root cause stated as cause, not symptom
- [ ] Single hypothesis, tested minimally
- [ ] Reproduction test written before the fix
- [ ] One fix at a time; suite green after
- [ ] Fix count < 3, or architecture under discussion

## Input → Process → Output

**INPUT** — a bug report: "Checkout returns 500 when the cart has 10 or more
items. Smaller carts work fine."

**PROCESS** (the skill applied):

1. **Investigate.** Read the full error: `index out of bounds` in
   `discount.rs:42`. Reproduce with a script: build a cart, add items
   1…9 (200 OK), 10…12 (500). Reliable reproduction confirmed.
2. **Trace to origin.** The handler calls `apply_tiered_discount(items)`;
   the function indexes `tiers[items.len()]` into a hardcoded 10-element
   array. Working examples (carts < 10) never reach the end of the array.
3. **Hypothesis.** Single, minimal: "the tier lookup assumes cart size <
   10; carts ≥ 10 index past the array." Test it by calling the function
   directly with 10 items — same panic. Root cause confirmed: a fixed-size
   table for a variable-size input.
4. **Implement.** Write the reproduction test first
   (`discount_applies_to_carts_over_10_items`), then one fix: replace the
   array index with a saturating lookup (`tiers.get(n).unwrap_or(&top_tier)`).

**OUTPUT** — a diagnosis ("fixed-size tier table indexed by cart size") plus
a regression test and a one-line fix. The report now documents cause, not
symptom.

## Proven pattern vs avoid pattern

- **Proven:** the hypothesis log. Keep a running list of hypotheses, each
  with the minimal test that would confirm or kill it, and the result.
  Example: `H1: array bounds (test: call fn with n=10 → confirmed)`. It turns
  debugging into a searchable record and stops you from re-testing dead
  theories.
- **Avoid:** patch-driven debugging — changing code until the symptom
  disappears. It fails because the symptom can vanish while the cause
  remains (e.g. capping carts at 9 "fixes" the 500). If a fix works but you
  can't state the root cause in one sentence, you haven't fixed it.

## Telemetry, testing & debugging

- **Record:** the hypothesis log (hypothesis, test, verdict); time from
  reproduction to root cause; number of fixes attempted. More than 2 failed
  fixes is the signal to stop and question the architecture.
- **Test the skill's output:** the reproduction test must fail on the
  pre-fix code and pass on the post-fix code — verify both directions.
  The diagnosis must name the component and the invariant that broke
  ("tier table assumed cart size < 10"), not restate the symptom.
- **Debug when it goes wrong:** if you're stuck past the 3-fix rule, you're
  probably debugging the wrong layer — re-trace from the error to the
  origin and re-check: is the reproduction actually reliable? Is the
  "working" example truly the same code path? Most stuck sessions trace to
  an unreliable reproduction or a hypothesis that was never actually
  tested.
