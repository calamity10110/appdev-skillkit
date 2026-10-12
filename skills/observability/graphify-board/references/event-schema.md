# Graphify event schema (JSONL, one event per line)

## `graph_def` — emitted once at run start

```json
{
  "t": 0.0,
  "event": "graph_def",
  "run_id": "etl-2026-10-12-001",
  "nodes": [
    {
      "id": "validate_rows",
      "label": "Validate rows",
      "kind": "process",
      "function": "pipeline::validate_rows (validate.rs:88)",
      "dependencies": ["fetch_rows"],
      "inputs": {"rows": "Vec<Row>"},
      "outputs": {"valid": "Vec<Row>", "rejected": "Vec<Row>"}
    }
  ]
}
```

`kind` is one of `input` | `process` | `output`.

## `node_start` / `node_end`

```json
{"t": 0.42, "event": "node_start", "node": "fetch_rows"}
{"t": 0.91, "event": "node_end", "node": "fetch_rows",
 "duration_ms": 490, "outputs": {"rows": "1000 rows"}}
```

## `node_error` — carries the five Ws

```json
{"t": 1.12, "event": "node_error", "node": "validate_rows",
 "duration_ms": 210,
 "error": {
   "what": "ValidationError: row 412 has negative price",
   "when": "t=1.12s, 3rd node in run etl-2026-10-12-001",
   "where": "pipeline::validate_rows (validate.rs:88), input record id=r412",
   "why": "supplier feed changed date format; price column misaligned",
   "how": "returned Err; run aborted, 588 rows never processed"
 }}
```

## `edge_data` — data summaries between nodes (never full payloads)

```json
{"t": 0.92, "event": "edge_data",
 "from": "fetch_rows", "to": "validate_rows", "summary": "1000 rows"}
```

## Rules

- `t` is seconds since run start, monotonically increasing.
- Summaries only: counts, shapes, durations, status codes. No PII, no secrets.
- Every `node_error` must include all five Ws; a missing W is a bug in the
  instrumentation, not in the run.
