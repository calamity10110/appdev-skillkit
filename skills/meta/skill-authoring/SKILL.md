---
name: skill-authoring
description: How to write, structure, and validate a new SKILL.md skill. Use when creating a skill from scratch, editing an existing one, improving a skill's trigger accuracy, or setting up test prompts to check a skill works.
---

# Skill Authoring — Writing a Good SKILL.md

A skill is a directory with a `SKILL.md` at its root plus optional supporting files. It teaches an agent a reusable workflow: what to do, when to do it, and how.

## Layout

```
skill-name/
├── SKILL.md        # required: frontmatter + instructions
├── scripts/        # optional: executable helpers for deterministic or repetitive steps
├── references/     # optional: deep docs loaded on demand
└── assets/         # optional: templates, icons, fonts used in output
```

- **scripts/** — things the agent should run, not read (formatters, validators, generators). Executed as black boxes; don't paste their contents into the body.
- **references/** — long-form material split by topic so only the relevant file gets loaded (e.g. one reference per domain variant). Give files over ~300 lines a table of contents.
- **assets/** — files the skill's output embeds or transforms.

## Required frontmatter

```yaml
---
name: skill-name          # lowercase, hyphens; matches the directory name
description: What it does AND when to trigger. This is the primary trigger mechanism.
---
```

The description carries the whole triggering burden — put all "when to use" information there, not in the body. Name the concrete situations and user phrases that should invoke it ("when building X", "when the user asks for Y", "whenever Z is mentioned"), not just a summary of the content. Lean slightly pushy: agents tend to under-trigger, so err toward listing more trigger contexts than fewer.

## Body structure

- Lead with **when to use** as bullet scenarios, then the workflow itself.
- Write instructions in the **imperative**: "Run the linter before committing", not "The linter should be run".
- Keep `SKILL.md` focused (under ~500 lines); push detail into `references/` and point to it with clear "read this when…" guidance.
- Use concrete **input → output examples** to pin formats (report templates, commit formats, file shapes). Show a minimal correct example rather than pages of prose.
- Explain the *why* behind important rules, not just the rule — reasoning survives longer in context than commands.
- Prefer general, portable guidance over hyper-specific examples that won't transfer.

## Progressive disclosure

Design for three loading levels:

1. **Metadata** (name + description) — always in context, ~100 words. This is what decides whether the skill fires.
2. **SKILL.md body** — loaded when the skill triggers. This is the whole workflow in one read.
3. **Bundled resources** — pulled in only as needed; scripts can execute without their contents ever entering context.

Practical consequences: keep the metadata tight and trigger-rich, the body self-contained for the common path, and long reference material out of the body entirely. If the body is growing past ~500 lines, add another hierarchy level with explicit pointers to it.

## Testing a skill

1. Write the draft, then come up with 2–3 realistic test prompts — things a real user would actually say in conversation.
2. Run an agent with access to the skill on each prompt, plus a baseline run without the skill for comparison.
3. While runs are in progress, draft what "good" looks like for each prompt: concrete, checkable expectations (not vibes).
4. Grade the outputs against those expectations, qualitatively and with any quantitative checks you have.
5. Rewrite the skill based on what the runs reveal — unclear instructions, missing edge cases, over- or under-triggering.
6. Repeat the loop, then expand the test set and try again at larger scale.

Skills with objectively verifiable outputs (transforms, code generation, fixed workflows) repay formal evals; style and taste skills are usually judged qualitatively. Match the rigor to the skill type, and let the user decide how far to take it.

## Improving trigger accuracy

If the skill fires too rarely, enrich the description with more trigger phrases and contexts; if it fires when it shouldn't, narrow the description to the specific situations. Generate a handful of candidate queries ("should trigger" and "should not trigger") and check the description against each one, then adjust wording and re-test.

## Safety

A skill must never contain malware, exploits, or anything that compromises the host, and must not mislead about what it does. Refuse to author skills designed for unauthorized access, data exfiltration, or other harm — benign roleplay or creative framing is fine, deception is not.

## Input → Process → Output

**INPUT** — a request: "Write a skill for our team's sqlx conventions —
query patterns, migration discipline, and the transaction rules we keep
repeating in reviews."

**PROCESS** (the skill applied):

1. **Gather the raw material.** Collect 3–5 real examples from the
   codebase: a correct query with `query_as!`, a migration pair, a
   transaction block that got it right — and one review comment where the
   convention was violated.
2. **Draft SKILL.md.** Frontmatter: `name: sqlx-conventions`,
   description naming the triggers ("Use when writing sqlx queries,
   migrations, or transactions in this repo…"). Body: input→output
   examples first (a query before/after), then the rules, each with a
   one-line rationale. Push long reference material into `references/`.
3. **Test the trigger.** Run 2–3 realistic prompts ("add a migration for
   the orders table", "why is this query slow") with and without the
   skill; check it fires when it should and stays quiet otherwise.
4. **Test the content.** Ask the agent to write a query using the draft —
   does it follow the transaction rules? Fix the wording where it
   drifted, repeat.

**OUTPUT** — `sqlx-conventions/SKILL.md` (plus `references/`), validated
against real prompts, ready to install in the repo's skills directory.

## Proven pattern vs avoid pattern

- **Proven:** example-first authoring. Write the input→output examples
  before the rules — agents imitate examples far more reliably than they
  follow abstract prose. Example: show the transaction block done right,
  then state the rule it demonstrates.
- **Avoid:** the encyclopedia skill — dumping every convention, edge case,
  and rationale into one 2000-line file. It fails because the agent
  can't find the relevant rule under load and the skill stops being
  read. Keep the body under ~500 lines; push detail into `references/`
  and let progressive disclosure do its job.

## Telemetry, testing & debugging

- **Record:** trigger accuracy (fires correctly / missed / false-fired)
  across your test prompts; per-test-prompt verdict of whether the skill
  changed the output for the better; iteration count on the draft.
- **Test the skill's output:** the with-skill vs baseline comparison is
  the test. If the skill doesn't measurably improve the output on your
  test prompts, it's not done — usually the description is too vague or
  the examples too abstract.
- **Debug when it goes wrong:** if the skill fires at the wrong times,
  narrow the description's trigger phrases (add "should not trigger"
  cases and re-test). If it fires correctly but the agent ignores it,
  the body is too long or too abstract — cut prose, add an example.
  If two skills overlap, merge them; competing skills produce
  inconsistent behavior.
