---
name: backend
description: Build/extend the Go API in playgrounds/elora-be-go. After the API contract is ready.
---

Consumes: `PRD.md`, `playgrounds/elora-be-go/api-documentation/*.yaml`. Produces: code in
`playgrounds/elora-be-go`.
Stack: Go (Go workspace — `pkg/` + `internal/` + `apps/server/`), Gin router, PostgreSQL via
`pgx`, Redis, JWT + Google OAuth (JWKS) auth — modular-monolith architecture designed to split
into microservices without rewriting business logic.

**Read `playgrounds/elora-be-go/CLAUDE.md` first and follow it.** It is the source of truth for
repository structure, the microservice-migration checklist, and session-log conventions.

**Then read every file in `.claude/skills/backend/` before writing any code** — these are a
synced copy of `playgrounds/elora-be-go/.claude/skills/`:
- `coding-style.md` — domain structure (`handler/usecase/repository/entity/dto/event/projection`),
  file naming (noun-first), layer responsibilities, cross-domain communication (no direct domain
  imports — consumer defines the interface), dependency injection (only `apps/` wires
  implementations), `pkg/` rules, `configs/` placement.
- `database-conventions.md` — `snake_case` DB naming, transactions owned by the usecase (never
  the repository), migrations live inside the owning `apps/*` module, append-only, zero-padded
  6-digit version numbers.
- `api-design.md` — `camelCase` JSON on the wire vs `PascalCase` Go fields, the `EventBus`
  interface and event-contract rules (events are a versioned public contract — never rename/
  remove a field, add a `V2` type instead), projection rules (partial copies only, UPSERT not
  INSERT), correlation-ID propagation.

If a project convention changes, the fix belongs in `playgrounds/elora-be-go/.claude/skills/`
first (the repo's own source of truth) — then mirror the change into this workspace's
`.claude/skills/backend/` copy so the two don't drift; note the sync in `notes/`.

Rules:
- Every commit message must start with the prefix `as backend:` (e.g. `as backend: implement /accounts endpoint`).
- Implement endpoints **exactly** as the OpenAPI contract specifies — request/response/error
  must match, or the front-end breaks.
- Validate input, handle errors per `api-design.md`'s error conventions.
- Domains must never import each other directly (see `coding-style.md`).
- If the implementation forces a contract change: update **both** copies of the affected
  `api-documentation/*.yaml`, note why in `notes/`, tell the manager. Never change it silently.
- Update the `build-backend` phase in `task-status.json`.

Report back to the manager with: endpoints implemented (method + path), any contract deviations
noted in `notes/`, and the result of `go build ./...` + `go test ./...` (run from the repo root
so the whole `go.work` compiles).

Done = endpoints work AND `@qa` tests pass.
