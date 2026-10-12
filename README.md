# appdev-skillkit

One merged, refined skill set for application development — Rust-first
full-stack. 24 agent skills distilled from the best public skill libraries on
GitHub, deduplicated and rewritten into a single coherent repo.

## Layout

```
skills/
├── methodology/            # how to work: TDD, debugging, planning, reviews
│   ├── tdd-workflow
│   ├── systematic-debugging
│   ├── planning-and-execution
│   └── code-review-workflow
├── rust/                   # Rust language + backend services
│   ├── rust-patterns        # ownership, async/tokio, errors, API design, anti-patterns
│   └── rust-axum-service   # hexagonal Axum services: layers, state, middleware, testing
├── rust-ui/                # pick ONE per project
│   ├── rust-gpui           # native Rust UI via GPUI / gpui-kit
│   └── rust-tauri          # Tauri v2: Rust core + webview frontend
├── frontend/               # web UI
│   ├── ui-ux-decisions     # style direction, palette/font discipline, pre-delivery checklist
│   ├── frontend-taste       # design judgment: avoid the generic AI look
│   ├── react-best-practices
│   ├── tailwind-patterns
│   ├── shadcn-patterns
│   └── web-design-audit    # post-build accessibility/UX audit checklist
├── api/
│   └── api-design          # REST/GraphQL/gRPC decisions, versioning, errors, auth
├── quality/
│   ├── testing-patterns    # test pyramid, AAA, Playwright page objects
│   └── webapp-testing      # unit/integration/E2E + visual regression + a11y
├── security/
│   └── security-audit      # OWASP/CWE audit, secrets, supply chain, STRIDE-lite
├── infra/
│   ├── architecture-review # quality attributes, one-way doors, failure modes, ADRs
│   ├── sql-patterns        # schema, indexing, transactions, migrations
│   └── container-patterns  # Dockerfiles, Compose, image security
└── meta/
    └── skill-authoring     # write your own SKILL.md for the gaps
├── observability/              # live dataflow visibility
│   └── graphify-board     # dataflow graph board: live updates + debug-capture playback
├── bridges/                    # app-to-app bridges
│   └── mcp-tools-bridge   # expose/consume MCP tools, CLI/REST/webhook adapters
└── conventions/
    └── api-table.md       # reusable API/function-name conventions
```

Each skill follows the Anthropic Agent Skills spec: `SKILL.md` with `name` +
`description` frontmatter. Every skill now carries three standard sections:

- **Input → Process → Output** — a concrete worked example per module
- **Proven pattern vs avoid pattern** — what to do and the failure mode to dodge
- **Telemetry, testing & debugging** — module-specific observability, test, and
  debug guidance

Shared conventions live in `conventions/api-table.md`: the predefined
reusable API/function-name vocabulary (system lifecycle, config, logging,
errors, UI, input devices, storage, network/IPC, background tasks, bridges)
that AI agents should use when writing code, so names stay consistent across
the whole codebase.

Copy a skill folder into your agent's skills directory
(`~/.claude/skills/`, `.claude/skills/`, `.agents/skills/`) or symlink the
whole repo.

## Recommended install order

1. `methodology/*` — the workflow backbone (4 skills)
2. `rust/rust-patterns` + `rust/rust-axum-service`
3. One of `rust-ui/*`
4. `frontend/ui-ux-decisions` + `frontend/frontend-taste` + `frontend/web-design-audit`
5. `frontend/react-best-practices` + `tailwind-patterns` + `shadcn-patterns`
6. `api/api-design`, `quality/*`, `security/security-audit`, `infra/*`
7. `meta/skill-authoring` — for what's missing (iced, sqlx-specific, project conventions)

## Gaps (no good upstream source found)

- No maintained dedicated Axum skill upstream — `rust-axum-service` is distilled
  from a stale source; verify against axum 0.8.
- No sqlx/diesel/sea-orm-specific skills; no iced skill. Use `skill-authoring`
  to write these.

## Provenance

Every skill is an original distillation (paraphrased, merged, condensed) — not a
copy. See ATTRIBUTION.md for the upstream sources and licenses.
