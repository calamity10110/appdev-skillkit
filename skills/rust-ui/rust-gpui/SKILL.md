---
name: rust-gpui
description: >
  Build native Rust desktop UIs with GPUI (via the gpui-kit crate): choosing
  components, state ownership, theming, actions/keybindings, async work.
  Use when writing or reviewing GPUI desktop app code.
metadata:
  distilled_from: jfspencer/duet gpui-kit skill (MIT)
  version: "1.0.0"
---

# Rust GPUI UI

GPUI is a Rust-native GPU-accelerated UI framework (Zed's foundation). The
`gpui-kit` crate adds a styled component library on top. Applications depend on
one crate, `gpui-kit`; GPUI itself is reached via `use gpui_kit::*;`, with
layers `gpui_kit::component` (styled), `gpui_kit::base` (unstyled behavior),
`gpui_kit::assets` (icons), `gpui_kit::platform`.

## Rules

1. **Never invent an API.** GPUI's surface changes fast. Search the current
   source (or the component docs) for the real signature — do not translate a
   React/CSS example by analogy. A plausible method that doesn't exist is the
   most common failure mode.
2. **Framework owns behavior, application owns presentation.** Don't put
   colors/sizing/layout in unstyled base components; don't put interaction
   behavior in application styling code.
3. **State ownership is explicit.** Stateless views render once (`RenderOnce`);
   stateful views hold `Entity<State>` created with `cx.new(...)` and pass a
   reference at render time. Decide where each piece of state lives before
   writing the view.
4. **Stable identity for repeated elements.** Lists and repeated elements need
   domain-derived `ElementId`s — never list indexes.
5. **Public API uses builders.** Public component types expose builder chains
   and reader methods, not `pub` fields.

## Core mechanics

- **Components:** `Button`, `Input`, `Select`, `Dialog`, `Sheet`, `Tabs`,
  `Sidebar`, `List`, `DataTable`, `Tree`, `Chart` — construct, set size
  (`.xsmall()` … `.large()`), attach `.on_click` handlers.
- **Theming:** read tokens from `cx.theme()` (`.primary`, `.background`,
  `.foreground`, `.border`, `.muted`) — never hardcode colors.
- **Events/actions/focus:** emit with `cx.emit`, subscribe with `cx.subscribe`,
  declare keybindings with `actions!`, manage focus with `FocusHandle`.
- **Async:** spawn work with `cx.spawn` / background tasks; never block the
  render path on I/O.
- **Layout:** flex-based; measure and scroll with scroll handles; use virtual
  lists for large collections.
- **Testing:** UI integration tests with a test app context; test behavior,
  not pixels.

## Docs

Upstream docs are LLM-friendly: `https://gpui-kit.com/llms.txt` for the index,
per-component pages at `https://gpui-kit.com/component/{name}.md`
(e.g. `button.md`, `data-table.md`). Prefer these over training knowledge.

## When to pick GPUI vs Tauri

GPUI: pure-Rust, native rendering, maximum control and performance — best when
the whole team is Rust and you want a native feel. Tauri (`rust-tauri` skill):
web-tech UI (React/Svelte/…) in a native webview with a Rust core — best when
reusing web components or an existing web frontend.

## Input → Process → Output

Worked example — a settings panel with a name field and a Save button bound
to Cmd+S.

**INPUT:** "Settings view: editable display-name field, Save button, Cmd+S
shortcut, invalid input shows an inline error, uses theme colors."

**PROCESS:**
1. Choose components: `Input` for the field, `Button::primary()` for Save —
   verify each exists in the component docs before writing code.
2. Decide state ownership: the input needs retained state →
   `Entity<InputState>` created once with `cx.new(...)`, passed by reference
   at render; the panel itself is `RenderOnce` (no retained state of its own).
3. Style with theme tokens only (`cx.theme().primary`, `.border`,
   `.foreground`) — no hardcoded hex colors.
4. Declare the save as an action with `actions!`, bind Cmd+S, wire it to the
   same code path as the button click (one behavior, two triggers).
5. Validate in the action handler; surface errors as the input's error state,
   not a dialog.

**OUTPUT:**

```rust
use gpui_kit::*;

struct SettingsView { name: Entity<InputState> }

impl Render for SettingsView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().flex().flex_col().gap_2().p_4()
            .child(Input::new(&self.name).placeholder("Display name"))
            .child(
                Button::new("save").primary().label("Save")
                    .on_click(cx.listener(|this, _, window, cx| {
                        this.save(window, cx);
                    })),
            )
    }
}
```

## Proven pattern vs avoid pattern

**Proven — explicit state ownership + stable identity:** decide up front what
is `RenderOnce` (stateless, rebuilt each render) vs `Entity<State>`
(retained, mutated through handles), and give repeated elements
domain-derived `ElementId`s. This is what keeps re-renders cheap and focus
behavior correct.

**Avoid — inventing component APIs and styling in the wrong layer:** calling
a method that "should" exist (it doesn't — GPUI's surface moves fast) or
putting colors and layout inside unstyled base components. It fails at
runtime or silently diverges from the design system. Do this instead: check
the real signature in the current source or at
`gpui-kit.com/component/{name}.md` first; keep behavior in `gpui-base`,
presentation in `gpui-component` and app code, colors only from `cx.theme()`.

## Telemetry, testing & debugging

- **Test:** UI integration tests with `#[gpui_kit::test]` and a
  `TestAppContext` — drive the real component, assert behavior (value
  committed, action fired), not pixels. Unit-test any pure state-transition
  logic separately.
- **Debug:** add `tracing` events inside action handlers to see what fired
  and with what state; when layout looks wrong, re-read the relevant
  section of the coding guide (flex rules, `ElementId` stability) rather
  than guessing. `gpui-kit.com/llms.txt` is the fastest path to the current
  component API — prefer it over training knowledge.
- **Telemetry:** log user-facing errors through the app's `tracing` setup
  with enough context (which view, which action) to reproduce from the log
  alone; keep render paths free of I/O so frame cost stays predictable.
