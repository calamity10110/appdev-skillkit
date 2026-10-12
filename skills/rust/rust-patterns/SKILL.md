---
name: rust-patterns
description: >
  Production Rust patterns distilled for writing, reviewing, and refactoring
  Rust code. Use when writing new Rust functions, modules, or crates;
  designing library APIs; adding error handling or async Tokio code; or
  reviewing for ownership, performance, and correctness issues. Covers error
  handling (thiserror/anyhow), ownership & borrowing, async with Tokio, API
  design, anti-patterns, testing, clippy tooling, and key crates.
metadata:
  distilled_from: leonardomso/rust-skills (MIT)
  version: "1.0.0"
  rust_edition: 2024
---

# Rust Patterns

Condensed production guidance for idiomatic, correct, fast Rust. Prefer the
cheapest construct that does the job: borrow over clone, `Result` over panic,
`thiserror` over ad-hoc strings, iterators over indexing.

## 1. Error handling

**Libraries: typed errors with thiserror.** Library crates should return
named, matchable error enums so callers can handle specific cases. Derive the
`Error` trait with `thiserror`, keep messages lowercase without trailing
punctuation, and wire causes with `#[from]` (auto-conversion) or `#[source]`
(chain without conversion):

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum StoreError {
    #[error("record not found: {id}")]
    NotFound { id: u64 },
    #[error("database failure")]
    Db(#[from] sqlx::Error),        // From<sqlx::Error> for free
    #[error("cache failure")]
    Cache(#[source] CacheError),    // keeps the chain, manual conversion
}
```

**Applications: context-rich errors with anyhow.** Binaries rarely need to
match error variants — they need to know *where* and *why* something failed.
Return `anyhow::Result`, attach context at each boundary, and propagate with
`?`. `bail!` and `ensure!` replace manual `return Err(...)` boilerplate:

```rust
use anyhow::{Context, Result};

fn load_config(path: &Path) -> Result<Config> {
    let raw = std::fs::read_to_string(path)
        .with_context(|| format!("reading config at {}", path.display()))?;
    toml::from_str(&raw).context("parsing config file")
}
```

Print full chains with `{:#}`. Hybrid is fine: `thiserror` types at the
library's public boundary, `anyhow` wrapping them inside the application.

**Rules:**
- Return `Result<T, E>` for anything recoverable; panic only on true
  programmer errors (broken invariants), never on user input or I/O.
- `?` for propagation everywhere. `expect("...")` only for invariants that
  would signal a bug; never `unwrap()` in production paths.
- Implement `From<E>` (or use `#[from]`) so `?` converts automatically.
- Document fallible behaviour in `# Errors` doc sections; log the *full*
  source chain once per error, never silently swallow errors (`let _ = ...`
  on a fallible result is a code smell).
- Never return `String` or `Box<dyn Error>` as a public error type — both
  are unmatchable and untestable.

## 2. Ownership & borrowing

Default to borrowing; pay for ownership only when you need it.

```rust
fn summarize(title: &str, rows: &[Row]) -> Summary  // &str, &[T] — never &String / &Vec<T>
```

- **Borrow by default**: parameters take `&str`/`&[T]`; don't clone what you
  can borrow. Move large values instead of copying; wrap them in `Box` if
  moves become expensive.
- **Conditional ownership**: `Cow<'a, str>` (or `Cow<'a, [T]>`) when a value
  is usually borrowed but sometimes transformed.
- **`Copy` for small plain types** (a couple of words); explicit `Clone`
  when copying has real cost.
- **Shared ownership**: `Rc<T>` single-threaded; `Arc<T>` across threads.
  Interior mutability: `RefCell<T>`/`Cell<T>` single-threaded,
  `Mutex<T>`/`RwLock<T>` across threads (prefer `RwLock` when reads dominate).
  Prefer `std::sync::Mutex` in async code unless the critical section must
  span an `.await` — and redesign before accepting that.
- **`mem::take` / `mem::replace`**: move a value out of `&mut` without
  cloning when a `Default` placeholder is acceptable.
- **Smart pointers for layout**: box large enum variants to shrink the enum,
  `Box<[T]>` for fixed-size heap data, `SmallVec`/`ArrayVec` for usually-small
  collections, `with_capacity()` when the size is known, and clear-and-reuse
  collections inside loops rather than reallocating.

## 3. Async with Tokio

- **Runtime setup**: `#[tokio::main]` with a tuned multi-threaded runtime
  (size the worker count to the workload); keep one runtime per process.
- **Never block the runtime**: CPU-heavy work, crypto, compression, and
  synchronous I/O go in `tokio::task::spawn_blocking` (consider rayon
  *inside* `spawn_blocking` for parallel CPU work). Use `tokio::fs` instead
  of `std::fs` inside async code.
- **No locks across `.await`**: release guards before suspending — extract
  or clone what you need, do the async work, then re-lock for a quick
  update. Holding a `Mutex`/`RwLock` across an await stalls every other
  waiter and can deadlock.
- **Channels pick by shape**:
  - `mpsc` — async queue between tasks; **bounded** (backpressure over
    unbounded memory growth)
  - `oneshot` — single request/response
  - `broadcast` — pub/sub, every receiver gets every message
  - `watch` — latest-value sharing with many observers
- **Combinators**: `join!` / `try_join!` for concurrent independent futures
  (early-exit on first error with `try_join!`); `select!` to race futures;
  `JoinSet` for a dynamic bag of spawned tasks. Branches inside `select!`
  must be cancellation-safe; use `CancellationToken` for graceful shutdown.
- **Traits**: native `async fn` in traits (stable since 1.75) instead of
  `async_trait`; bound callbacks with `AsyncFn*` rather than
  `Fn() -> Fut` + `Future` pairs. Clone `Arc` handles *before* await points
  instead of holding borrowed references across suspensions.

```rust
async fn good_update(state: &Mutex<State>) {
    let config = state.lock().await.config.clone(); // clone out, drop guard
    let result = tokio::task::spawn_blocking(move || heavy(config))
        .await
        .expect("blocking task panicked");
    state.lock().await.result = result; // quick re-lock
}
```

## 4. API design

- **Builder pattern** for constructors with several options: fluent
  consuming setters, `impl Into<String>` for flexible text args, defaults
  inside `build()`, and `#[must_use]` on the builder and its methods so a
  half-built value can't be silently dropped.
- **Implement `From`, not `Into`**: `impl From<T> for U` hands you
  `Into<U> for T` automatically. Reach for `TryFrom` for fallible
  conversions, `FromStr` to plug into `str::parse`, and `AsRef<T>` when a
  function only needs to borrow the inner data.
- **Newtypes for safety**: wrap primitives in semantic types
  (`UserId(u64)`, `Email(String)`); use enums for mutually exclusive states;
  `#[non_exhaustive]` on public enums/structs for forward compatibility;
  sealed traits when outsiders may use but not implement a trait.
- **Parse, don't validate**: at system boundaries, convert raw input into
  validated types immediately (e.g. `#[serde(try_from = "Raw")]`) so
  downstream code never re-checks invariants.
- **Typestate** when state-machine misuse must be a compile error, not a
  runtime one; extension traits to add methods to types you don't own.
- **Annotations that pay**: `#[must_use]` on types/functions where ignoring
  the result is likely a bug; `Default` for types with a sensible zero;
  standard traits (`Debug`, `Clone`, `PartialEq`, `Serialize`) on public
  types; `Deref` only for genuine smart pointers, not as a shortcut.
- **Serde**: `rename_all` to match the wire convention, `#[serde(default)]`
  for backward-compatible fields, `skip_serializing_if` for empties,
  `deny_unknown_fields` when strictness matters, explicit enum tagging
  (externally / internally / adjacently / untagged), and keep `serde` an
  optional feature for library crates rather than a hard dependency.

## 5. Anti-patterns checklist

- No `unwrap()` / `expect()` on recoverable paths in production code; no
  silent error swallowing; no panics for expected failures.
- No gratuitous clones — borrow, slice, or `Cow` first.
- No `&String` / `&Vec<T>` parameters; no strings where an enum or newtype
  carries the meaning ("stringly typed").
- No locks held across `.await`; no `Box<dyn Trait>` where `impl Trait`
  or generics keep static dispatch.
- No indexing when iterators do the job; no intermediate `.collect()`s;
  `entry()` for map insert-or-update; prefer `extend` over repeated pushes.
- No `format!()` in hot paths or where a literal/`write!` into a buffer
  works; don't profile-blindly optimize — measure first.
- No over-abstraction: don't genericize before the second use case; macros
  only where functions/generics can't express it.

## 6. Testing & tooling

- **Layout**: unit tests in `#[cfg(test)] mod tests` per module;
  integration tests under `tests/`; async tests with `#[tokio::test]`;
  doctests double as runnable documentation examples.
- **Practice**: descriptive names stating behaviour, arrange/act/assert
  structure, property-based tests (proptest) for invariants, mockall or
  hand-rolled trait fakes for boundaries, RAII fixtures (`Drop`) for
  cleanup, `insta` snapshots for serialized/complex output, `criterion`
  with `black_box` for benchmarks, `loom` for concurrent code.
- **Lints**: `#![deny(clippy::correctness)]` plus `warn` on suspicious,
  style, complexity, and perf groups; pedantic/nursery lints enabled
  selectively; `missing_docs` for public items; workspace-level lint config
  so every crate enforces the same floor; `cargo fmt --check` in CI.
- **CI basics**: `cargo test`, `cargo clippy`, `cargo fmt --check`,
  `cargo miri test` for any crate with `unsafe`; declare MSRV
  (`rust-version`) and keep features strictly additive.

## 7. Key crates

| Crate | Purpose |
|---|---|
| tokio | Async runtime, tasks, timers, sync primitives |
| axum | HTTP services on Tower; routers, extractors, middleware |
| serde / serde_json | Serialization framework and JSON (de)serialization |
| anyhow | Ergonomic app-level error handling with context chains |
| thiserror | Derive-based typed errors for library APIs |
| tracing | Structured, span-aware logging and diagnostics |
| sqlx | Async SQL with compile-time query checking |
| clap | CLI argument parsing (derive API) |
| reqwest | Async HTTP client |
| tonic | gRPC client/server implementation |
| rayon | Data-parallel CPU work via parallel iterators |
| criterion | Statistical benchmarking harness |

## When to reach for more

This is the high-signal core. For deeper cuts see the source library:
`unsafe` semantics (The Rustonomicon), proc-macro authoring, const
generics, SIMD/portable-vectorization, PGO/LTO release tuning, and
full project-structure conventions.

## Input → Process → Output

Worked example — a CLI that fetches three config endpoints concurrently, each
with a 5s timeout, failing loudly with context.

**INPUT:** "Fetch three config URLs concurrently. On failure the error must
name the URL and say what stage failed (connect, timeout, parse)."

**PROCESS:**
1. Define a `Config` struct with a validated constructor — parse the JSON
   into `Config` once, reject bad data there, never downstream.
2. Write one `async fn fetch_config(url: &str) -> anyhow::Result<Config>`,
   attaching `.context()` at each fallible step (connect, read, parse).
3. Spawn one task per URL with `tokio::spawn`, wrap each in
   `tokio::time::timeout`, collect with a `JoinSet`.
4. `main` returns `anyhow::Result<()>`; on error print the chain with `{:#}`.

**OUTPUT:**

```rust
use anyhow::{Context, Result};

#[tracing::instrument]
async fn fetch_config(url: &str) -> Result<Config> {
    let body = tokio::time::timeout(
        Duration::from_secs(5),
        reqwest::get(url).await.context("sending request")?,
    )
    .await
    .with_context(|| format!("request to {url} timed out"))?
    .text()
    .await
    .with_context(|| format!("reading body from {url}"))?;
    Config::parse(&body).with_context(|| format!("parsing config from {url}"))
}
```

## Proven pattern vs avoid pattern

**Proven — validated constructors (parse, don't validate):** put invariants in
the constructor so every consumer inherits them. This makes invalid states
unrepresentable and concentrates edge-case tests in one place.

```rust
pub struct Port(u16);
impl Port {
    pub fn new(raw: u16) -> Result<Self, ConfigError> {
        (1..=65535).contains(&raw).then_some(Self(raw))
            .ok_or(ConfigError::BadPort(raw))
    }
}
```

**Avoid — `unwrap`/`expect` on production paths and clone-everywhere:**
`unwrap` turns a recoverable error into a crash at 3am; defensive `.clone()`
sprinkled to satisfy the borrow checker hides ownership bugs and costs
performance. Do this instead: propagate with `?` (plus `.context()`), and
borrow with `&str`/`&[T]` parameters — clone only at a true ownership
boundary.

```rust
// bad: crashes on bad input, clones pointlessly
let name = map.get(&key).unwrap().clone();

// good: error is data, borrow is free
let name = map.get(&key).ok_or(ConfigError::Missing(key.clone()))?;
```

## Telemetry, testing & debugging

- **Instrument:** `#[tracing::instrument]` on public async fns gives you
  spans with arguments for free; emit `tracing::info!/warn!` at decision
  points. Set `RUST_LOG=info,my_crate=debug` per-crate to keep noise down.
- **Metrics:** the `metrics` crate for counters/gauges/histograms
  (requests, queue depth, latencies) — cheap, and exportable later.
- **Test:** unit tests per module for constructors and edge cases; `proptest`
  for invariants ("parse never panics on arbitrary bytes"); `#[tokio::test]`
  for async paths; `insta` snapshots for serialized output; `criterion` with
  `black_box` before claiming a perf win; `loom` for code with real
  concurrency; `cargo miri test` for anything with `unsafe`.
- **Debug:** reproduce first with a failing test, then bisect. For async
  stalls, `tokio-console` shows live tasks and where they're parked
  (needs the `tokio_unstable` cfg + console-subscriber). `RUST_BACKTRACE=1`
  on panics; `cargo clippy` before you ask a human.
