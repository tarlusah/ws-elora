# PRD — Online-required writes

- **Cycle:** next feature/change cycle
- **Date:** 2026-08-01 (revised — see revision note below)
- **Baseline:** `shared/context/PRD.md` (consolidated status — read that first for where the
  project stands today)
- **Architecture decision:** `notes/ADR-0004-online-required-writes.md` — the authoritative
  record for *why* each choice was made. This PRD states *what* to build.
- **Status:** ✅ Approved 2026-08-02 (all §7 open decisions confirmed) → next: `@pm` writes
  `user-stories.md`, then `@architect`

**Revision note:** this PRD originally described full bidirectional offline-first sync (see
`notes/ADR-0003-offline-first-sync-and-receipt-scan.md`). On 2026-08-01 the manager stepped that
back to an online-required-write model before any production user exists — implementation cost
was judged too high for the product's current stage. This revision reflects that decision.

**2026-08-02 — receipt scanning cut from scope.** This PRD originally bundled LLM receipt
scanning alongside the online-required-writes change, because both touched the same data paths
(see ADR-0004's prior revision, and `notes/ADR-0003-offline-first-sync-and-receipt-scan.md` §3
for the original design). The manager cut it entirely — not deferred, not split out for later,
just removed from the plan. This PRD now covers online-required writes only.

---

## 1. Summary

**Writes require a live connection; reads work offline from a local cache.** Spendos already
stores locally (Drift). That cache keeps its role for fast, offline-capable *reading* — accounts,
transactions, budgets, categories, dashboard aggregates all display the last data fetched from the
server. Creating, editing, or deleting anything requires connectivity at that moment. There is no
local write queue, no sync protocol, no conflict resolution.

There is **no production data and no backward-compatibility requirement**, so this cycle uses
plain, server-authoritative CRUD rather than the sync protocol previously scoped.

---

## 2. Goals

| # | Goal | Success looks like |
|---|---|---|
| G1 | The app is usable offline for **reading** | Viewing Capture history, Review, History, Dashboard, and Budget all work with the network off, showing the last-fetched data |
| G2 | Writing is honest about connectivity | Every create/edit/delete action fails immediately and explicitly ("needs connection") when offline — never silently queued, never silently lost |
| G3 | Data survives a lost phone | A fresh install + login restores every account, category, budget, and transaction from the server |

**Dropped from the previous cycle's goals** (see ADR-0004 §3): "fully usable offline including
writes," "no user data ever lost via a local write queue," and "sync status indicator / rejected
records visible" — these depended on the offline-write mechanism this cycle no longer builds.

## 3. Non-goals (explicitly out of scope this cycle)

- **Offline writes of any kind.** Creating, editing, or deleting anything requires connectivity.
  This is the one that changed — see ADR-0004.
- **Multi-device concurrent use.** One account, sequential device use, as before.
- **Anonymous / no-account mode.** Login is required.
- **Receipt scanning.** Cut from scope entirely (2026-08-02) — see the revision note above. Not
  part of this or any currently planned cycle.
- **Web client.** Server dashboard endpoints are retained for it, but it is not built here.

---

## 4. Scope — Online-required writes, local read cache

### 4.1 What the user experiences

- Viewing data — Capture history, Review, History, Dashboard, Budget — works offline, showing
  whatever was last fetched.
- Creating, editing, or deleting a transaction, account, budget, or category **requires an active
  connection**. Attempting it offline shows an explicit "needs connection" message immediately —
  the action does not happen and nothing is queued to happen later.
- Session length (refresh-token TTL) is a normal "how long before you must log in again" — not an
  offline-window promise. **Confirmed: 60 days, sliding** (§7).
- Logging in as a different user on the same device simply replaces the cached data — the
  previous user's cache is cleared before the new user's data is fetched.

### 4.2 What gets built — backend (`elora-be-go`)

| Item | Detail |
|---|---|
| No sync domain | The old sync log (`POST /v1/sync`, `GET /v1/sync/status`, `PUT /v1/sync/{id}`) stays retired; no delta endpoints replace it |
| Standard CRUD | Existing per-domain REST endpoints (`transactions`, `accounts`, `budgets`, `categories`) are the only write surface |
| `id` generation | Server-generated, as it was before ADR-0003 — no client-supplied UUID |
| Delete semantics | Plain `DELETE`. No `deleted_at` tombstone requirement (a separate "undo" UX could reintroduce one later, independent of this decision) |
| No `server_seq` | No pull cursor, no push staleness check — neither concept applies without a sync protocol |
| `default_account_id` on user | Replaces the per-account `is_default` flag (avoids a two-record atomic update) — unaffected by this pivot, still worth doing |
| `/me` endpoint | Completes the Profile placeholders noted in the baseline PRD — unaffected by this pivot |
| Auth | Refresh-token TTL confirmed at 60 days, sliding (§7); rotation + grace window is a nice-to-have against spurious logouts, not a correctness requirement anymore |

