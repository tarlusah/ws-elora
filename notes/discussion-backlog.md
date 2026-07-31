# Discussion backlog — offline-first sync + receipt scanning

Working list for the manager and the architect. Ordered by dependency: items inside a track
should be settled in order; tracks 1–3 can run in parallel.

**Weight:** 🔴 deep design discussion · 🟡 medium · 🟢 quick decision (mostly manager's call)
**Status:** ☐ open · ☑ settled (link the ADR section or note when closing)

Context already settled: `notes/ADR-0003-offline-first-sync-and-receipt-scan.md`,
`shared/context/PRD-offline-first-and-receipt-scan.md`.
Architecture rules in play: `.claude/skills/backend/*.md`.

---

## Track 1 — Sync core (backend)

Blocking. Everything in Track 4 waits on this, and the API contract can't be written until
1–5 are settled.

| # | Topic | What it decides | Weight | Status |
|---|---|---|---|---|
| 1.1 | **Shape of the new `sync` domain** | Is `sync` an orchestrator or still a peer domain? Which interfaces does it define (consumer-defined, per house rule)? How do the 4 domains satisfy them? **Start here — 1.2–1.5 all flow from it.** | 🔴 | ☐ |
| 1.2 | **Where changes come from** | Event + projection (idiomatic, but in-memory bus isn't durable) vs. reading entity rows directly. Ties to whether `server_seq` is authoritative on the row. | 🔴 | ☐ |
| 1.3 | **`server_seq`: assignment point + generator** | Must it live in the repository layer so non-sync write paths also get a sequence? Global Postgres sequence vs per-user counter. | 🟡 | ☐ |
| 1.4 | **Read model for `GET /sync/changes`** | `UNION ALL` view over 5 tables vs a changelog table. Determines pagination and indexing. | 🟡 | ☐ |
| 1.5 | **Push semantics** | Per-record `applied` / `conflict` / `rejected`. What even counts as a conflict on a single-device product? Is the whole batch atomic or per-record? | 🔴 | ☐ |
| 1.6 | **Soft-delete enforcement** | `deleted_at IS NULL` touches every read in 4 domains. Shared helper, base repo, or discipline? One miss = deleted records resurrect. | 🟡 | ☐ |
| 1.7 | **Client-generated UUIDv7** | Who validates the format? What happens on ID collision or an ID that already belongs to another user? | 🟡 | ☐ |
| 1.8 | **Fate of existing per-domain CRUD endpoints** | Keep them (future web client) or route everything through sync? If kept, they're a second write path that must also assign `server_seq`. | 🟡 | ☐ |

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

Needs Track 1 (items 1.1–1.5) settled first, because it builds against the contract.

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
