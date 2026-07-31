# reports/ — Per-phase orchestrator reports

`bowo` (the orchestrator) writes one report here per completed phase, named `<phase>.md`
(e.g. `design.md`, `build-backend.md`, `build-frontend.md`, `testing.md`).

Each report records: what was done, by which agent, the artifacts produced, the build/test
result, any deviations noted in `notes/`, and the resulting status.

Live status remains in `shared/context/task-status.json`; these files are the human-readable
trail of each phase.