### 4.3 What gets built — frontend (`elora_spendos`)

| Item | Detail |
|---|---|
| Read-cache refresh | Paginated `GET` fetch on app foreground and after every successful write, upserted into Drift. Not cursor/delta based. |
| Write connectivity gate | Every create/edit/delete action checks connectivity first and fails fast with an explicit message if offline |
| No dirty tracking, no retry/backoff, no conflict UI, no rejected-record UI | None of these apply without a write queue |
| Cache freshness indicator (optional) | A simple "data as of [time]" label on cached views, if wanted — much lighter than the previous `syncing`/`synced`/`failed` state machine |
| Login flow | Authenticate → fetch from server → populate cache. One case, not three. |
| Logout flow | Plain wipe — no "warn if unsynced records exist" check, since nothing local is ever unsynced |
| **Repository layer cleanup** | Budget, History, Review, Dashboard, Profile currently call Drift directly. *No longer bundled into this cycle by necessity* (see ADR-0004 §2.5) — worth doing, schedule independently at whatever priority the manager wants |
| Budget feature | Still a 9-line stub. Unaffected by this pivot; still its own open item |

### 4.4 What syncs (i.e., what the read cache holds)

Transactions (incl. `000` placeholders, review state, splits), accounts, budgets, categories, and
user profile are all fetched and cached for offline reading. Dashboard aggregates are computed
locally from the cache, as before. Not cached: sessions/tokens.

---

## 5. Acceptance criteria

- [ ] With the network enabled, a user can create, edit, delete, review, split, and categorise
      transactions; manage accounts, categories, and budgets.
- [ ] With the network disabled, the same actions fail immediately with an explicit "needs
      connection" message — nothing is silently queued or silently lost.
- [ ] With the network disabled, the user can still view Capture history, Review, History,
      Dashboard, and Budget using the last-fetched data.
- [ ] A fresh install + login restores the complete dataset from the server.
- [ ] Logging in as a different user on the same device replaces the cache — no data mixing.
- [ ] Refresh-token expiry produces a "log in again" state without corrupting the local cache.

---

## 6. Dependencies and sequencing

1. Manager approves this PRD and answers §7.
2. `@pm` writes `user-stories.md`.
3. `@architect` updates **both** copies of `api-documentation/*.yaml` — whatever schema changes
   fall out of dropping client-generated ids and tombstones. The two copies must not drift.
4. `@backend` and `@frontend` run in parallel against the agreed contract, each in its own
   worktree. The frontend uses contract mocks until the backend endpoint lands.
5. `@qa` tests both surfaces: the online-write happy path and the offline "needs connection" gate.
   No conflict/rejected-record matrix — that mechanism doesn't exist.
6. Manager review → `@devops` deploy.

**Sequencing note:** Budget is a clean slate on the frontend and should be built once — not now
and rewritten again later, unaffected by this pivot.

---

## 7. Open decisions blocking `@architect`

| # | Decision | Status |
|---|---|---|
| 1 | ~~Refresh-token / session TTL~~ | **☑ Confirmed 2026-08-02 — 60 days, sliding.** |
| 2 | ~~Hard delete vs. soft delete~~ | **☑ Confirmed 2026-08-02 — hard delete.** Default accepted (no longer forced by sync; free choice — see ADR-0004 §4). |

**All open decisions confirmed 2026-08-02 — this PRD is approved.** Next: `@pm` writes
`user-stories.md`, then `@architect` starts the contract (backlog item 5.1).

---

## 8. Notes for whoever picks this up

- The remote GitHub copies of both repos are roughly 1–2 weeks behind what the baseline PRD
  describes (backend last pushed 2026-05-09, frontend 2026-05-02, versus status documents dated
  2026-05-15/16). Push local work before relying on the remote as the source of truth.
- `notes/ADR-0003-offline-first-sync-and-receipt-scan.md` remains as historical record of the
  fuller offline-first design. It is not wrong — it was superseded for cost reasons at the
  product's current pre-production stage, and remains available to revisit in a future cycle. Its
  §3 (receipt scanning) is historical only — that feature is cut, not deferred (see the revision
  note above).
- Field names and details (e.g. whether system categories can be renamed by users) were derived
  from the domain list in `shared/context/PRD.md`, not from the live schema. They must be
  verified against the actual code before entering the API contract.
