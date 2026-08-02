# Discussion backlog — offline-first sync

**2026-08-01 — `ADR-0004-online-required-writes.md` superseded the offline-sync scope of this
backlog.** The manager stepped back to an online-required-write model before any production user
exists — implementation cost was judged too high for the product's current stage. Tracks 1, 2,
and 4 are now mostly **superseded**, marked below with the reason; the settled reasoning is kept,
not deleted, since the design itself wasn't wrong. Track 5 shrinks a lot — the contract is now
standard CRUD, not a sync protocol.

**2026-08-02 — Track 3 (receipt scanning) removed.** The manager cut receipt scanning from
product scope entirely — not deferred, not tracked elsewhere, just no longer a planned feature.
The track's discussion items are removed from this backlog rather than kept as superseded, since
there is no successor decision or document carrying the topic forward. If the feature is ever
reconsidered, the prior design lives in `notes/ADR-0003-offline-first-sync-and-receipt-scan.md`
§3 and in this file's git history.

Working list for the manager and the architect. Ordered by dependency: items inside a track
should be settled in order; tracks 1–2 can run in parallel.

**Weight:** 🔴 deep design discussion · 🟡 medium · 🟢 quick decision (mostly manager's call)
**Status:** ☐ open · ☑ settled (link the ADR section or note when closing) · 🚫 superseded
(no longer applicable — see note)

Context: `notes/ADR-0004-online-required-writes.md` (current) and
`notes/ADR-0003-offline-first-sync-and-receipt-scan.md` (superseded for sync, historical
reference for the reasoning trail). `shared/context/PRD-online-required-writes.md` is
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
| 2.1 | ~~**Refresh-token / session TTL**~~ | **☑ Settled 2026-08-02 — 60 days, sliding.** *Reframed by ADR-0004 §2.4: just "how long before you must log in again," not an offline-window promise.* Manager confirmed the carried-over default. | 🟢 | ☑ |
| 2.2 | ~~**Rotation + grace window**~~ | **☑ Settled 2026-08-02 — build it as described, no further spec needed.** Was: session table shape to prevent lockout when the network drops mid-rotation while the device holds unsynced writes. **Downgraded by ADR-0004 §2.4 to an ordinary UX nicety** (avoid a spurious logout on a network blip) — no longer a correctness requirement, since there's no unsynced data at risk. Manager confirmed the default: worth doing, not urgent, no specific TTL/grace-window number beyond that. | 🟢 | ☑ |
| 2.3 | ~~**Session soft-limit (max 10) vs long-offline device**~~ | Was: an evicted session must land in the push-first path, never the clean-install path. **Superseded by ADR-0004 §2.4 — eviction is just "log in again," identical to any other token expiry; there's no unsynced local data to protect.** | 🟡 | 🚫 |
| 2.4 | ~~**Three login cases**~~ | Was: clean install / same user with local data / different user, and how much the server needs to know. **Superseded by ADR-0004 §2.3 — login collapses to one case: authenticate, fetch from server, populate cache.** | 🔴 | 🚫 |

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

Downstream of Tracks 1–2. **Scope shrank a lot on 2026-08-01, and further on 2026-08-02** — the
contract is now standard per-domain CRUD (mostly already exists), not a sync protocol, and there
is no receipt-scan endpoint since that feature was cut.

| # | Topic | What it decides | Weight | Status |
|---|---|---|---|---|
| 5.1 | **OpenAPI changes** | *Reframed by ADR-0004: no sync endpoints, no receipt-scan endpoint.* Whatever schema/CRUD adjustments fall out of dropping client-generated ids and tombstones (§2.2). **Both copies** must move together. | 🟢 | ☐ |
| 5.2 | **Migrations** | *Reframed: no `server_seq`/`deleted_at` columns to add for sync purposes.* Append-only per house rule regardless — new files, never edit existing ones, `.up` + `.down` each. | 🟢 | ☐ |
| 5.3 | ~~**QA matrix**~~ | Was: offline → sync → conflict → resolve, plus three login cases and rejected-record path. **Superseded by ADR-0004 — replaced by a much smaller matrix: online-write happy path and the "needs connection" gate when offline.** | 🟡 | 🚫 |
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
