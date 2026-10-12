---
name: code-review-workflow
description: Use when work is ready for review or when review feedback arrives — how to request useful reviews and how to receive them with technical rigor.
---

# Code Review Workflow

## Part 1 — Requesting a Review

Get a second set of eyes before work lands. The reviewer judges the work
product, never your session history — hand them crafted context, not a
conversation dump.

### When to request

**Mandatory:** before merging to main, after a major feature, after each
plan task in multi-agent execution.

**Worthwhile:** when stuck (fresh perspective), before a large refactor
(baseline check), after a complex bug fix.

Never skip because "it's simple."

### The request package

1. **Diff range:** identify the base and head commits
   (`git merge-base <main> HEAD` … `HEAD`), or the base/head SHAs for a
   task-scoped review.
2. **Description:** one short paragraph on what was built.
3. **Requirements:** the plan, design doc, or spec it should satisfy —
   the reviewer grades against this, not against the diff's own logic.
4. **Expectation of form:** strengths noted, findings with severity, an
   overall readiness assessment.

### Acting on findings

- **Critical:** fix immediately, with a test proving the fix.
- **Important:** fix before proceeding to the next task or merge.
- **Minor:** note in the ledger for later; don't expand the task.
- **Disagreement:** push back with technical reasoning and evidence —
  code or tests that show it works. Valid feedback is never argued away
  on style grounds.

## Part 2 — Receiving Review Feedback

Technical evaluation, not performance. Verify before implementing, ask
before assuming, and keep correctness ahead of comfort.

### The response pattern

1. **Read** the complete feedback without reacting.
2. **Understand** — restate each item in your own words; if anything is
   unclear, stop and ask about *all* unclear items before implementing
   any of them. Partial understanding produces wrong implementations.
3. **Verify** each item against codebase reality: is it correct for this
   codebase, does it break anything, does the reviewer have full context?
4. **Evaluate** — sort into must-fix (bugs, security, broken contracts)
   vs. suggestions (style, optional improvements).
5. **Respond** with a technical acknowledgment or a reasoned pushback.
6. **Implement** one item at a time, testing each, watching for
   regressions.

### Rules of conduct

- No performative agreement ("You're absolutely right!", "Great point!").
  State what changed or just make the change — the code is the receipt.
- Never implement before verifying. If you can't verify an item, say what
  you need to check it and ask how to proceed.
- Push back when a suggestion is wrong: breaks existing behavior, rests
  on missing context, violates YAGNI, or conflicts with the human's
  architectural decisions. Use evidence, not defensiveness. If you were
  wrong to push back, say so factually and fix it — no long apologies.
- **YAGNI check:** when a reviewer asks to "do it properly," first grep
  for actual usage. Unused code gets removed, not polished.

### Implementation order for multi-item feedback

1. Blocking issues (breaks, security)
2. Simple fixes (typos, imports)
3. Complex fixes (logic, refactoring)

### Handling conflict with prior decisions

If feedback conflicts with decisions your human partner already made,
stop and discuss with them first. External feedback is checked carefully
but skeptically — it doesn't overrule the human's architecture.

### Reply where the comment lives

For inline comments on a pull request, reply in that comment's thread,
not as a top-level comment.

## Checklist

- [ ] Review requested with diff range + requirements, not session history
- [ ] Every feedback item read fully before any implementation
- [ ] Unclear items clarified before acting on any item
- [ ] Each item verified against the codebase before implementing
- [ ] Must-fix separated from suggestions; suggestions don't expand scope
- [ ] Pushback given with technical reasoning where warranted
- [ ] Each fix tested individually; suite still green

## Input → Process → Output

**INPUT** — review feedback on a pull request, three items:

1. "The retry loop has no upper bound on total wait time — a slow endpoint
   could hang checkout for minutes." (correctness)
2. "Nit: rename `do_thing` to `fetch_orders`." (naming)
3. "Why not use the existing `backoff` crate instead of hand-rolling?"
   (design suggestion)

**PROCESS** (the skill applied):

1. **Read everything first.** All three items, fully, before touching code.
2. **Restate and verify.** Item 1: check the code — the loop multiplies
   delay by 2 each attempt with no cap. Confirmed real. Item 2: trivially
   verifiable. Item 3: check the crate — it does cover our case, but adds
   a dependency for 20 lines of code.
3. **Classify.** Item 1 = must-fix (hangs a user-facing path). Item 2 =
   trivial accept. Item 3 = suggestion — push back: the hand-rolled version
   is small, dependency-free, and already tested; adopting the crate adds
   supply-chain surface for no behavioral gain.
4. **Clarify before implementing** anything unclear — none here.
5. **Implement one item at a time**, with tests: cap total wait with
   `max_elapsed`, test `stops_retrying_after_max_elapsed`; rename; reply
   to item 3 with the reasoning.

**OUTPUT** — a review response: item 1 fixed with a test, item 2 renamed,
item 3 declined with technical justification. The reviewer sees every item
addressed, none silently dropped.

## Proven pattern vs avoid pattern

- **Proven:** the severity split. Handle must-fix items immediately,
  suggestions on their own merits, nits in bulk — and say so explicitly.
  Example: "Fixed (1), renamed (2), declining (3) because…". It keeps
  reviews honest: critical issues never hide inside a pile of nits.
- **Avoid:** performative agreement — implementing every suggestion to
  "be agreeable," including ones that expand scope or weaken the design.
  It fails because the codebase pays for politeness: unnecessary
  dependencies, diluted ownership. Push back with evidence instead.

## Telemetry, testing & debugging

- **Record:** feedback items received, per item: verdict (fixed / declined /
  deferred), and for fixes the test that covers it. Track declined items —
  a pattern of declined feedback from one reviewer signals a real
  disagreement worth a conversation, not more reviews.
- **Test the skill's output:** after the review cycle, each must-fix item
  must map to a code change plus a test; each declined item must have a
  written reason. If an item has neither, the review wasn't finished.
- **Debug when it goes wrong:** if reviews keep producing the same class
  of findings (e.g. repeated "no upper bound" style bugs), the problem is
  upstream — the planning or TDD step is skipping that concern. Feed the
  pattern back into the plan template or test checklist instead of fixing
  it review after review.
