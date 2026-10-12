---
name: rust-axum-service
description: Build an Axum web service with a hexagonal (ports-and-adapters) layout. Use when structuring a new Rust API, separating domain logic from Axum handlers and database code, wiring shared state, adding tower middleware, mapping errors to HTTP responses, handling graceful shutdown, or testing service layers.
---

# Rust Axum Service — Hexagonal Layout

Organize the service so business rules never import web-framework or database crates. Dependencies point inward: web and persistence adapters depend on the domain, never the reverse.

## Layout

```
src/
├── domain/          # entities + validation + repository ports (no axum, no sqlx)
├── application/     # use cases / services orchestrating domain logic
├── infrastructure/
│   ├── web/         # axum handlers, DTOs, routes
│   └── persistence/ # repository trait implementations (sqlx, in-memory, ...)
└── main.rs          # dependency wiring only
```

### domain/entities

- Plain structs with constructor validation returning `Result`.
- Invariants live here (e.g. empty email rejected), so every layer above inherits them.
- Business rules (state transitions, calculations) are methods on the entity.
- Zero dependencies on frameworks, databases, or serialization crates where avoidable.

### domain/repositories — the ports

- Async traits (`Send + Sync`) defining what the domain needs:
  `find_by_id`, `save`, `delete`, and any query the use cases require.
- Traits name domain concepts, not storage concepts — no "query_by_index" leaking through.
- For Rust <1.75-era async traits or trait objects, use the `async-trait` crate.

### application — use cases

- Structs holding `Arc<dyn RepositoryPort>`; implement cross-entity workflow rules
  (uniqueness checks, authorization gates, state transitions).
- One method per meaningful operation; keep orchestration here, not in handlers.
- Return domain errors; translation to HTTP happens at the web edge.

### infrastructure/web — adapters

- Handlers stay thin: extract inputs, call the use case, map the result to a response.
- DTOs (`*Request` / `*Response`) translate between JSON and domain types;
  never leak domain internals into JSON shapes.
- Route tables are assembled in one place (a `create_router(state)` function).

### main.rs — wiring only

- Constructs the pool, adapters, services, and router, then serves.
- Reads config/secrets from the environment here, once.
- No business logic in `main`.

## State sharing

- Bundle dependencies into one `AppState` struct and hand it to the router with `.with_state(state)`.
- Handlers pull it via `State(state): State<AppState>`.
- Hold services as `Arc<dyn Trait>` so they are cheaply cloneable across tasks
  (the router requires `Clone` state anyway).
- For mutable shared config use `Arc<RwLock<_>>`; prefer read-heavy locking.
- Never hold a lock across an `.await` — drop the guard before awaiting I/O
  to avoid deadlocks and starving the runtime.

## Extractors and handlers

- `State` — services, config.
- `Path` — route params (parses and validates, e.g. `Path<Uuid>`, before your code runs).
- `Query` — query strings; `Json` — request bodies; `HeaderMap` — headers.
- Return `impl IntoResponse` with explicit `StatusCode`s:
  - 200/201 for success, 204 for delete with no body,
  - 400 for validation errors, 404 when a resource is missing,
  - 500 for internal failures.
- Prefer signatures that make invalid states unrepresentable —
  extractor parsing is validation you don't have to write.

## Error-to-HTTP mapping

- Define one error enum with `thiserror` covering domain, infra, and not-found cases.
- Implement `IntoResponse` for it so every handler funnels errors through one mapping:
  - validation / user errors → 400 with a clear message,
  - not found → 404,
  - unexpected (DB, IO) → 500 with a generic message.
- Log the real error server-side with `tracing`; never leak internals to the client.
- Keep error responses shape-consistent (`{"error": "..."}`) across endpoints.

## Tower middleware

Compose cross-cutting behavior with `.layer(...)`:

- `TraceLayer` (tower-http) for request/response logging.
- `CorsLayer` with explicit allowed origins — wildcard only for local dev.
- `RequestBodyLimitLayer` to cap body size on public endpoints.
- Auth via `middleware::from_fn`: inspect the `Authorization` header,
  reject early with 401, stash validated claims in request extensions
  for handlers to read.
- Rate limiting (token-bucket style) on public endpoints.

Ordering matters: put cheap, reject-early layers (auth, rate limit, body limit)
outside expensive ones (tracing, compression) so rejected requests cost little.

## Graceful shutdown

- Serve with `axum::serve` combined with `with_graceful_shutdown`,
  listening for SIGTERM/SIGINT.
- On signal: stop accepting new connections, drain in-flight requests
  within a timeout, then close the connection pool.
- Only exit the process after the drain completes — killed deploys
  should not drop mid-request work.

## Concurrency patterns

- `Arc` for shared ownership of services/repos across handlers and spawned tasks.
- `tokio::sync::Mutex` / `tokio::sync::RwLock` for state touched inside async code
  (they yield instead of blocking threads); `std::sync` variants only for data
  never held across `.await`.
