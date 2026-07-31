---
name: frontend
description: Build/extend the Flutter app in playgrounds/elora_spendos. After the API contract is ready.
---

Consumes: `PRD.md`, `playgrounds/elora-be-go/api-documentation/*.yaml` (mirrored copy in
`playgrounds/elora_spendos/api-documentation/`), `playgrounds/elora_spendos/design-system/`.
Produces: code in `playgrounds/elora_spendos`.
Stack: Flutter (Dart), offline-first — `flutter_riverpod` for state, `go_router` for
navigation, `drift` for local storage, `dio` for HTTP, `flutter_secure_storage` for tokens.

**Read `playgrounds/elora_spendos/CLAUDE.md` first and follow it.** It is the source of truth
for the folder structure and session-log conventions.

**Then read every file in `.claude/skills/frontend/` before writing any code** — these are a
synced copy of `playgrounds/elora_spendos/.claude/skills/`:
- `coding-style.md` — feature architecture is strictly **Repository → Provider → Screen**:
  Repository does all API/local-DB access, Provider (`Notifier`/`AsyncNotifier`, `@freezed`
  state) calls only the repository, Screen (`ConsumerWidget`) watches providers and holds zero
  business logic. Never call `dio` or `drift` directly from a screen or provider.
- `database-conventions.md` — every synced Drift table needs identity + business fields +
  **soft delete** (`isDeleted`, hard delete is forbidden) + sync metadata (`isSynced`,
  `updatedAt`, `idempotencyKey`). All writes go to Drift first, never directly to the server;
  sync is push-then-pull, batched, idempotent, server-wins on conflict.
- `api-design.md` — base URL and full endpoint list live here; read the relevant
  `api-documentation/*.yaml` before implementing or calling any endpoint. Every authenticated
  request carries `Authorization: Bearer {jwt}`; timestamps are ISO 8601 UTC; IDs are UUID v4;
  amounts are integers (IDR, no decimals); success/error envelope is `{data: ...}` /
  `{error: {code, message}}`; deletes on synced entities are always soft.
- `design-system.md` — **never hardcode a color, size, spacing, or radius.** Always use the
  `AppColors` / `AppTypography` / `AppSpacing` / `AppRadius` token constants; check
  `design-system/styles/tokens.css` first if a value isn't covered, then add the constant.

Workflow skills (superpowers) — invoke these, don't reinvent them:
- **Before building any feature:** `superpowers:test-driven-development` — write a failing widget/unit test first, then make it pass.
- **When a requirement or UI behaviour is ambiguous** (the contract or design system doesn't say): `superpowers:brainstorming` to settle intent before writing code — don't guess, and don't change the contract.
- **When a test fails or the UI misbehaves:** `superpowers:systematic-debugging` — find the cause before patching, don't paper over symptoms.
- **Before claiming any task done:** `superpowers:verification-before-completion` — actually run `flutter analyze` and `flutter test`, confirm the output, then report. No "should pass."
- **Isolation:** work in your own git worktree (`superpowers:using-git-worktrees`) so `build-frontend` and `build-backend` don't collide.
- **When the build is complete:** `superpowers:requesting-code-review`, then `superpowers:finishing-a-development-branch` to merge/PR. Handle review or `@qa` feedback with `superpowers:receiving-code-review` — verify suggestions, don't agree blindly.
- *Optional:* `frontend-design:frontend-design` for visual judgment on states the design system doesn't cover — but always within the design tokens; `design-system.md` and `app_theme`/`AppColors` win on anything they do specify.

Rules:
- Every commit message must start with the prefix `as frontend:` (e.g. `as frontend: implement transaction split screen`).
- Consume the API contract exactly as specified — request/response shapes must match; never alter the contract here.
- Use mock responses (hardcoded or from a local mock service) until the backend endpoint it needs is `done`.
- If a UI decision forces a contract question, note it in `notes/` and tell the manager. Never change the contract silently.
- Update the `build-frontend` phase in `task-status.json`.

Done = `flutter analyze` clean + `flutter test` passes + UI matches design tokens + `@qa` tests pass — and per `superpowers:verification-before-completion`, all four are *run and confirmed*, never assumed.
