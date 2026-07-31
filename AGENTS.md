# AGENTS.md — Rules for all agents

Applies to every agent. Short — only what you can't infer yourself.

0. **Write files in English.** Every artifact you write to disk (PRD, api-docs, reports, notes, code, comments, commit messages) is in English, regardless of the chat language.

1. **Read context first.** Before working, read the artifacts you consume in `shared/context/`. Also read the nested `CLAUDE.md` of the `playgrounds/` project you are working in (e.g. `playgrounds/elora-be-go/CLAUDE.md`, `playgrounds/elora_spendos/CLAUDE.md`), and follow it — plus every file inside its own `.claude/skills/` first.

2. **Update `task-status.json` (required).** On start → set your phase `in-progress` + `started_at`. On finish → `done` + `completed_at` + a summary in `notes`. Stuck → `blocked` + reason.

3. **Don't change contracts silently.** `PRD.md`, `api-docs.md`, and `design-system.json` are shared contracts. If a change is needed, note it in `notes/` and tell the manager first.

4. **Stay in your lane.** Only do and write your own artifact. Findings outside your scope → write them in `notes/`, don't fix them yourself.

5. **Respect dependencies.** See `workflow.md`. Don't start if your prerequisite isn't `done`; if so, set `blocked`.

6. **`done` means verified.** A build phase is only `done` when `@qa` tests pass — not on a one-sided claim.

The manager (human) directs and reviews. Specialist agents do not call each other on their own — only **`bowo`** (the orchestrator) dispatches other agents, and only by following `workflow.md` gates. Run it with `claude --agent bowo`. Phases owned by **manager** are human gates: Bowo stops and asks.
