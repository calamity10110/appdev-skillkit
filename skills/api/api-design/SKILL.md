---
name: api-design
description: Designing HTTP API surfaces — choosing REST vs GraphQL vs gRPC, resource naming, versioning, pagination, error envelopes, authentication, rate limiting, and idempotency.
---

# API Design

Guidance for designing HTTP API surfaces that stay consistent, evolvable,
and safe to operate. This is platform-neutral; examples use REST-style
HTTP because it is the common case, with a decision framework for when
another protocol fits better.

## Choosing the protocol

| Approach | Choose when | Costs |
|---|---|---|
| **REST** | Public or partner APIs, CRUD-heavy domains, cacheability matters, widest client compatibility | Over/under-fetching, chatty clients, N+1 round trips |
| **GraphQL** | Rich client-driven data needs, many consumer types with different shapes, rapid UI iteration | Query complexity attacks, caching is harder, backend must enforce depth/cost limits |
| **gRPC** | Internal service-to-service, strongly-typed contracts, streaming, polyglot backends | Poor browser support, opaque payloads for debugging, proxy/load-balancer complexity |
| **tRPC / RPC-style** | TypeScript monorepo with a single trusted client | Locks you to one language ecosystem |

Default to REST for anything external unless a concrete pain justifies the
alternatives; do not expose GraphQL publicly without query cost analysis
and depth limiting.

## Resource naming and HTTP semantics

- Nouns for resources, not verbs: `POST /orders`, not `POST
  /createOrder`. Collections are plural; nested resources reflect
  ownership (`/orders/{id}/items`), but flatten beyond two levels.
- Use HTTP methods for their semantics: `GET` reads (safe, idempotent,
  cacheable), `POST` creates or performs non-idempotent actions, `PUT`
  full replace (idempotent), `PATCH` partial update, `DELETE` removes
  (idempotent). Never tunnel state changes through `GET`.
- Return the right status codes: `201` with a `Location` header on
  create, `204` on empty success, `400` for client errors, `401` for
  missing/invalid auth, `403` for forbidden-but-authenticated, `404` for
  unknown resources (not for "not allowed" — that is `403`), `409` for
  conflicts, `422` for semantic validation failures, `429` when rate
  limited. `500` means your bug, never the client's.
- Filter, sort, and sparse-field via query parameters
  (`?status=open&sort=-created&fields=id,name`); keep the grammar
  consistent and documented.

## Versioning

Version the contract, not the URL prefix alone. Options in preference
order: additive evolution with no version bump (new optional fields, new
endpoints); explicit version in the path (`/v1/…`) when breaking changes
are unavoidable; content negotiation (`Accept: application/vnd.api.v2+json`)
for mature public APIs. Never break existing consumers without a version
boundary; deprecate with `Deprecation`/`Sunset` headers and a published
timeline. Additive-only discipline keeps most APIs on one version for a
long time.

## Pagination

Never return unbounded collections. Prefer cursor/keyset pagination on a
stable indexed key (`?cursor=…&limit=…`) — it is stable under inserts and
cheap at depth. Offset pagination (`?page=&per_page=`) is acceptable only
for shallow, low-volume browsing; deep offsets are linear-cost scans.
Return pagination metadata in a consistent envelope (next cursor, has-more)
and cap `limit` server-side.

## Error envelopes (RFC 9457 problem details)

Errors are part of the contract. Return machine-readable problem details:
`type` (URI identifying the problem class), `title`, `status`, `detail`,
and `instance`, with extension fields for validation errors (field-level
`errors` array). Keep one envelope across the whole API. Never leak
internals: stack traces, SQL, and file paths stay in server logs, never in
responses. Validation failures get `422` with per-field messages the
client can render; authentication failures stay generic (`401`/`403`
without distinguishing "user not found" from "wrong password" in ways that
aid enumeration).

## Authentication and authorisation

