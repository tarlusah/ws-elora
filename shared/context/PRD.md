# Elora / Spendos — Consolidated PRD & Status

**Consolidated:** 2026-07-31
**Purpose:** Single up-to-date entry point before any large change. Original PRD + status
across both repos were scattered and partly stale (see "Document Map" at the bottom) — this
file supersedes them as the thing to read first. It does not replace the detailed docs, it
tells you which ones are still authoritative and summarizes where the project actually stands.

---

## 1. Product

**Spendos** — a personal expense tracker mobile app, part of the Elora ecosystem
(`spendos.elora.work`). Backend: Go (`playgrounds/elora-be-go`). Frontend: Flutter, offline-first
(`playgrounds/elora_spendos`).

### Philosophy (unchanged since original PRD, still the design compass)
| Principle | Description |
|---|---|
| Awareness over Accuracy | Know roughly where money goes, don't obsess over exact records |
| Reflective, not Rushed | Calm periodic review, not frantic real-time tracking |
| Minimum Friction, Maximum Meaning | Capture is effortless; review is meaningful |
| Two Modes of Time | Capture = fast & focused. Review = slow & reflective |

### Core loop
**Capture** (amount + account only, category deferred, `000` = placeholder-save-for-later) →
**Review** (flash-card queue, swipe/tap to categorize, parent-child split) → **History** (search
+ filter) → **Dashboard** (budget-aware summary) → **Budget** (per-category monthly limit).

### ⚠️ Scope has evolved since the original PRD — read this before planning changes
The original `spendos-prd.md` (v1.0, May 2026) explicitly lists **"Offline mode (full)" as
Out-of-Scope for MVP** (§10). The team nonetheless built the app **offline-first from the start**
(Drift local DB, sync engine, `isSynced`/soft-delete metadata on every entity) — this is now
core architecture, not an add-on. The PRD document itself was never updated to reflect this
pivot. Treat the **original PRD as vision/features reference only** — for what's actually built,
use §3–5 below, not `spendos-prd.md` directly.

---

## 2. Backend status — `elora-be-go`

**Overall: mature, functionally complete for its 7 domains, no pending/blocked backend work as
of the last session (2026-05-15).**

| Domain | Layers (handler/usecase/repo/entity/dto) | Notes |
|---|---|---|
| user | ✓ complete | Auth (bcrypt cost 12, JWT), Google OAuth w/ JWKS verification + caching, session soft-limit (max 10) |
| account | ✓ complete | CRUD + archive/restore/set-default |
| category | ✓ complete | Hierarchical (parent/child), system vs user categories, hide/show |
| transaction | ✓ complete | CRUD, review, split, idempotency key |
| budget | ✓ complete | CRUD + copy |
| dashboard | ✓ complete (no entity — aggregation-only) | `/dashboard/home`, `/dashboard/stats` |
| sync | ✓ implemented, but **intentionally simplified** — see §4 | Generic idempotent log, not the original push/pull/devices spec |

All domains have real logic + unit/integration tests (150–750 line usecases, matching repo
tests). No stubs found except one explicit dev-only email sender fallback (prod path is real).

**API surface** (`api-documentation/*.yaml`, 9 files, kept in **two synced copies** — also under
`elora_spendos/api-documentation/`): auth (9 endpoints), users, accounts, categories,
transactions, budgets, dashboard, sync. Full list in `.claude/skills/backend/api-design.md`
(this workspace) or the YAML files directly.

**Alignment effort closed:** `docs/task-backend-alignment.md` tracked 7 priorities against
`docs/backend-discrepancies.md`; priorities 1, 2, 3, 5, 7 are done (auto-seed on register,
per-user protected "Uncategorized" category, transaction idempotency key, user profile
endpoints, unreviewed-count endpoint). Priorities 4 and 6 were **deliberately deferred** — see §4.

---

## 3. Frontend status — `elora_spendos`

**Overall: 6 of 7 features built and design-aligned; Budget is the one real gap.**

