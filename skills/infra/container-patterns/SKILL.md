---
name: container-patterns
description: Building and reviewing container images and local container setups — multi-stage Dockerfiles, Compose for dev parity, health checks, secrets handling, and image security basics.
---

# Container Patterns

Standards for building, securing, and running containers with the OCI/Docker
toolchain. This covers the Dockerfile, image hygiene, and local development
with Compose; full Kubernetes platform engineering and pipeline wiring are
separate concerns.

## Multi-stage Dockerfiles

- Build/SDK tooling never ships in the runtime image. Separate stages for
  dependency install, build, and runtime; the final stage starts from a
  minimal base and receives only built artefacts via `COPY --from=<stage>`.
  A single-stage image carrying compilers or dev dependencies is a defect.
- Order layers cache-friendliest-first: dependency manifests and installs
  before source `COPY`, so code edits don't re-trigger dependency
  resolution. Collapse related `RUN` steps and clean package caches in the
  same layer; use BuildKit cache mounts for package caches.
- Use exec form (`["executable","arg"]`) for `ENTRYPOINT`/`CMD` so the
  process is PID 1 and receives `SIGTERM` directly for graceful shutdown;
  shell form swallows signals.

## Base images and pinning

- Pin the base image to a specific tag, and to a digest (`@sha256:…`) for
  anything reproducibility-critical. Never `:latest` in a committed
  Dockerfile — it is non-reproducible base drift.
- Smallest viable runtime base first: distroless / chiselled for compiled
  or runtime-only apps (no shell, no package manager — smallest CVE
  surface), then `-slim` when you need a shell or glibc-linked native
  deps, then Alpine only when the stack is musl-safe and verified (musl
  breaks some native dependencies and DNS edge cases).
- One concern per image; no debugging tools (`curl`, `netcat`, compilers)
  in the runtime stage.

## Non-root and .dockerignore

- Run as non-root: an explicit `USER` with a numeric UID, never the
  implicit root. Set ownership on `COPY --chown=…` where needed.
- A `.dockerignore` always exists and excludes `.git`, secrets, build
  output, and dependency directories — it shrinks the build context and
  prevents leaking files into the image.

## Secrets handling

- No secrets in the image: not in layers, not in `ENV`, not in
  `--build-arg` — layer history is forever, even after `rm`. Build-time
  secrets via BuildKit `--mount=type=secret`; runtime secrets via the
  orchestrator's secret store (Kubernetes Secret, container-host secrets,
  or a vault reference). Secrets in ConfigMaps are still exposed —
  ConfigMaps are not encrypted.

## Health checks and limits

- Define health: a Dockerfile `HEALTHCHECK` or, on orchestrators, explicit
  liveness/readiness probes. Liveness answers "am I alive" (keep it cheap
  — a heavy liveness check restarts healthy pods during downstream blips);
  readiness answers "route traffic to me". Pair Compose's `depends_on:
  condition: service_healthy` with a real healthcheck so it means
  something.
- Set explicit CPU/memory requests and limits. An unbounded container is a
  noisy-neighbour incident waiting to happen; in Kubernetes, missing
  limits also pins you to the worst eviction class.
- Handle termination gracefully: honour SIGTERM and drain connections on
  rollout rather than being cut mid-flight.

## Image security basics

- Scan both the Dockerfile and the built image in CI (Trivy, Grype, or
  Docker Scout); fail the build on fixable critical/high findings, using
  `--ignore-unfixed` so the gate reflects actionable results.
- Produce an SBOM and provenance attestation for every released image
  (`--sbom=true --provenance=mode=max`) so contents and origin are
  verifiable; sign released images (Cosign keyless/OIDC) and verify before
  deploy.
- Keep base images current — yesterday's clean scan means nothing after
  a new CVE drops. Automate rebuilds on base-image updates and track scan
  exceptions with expiry dates rather than disabling the gate.
- Lint Dockerfiles (`hadolint`) and review the checklist before shipping:
  multi-stage, pinned minimal base, non-root numeric UID, cache-friendly
  ordering, `.dockerignore` present, no secrets baked in, health defined,
  image scanned.

## Compose for local dev parity

- Compose is for local development and single-host scenarios — one command
  brings up the whole stack. It is not a production orchestrator; translate
  the topology into real manifests for production, don't promote Compose
  directly.
- Use `compose.yaml` (the top-level `version:` key is obsolete — omit it).
- Real `healthcheck` blocks on stateful services so `depends_on:
  service_healthy` gates on readiness, not just container start.
- Named volumes for persistent data, never the container's writable layer;
  bind-mount source only in development.
- Credentials via file secrets (`*_FILE` env vars or app-side file reads),
  never hardcoded env values; keep secret files out of git.
- Put the database on an internal-only network; attach host-facing services
  to an external network as well, and publish only the ports the host must
  reach. Use `profiles` to gate optional services so plain `up` stays lean,
  and set CPU/memory limits so local runs mirror production.

## Input → Process → Output

**INPUT:** A Node API + Postgres + Redis app whose Dockerfile is one giant
`FROM node:20`, 1.2 GB image, secrets baked in as ENV, and Compose that
mounts the host's Docker socket.

**PROCESS:**
1. Multi-stage build: `builder` stage installs deps and compiles;
   `runtime` stage copies only `dist/` + production deps onto a pinned
   minimal base (`node:20-slim`). Drops the image to ~180 MB.
2. Layering: copy `package.json` + lockfile first, install, then copy
   source — dependency layers cache across code changes.
3. Secrets: remove `ENV API_KEY=...`; use BuildKit `--mount=type=secret`
   at build time and Compose file secrets at runtime; non-root `USER`.
4. Health: add a real `/healthz` probe and `HEALTHCHECK` in the Dockerfile;
   `depends_on` with `condition: service_healthy` in Compose.
5. Network: database on an internal-only network; publish only the API
   port; remove the Docker-socket mount.

**OUTPUT:** A hardened multi-stage Dockerfile (~180 MB, non-root, no
secrets in layers) plus a Compose file with health-gated startup and an
internal database network.

## Proven pattern vs avoid pattern

- **Proven — cache-friendly multi-stage layers:** stable instructions
  (dependency install) before volatile ones (source copy). Rebuilds after a
  one-line code change take seconds, not minutes, and the runtime image
  carries no build tooling.
- **Avoid — single-stage images with secrets in ENV:** everything in one
  layer means every code change reinstalls the world, and `ENV` secrets
  persist in the image history forever — anyone with the image has the key.
  Multi-stage + secret mounts + non-root user instead.

## Telemetry, testing & debugging

- **Instrument:** image size trend per build; vulnerability counts from the
  scanner; container restart counts and OOM kills (resource limits doing
  their job or too tight).
- **Test:** `docker build` in CI with `--no-cache` periodically to catch
  cache-rot; scan the image (`trivy`/`grype`) and fail on critical CVEs;
  run Compose with healthchecks and assert every service reports healthy.
- **Debug:** `docker logs --tail` and `docker inspect` on the failing
  container first; exec in only to reproduce — `docker run --rm -it
  <image> sh` with the same env to isolate image vs orchestration issues.
