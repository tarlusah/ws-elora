# ADR 0004 — Online-required writes (no offline sync) + LLM receipt scanning

- **Status:** Accepted
- **Date:** 2026-08-01
- **Decided by:** Manager (human)
- **Scope:** Both repos — `elora-be-go` and `elora_spendos`
- **Destination:** copy to `elora-be-go/docs/decisions/0004-online-required-writes-and-receipt-scan.md`
  once that repo is checked out. This workspace clone has no access to the `playgrounds/*`
  repos, so the record lives here first.
- **Supersedes:** the offline-sync portion of
  `notes/ADR-0003-offline-first-sync-and-receipt-scan.md` (§2.1–§2.14 and the login-case
  content in §2.9). The receipt-scanning decision (§3 there) is carried forward and restated
  here, with one flow simplification — see §6.

---

## 1. Context

ADR-0003 committed to full bidirectional offline-first sync: cursor-based delta pull, push with
`base_seq` staleness detection, client-generated UUIDv7 identity from birth, soft-delete
tombstones, a dedicated `sync` domain, a conformance test suite, and a rejected-record UI surface.
Discussion items 1.1–1.7 worked through the mechanics of that design and settled cleanly — the
design was sound.

What changed is not the design's soundness but its **cost relative to the product's stage**: there
is no production user yet, and the manager judged the implementation cost of full bidirectional
sync — on both `elora-be-go` and `elora_spendos` — too expensive to take on before a single user
exists. Because there is no production data or existing user base to protect, stepping back now
costs nothing that stepping back later would not cost far more.

**Decision: step back to an online-required-write model for this cycle.** Local read caching
stays — the app still shows the last data it fetched when offline — but creating, editing, or
deleting anything requires an active connection at the moment of the action. This is explicitly
staged, not final: full offline-first sync (ADR-0003's design) remains available to revisit in a
future cycle if the product needs it later, unchanged in its technical soundness, just deferred.

---

## 2. Decision

### 2.1 Read-through cache, online-required writes

- The local Drift database keeps its role as a **cache of what was last fetched from the
  server** — not a source of truth, and not a write buffer. Viewing accounts, transactions,
  budgets, categories, and dashboard aggregates works offline, showing whatever was last pulled.
- **Business writes require a live connection at the moment of the action**: creating, editing,
  or deleting a transaction, account, or budget; creating, editing, or hiding a category.
  If the device is offline, the action fails immediately with an explicit "needs connection"
  message. **It is not queued, and no local draft is created for it to resolve later.**
- There is therefore no local write queue, no dirty tracking, no retry/backoff for writes, no
  conflict detection, and no rejected-record concept. A write either succeeds against the server
  in that request, or the user is told to reconnect and try again — the same failure mode any
  ordinary online-only app has.
- Refreshing the local cache is a plain read: on app foreground and after every successful write,
  the client re-fetches the relevant lists via ordinary paginated REST `GET` endpoints and
  upserts into Drift. This is **not** cursor/delta based — there is no `server_seq`, no cursor,
  nothing to reconcile. At this data volume (§2.6 of ADR-0003 estimated ~18,000 transaction rows
  over 5 years), a plain paginated re-fetch is cheap enough that a smarter incremental scheme
  isn't worth building.

### 2.2 Data model reverts to plain server-authoritative CRUD

- **`id` is server-generated again.** The reason it had to be client-generated in ADR-0003 —
  "records are identified from birth," so a locally-created record could reference another
  locally-created record before either had ever synced — doesn't apply. Every record's first
  write is itself a round trip to the server, so the server can safely mint the id the same way
  any ordinary REST API does.
- **Hard delete is fine again.** It was banned in ADR-0003 §2.13 because a client pulling deltas
  would never learn about a row removed outside the tombstone path, leaving a permanent zombie
  record. Without a delta-pull protocol reading rows, that risk doesn't exist. Default: plain
  `DELETE`, no `deleted_at` column. If a product reason surfaces later to want an "undo delete"
  UX, that is a small, separate decision — not something this ADR needs to force.
- **`server_seq` is removed entirely.** It existed only to give pull a cursor and to give push a
  staleness check. Neither concern exists without bidirectional sync.
- **The `sync` domain, the delta endpoints (`GET`/`POST /v1/sync/changes`), and the old retired
  sync log (`POST /v1/sync`, `GET /v1/sync/status`, `PUT /v1/sync/{id}`) are all unnecessary.**
  Standard per-domain REST CRUD is the only write surface for every domain, pushable or not —
  which also resolves discussion item 1.8 by default: there is no competing write path to
  reconcile CRUD against, because CRUD is simply the only write path there ever is.

### 2.3 The "three login cases" collapse to one

ADR-0003 §2.9 spent real design weight on "clean install" vs. "same user, device holds unsynced
writes" vs. "different user on the same device," because the middle case risked silently losing
data that existed only on the device. That risk doesn't exist here: a write that succeeded already
landed on the server: a write that failed because the device was offline was never recorded
anywhere on the device either, so there is nothing on the device to lose or reconcile.

Login is now uniformly: authenticate, fetch current data from the server, populate (or replace)
the local cache. Switching users on the same device: the previous user's cached rows are simply
cleared before hydrating the new user's data — no risk calculus needed, because the cache is
disposable and derived, never authoritative.

### 2.4 Auth and session

- **Refresh-token TTL is still an open decision** (see §5), but it is no longer "the public
  promise of how long you can work offline with unsynced data at risk" — it is just an ordinary
  session length: how long before the user must log in again. The stakes are materially lower
  than in ADR-0003.
- **Session eviction (soft-limit 10)** no longer needs the push-first handling ADR-0003 §2.9
  required — an evicted session just means "log in again," identical to any other token expiry,
  because there is no unsynced local data at risk of being orphaned.
- **Refresh-token rotation + grace window** is still worth having, to avoid spurious logouts on a
  transient network blip during rotation — but as an ordinary UX nicety, not the correctness
  requirement it was when unsynced writes hung in the balance.

### 2.5 Frontend

- The `lib/core/sync/` rewrite (ADR-0003 item 4.1) is no longer a bidirectional sync engine. What
  replaces it is much smaller: a read-cache refresh routine (paginated fetch + upsert into Drift)
  and a connectivity gate in front of every write action.
- No dirty tracking, no retry/backoff for writes, no rejected-record UI, no sync status indicator
  beyond perhaps a simple "data as of [time]" freshness label on cached views.
- No deferred-FK handling on batch apply (ADR-0003 item 4.2) — writes are single-record and
  synchronous; the server validates foreign keys the same way any ordinary REST API already does.
- **Repository-layer cleanup** (Budget/History/Review/Dashboard/Profile calling Drift directly)
  is still worth doing on its own merits, but it is no longer bundled into this cycle by
  necessity — ADR-0003 folded it in because the same data paths were already being torn open for
  the sync rewrite. It can now be scheduled independently, at whatever priority the manager wants.
- The **budget feature** (still a 9-line stub) is unaffected either way by this decision and
  remains its own open item.

---

## 3. LLM receipt scanning

Carried forward from ADR-0003 §3, unchanged except for §3.1's flow (no more offline scan queue,
consistent with §2.1 above — everything requires connectivity now, so scanning is no longer a
special case).

