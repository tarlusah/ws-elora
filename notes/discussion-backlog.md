# Discussion backlog — offline-first sync + receipt scanning

**2026-08-01 — `ADR-0004-online-required-writes-and-receipt-scan.md` superseded the offline-sync
scope of this backlog.** The manager stepped back to an online-required-write model before any
production user exists — implementation cost was judged too high for the product's current
stage. Tracks 1, 2, and 4 are now mostly **superseded**, marked below with the reason; the
settled reasoning is kept, not deleted, since the design itself wasn't wrong. Track 3 (receipt
scanning) is essentially unaffected and now lives in ADR-0004 §3. Track 5 shrinks a lot — the
contract is now standard CRUD + one receipt-scan endpoint, not a sync protocol.

Working list for the manager and the architect. Ordered by dependency: items inside a track
should be settled in order; tracks 1–3 can run in parallel.

**Weight:** 🔴 deep design discussion · 🟡 medium · 🟢 quick decision (mostly manager's call)
**Status:** ☐ open · ☑ settled (link the ADR section or note when closing) · 🚫 superseded
(no longer applicable — see note)

Context: `notes/ADR-0004-online-required-writes-and-receipt-scan.md` (current) and
`notes/ADR-0003-offline-first-sync-and-receipt-scan.md` (superseded for sync, historical
reference for the reasoning trail). `shared/context/PRD-offline-first-and-receipt-scan.md` is
being updated to match ADR-0004.
Architecture rules in play: `.claude/skills/backend/*.md`.

**Superseded 2026-08-01 — no longer load-bearing:** sync direction (was settled with 1.1), pull
source / `server_seq` (was settled with 1.2), push semantics / `base_seq` / conflict (was settled
with 1.5). See ADR-0004 §2 for what replaces all three: plain server-authoritative CRUD, no sync
protocol.

---

## Track 1 — Sync core (backend)

**🚫 Superseded 2026-08-01 by ADR-0004.** There is no sync protocol to build — every domain's
write path is plain CRUD (ADR-0004 §2.2), which also resolves 1.8 by default: there was never a
competing write path to reconcile it against. The reasoning below is kept for the record; none of
it is load-bearing anymore.

| # | Topic | What it decides | Weight | Status |
|---|---|---|---|---|
| 1.1 | ~~**Shape of the new `sync` domain**~~ | Was settled → ADR-0003 §2.11 (thin orchestrator; opt-in `SyncPushable`/`SyncPullable`; `pkg/synccontract`; conformance suite). **Superseded by ADR-0004 §2.2 — no `sync` domain exists.** | 🔴 | 🚫 |
| 1.2 | ~~**Where changes come from**~~ | Was settled → ADR-0003 §2.12 + §2.13 (`ChangesSince` reads entity rows; `server_seq` from a per-user in-transaction counter). **Superseded by ADR-0004 §2.2 — no pull-cursor, no `server_seq`.** | 🔴 | 🚫 |
| 1.3 | ~~**`server_seq`: assignment point**~~ | Was settled → ADR-0003 §2.15 (assignment inside the repository, not the usecase). **Superseded — `server_seq` doesn't exist under ADR-0004.** | 🟡 | 🚫 |
| 1.4 | ~~**Read model for `GET /sync/changes`**~~ | Was collapsed by 1.1 + 1.2. **Superseded — the endpoint doesn't exist under ADR-0004.** | 🟡 | 🚫 |
| 1.5 | ~~**Push semantics**~~ | Was settled → ADR-0003 §2.14 (`base_seq`, LWW, conflict = applied, sticky tombstones, budget natural key). **Superseded by ADR-0004 §2.1/§2.2 — no push, no conflict detection; writes are synchronous and either succeed or fail per request.** | 🔴 | 🚫 |
| 1.6 | ~~**Soft-delete enforcement**~~ | Was settled → ADR-0003 §2.16 (read-time filter + `BEFORE DELETE` trigger). **Superseded by ADR-0004 §2.2 — hard delete is fine again; no delta-pull reading rows means no zombie-record risk.** | 🔴 | 🚫 |
| 1.7 | ~~**Client-generated UUIDv7**~~ | Was open — format validation, collision, cross-user id reuse. **Superseded by ADR-0004 §2.2 — `id` is server-generated again; the whole class of problem (client picks an id before ever syncing) doesn't exist.** | 🟡 | 🚫 |
| 1.8 | ~~**Fate of existing per-domain CRUD endpoints**~~ | Was open — keep vs. route through sync. **Resolved by default under ADR-0004 §2.2 — CRUD is the only write path there ever is; nothing to reconcile it against.** | 🟡 | 🚫 |

## Track 2 — Auth & session

Mostly independent of Track 1 — can run in parallel. **Reframed 2026-08-01 by ADR-0004 §2.4**:
this is no longer about protecting unsynced local writes — there aren't any. Stakes are lower
across the board.

| # | Topic | What it decides | Weight | Status |
|---|---|---|---|---|
| 2.1 | **Refresh-token / session TTL** | *Reframed by ADR-0004 §2.4: just "how long before you must log in again," not an offline-window promise.* Still a number the manager needs to pick. Proposed 60 days sliding, carried over as the default. | 🟢 | ☐ |
| 2.2 | ~~**Rotation + grace window**~~ | Was: session table shape to prevent lockout when the network drops mid-rotation while the device holds unsynced writes. **Downgraded by ADR-0004 §2.4 to an ordinary UX nicety** (avoid a spurious logout on a network blip) — no longer a correctness requirement, since there's no unsynced data at risk. Worth doing, but not urgent. | 🟢 | ☐ |
| 2.3 | ~~**Session soft-limit (max 10) vs long-offline device**~~ | Was: an evicted session must land in the push-first path, never the clean-install path. **Superseded by ADR-0004 §2.4 — eviction is just "log in again," identical to any other token expiry; there's no unsynced local data to protect.** | 🟡 | 🚫 |
| 2.4 | ~~**Three login cases**~~ | Was: clean install / same user with local data / different user, and how much the server needs to know. **Superseded by ADR-0004 §2.3 — login collapses to one case: authenticate, fetch from server, populate cache.** | 🔴 | 🚫 |

## Track 3 — Receipt scanning

Fully independent of Tracks 1–2. Can be discussed or built at any time. **Essentially unaffected
by the 2026-08-01 ADR-0004 pivot** — now lives in ADR-0004 §3 instead of ADR-0003 §3, with one
flow simplification (no offline scan queue, since nothing is offline-capable for writes anymore).
All items below are still open exactly as before.

| # | Topic | What it decides | Weight | Status |
|---|---|---|---|---|
| 3.1 | **Domain shape** | No repository for receipts (nothing persisted) — but quota needs one. Precedent: `dashboard` is aggregation-only with no entity. | 🟡 | ☐ |
| 3.2 | **Extraction contract** | The JSON Schema for `output_config.format`: which fields, which are optional, how confidence is expressed. | 🔴 | ☐ |
| 3.3 | **Indonesian number rules + server validation** | `Rp 1.500,00` handling, PPN 11%, service charge. Range-check against line items before returning. | 🔴 | ☐ |
| 3.4 | **Quota + image-hash cache** | Per-user monthly cap (number needed). Redis for `hash → extraction`. | 🟢 | ☐ |
| 3.5 | **Synchronous vs async job** | Sync + spinner is simpler; async only if measurement demands it. Affects Gin tuning — a request held for seconds is a new load profile. | 🟡 | ☐ |
| 3.6 | **Model tier** | Default `claude-opus-5`. Stepping down is a quality/cost tradeoff — manager's call. | 🟢 | ☐ |

## Track 4 — Frontend

**Reframed 2026-08-01 by ADR-0004 §2.5.** No bidirectional sync engine to build — what's needed
is a read-cache refresh routine plus a connectivity gate in front of writes (ADR-0004 §2.1).

| # | Topic | What it decides | Weight | Status |
|---|---|---|---|---|
| 4.1 | ~~**`lib/core/sync/` rewrite**~~ | Was: cursor state, dirty tracking, retry/backoff for a bidirectional engine. **Superseded by ADR-0004 §2.5 — replaced by a much smaller read-cache refresh (paginated fetch + upsert into Drift) plus a write-side connectivity gate.** | 🔴 | 🚫 |
| 4.2 | ~~**Deferred FK on batch apply**~~ | Was: pull ordering by `server_seq` isn't FK-safe on hydration; defer FK checks in the batch transaction. **Superseded by ADR-0004 §2.5 — writes are single-record and synchronous; the server validates FKs the way any ordinary REST API does.** | 🟡 | 🚫 |
| 4.3 | ~~**Rejected-record UX**~~ | Was: new surface for a transaction the server refuses days after entry. **Superseded by ADR-0004 §2.1 — a write either succeeds in that request or fails immediately with "needs connection"; there's no delayed rejection to surface.** | 🔴 | 🚫 |
| 4.4 | **Repository layer cleanup scope** | Budget/History/Review/Dashboard/Profile currently call Drift directly. *Reframed by ADR-0004 §2.5: no longer bundled into this cycle by necessity (it rode along with the sync rewrite before) — still worth doing, now an independently-scheduled decision on its own merits.* | 🟡 | ☐ |
| 4.5 | ~~**Three login cases, client side**~~ | Was: push-first-then-pull, isolate-on-different-user, never-wipe-on-token-expiry. **Superseded by ADR-0004 §2.3 — login collapses to one case: authenticate, fetch, populate cache.** | 🔴 | 🚫 |
| 4.6 | ~~**Sync status indicator**~~ | Was: `syncing` / `synced` / `failed`. **Superseded by ADR-0004 §2.5 — at most a simple cache-freshness label ("data as of [time]"); no sync state machine to indicate.** | 🟢 | 🚫 |
| 4.7 | **Budget feature** | Still a 9-line stub. Unaffected by the ADR-0004 pivot either way — still its own open item. | 🟡 | ☐ |

## Track 5 — Contract & process

Downstream of Tracks 1–3. **Scope shrank a lot on 2026-08-01** — the contract is now standard
per-domain CRUD (mostly already exists) plus one new receipt-scan endpoint, not a sync protocol.

| # | Topic | What it decides | Weight | Status |
|---|---|---|---|---|
| 5.1 | **OpenAPI changes** | *Reframed by ADR-0004: no sync endpoints.* Just `POST /v1/receipts/scan`, plus whatever schema/CRUD adjustments fall out of dropping client-generated ids and tombstones (§2.2). **Both copies** must move together. | 🟢 | ☐ |
| 5.2 | **Migrations** | *Reframed: no `server_seq`/`deleted_at` columns to add for sync purposes.* Append-only per house rule regardless — new files, never edit existing ones, `.up` + `.down` each. | 🟢 | ☐ |
| 5.3 | ~~**QA matrix**~~ | Was: offline → sync → conflict → resolve, plus three login cases and rejected-record path. **Superseded by ADR-0004 — replaced by a much smaller matrix: online-write happy path, "needs connection" gate when offline, and the receipt-scan flow.** | 🟡 | 🚫 |
| 5.4 | **Work sequencing** | What runs in parallel, in which worktree, and where the merge points are. Unaffected by the pivot, still open. | 🟡 | ☐ |

---

## Verification still owed

Not discussion items — facts to confirm against the real code before they enter the contract.
Blocked on `elora-be-go` / `elora_spendos` being reachable (GitHub remotes are ~1–2 weeks
behind the status docs). **Trimmed 2026-08-01** — items that only mattered for the sync protocol
are removed; general facts worth knowing regardless of ADR-0004 are kept.

- [ ] Does `TxManager` propagation actually work across domains in practice, not just by design?
- [ ] Can system categories be renamed by users today?
- [ ] Is `is_default` currently a per-account flag (moving it to `user.default_account_id`)?
- [ ] What does `lib/core/sync/` actually implement today? *Reframed: no longer "what to rebuild
      into a cursor engine" — now "what to strip down to a plain read-cache refresh" (ADR-0004 §2.5).*
- [ ] Is the `event/` + `projection/` mechanism used by any domain yet, or purely aspirational?
