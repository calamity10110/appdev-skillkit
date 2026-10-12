---
name: graphify-board
description: >
  Build and use a live dataflow graph board: nodes for each input/process/output
  stage carrying dependencies, function, timing and error detail; the board
  updates as the program runs and replays debug captures to show what, when,
  where, why and how each node behaved or failed. Use when instrumenting a
  pipeline, debugging a dataflow, or keeping a living architecture map.
metadata:
  version: "1.0.0"
  relates_to: conventions/api-table.md
---

# Graphify Board

A Graphify board is a **live graph of your program's dataflow**. Each node is
one stage — an input, a process step, or an output — and edges show data
moving between stages. The board updates while the program runs, and it can
replay a recorded run like a flight recorder: scrub the timeline, click any
node, and see exactly what happened, when, where, why, and how — especially
where it errored out.

## Node data model

Every node carries the same detail record:

| Field | Meaning |
|---|---|
| `id` | Stable node key, e.g. `validate_rows` |
| `label` | Human label, e.g. "Validate rows" |
| `kind` | `input` \| `process` \| `output` |
| `function` | Code reference: `module::function` or `file:line` |
| `dependencies` | Upstream node ids that must complete first |
| `inputs` / `outputs` | Named ports with type summaries, e.g. `rows: Vec<Row>` |
| `status` | `pending` \| `running` \| `ok` \| `error` \| `skipped` |
| `started_at` / `ended_at` / `duration_ms` | Timing |
| `error` | On failure: the five Ws — see below |

Edges carry a data summary (`1000 rows`, `2.1 MB`, `status 200`) — shapes and
counts, never full payloads.

## The five Ws of a failed node

When a node errors, record all five — this is what makes playback debugging
work:

- **What** — the failure itself: error type + message (`ValidationError:
  row 412: negative price`).
- **When** — timestamp and position in the run (t=0.45s, 3rd node).
- **Where** — code location (`pipeline::validate_rows`, `validate.rs:88`) and
  which input record triggered it.
- **Why** — the root cause in domain terms (supplier feed changed its date
  format), not just the exception chain.
- **How** — how it surfaced: returned `Err`, panicked, timed out, rejected
  by a downstream schema — and what the node did about it (retried, skipped
  the record, aborted the run).

## Live update

Instrument the program to emit graph events as it runs:

1. At each stage boundary, emit `node_start` / `node_end` / `node_error`
   events (see `references/event-schema.md`). In Rust, do this with a
   `tracing` layer so instrumentation is one attribute
   (`#[tracing::instrument]`) per function — no manual event code in business
   logic.
2. The board subscribes to the event stream (websocket, file tail, or message
   queue) and updates node status, timing, and edge summaries in real time.
3. Map stage functions to the shared vocabulary in
   `conventions/api-table.md` (`poll_input`, `update`, `invoke`, `write`,
   …) so node labels stay consistent across the codebase.

## Debug capture & playback

1. **Capture:** every run writes its event stream to a JSONL capture file
   (`captures/<run-id>.jsonl`). Captures are small — summaries, not payloads.
2. **Replay:** the board loads a capture and rebuilds the graph at any
   timestamp. Scrub the timeline; nodes light up as they ran. Click a node for
   its full detail record, including the five Ws on errors.
3. **Compare:** diff two captures (last green run vs failing run) to see
   which node diverged first — the board highlights the earliest behavioral
   difference, which is usually the root cause's neighborhood.
4. **CI:** on test failure, attach the capture as a build artifact. A failing
   run you can replay beats a stack trace you have to imagine.

A minimal single-file playback board and a sample capture ship in
`references/` — open `board.html`, load the sample, scrub, click nodes.

## Continuous development

Treat the board as a **living architecture map**: regenerate it from
instrumentation on every run rather than drawing it by hand. When the code
changes, the graph changes with it — stale diagrams become impossible. Review
the graph diff in PRs the same way you review code diffs: a new edge is a new
data dependency and deserves the same scrutiny.

## Input → Process → Output

**Input:** "Our ETL job fails intermittently and nobody can tell which stage
drops records."

**Process:**
1. Add `#[tracing::instrument]` to each stage function; add a graphify
   tracing layer emitting `node_start`/`node_end`/`node_error` + `edge_data`
   summaries (row counts in, rows out).
2. Run the job; the board shows live counts per edge. The `validate` node's
   out-edge consistently shows fewer rows than its in-edge.
3. Save the capture; replay the failing run; click `validate` — the five Ws
   show *what* (dropped 3% of rows), *when* (t=12.4s), *where*
   (`validate.rs:88`), *why* (supplier date format changed), *how* (records
   skipped silently, no error raised).

**Output:** the silent skip becomes a `node_error` with full five-W detail;
the board now catches the next format change the day it ships.

## Proven pattern vs avoid pattern

- **Proven:** instrument at stage boundaries with summaries (counts, shapes,
  durations). The graph stays cheap, the signal stays high, and captures are
  small enough to keep forever.
- **Avoid:** dumping full payloads into events, or hand-drawing the graph in
  a wiki. Payloads make captures huge and leak PII; hand-drawn graphs rot
  within a week. Summaries + generated-from-code is the combination that
  survives.

## Telemetry, testing & debugging

- **Telemetry:** the event stream *is* telemetry — aggregate node durations
  into p50/p95 per stage, alert when a stage's duration or error rate drifts
  from baseline. The board is the dashboard; the capture log is the audit
  trail.
- **Testing:** unit-test the event schema (every emitted event validates);
  golden-test the graph builder (a scripted run produces the expected node
  set and edges); replay-test playback (loading a capture reproduces the
  exact node states).
- **Debugging the board itself:** if nodes never light up, check the event
  transport first (is the subscriber connected?); if edges are wrong, check
  `dependencies` declarations — the graph renders what you declare, so a
  wrong edge means a wrong or missing dependency, which is itself a finding.