### 3.1 User flow

1. User captures or picks a receipt image from Capture.
2. Image is compressed and uploaded to the server — this requires connectivity, same as any other
   write now.
3. The server calls the model and returns structured extraction.
4. A transaction draft is created **with the extracted fields already populated**, landing in the
   existing Review queue as a suggestion.
5. The user confirms. Nothing is auto-committed.

If the device is offline, the scan action is simply unavailable — the same "needs connection"
gate as every other write (§2.1), not a queued draft that resolves later. This is the one
substantive change from ADR-0003 §3.1, which had scanning fall back to a local `scan_pending`
queue processed on reconnect; that fallback no longer has anything to be an exception to.

### 3.2 Accuracy requirements (Indonesian receipts) — unchanged

- Thousands separator `.`, decimal separator `,` (`Rp 1.500,00`) — the most likely source of a
  100× or 1000× error, handled explicitly in the prompt and validated server-side.
- PPN 11% and service charge lines must not be mistaken for the total.
- The server range-checks the extracted total against recognised line items before returning.
- Low-confidence extractions are marked as such, never presented as certain.
- Extraction is always a suggestion; the user always confirms. Nothing is auto-committed.

### 3.3 Images are never synced — unchanged, restated more simply

Still local-only, discarded server-side after extraction. ADR-0003's rationale ("would otherwise
contaminate a clean row-sync design") no longer applies — there is no row-sync design to
contaminate — but the simpler reason still holds: most users never revisit a receipt photo, so
building cross-device image transport for it isn't worth it.

### 3.4 The model call runs server-side — unchanged

`claude-opus-5` via `github.com/anthropics/anthropic-sdk-go`, structured output via
`output_config.format`, vision input, prompt caching on the extraction system prompt, idempotency
by image hash, synchronous request with a spinner. API key never ships in the app. See ADR-0003
§3.4 for the full detail — none of it changes here.

### 3.5 Cost — unchanged

Roughly $0.02–0.04 per scan at `claude-opus-5` rates; a per-user monthly cap and image-hash cache
are v1 requirements, not deferred. See ADR-0003 §3.5.

