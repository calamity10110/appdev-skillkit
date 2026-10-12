---
name: rust-tauri
description: >
  Build desktop/mobile apps with Tauri v2: Rust core + webview frontend
  (React/Vue/Svelte/etc.), IPC commands and events, capabilities/permissions,
  configuration, and distribution. Use when scaffolding or extending a Tauri app.
metadata:
  distilled_from: dchuk/claude-code-tauri-skills (MIT)
  version: "1.0.0"
---

# Rust Tauri Apps

Tauri builds desktop (and mobile) apps from a **Rust core** and a **webview
shell**: the frontend is HTML/CSS/JS (any web framework) rendered in the
platform's native webview (WebView2 on Windows, WKWebView on macOS, WebKitGTK
on Linux); the backend is Rust for system access, file I/O, and native
features. The two sides communicate over IPC.

## Architecture

- **Core (Rust):** system operations, business logic, plugins. Exposes
  `#[tauri::command]` functions; manages app state with `tauri::State`.
- **Shell (frontend):** UI in your web framework of choice. Calls Rust via
  `invoke('command_name', args)` from `@tauri-apps/api/core`.
- **IPC primitives:**
  - *Commands* — request/response. Frontend `invoke()` → Rust command →
    serialized return. Use for actions and queries.
  - *Events* — fire-and-forget, bidirectional. Rust emits with
    `app.emit("event", payload)`; frontend listens with
    `listen("event", handler)`. Use for notifications and state changes.
- **Capabilities/permissions:** Tauri v2 gates every API behind capability
  files — declare exactly which windows get which permissions. Default-deny;
  never grant broad permissions to work around a missing capability.

## Project setup

- Scaffold with the Tauri CLI (`create-tauri-app`); keep frontend and
  `src-tauri/` in one repo.
- Configure in `tauri.conf.json`: window defaults, bundle identifiers,
  security (CSP), updater.
- Frontend framework choice is free (React, Svelte, Vue, Solid, or Rust-WASM
  frameworks like Leptos/Yew/Dioxus via trunk) — the IPC contract is the
  same.

## Patterns

- **Commands:** small, typed, fallible (`Result<T, String>` or a serializable
  error type). Validate inputs at the command boundary.
- **State:** `app.manage(MyState::new())`, inject with `State<'_, MyState>`.
- **Long-running work:** run in `tauri::async_runtime::spawn`, report progress
  via events — never block a command on slow I/O.
- **Security:** treat the frontend as untrusted input; validate in Rust.
  Enable CSP; scope file-system and shell access narrowly.
- **Distribution:** platform bundles via `tauri build`; sign binaries
  (required on macOS/Windows); keep binary size in check by trimming unused
  features.

## When to pick Tauri vs GPUI

Tauri: reuse web UI skills/components, cross-platform with one codebase,
fastest path to a polished UI — at the cost of a webview runtime. GPUI
(`rust-gpui` skill): pure Rust, native rendering, no web stack — at the cost
of a smaller component ecosystem.

## Input → Process → Output

Worked example — the frontend needs a native "Open file" dialog and the
selected path back in the UI.

**INPUT:** "Add an 'Open file' button. It must open the OS native file
dialog, return the chosen path to the frontend, and show an error toast if
the user cancels or the dialog fails."

**PROCESS:**
1. Write a `#[tauri::command]` `pick_file(app: AppHandle) ->
   Result<Option<String>, String>` using the dialog plugin; cancellation is
   `Ok(None)`, real failures are `Err`.
2. Grant the capability: allow the `dialog` plugin **only for the main
   window** in the capability file — default-deny everything else.
3. Frontend: `invoke('pick_file')` from `@tauri-apps/api/core`; `null`
   → "no file chosen" state; string → display path; thrown error → toast.
4. Unit-test the command as a plain Rust function (no webview needed).

**OUTPUT:**

```rust
#[tauri::command]
async fn pick_file(app: tauri::AppHandle) -> Result<Option<String>, String> {
    use tauri_plugin_dialog::DialogExt;
    app.dialog()
        .file()
        .blocking_pick_file()          // native dialog; None = cancelled
        .map(|f| f.into_path().map(|p| p.to_string_lossy().into_owned()))
        .map_err(|e| format!("file dialog failed: {e}"))
}
```

```typescript
import { invoke } from '@tauri-apps/api/core';
const path = await invoke<string | null>('pick_file');
if (path === null) { /* user cancelled — quiet state, not an error */ }
```

## Proven pattern vs avoid pattern

**Proven — small typed commands + default-deny capabilities:** commands take
serializable args and return `Result<T, E>` with a serializable error; every
plugin API is gated by a capability file naming exactly which windows may use
it. The IPC contract stays reviewable and the attack surface stays minimal.

**Avoid — broad permissions and blocking commands:** granting `fs:allow-*`
to "make it work" and running slow I/O directly in a command handler. It
fails because any frontend compromise becomes full filesystem access, and a
blocked command freezes the IPC bridge for every other call. Do this instead:
scope each capability to the window and plugin that needs it; run slow work
in `tauri::async_runtime::spawn` and report progress back with events
(`app.emit("copy-progress", pct)`), keeping commands fast and non-blocking.

## Telemetry, testing & debugging

- **Logs:** `tauri-plugin-log` routes Rust `log`/`tracing` output to the
  frontend console, stdout, and a rotating file — one place to read both
  sides. Set `RUST_LOG=debug` in dev; keep release logs at `info` with
  `warn` for IPC errors.
- **Debug the UI:** debug builds enable the webview devtools — right-click →
  Inspect gives you the real DOM, console, and network for the shell.
  Debug the bridge by logging command inputs/outputs at the Rust boundary.
- **Test:** commands are plain Rust functions — unit-test them without a
  webview (inject a fake `AppHandle` or factor logic into a testable fn).
  Frontend: mock `invoke` in component tests; reserve full end-to-end runs
  for critical user journeys.
- **Telemetry:** emit structured events for user-visible outcomes
  (dialog opened/cancelled/completed) so product behavior is measurable
  without reading raw logs.
