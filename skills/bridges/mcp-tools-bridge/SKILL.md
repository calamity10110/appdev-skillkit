---
name: mcp-tools-bridge
description: >
  Design and build bridges from your app to other apps: expose capabilities
  as MCP tools, consume external MCP servers, and adapt CLIs/REST/webhooks
  as tools. Use when the app must talk to other apps, agents, or services.
metadata:
  version: "1.0.0"
  relates_to: conventions/api-table.md
---

# MCP & Tools Bridge

Bridges let your app exchange capabilities with other apps. Two directions:

- **Serve:** your app exposes tools (an MCP server, a CLI, a webhook) so
  other apps and AI agents can call into it.
- **Consume:** your app calls tools exposed by others (external MCP servers,
  CLIs, REST APIs) through one uniform `call_tool` interface.

Both directions share one contract vocabulary — see
`conventions/api-table.md` §10 (`register_tool`, `list_tools`, `call_tool`,
`register_bridge`, `bridge_event`).

## Bridge patterns

| Pattern | When | Notes |
|---|---|---|
| Embedded MCP server | Other agents/apps must call your app's capabilities | stdio for local, Streamable HTTP for remote |
| MCP client | Your app needs tools from other servers (files, browsers, DBs) | One client, many servers; uniform `call_tool` |
| CLI adapter | Wrapping an existing CLI as a tool | Parse `--json` output; never scrape human text |
| REST/webhook adapter | Talking to web services | OpenAPI → tool definitions; webhooks → `bridge_event` |
| Event bridge | Fire-and-forget cross-app notifications | At-least-once delivery, idempotent handlers |

## MCP essentials

MCP is JSON-RPC 2.0 with a capability handshake:

1. **Initialize:** client sends `initialize` (protocol version, capabilities);
   server responds with its own. Reject mismatched versions loudly.
2. **Tools:** server advertises via `tools/list`; client invokes via
   `tools/call` with `{name, arguments}`. Arguments are validated against the
   tool's JSON Schema **before** the handler runs.
3. **Resources & prompts:** expose read-only data (`resources/list|read`) and
   reusable prompt templates separately from tools — tools *do*, resources
   *show*.

Transports: `stdio` (subprocess, local) or Streamable HTTP (remote). Pick
stdio when the bridge lives on the same machine; HTTP when it crosses one.

**Libraries:** Rust → `rmcp`; TypeScript → `@modelcontextprotocol/sdk`;
Python → `mcp` package. All three follow the same handshake.

## Dev instructions

1. **Contract first.** Write the tool definition before the handler: `name`
   (verb_noun, e.g. `search_files`), one-sentence description stating
   exactly when to use it, JSON Schema for arguments with required fields,
   and the error cases. Review the contract; the handler is then mechanical.
2. **Validate at the boundary.** Schema-validate arguments, clamp numeric
   ranges, reject unknown fields (or explicitly allow them). Never let raw
   caller input reach a shell, SQL string, or file path unchecked.
3. **Keep handlers small and total.** One tool = one job. Time-box execution
   (timeouts), make handlers idempotent where retry is possible, and return
   structured results — data plus a short human summary, not prose dumps.
4. **Version capabilities.** `list_tools` includes a version per tool.
   Additive changes only; rename = new tool + deprecation note on the old.
5. **Authenticate the channel.** stdio bridges inherit process trust; HTTP
   bridges need bearer tokens or mTLS. Log *which principal* called *which
   tool*.
6. **Test with a mock client.** Drive `tools/list` → `tools/call` against a
   scripted client in CI: happy path, schema violations, timeouts, and error
   mapping. Record/replay real sessions for regression.

## Input → Process → Output

**Input:** "Let an external agent query our app's task list and mark tasks
done."

**Process:**
1. Contract: `list_tasks({status?}) -> {tasks[]}`, `complete_task({id}) ->
   {ok}`; errors: `not_found`, `already_done`.
2. Implement an MCP server (stdio) with the two tools; handlers call the
   existing `list`/`write` store functions — no new business logic.
3. Schema validation at the boundary; `id` checked against the store.
4. CI test: mock client lists, completes, re-completes (expects
   `already_done`), calls with bad schema (expects validation error).

**Output:** a `task-bridge` binary speaking MCP on stdio; any MCP-capable
agent can now drive the task list with two audited tools.

## Proven pattern vs avoid pattern

- **Proven:** thin tool handlers over existing domain functions. The bridge
  owns transport, validation, and observability; the domain owns logic. One
  codebase, two front doors (UI + tools).
- **Avoid:** reimplementing business logic inside the bridge ("the MCP
  version of X"). It drifts, duplicates bugs, and doubles the audit surface.
  If a tool needs logic the app doesn't have, add it to the domain first.

## Telemetry, testing & debugging

- **Telemetry:** log every `call_tool` with tool name, principal, duration,
  argument hash (never raw secrets), and outcome. Alert on error-rate spikes
  per tool — a new spike means a bad deploy or an abusive caller.
- **Testing:** contract tests (schema accept/reject), conformance against a
  reference MCP client, replayed sessions in CI, chaos (kill the server
  mid-call → client must surface a clean error).
- **Debugging:** reproduce with the recorded session; enable wire-level
  JSON-RPC logging behind a flag; check handshake first (90% of "it doesn't
  connect" is version/capability mismatch, not logic).

## Security floor

Least privilege per tool (a read-only tool must not import write helpers);
timeouts and rate limits on every bridge; secrets via env/secret mounts,
never in tool arguments or logs; treat all caller input as untrusted —
validate, don't trust.
