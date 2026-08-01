# Discussion backlog — offline-first sync + receipt scanning

Working list for the manager and the architect. Ordered by dependency: items inside a track
should be settled in order; tracks 1–3 can run in parallel.

**Weight:** 🔴 deep design discussion · 🟡 medium · 🟢 quick decision (mostly manager's call)
**Status:** ☐ open · ☑ settled (link the ADR section or note when closing)

Context already settled: `notes/ADR-0003-offline-first-sync-and-receipt-scan.md`,
`shared/context/PRD-offline-first-and-receipt-scan.md`.
Architecture rules in play: `.claude/skills/backend/*.md`.

**Sync direction, settled with 1.1:** two-way = account, budget, transaction. Pull-only =
category, user profile. Creating/editing/hiding a category requires connectivity.

**Pull source, settled with 1.2:** domains answer `ChangesSince` from their own entity rows.
`server_seq` is authoritative on the row and comes from a per-user counter incremented inside
the write transaction. No changelog table, no event/projection feed for sync.

**Push semantics, settled with 1.5:** every pushed record carries `base_seq`; a write standing
on a stale version is applied under LWW and returned as `conflict` (which means *applied*).
Per-record granularity — one rejected record never blocks the batch. Tombstones are sticky;
budgets have a unique natural key.

---

## Track 1 — Sync core (backend)

**Items 1.1–1.6 are settled, so the API contract is now unblocked and Track 4 can start.**
What remains (1.7, 1.8) is narrowed and none of it is wire-visible.

| # | Topic | What it decides | Weight | Status |
|---|---|---|---|---|
| 1.1 | ~~**Shape of the new `sync` domain**~~ | **☑ Settled → ADR §2.11.** Thin orchestrator; both directions via domain usecases; opt-in `SyncPushable`/`SyncPullable`; value types in `pkg/synccontract`; envelope carries sync metadata, payload = existing domain DTO; `ApplyBatch` not per-record; no entity/repository; conformance suite required. | 🔴 | ☑ |
| 1.2 | ~~**Where changes come from**~~ | **☑ Settled → ADR §2.12 + §2.13.** `ChangesSince` reads the domain's own entity rows; `server_seq` authoritative on the row. No changelog, no event/projection feed — it would either give `sync` a cross-domain table (contradicting 1.1) or fragment into 5 duplicate tables. `server_seq` from a per-user counter incremented **inside the write transaction**; global Postgres sequence rejected (commit-order gap silently skips records). Reversible: the wire envelope is identical either way. | 🔴 | ☑ |
| 1.3 | ~~**`server_seq`: assignment point**~~ | **☑ Settled → ADR §2.15.** Assignment happens inside each pushable domain's repository (`Create`/`Update`/soft-delete), via a shared `pkg/syncseq` helper — not as a parameter the usecase passes in. Closes "a usecase forgot to call it" structurally, since no repo write path exposes `server_seq` as settable input. Does not close raw-SQL/migration bypass — see 1.6. | 🟡 | ☑ |
| 1.4 | ~~**Read model for `GET /sync/changes`**~~ | **☑ Collapsed by 1.1 + 1.2.** Both candidates were cross-domain constructs already excluded — a `UNION ALL` view over 5 domain tables *is* sync reading other domains' tables. What's left is not a discussion: index `(user_id, server_seq)` per table; pagination per ADR §2.11 (merge-sort, single `int64` cursor). | 🟡 | ☑ |
| 1.5 | ~~**Push semantics**~~ | **☑ Settled → ADR §2.14.** Conflict is real and is a *staleness* problem, not a concurrency one — device keying, ingest timestamps, and advisory locks were each tested and moved none of the three cases. Every pushed record carries `base_seq`; `row.server_seq > base_seq` = stale. Policy: **detect, apply under LWW, flag** — `conflict` means the write **did** land. Batch is **per record** (domain may group what must land together); a rejected record never blocks the rest. Plus two defect fixes: tombstones are sticky, budgets get natural key `(user_id, category_id, period)`. Plus sync log + `ingested_at`, operational only. | 🔴 | ☑ |
| 1.6 | ~~**Soft-delete enforcement**~~ | **☑ Settled → ADR §2.16.** Two separate answers: (a) read-time filter — domain `List`/`Get` always exclude `deleted_at`, sync's `ChangesSince` uses a separately named `ListAllIncludingDeleted`; (b) hard-delete ban — a `BEFORE DELETE` Postgres trigger on all 4 tables, not `REVOKE DELETE` (the app role is almost certainly table owner, so revoke has no effect on it). | 🔴 | ☑ |
| 1.7 | **Client-generated UUIDv7** | Who validates the format? What happens on collision, or an ID already owned by another user? *Narrowed by 1.1: applies only to pushable units — categories keep server-generated IDs.* | 🟡 | ☐ |
| 1.8 | **Fate of existing per-domain CRUD endpoints** | Keep them (future web client) or route everything through sync? *Constrained by 1.1: if kept they share the same DTO and must accept a client-supplied ID — two ID policies would be worse than either.* | 🟡 | ☐ |

## Track 2 — Auth & the offline window

Mostly independent of Track 1 — can run in parallel.

| # | Topic | What it decides | Weight | Status |
|---|---|---|---|---|
| 2.1 | **Refresh-token TTL** | The public promise of "how long you can stay offline". Proposed 60 days sliding. | 🟢 | ☐ |
| 2.2 | **Rotation + grace window** | Session table shape. Prevents lockout when the network drops mid-rotation while the device holds unsynced writes. | 🟡 | ☐ |
| 2.3 | **Session soft-limit (max 10) vs long-offline device** | An evicted session must land in the push-first path, never the clean-install path. | 🟡 | ☐ |
| 2.4 | **Three login cases** | Clean install / same user with local data / different user. How much does the server need to know, vs. client-side only? | 🔴 | ☐ |

## Track 3 — Receipt scanning

Fully independent of Tracks 1–2. Can be discussed or built at any time.

| # | Topic | What it decides | Weight | Status |
|---|---|---|---|---|
| 3.1 | **Domain shape** | No repository for receipts (nothing persisted) — but quota needs one. Precedent: `dashboard` is aggregation-only with no entity. | 🟡 | ☐ |
| 3.2 | **Extraction contract** | The JSON Schema for `output_config.format`: which fields, which are optional, how confidence is expressed. | 🔴 | ☐ |
| 3.3 | **Indonesian number rules + server validation** | `Rp 1.500,00` handling, PPN 11%, service charge. Range-check against line items before returning. | 🔴 | ☐ |
| 3.4 | **Quota + image-hash cache** | Per-user monthly cap (number needed). Redis for `hash → extraction`. | 🟢 | ☐ |
| 3.5 | **Synchronous vs async job** | Sync + spinner is simpler; async only if measurement demands it. Affects Gin tuning — a request held for seconds is a new load profile. | 🟡 | ☐ |
| 3.6 | **Model tier** | Default `claude-opus-5`. Stepping down is a quality/cost tradeoff — manager's call. | 🟢 | ☐ |

## Track 4 — Frontend

**Unblocked** — Track 1 items 1.1–1.5 are settled, so the contract this builds against is stable.

| # | Topic | What it decides | Weight | Status |
|---|---|---|---|---|
| 4.1 | **`lib/core/sync/` rewrite** | Cursor state, dirty tracking, retry/backoff (specified in TASK-FE-01, never built). | 🔴 | ☐ |
| 4.2 | **Deferred FK on batch apply** | Pull ordering by `server_seq` is not FK-safe on hydration. Defer FK checks inside the batch transaction. | 🟡 | ☐ |
| 4.3 | **Rejected-record UX** | **New surface.** What the user sees when the server refuses a transaction they entered days ago. | 🔴 | ☐ |
| 4.4 | **Repository layer cleanup scope** | Budget/History/Review/Dashboard/Profile currently call Drift directly. How far do we take it in this cycle? | 🟡 | ☐ |
| 4.5 | **Three login cases, client side** | Push-first-then-pull, isolate-on-different-user, never-wipe-on-token-expiry. | 🔴 | ☐ |
| 4.6 | **Sync status indicator** | `syncing` / `synced` / `failed` — specified but never built. | 🟢 | ☐ |
| 4.7 | **Budget feature** | Still a 9-line stub. Build once, on the new data layer — not now and again later. | 🟡 | ☐ |

## Track 5 — Contract & process

Downstream of Tracks 1–3.

| # | Topic | What it decides | Weight | Status |
|---|---|---|---|---|
| 5.1 | **OpenAPI changes** | New sync + receipt endpoints, schema changes. **Both copies** must move together. | 🟡 | ☐ |
| 5.2 | **Migrations** | Append-only per house rule — new files, never edit existing ones, `.up` + `.down` each. | 🟢 | ☐ |
| 5.3 | **QA matrix** | Offline → sync → conflict → resolve, plus the three login cases and the rejected-record path. | 🟡 | ☐ |
| 5.4 | **Work sequencing** | What runs in parallel, in which worktree, and where the merge points are. | 🟡 | ☐ |

---

## Verification still owed

Not discussion items — facts to confirm against the real code before they enter the contract.
Blocked on `elora-be-go` / `elora_spendos` being reachable (GitHub remotes are ~1–2 weeks
behind the status docs).

- [ ] Does `TxManager` propagation actually work across domains in practice, not just by design?
- [ ] Can system categories be renamed by users today?
- [ ] Is `is_default` currently a per-account flag (moving it to `user.default_account_id`)?
- [ ] What does `lib/core/sync/` actually implement today?
- [ ] Is the `event/` + `projection/` mechanism used by any domain yet, or purely aspirational?
      *No longer blocks 1.2 — sync does not use it either way — but still worth knowing.*
- [ ] **Is per-user category hide/show stored in a table separate from the category definition?**
      If so, that table's writes must bump the category row's `server_seq` (ADR §2.12), or hiding
      a category never reaches the device.
- [ ] **Which write paths hard-delete today** across `transactions`, `accounts`, `categories`,
      `budgets` — including migrations, cleanup jobs, and cascading FKs? Each is a zombie-record
      source under ADR §2.13.
