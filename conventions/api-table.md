# Reusable API / Function Name Conventions

A predefined vocabulary of function and API names to use **throughout the
development process** — system, UI, input devices, storage, networking, and
bridges. When an AI agent writes code in this project, it should reach for
these names first instead of inventing new ones. Consistent names make code
searchable, reviewable, and portable across languages.

**Language mapping:** names below are `snake_case`. In TypeScript/JavaScript
use `camelCase` equivalents (`load_config` → `loadConfig`); in Rust keep
`snake_case`. Semantics stay identical.

## Naming rules

- Verbs for actions: `start`, `stop`, `send`, `load`, `save`.
- Nouns for accessors: `config()`, `state()`, `version()`.
- `on_` prefix for event handlers: `on_key`, `on_message`.
- `try_` prefix for fallible attempts that return a recoverable result.
- `is_` / `has_` prefix for boolean predicates.
- No abbreviations except standard ones (`config`, `msg`, `ctx`, `tx`).
- Async variants keep the same name; the signature carries async-ness.

## 1. System / app lifecycle

| Name | Sketch | Use when |
|---|---|---|
| `init` | `init() -> Result<()>` | One-time startup: parse args, set up logging, load config |
| `configure` | `configure(opts: Options)` | Apply runtime options before start |
| `start` | `start() -> Result<()>` | Begin serving / running the main loop |
| `shutdown` | `shutdown() -> Result<()` | Graceful stop: drain work, flush telemetry, close handles |
| `health_check` | `health_check() -> Health` | Liveness/readiness probes |
| `version` | `version() -> &str` | Build version string |
| `reload` | `reload() -> Result<()>` | Hot-reload config without restart |

## 2. Configuration

| Name | Sketch | Use when |
|---|---|---|
| `load_config` | `load_config(path) -> Result<Config>` | Read + parse + validate config file |
| `get_config` | `get_config() -> &Config` | Access the loaded config |
| `watch_config` | `watch_config(cb)` | Reload on file change |

## 3. Logging / telemetry

| Name | Sketch | Use when |
|---|---|---|
| `log_event` | `log_event(level, msg, fields)` | Structured log line |
| `trace_span` | `trace_span(name) -> Guard` | Timed operation span |
| `record_metric` | `record_metric(name, value, labels)` | Counter/gauge/histogram |
| `flush_telemetry` | `flush_telemetry()` | Push buffered logs/metrics on shutdown |

## 4. Errors

| Name | Sketch | Use when |
|---|---|---|
| `AppError` | enum / class | Single crate-level error type |
| `to_problem` | `to_problem(err) -> ProblemDetails` | Map errors to RFC 9457 HTTP responses |
| `report_error` | `report_error(err, ctx)` | Send to error tracking with context |

## 5. UI (screens, components, views)

| Name | Sketch | Use when |
|---|---|---|
| `render` | `render(ctx) -> View` | Build/refresh the visible output |
| `update` | `update(msg) -> Effect` | Handle a message, produce new state |
| `mount` | `mount(parent)` | Attach a component to the tree |
| `unmount` | `unmount()` | Detach + release resources |
| `on_event` | `on_event(evt)` | Component-level event handler |
| `set_state` / `get_state` | `set_state(patch)` | Explicit state transitions |
| `invalidate` | `invalidate()` | Mark dirty, schedule re-render |
| `focus` / `blur` | `focus()` | Keyboard focus management |

## 6. Input devices

| Name | Sketch | Use when |
|---|---|---|
| `poll_input` | `poll_input() -> Vec<InputEvent>` | Game-loop style input polling |
| `on_key` | `on_key(evt: KeyEvent)` | Keyboard handler |
| `on_pointer` | `on_pointer(evt: PointerEvent)` | Mouse/touch-pointer handler |
| `on_touch` | `on_touch(evt: TouchEvent)` | Multi-touch gestures |
| `on_gamepad` | `on_gamepad(evt: GamepadEvent)` | Gamepad handler |
| `on_voice` | `on_voice(evt: VoiceEvent)` | Mic/STT input handler |
| `bind_action` | `bind_action(name, trigger)` | Map input trigger → named action |
| `unbind_action` | `unbind_action(name)` | Remove a binding |
| `get_input_state` | `get_input_state() -> InputState` | Current device state snapshot |

Input events normalize to one `InputEvent` enum: `Key`, `Pointer`, `Touch`,
`Gamepad`, `Voice`, `System` — handlers switch on the variant, never on raw
device APIs.

## 7. Storage

| Name | Sketch | Use when |
|---|---|---|
| `open_store` | `open_store(path) -> Result<Store>` | Open DB / file store |
| `read` / `write` / `delete` | `read(key) -> Result<Value>` | CRUD on the store |
| `list` | `list(prefix) -> Vec<Key>` | Key listing / scans |
| `migrate` | `migrate(to_version)` | Schema migrations (expand→migrate→contract) |
| `transaction` | `transaction(f) -> Result<T>` | Atomic multi-op block |

## 8. Network / IPC

| Name | Sketch | Use when |
|---|---|---|
| `connect` / `disconnect` | `connect(addr) -> Result<Conn>` | Manage connections |
| `send` / `on_message` | `send(msg)` | Message passing |
| `subscribe` / `unsubscribe` | `subscribe(topic, cb)` | Pub/sub |
| `emit` | `emit(event, payload)` | Fire-and-forget event |
| `invoke` | `invoke(cmd, args) -> Result<Value>` | Request/response call (Tauri-style IPC, tool calls) |
| `listen` | `listen(event, cb)` | Subscribe to pushed events |

## 9. Background tasks

| Name | Sketch | Use when |
|---|---|---|
| `spawn_task` | `spawn_task(name, fut) -> TaskId` | Launch background work |
| `cancel_task` | `cancel_task(id)` | Cooperative cancellation |
| `task_status` | `task_status(id) -> Status` | Query task state |

## 10. Bridges / tools (see `bridges/mcp-tools-bridge`)

| Name | Sketch | Use when |
|---|---|---|
| `register_tool` | `register_tool(def)` | Expose a capability to other apps |
| `list_tools` | `list_tools() -> Vec<ToolDef>` | Advertise capabilities |
| `call_tool` | `call_tool(name, args) -> Result<Value>` | Invoke a tool (local or remote) |
| `register_bridge` | `register_bridge(cfg)` | Connect to an external app/service |
| `bridge_event` | `bridge_event(name, payload)` | Cross-app event |
