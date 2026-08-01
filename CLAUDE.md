# CLAUDE.md — Workspace configuration

## Product
Elora — two independent repositories, not a monorepo. Both are symlinked into
`playgrounds/`, never copied:
- `playgrounds/elora-be-go/` — Go backend API. **Has its own `CLAUDE.md` + `.claude/skills/` —
  agents working on it MUST read and follow those first.**
- `playgrounds/elora_spendos/` — Flutter app "Spendos" (personal expense tracker). **Has its
  own `CLAUDE.md` + `.claude/skills/` — agents working on it MUST read and follow those first.**

## Stack (as implemented in the two repos — do not change without discussing)
- **Backend:** Go, Go workspace (`go.work`: `pkg/` + `internal/` + `apps/server/`), Gin router,
  PostgreSQL via `pgx`, JWT + Google OAuth (JWKS) auth. **No Redis** — everything, including
  caches (e.g. idempotency keys), lives in Postgres; confirmed 2026-08-01, do not reintroduce
  Redis without discussing. Modular-monolith architecture — see `playgrounds/elora-be-go/CLAUDE.md`
  for the full domain/layering rules.
- **Mobile:** Flutter (Dart), Riverpod (`flutter_riverpod` + `riverpod_annotation`) for state,
  `go_router` for navigation, `drift` for offline-first local storage, `dio` for HTTP,
  `flutter_secure_storage` for tokens. See `playgrounds/elora_spendos/CLAUDE.md`.
- **Deploy target:** VPS (Go binary + PostgreSQL via `docker-compose`), not a managed
  serverless platform — see `playgrounds/elora-be-go/deploy/docker-compose.yml`.
- **API contract:** `playgrounds/elora-be-go/api-documentation/*.yaml` (OpenAPI, one file per
  domain). **This same set of files is duplicated in `playgrounds/elora_spendos/api-documentation/`**
  — the two repos are kept in sync manually. Treat any drift between the two copies as a bug:
  flag it in `notes/`, don't silently pick one.

## Language
- **Every file in this workspace is written in English** — all artifacts, reports, notes, code, and comments.
- Live chat with the manager can be in any language; only files are English.

## Rules
- Follow `AGENTS.md`.
- To run the build flow automatically, start the orchestrator **Bowo**: `claude --agent bowo`. It dispatches each agent per `shared/context/workflow.md`, updates `task-status.json`, and writes a report per phase to `reports/`.
- `@backend` and `@frontend` may run in parallel once the API contract is agreed. Ideally each agent works in its own git worktree/branch (inside the respective `playgrounds/*` repo) to avoid collisions, then merge.
- The front-end uses mock responses from the API contract until the backend endpoint it needs is ready.
- Don't change the API contract silently — note changes in `notes/` and tell the manager, and update **both** copies of the affected `api-documentation/*.yaml` file.
- Small, frequent commits.

## Definition of done (per feature)
Code works + `@qa` tests pass + `task-status.json` updated to `done` + you (the manager) approve.