| Feature | Status |
|---|---|
| Auth | ✓ Done — splash, onboarding, login, register, forgot password, email verification |
| Capture | ✓ Done — rebuilt to match design HTML 1:1 (s9), calculator keypad, `000` placeholder flow, offline-first save |
| Review | ✓ Done — flash-card swipe-to-skip gesture (s9), split-transaction sheet |
| History | ✓ Done — filters, search, edit-transaction screen |
| Dashboard | ✓ Done — real aggregation over Drift (current + previous month), sparkbar, top-3 categories, empty state, skeleton loading, error-retry. Budget-aware variants deferred (need Budget data, see below) |
| Profile | ✓ Mostly done — identity card, account mgmt, category mgmt, sign-out wired. "Coming soon" placeholders remain for: edit profile, preferences, add-account, add-sub-category; identity/stat numbers are placeholders pending a `/me` endpoint |
| **Budget** | ❌ **Not started** — `budget_screen.dart` is a 9-line stub, no provider, no data layer. Design HTML (`Screens · Budget.html`, 810 lines) shipped 2026-05-15, so this is unblocked whenever picked up |

**Architecture note:** the `Repository → Provider → Screen` pattern (mandatory per
`.claude/skills/frontend/coding-style.md`) is followed cleanly in **Auth** and **Capture**.
**Budget, History, Review, Dashboard, Profile** have providers that call Drift directly,
skipping a repository layer. Not broken, but a drift from the documented convention — worth
tidying if a large refactor touches those features anyway.

**Known open issues** (see `elora_spendos/CLAUDE.md` "Status & Progress" for full detail):
- `CategoriesNotifier.retry()` untested (test-harness limitation, not reproduced in prod)
- `CaptureNotifier.saveSuccess` flag is non-introspectable after `save()` returns (cosmetic, UI works)
- Dashboard has no unit tests yet
- Fraunces font only partially wired (headings fall back to DM Sans in a few places)
- Dashed-border empty-state detail not implemented (Flutter has no native dashed border)

**Immediate next steps per last session (2026-05-16):** Budget screens → Dashboard
budget-aware variants → Profile sub-screens (edit profile, preferences).

---

## 4. Open architectural decision: Sync — read this before any big change touching data sync

> **✅ RESOLVED 2026-07-31, revised 2026-08-01.** This fork was first closed by
> `notes/ADR-0003-offline-first-sync-and-receipt-scan.md` (login-required, single-device,
> bidirectional cursor delta sync with soft delete and server-assigned sequencing). Before any
> production user existed, the manager judged that implementation too expensive for this stage
> and stepped back to an online-required-write model —
> `notes/ADR-0004-online-required-writes-and-receipt-scan.md` is now the current decision:
> plain server-authoritative CRUD, no sync protocol, local cache for offline **reading** only.
> ADR-0003's design was not wrong and remains available to revisit in a future cycle. The section
> below is retained as the historical framing of the problem — read ADR-0004 for the current
> decision, and `shared/context/PRD-offline-first-and-receipt-scan.md` for what to build.

This is the single most consequential open fork in the whole project, and it spans **both**
repos:

- **Backend** (`docs/decisions/0001-defer-sync-rewrite.md`, 2026-05-08): the original
  `tasks/task-sync.md` spec wanted full push/pull/multi-device sync — 3 new tables
  (`devices`, `idempotency_keys`, audit log), soft-delete + versioning across **transactions,
  accounts, categories, budgets**, a `X-Device-ID` convention, background cleanup jobs.
  Estimated 1–2 weeks, non-trivial regression risk. **Deferred** — kept as a simple generic
  sync log (`POST /v1/sync`, `GET /v1/sync/status`, `PUT /v1/sync/{id}`) until the owner
  explicitly picks: **(a)** full rewrite, **(b)** keep current + adjust frontend, or **(c)**
  hybrid (keep as event log, add separate push/pull endpoints).
- **Frontend** (`tasks/TASK-FE-01-sync-mechanism.md`): spec calls for UUID sync IDs, SHA-256
  checksums, exponential-backoff retry (max 3), debounce, and a `syncing/synced/failed` UI
  status indicator. What's actually built (`lib/core/sync/`) is a basic push/pull with a dirty
  flag and a last-sync timestamp — **none of the retry/backoff/checksum/UI-status requirements
  are implemented.**

**Both sides independently simplified the same feature and never reconciled.** If "large
changes" touch multi-device use, conflict resolution, or reliability of sync, this decision
needs to be made explicitly first — it drives schema changes (soft-delete + versioning on 4
tables), breaks current DELETE semantics, and determines how much frontend retry/backoff work
is actually needed. See decision 0001 for the exact unblocking questions (multi-device needed?
offline-first/conflict-resolution in scope this quarter? OK breaking current DELETE semantics?).

**Resolved, no action needed:** Decision 0002 (REST path conventions — action-style POST for
archive/hide/show/set-default) — kept as implemented, documentation-only, no code change.

---

