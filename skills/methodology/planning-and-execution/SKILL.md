---
name: planning-and-execution
description: Use for any non-trivial build — brainstorm intent first, write a plan with checkable steps, execute task by task with tests, then finish the branch cleanly.
---

# Planning and Execution

## Part 1 — Brainstorm Before Building

You are good at building, which makes it easy to build the wrong thing
well. Before anything gets made, find out what the human actually wants
and why.

1. **Check what you know.** If the request is complete and unambiguous,
   just do it. If anything non-trivial is missing, ask.
2. **Ask open questions, one at a time.** Get them describing real
   moments and concrete outcomes, not picking from your options. ("Walk
   me through the last time this came up." "What would make you wince if
   I got it wrong?")
3. **Cover the territory:** what this is and what makes it special; goals,
   non-goals, and anti-goals; prior art they've loved or hated; current
   state and what's been tried; the tooling choices they want to make vs.
   hand to you.
4. **Play it back.** Summarize what you understood, mark your guesses as
   guesses, and ask what's wrong or missing.
5. **Size the work** and say it plainly:
   - Quick, clear task → do it now
   - Small change → describe it briefly in chat, then build through the
     normal workflow
   - Project → write a design document covering intent, goals/non-goals,
     constraints, and which decisions are the human's vs. the builder's
6. **Hard gate:** nothing gets built until the human approves the
   description (chat for a small change, the document for a project).
   Building means product code, scaffolding, dependencies — reading and
   exploring are fine. A throwaway spike is the one exception: offer it,
   keep it minimal and untested, and bring its lessons back through the
   gate before building for real.

## Part 2 — Write the Plan

For a project: turn the approved design into an implementation plan aimed
at an implementer who hasn't seen the codebase. Lock down what they can't
infer: exact files, names, signatures, the spec's exact values, and the
tests that prove each task.

### Plan structure

- **Header:** goal (one sentence), architecture (2–3 sentences), tech
  stack, link to the design doc, global constraints (version floors,
  naming rules — copied verbatim from the design).
- **File map:** which files are created/modified and what each owns.
  Small, focused files; follow existing codebase conventions.
- **Tasks:** each task is the smallest unit with its own test cycle —
  independently testable, worth a reviewer's separate verdict.
- **Steps:** each step is one action with a checkable result. A test
  step carries the test name and assertions. A code step carries the
  exact signature and file; only include a body for an algorithm the
  signature can't determine. A verification step carries the command and
  the expected output. Steps stay in RED-GREEN order: write the failing
  test, watch it fail, implement, watch it pass, commit.

### Self-review the plan

1. **Coverage:** every design requirement maps to a task. Name the gaps.
2. **Step scan:** every step lets the implementer write exactly one
   reasonable thing. Remove lines that decide nothing ("handle edge
   cases") and code blocks the signature and tests already determine.
3. **Consistency:** names, types, and signatures match across tasks.
4. **Proportion:** the plan is shorter than the code it describes. If
   it's mostly code blocks, replace bodies with signatures and tests.

Then hand the plan to the human: does it capture what they want? No code
until they say yes.

## Part 3 — Execute the Plan

1. **Set up isolation.** Work on a branch, ideally in a separate worktree,
   so the main line stays untouched. Never start on main/master without
   explicit consent.
2. **Read the plan and the design** before Task 1. The design outranks the
   plan when they conflict.
3. **Work tasks in order**, following each step's expected output. For
   every command step: run it, read the output, compare against Expected.
4. **Deviate only by ruling.** If a step contradicts the design or an
   interface doesn't match, decide the smallest spec-satisfying change,
   record it as `Ruling: <finding> — <decision and why> — <cost if wrong>`,
   and continue. Unrecorded deviation is a decision made in secret.
5. **Test discipline holds.** Every task goes through the TDD workflow;
   a test the executor never saw fail proves nothing.
6. **Keep a progress ledger** (file, not just memory): task completions,
   rulings, test commands and results. Commit per the plan's commit steps.
7. **Stop only for four things:** an irreversible/destructive operation, a
   security-sensitive action, a side effect outside the workspace (merge,
   push, publish), or a plan so broken every path is a guess. Everything
   else is decided and ledgered — no mid-task check-ins.
8. **Final review.** When all tasks are done, have a fresh reviewer (a
   separate agent or a clean read) check the whole branch against the
   plan and design. Fix critical/important findings in one pass, each with
   its own red-green test cycle and a green suite. Minor findings go in
   the ledger as deferred. There is no second fix pass.
9. **Finish the branch** (Part 4).

## Part 4 — Finish the Branch

1. **Verify first:** run the full test suite on the tree you're about to
   integrate. A green run only proves the tree it ran on.
2. **Confirm the base branch.** Merging into the wrong base is expensive
   to undo.
3. **Present the options; the integration decision belongs to the human:**
   - Merge locally
   - Push and open a pull request
   - Keep the branch as-is for later
4. **Execute the choice.** After a local merge, re-run tests on the merged
   result before declaring success. For a PR: push, open it following the
   repo's template, report the URL, and keep the worktree — feedback gets
   fixed there.
5. **Clean up.** Remove worktrees you created and delete merged branches.
   Discarding work entirely requires the human to say so explicitly; never
   force-remove a worktree holding uncommitted files without asking what
   to do with them.

## Checklist (whole run)

- [ ] Intent confirmed before any building
- [ ] Plan written, self-reviewed, human-approved
- [ ] Executed on an isolated branch with TDD at every step
- [ ] Rulings recorded; deviations traceable
- [ ] Final review done; critical/important findings fixed with tests
- [ ] Full suite green on the final tree
- [ ] Human chose the integration path; cleanup complete

## Input → Process → Output

**INPUT** — a feature request: "Add CSV export of the order list to the
admin panel, with the same filters as the on-screen table."

**PROCESS** (the skill applied):

1. **Brainstorm.** Clarify scope with the human: which columns? date range
   handling? filename convention? Agree: reuse the table's filter query,
   stream rows to avoid memory blowup on large exports, filename
   `orders-YYYYMMDD.csv`. No build until the human approves the scope.
2. **Write the plan.** Goal, architecture (new `export.rs` module reusing
   the filter builder, streaming serializer), file map, tasks sized to
   single steps: (a) extract filter query into a shared builder, (b) CSV
   serializer over an async stream, (c) admin route + auth check, (d)
   filename + content-disposition headers. Each task lists its checkable
   verification (e.g. "export of 100k rows stays under 50 MB RSS").
3. **Self-review the plan.** Catch: the streaming serializer needs the same
   column order as the UI — add a shared column-definition list instead of
   duplicating it.
4. **Execute on an isolated branch**, TDD per step, logging rulings
   ("used `text/csv` content type; kept UTF-8 BOM for Excel compat").
5. **Final review** with fresh context, then the human picks merge vs PR.

**OUTPUT** — an approved plan document, a branch with tested incremental
commits, a rulings ledger, and a review record. The feature lands with its
decisions traceable.

## Proven pattern vs avoid pattern

- **Proven:** the rulings ledger. During execution, every judgment call gets
  one line: what was decided, why, what was rejected. Example:
  "BOM included — Excel compat; rejected BOM-less after checking admin
  machines run Excel." Six months later this is the project's memory.
- **Avoid:** planning in your head and coding directly. It fails because
  scope decisions made mid-code are invisible — the human never approved
  them and they can't be reviewed. If you find yourself "just starting" a
  non-trivial task, stop and write the plan.

## Telemetry, testing & debugging

- **Record:** plan approval checkpoint (approved / changed / rejected);
  task-level completion with verification evidence; deviations from the
  plan with reasons; review findings by severity.
- **Test the skill's output:** a good plan is checkable — each task's
  "done" condition must be verifiable without judgment calls ("100k rows
  under 50 MB RSS", not "works well"). If a task's completion can't be
  checked, the plan is too vague.
- **Debug when it goes wrong:** if execution keeps drifting off-plan, the
  plan was underspecified, not the execution undisciplined — pause, update
  the plan with what you learned, re-approve, continue. If the human
  rejects the plan twice, the brainstorming was skipped: go back and
  re-scope before rewriting.