- Authenticate with short-lived bearer tokens (JWT or opaque) over TLS;
  never accept credentials in query strings. Validate signatures against
  a trusted key, enforce expiry, and rotate signing keys.
- Prefer OAuth2/OIDC for delegated access; use PKCE for public clients,
  validate redirect URIs and state, and scope tokens to least privilege.
  Service-to-service: mutual TLS or signed workload identity beats shared
  static secrets.
- Authorise at the resource level on every request: authenticate first,
  then check ownership/role for the specific object — missing
  function-level checks and IDOR are the most common API flaws. Deny by
  default; log auth decisions for audit.

## Rate limiting and idempotency

- Rate limit at the edge by client identity (API key, user, IP as a
  fallback): return `429` with `Retry-After`, and document quotas. Tighten
  limits on expensive or sensitive endpoints (auth, search, exports).
- Make every mutating endpoint safe to retry: accept client-generated
  idempotency keys on `POST` (store key → result, return the stored result
  on replay), keep `PUT`/`DELETE` naturally idempotent, and design
  handlers to tolerate duplicate delivery from queues and client retries.
- For long operations, return `202 Accepted` with a status endpoint or
  webhook callback rather than holding the connection open; expose
  progress and cancellation where it matters.

## Operability of the surface

- Publish an OpenAPI (or GraphQL/gRPC reflection) description and keep it
  generated from the implementation, not hand-written — drift between
  docs and behaviour is a bug.
- Version your error codes and document every status a client can see.
- Instrument per-endpoint latency, error rate, and payload size; alert on
  error-budget burn, not just 5xx spikes.
- Never log secrets, tokens, or PII at the edge; redact before the access
  log. Treat request logs as a data store with retention and access
  controls.

## Input → Process → Output

**INPUT:** "We need an API to list a user's orders with filters (status, date
range) for the mobile app and admin dashboard."

**PROCESS:**
1. Decide style: reads only, page-based, filters are query params — REST fits;
   GraphQL would be overkill for two fixed clients.
2. Resource naming: `GET /v1/users/{userId}/orders?status=&from=&to=&page[cursor]=`.
3. Versioning: `v1` in path; promise additive-only changes under v1.
4. Pagination: cursor-based (`page[cursor]`, `page[limit]` default 20, max
   100) — offsets would drift as orders arrive.
5. Errors: RFC 9457 `application/problem+json` envelopes, typed `code`
   strings (`order:invalid_status`), never bare 500s for client errors.
6. Auth: OAuth2 bearer JWT; `users/{userId}` authorized against the token
   subject unless admin scope.
7. Operability: generate OpenAPI from the implementation, idempotency key on
   the create-order sibling endpoint.

**OUTPUT:** A versioned endpoint contract with named resources, cursor
pagination, typed problem-detail errors, per-endpoint auth rules, and a
generated OpenAPI doc — ready for contract tests.

## Proven pattern vs avoid pattern

- **Proven — cursor pagination with opaque cursors:** encode the sort key
  (`created_at,id`) in a base64 cursor; every page is stable even as rows are
  inserted. Clients treat it as an opaque token, so the encoding can change.
- **Avoid — offset pagination at scale:** `?page=50&limit=100` skips rows as
  new orders arrive and degrades into full scans. It also invites deep-page
  DoS. Use cursors for anything with writes happening during reads.

## Telemetry, testing & debugging

- **Instrument:** per-endpoint RED metrics (rate, errors, duration) via the
  HTTP middleware (tower, axum `TraceLayer`); payload size histograms; 429
  counts per client to tune rate limits.
- **Test:** contract tests against the OpenAPI spec (schemathesis or
  hand-rolled golden requests); version-compat tests that old clients' payloads
  still parse after additive changes.
- **Debug:** correlate requests with a trace id (`X-Request-Id` in → response
  header out); tail structured logs filtered by endpoint + status class to find
  which filter combination 500s.