### 3.6 Contract impact

```
POST /v1/receipts/scan     multipart image → structured extraction
```

Unchanged from ADR-0003 §3.6.

### 3.7 Domain shape — everything in Postgres, no Redis

*Settled in discussion item 3.1.* The `receipt` domain has no entity or repository for the
receipt/extraction content itself — nothing about it is ever persisted (§3.3). But two smaller,
genuinely-owned pieces of state do need a home:

- **Monthly scan quota**, `(user_id, period)` unique, a `count` column. Must be durable — a
  cost-control cap that silently resets on a cache restart defeats its own purpose.
- **Idempotency by image hash** (§3.4), `(user_id, image_hash) → extraction_result` (JSONB),
  unique per pair. No active expiry needed at this data volume; rows are small and low-frequency
  given the monthly cap above.

**Both live in Postgres, in `internal/receipt/repository/receipt_repo.go` — there is no Redis
anywhere in this stack** (confirmed 2026-08-01; `CLAUDE.md` updated to match). This also makes
the idempotency table more consistent with the codebase's existing convention than a new Redis
dependency would have been — §3.4 already pointed at "the existing transaction idempotency-key
pattern," which is itself a Postgres pattern.

**Why not client-side / BYOK.** Considered and rejected: Spendos's users are casual, non-technical
personal-finance users, not developers who already hold an Anthropic API key — requiring one to
use the marquee scanning feature would be a significant adoption barrier. It would also move the
extraction prompt and validation rules into the shipped APK, giving up the ability to fix a
systematic misread (e.g. an Indonesian number-format bug) via server deploy instead of an app
release. The image still passes through the server on every scan, but only transiently — it is
never written to disk, which is what "not stored on the server" actually requires; the server's
role is to hold the shared API key safely, not to persist the photo.

---

## 4. Consequences

### Positive

- Large scope reduction for this cycle: no cursor-delta protocol, no conflict resolution, no
  rejected-record UX, no client-generated-ID collision handling, no per-user `server_seq` counter,
  no hard-delete-ban trigger, no conformance test suite.
- Both surfaces get a standard, well-understood REST CRUD implementation instead of a novel sync
  protocol either team would be building for the first time.
- The local Drift cache still earns its keep — fast, offline-capable reads — without needing to
  be a write source of truth, which is the harder half of what made ADR-0003 expensive.

### Negative / accepted costs

- **The product can no longer capture a transaction — typed or receipt-scanned — without a live
  connection at that moment.** This is a real capability regression from the offline-first vision
  ADR-0003 described, accepted explicitly because there is no production user base yet to be
  affected by it, and because the cost of building it now was judged higher than the value of
  having it before any user exists.
- If offline writes are wanted in a future cycle, this is not a small patch to reverse — it
  touches ID generation, delete semantics, the auth/session model, and the entire frontend data
  layer. ADR-0003's design remains available and sound if that cycle comes.

---

## 5. Open items requiring manager confirmation

Carried over from ADR-0003, still unanswered:

1. **Refresh-token / session TTL.** Now just "how long before the user must log in again" —
   materially lower-stakes than ADR-0003's "offline window" framing, but still a number to pick.
2. **Model tier for receipt scanning.** Default `claude-opus-5`. Stepping down to `claude-sonnet-5`
   or `claude-haiku-4-5` is a quality/cost tradeoff — manager's call.
3. **Per-user monthly scan cap.** A number is needed.

No longer forced by this ADR (now a free, independent choice if it ever comes up):

4. Hard delete vs. soft delete. Default here is hard delete; soft delete is only worth
   reconsidering if a separate "undo delete" UX is wanted.

---

## 6. Supersedes

- **`notes/ADR-0003-offline-first-sync-and-receipt-scan.md` §2.1–§2.14** (offline level, sync
  protocol, schema for sync, sync domain architecture, pull source, push semantics, soft-delete
  enforcement, `server_seq` assignment) and the login-case content in §2.9 — **closed**. Reason:
  implementation cost judged too high for the product's pre-production stage; the design itself
  was not wrong, and remains available to revisit in a future cycle.
- **ADR-0003 §3** (receipt scanning) — carried forward into this ADR's §3 with one simplification
  (no offline scan queue). ADR-0003 §3 is not wrong, just consolidated here so there is one
  authoritative document going forward.
- **Discussion backlog items 1.1–1.8, 2.2, 2.3, 2.4, 4.1, 4.2, 4.3, 4.5, 4.6** — superseded, no
  longer applicable. See `notes/discussion-backlog.md` for the per-item note.
- ADR-0001's closure by ADR-0003 is untouched — this ADR does not reopen it.
