# Elora — Context-Engineering Workspace

This repo is the **orchestration workspace** for the Elora projects. It holds no product
code itself — `playgrounds/` symlinks to the two real repos:
- `playgrounds/elora-be-go` → Go backend
- `playgrounds/elora_spendos` → Flutter app (Spendos)

You are the **manager / orchestrator + reviewer**; each agent owns one job and hands off
through `shared/context/`.

> Language rule: **everything written to files in this workspace is in English.**
> Live chat with the agents can be in any language you like — only files are English.

## Agents

| Agent | Job |
|-------|-----|
| `@pm` | Write the PRD + trim scope to the thinnest version that's still real |
| `@architect` | Analysis + reconcile the API contract the two surfaces build against |
| `@backend` | Build/extend the Go API in `playgrounds/elora-be-go` |
| `@frontend` | Build/extend the Flutter app in `playgrounds/elora_spendos` |
| `@qa` | Test both surfaces against the acceptance criteria |
| `@devops` | Deploy & operate — VPS, docker-compose, secrets, CI/CD |
| `bowo` | Orchestrator — dispatches the above per `shared/context/workflow.md` |

Review/approve is done by **you** (the human), not an agent.

## Flow

```
@pm  →  [you approve scope]  →  @architect (api contract)
                                       │
                          @backend  ──┴──  @frontend
                                       │
                    @qa  →  [you review & approve]  →  @devops (deploy)
```

Both surfaces hold to the OpenAPI contract in `playgrounds/elora-be-go/api-documentation/`
(mirrored in `playgrounds/elora_spendos/api-documentation/`). The front-end uses mocks from
the contract until the backend endpoint it needs is available.

## How to run

1. `cd` into this repo && `claude --agent bowo` to drive the full flow, or invoke a single
   agent directly (e.g. `claude --agent backend`) for a scoped task.
2. Open `shared/context/task-status.json` anytime to see where each phase stands.
3. Notes and decisions land in `notes/`; per-phase orchestrator reports land in `reports/`.

## Definition of done (per feature)

A phase is `done` not because an agent says so, but because the code works **and**
`@qa` tests pass. That's your gate.

## Project provenance

The agent roles, workflow mechanics, and process skills in this workspace are adapted
(abstracted — no borrowed project content) from an earlier workspace (`ws001`). The
`@backend` and `@frontend` architecture rules under `.claude/skills/` are copied from the
existing `.claude/skills/` in `playgrounds/elora-be-go` and `playgrounds/elora_spendos`
respectively — this workspace is a second, synced copy for the agents that operate here;
the originals in each repo remain the source repos' own local copy.