- `tokio::sync::mpsc` channels for background jobs and domain events —
  prefer message passing over shared mutable state where feasible.
- `parking_lot` locks when uncontended-lock performance matters;
  `OnceLock` for one-time config initialization.
- Keep critical sections tiny; never nest locks in inconsistent order;
  profile real contention before optimizing.

## Testing per layer

- **Domain** — plain unit tests, no mocks. Assert constructors accept/reject
  edge cases and that entity methods enforce invariants.
- **Application** — in-memory or mock repository behind the trait
  (a `HashMap`-backed struct is enough). Assert use-case rules:
  duplicates rejected, transitions valid, authorization enforced.
  Use `#[tokio::test]` for async tests.
- **Persistence adapters** — integration tests against a real test database
  (e.g. `#[sqlx::test]` spinning up a fresh DB per test).
- **HTTP layer** — drive the real `Router` with `tower::ServiceExt::oneshot`,
  asserting status codes and response shapes for success,
  validation failure, and not-found paths.

## Security floor

- Validate at domain boundaries; parameterize all SQL (sqlx macros do this).
- Sanitize error output; read secrets from the environment, never hardcode.
- Auth in middleware, authorization checks in use cases.
- Rate-limit public endpoints; enable TLS in production; keep dependencies updated.

## Dependencies (typical)

axum, tokio (full), tower, tower-http (cors, trace, limit), serde/serde_json,
thiserror (domain errors), anyhow (app-level), async-trait, sqlx (tokio runtime),
tracing, uuid.

## Input → Process → Output

Worked example — `POST /orders {"item_id": "…", "qty": 2}` creates an order.

**INPUT:** "Accept an order request, reject qty ≤ 0 with 400, reject unknown
items with 404, persist the order, return 201 with the order JSON."

**PROCESS:**
1. **Domain:** `Order::new(item_id, qty)` validates qty > 0, returns a typed
   `DomainError`. Declare the port: `trait OrderRepo: Send + Sync` with
   `save` and `find_item`.
2. **Application:** `PlaceOrder` use case holds `Arc<dyn OrderRepo>`; it
   builds the entity, checks the item exists, saves, returns the order.
3. **Web adapter:** `CreateOrderRequest` DTO (serde) → validated domain types;
   handler calls the use case, maps the one error enum via `IntoResponse`;
   `OrderResponse` DTO for the 201 body. Route in `create_router(state)`.
4. **Wire:** `AppState { place_order: Arc<PlaceOrder> }` via `.with_state`;
   auth/rate-limit/body-limit middleware outside `TraceLayer`.
5. **Test each layer** (domain unit, use-case with HashMap fake repo,
   `oneshot` router test asserting 201/400/404).

**OUTPUT (handler):**

```rust
async fn create_order(
    State(state): State<AppState>,
    Json(req): Json<CreateOrderRequest>,
) -> Result<impl IntoResponse, AppError> {
    let order = state.place_order.execute(req.item_id, req.qty).await?;
    Ok((StatusCode::CREATED, Json(OrderResponse::from(order))))
}
```

## Proven pattern vs avoid pattern

**Proven — one error enum, one `IntoResponse`:** a single `AppError` with
`thiserror` variants, mapped to status codes in one place. Handlers stay
three lines; every endpoint returns the same `{"error": "…"}` shape; real
details go to `tracing::error!` server-side.

**Avoid — framework and SQL leaking into handlers:** handlers that run SQL,
validate business rules, or serialize domain structs directly. It fails
because every endpoint re-implements the same rules inconsistently and a
schema change ripples through JSON contracts. Do this instead: validation in
`domain` constructors, persistence behind the port trait, DTOs at the web
edge — handlers only extract, call, and map.

```rust
// bad: SQL + business rule in the handler
let row = sqlx::query!("INSERT INTO orders ...").execute(&pool).await?;
if req.qty <= 0 { return Err(...) } // rule duplicated per endpoint

// good: handler delegates; rule lives in Order::new, SQL in the adapter
let order = state.place_order.execute(req.item_id, req.qty).await?;
```

## Telemetry, testing & debugging

- **Instrument:** `TraceLayer::new_for_http()` from tower-http logs every
  request/response; add a request-id layer (`SetRequestIdLayer` +
  `PropagateRequestIdLayer`) and include the id in `tracing` spans via
  `#[tracing::instrument(skip(state), fields(request_id))]`. Emit metrics
  (latency histograms, 5xx counters) per route.
- **Test:** domain unit tests (constructors accept/reject); application tests
  with a `HashMap`-backed fake repo; persistence tests with `#[sqlx::test]`
  on a fresh DB; HTTP tests driving the real `Router` with
  `tower::ServiceExt::oneshot` asserting status + body for 201/400/404.
- **Debug:** `RUST_LOG=debug,tower_http=trace` to see extractor/middleware
  behavior; `#[axum::debug_handler]` on a handler surfaces extractor
  signature mismatches at compile time. Reproduce a bad request with a
  failing `oneshot` test before touching code; never debug against a live DB.
