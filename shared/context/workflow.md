# workflow.md — Flow & dependencies

Two surfaces: Go backend (`playgrounds/elora-be-go`) + Flutter app (`playgrounds/elora_spendos`).
Both already have substantial existing code (see each repo's own `docs/progress/` and
`CLAUDE.md` "Status & Progress" section for history) — this file governs the flow for the
**next** feature/change cycle, not a from-scratch build.

| Phase | Owner | Consumes | Produces | Starts after |
|-------|-------|----------|----------|--------------|
| planning | `@pm` | manager's idea | `PRD.md`, `user-stories.md` | — |
| design | `@architect` | `PRD.md`, `user-stories.md`, existing `api-documentation/*.yaml` | updated `api-documentation/*.yaml` (both copies) + data model | planning `done` |
| build-backend | `@backend` | `PRD.md`, `api-documentation/*.yaml` | code in `playgrounds/elora-be-go` | design `done` |
| build-frontend | `@frontend` | `PRD.md`, `api-documentation/*.yaml`, `playgrounds/elora_spendos/design-system/` | code in `playgrounds/elora_spendos` | design `done` |
| testing | `@qa` | `PRD.md` + code | `test-report.md` | build-backend `done` AND build-frontend `done` |
| review | **manager** | everything + code | approve / reject | testing `done` |
| deploy | `@devops` | `playgrounds/elora-be-go`, `api-documentation/*.yaml` | live backend on the VPS (URL/IP recorded) + infra/CI configs | testing `done` |

**Deploy is plan-then-confirm.** `@devops` may read state, write IaC/CI/scripts, and dry-run
freely, but every real mutation to the live server is a **human gate** — Bowo stops and asks
the manager, like any manager-owned phase. It runs on demand when the manager wants a deploy;
it does not auto-fire every build.

## Who drives this
**`bowo`** (the orchestrator) drives the flow: it reads this file + `task-status.json`,
dispatches the owning agent for each ready phase, updates status, and writes a report per phase
to `reports/`. Start it with `claude --agent bowo`. Phases owned by **manager** are human gates
— Bowo stops and asks.

## Parallel
`build-backend` and `build-frontend` run simultaneously once `design` is `done` — both hold to
the same API contract. Run each in its own git worktree, inside the respective `playgrounds/*`
repo, so they don't collide. The front-end uses mocks from the contract until `build-backend`
is `done`.

## Gates & corrections
- A phase must not start before its prerequisite is `done`.
- If you reject at review: send it back to `@pm` (requirement issue) or `@architect` (contract
  issue), reset the related entries in `task-status.json` to `todo`, then re-run the downstream
  phases.
- If the API contract changes after a build started: any `done` build-frontend (and
  build-backend, if affected) becomes stale — set it back to `todo` and re-run.
- The API contract exists as **two copies** (`playgrounds/elora-be-go/api-documentation/` and
  `playgrounds/elora_spendos/api-documentation/`). Any phase that changes it must update both,
  or note the drift in `notes/` immediately.