## 5. Document map — what's current vs historical

Don't re-read the superseded ones for current state; they're listed only so you know they exist
and can be skipped.

| Location | What it is | Status |
|---|---|---|
| `elora_spendos/spendos-prd.md` (+ mirrored in `elora-be-go/temp_storage/`) | Original product PRD v1.0 | Vision/features reference only — see §1 scope-evolution note |
| `elora_spendos/project-knowledge.md` | Project knowledge base | Reference, broadly still relevant |
| `elora_spendos/CLAUDE-indo.md` | Early Indonesian-language architecture guide | Historical — superseded by root `CLAUDE.md` |
| `elora_spendos/SYSTEM_FLOW.md` | Backend API reference (FE's copy) | Reference for API contract |
| `elora_spendos/FLUTTER_SYSTEM_FLOW.md` + `prompt-generate-flutter-flow.md` | Generated FE flow doc, 2026-05-09 | **Stale** — predates sessions 5-9 |
| `elora_spendos/docs/CHAT-HANDOFF.md` | Session hand-off doc | Current as of its session |
| `elora_spendos/docs/progress/session-1..9.md` | Dated FE dev session logs | **Authoritative** for FE history |
| `elora_spendos/task-alignment/a01/*` | Early gap analysis (2026-05-09) | **Superseded** — said only Auth+Capture done; long outdated |
| `elora_spendos/task-alignment/a02/screen-alignment-progress.md` | Screen alignment tracker (through 2026-05-16) | Current, second most-reliable status doc after `CLAUDE.md` |
| `elora_spendos/update-tasks/*.md` | Capture screen build/design-review log | Historical, resolved |
| `elora_spendos/tasks/flutter-00..07-*.md` | Original per-feature specs (2026-05-02) | Historical intent; `flutter-07-dashboard.md` explicitly marked SUPERSEDED |
| `elora_spendos/tasks/TASK-FE-01-sync-mechanism.md` | Sync spec | Not fully implemented — see §4 |
| `elora_spendos/design-system/PROGRESS.md` | Design-system decision tracker | **Stale/contradicted** — shows brand color etc. as "pending" when already decided and shipped; trust `CLAUDE.md` + a02 instead |
| `elora_spendos/design-system/DESIGN_SYSTEM_REPORT.md` | Design token audit (2026-05-09) | Reference, roughly current |
| `elora_spendos/CLAUDE.md` "Status & Progress" | FE status summary | **Authoritative**, updated 2026-05-16 |
| `elora-be-go/tasks/task-*.md` | Original per-domain BE specs | Historical artifacts per decision 0002 |
| `elora-be-go/tasks/task-user-progress.md` | User-domain completion checklist | Done, no pending items |
| `elora-be-go/docs/task-backend-alignment.md` | 7-priority remediation vs `backend-discrepancies.md` | Mostly done; P4/P6 tracked as decisions |
| `elora-be-go/docs/backend-discrepancies.md` | Gap list that alignment doc responds to | Historical input, resolved via alignment doc |
| `elora-be-go/docs/decisions/0001-*.md`, `0002-*.md` | Deferred-decision records | **Authoritative** — see §4 |
| `elora-be-go/docs/progress/session-*.md` | Dated BE dev session logs | **Authoritative** for BE history |
| `elora-be-go/CLAUDE.md` "Status & Progress" | BE status summary | **Authoritative**, updated 2026-05-15 |

---

## 6. Before making large changes

> **Current cycle:** online-required writes (CRUD, no sync protocol) + LLM receipt scanning are
> specified in `shared/context/PRD-offline-first-and-receipt-scan.md` (decision record:
> `notes/ADR-0004-online-required-writes-and-receipt-scan.md`, which supersedes the offline-sync
> portion of `notes/ADR-0003-offline-first-sync-and-receipt-scan.md`). This file remains the
> baseline you are changing *from*; that PRD defines the change itself.

1. Resolve the sync-architecture fork (§4) first if the change touches sync/offline/multi-device
   at all — it's the one place both surfaces already diverged from spec independently.
2. If the change touches Budget, note the frontend has zero existing code there — it's a clean
   slate, not a refactor.
3. If the change touches Profile, History, Review, or Dashboard on the frontend, consider folding
   in the repository-layer cleanup (§3 architecture note) while you're in there.
4. Use `@pm` to write a fresh `user-stories.md` for the new scope once direction is picked —
   this file intentionally does not attempt to define the large change itself, only the baseline
   you're changing from.
