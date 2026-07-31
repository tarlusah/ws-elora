# ADR 0003 — Offline-first sync (cursor delta) + LLM receipt scanning

- **Status:** Accepted (supersedes and closes `elora-be-go/docs/decisions/0001-defer-sync-rewrite.md`)
- **Date:** 2026-07-31
- **Decided by:** Manager (human)
- **Scope:** Both repos — `elora-be-go` and `elora_spendos`
- **Destination:** copy to `elora-be-go/docs/decisions/0003-offline-first-sync-and-receipt-scan.md`
  once that repo is checked out. This workspace clone has no access to the `playgrounds/*`
  repos, so the record lives here first.

---

## 1. Context

Decision 0001 (2026-05-08) deferred the sync rewrite pending four answers from the owner.
Those answers are now in:

| Question from 0001 | Answer |
|---|---|
| Is multi-device needed? | **No.** One account, sequential device use. |
| Must the app work before any login? | **No.** Login is required. |
| Is offline-forever required? | **No.** Bounded offline window, tied to refresh-token lifetime. |
| OK to break current DELETE semantics? | **Yes.** No production data exists — no backward compatibility required. |

Additionally, a new capability enters scope in the same cycle: **LLM-based receipt/invoice
scanning**, which introduces image attachments that Spendos did not previously have.

Both surfaces had independently simplified the sync feature and never reconciled (see
`shared/context/PRD.md` §4). This ADR reconciles them.

---

## 2. Decision

### 2.1 Offline-first level

Spendos targets **local-as-source-of-truth with mandatory sync** ("L2"):

- Login is required. The app is never usable without an account.
- Once logged in, all business flows work offline for the lifetime of the refresh token.
- Sync is **bidirectional**, cursor-based, and mandatory — not optional backup.
- Multi-device is **out of scope**; conflict handling covers only the sequential-device
  window (device A holds unsynced writes while the user works on device B).

### 2.2 No concurrent writers → no clock trust required

Because there is never more than one active writer, ordering does not require a hybrid
logical clock, version vectors, or CRDTs. **The server assigns a monotonically increasing
`server_seq` per user at ingest.** Device clocks are never trusted for conflict resolution.

Conflict policy: **last-write-wins per record**, adjudicated by server ingest order.

### 2.3 Sync protocol

The existing generic sync log (`POST /v1/sync`, `GET /v1/sync/status`, `PUT /v1/sync/{id}`)
is **retired, not extended**. It is replaced by a cursor-based delta pair:

```
GET  /v1/sync/changes?since=<seq>&limit=<n>
     → { protocol_version, changes: [...], next_cursor, has_more }

POST /v1/sync/changes
     → { protocol_version, results: [ { client_id, status, server_seq } ], next_seq }
     status ∈ applied | conflict | rejected
```

- A fresh login is the same endpoint with `since=0`. Bootstrap is not a special code path;
  it is the extreme case of ordinary sync.
- `protocol_version` is present from day one. The server rejects clients that are too old
  with an explicit error rather than failing silently.
- Pagination (`limit`, `next_cursor`, `has_more`) is mandatory from day one. No response
  may assume the whole dataset fits.

### 2.4 Push ordering is part of the contract

Foreign keys must resolve at ingest. The client MUST push in this order, and the server
MUST hydrate in this order:

```
accounts → categories (parent before child) → budgets → transactions (parent before split child)
```

This is a contract requirement, not an implementation detail.

### 2.5 Schema changes

Applied to `transactions`, `accounts`, `categories`, `budgets` in **both** PostgreSQL and
Drift:

| Column | Owner | Purpose |
|---|---|---|
| `id` | client | UUIDv7, generated client-side. Records are identified from birth. |
| `updated_at` | client | Informational / display ordering only. **Never** used to resolve conflicts. |
| `deleted_at` | either | Tombstone. Replaces hard delete. |
| `server_seq` | server | Assigned at ingest. Client stores it as its cursor position. |

Client-side additionally: `sync_state` (`clean` / `dirty` / `rejected`).

**DELETE becomes soft delete.** This is breaking and is accepted — no production data exists.

Because there is no data to preserve, Drift requires **no migration steps**: bump the schema
version and recreate the local database. PostgreSQL migrations may drop and recreate freely.

### 2.6 What syncs

Test applied: *would the user be upset to lose this if their phone died?*

| Data | Direction | Rationale |
|---|---|---|
| Transactions | ↕ two-way | Core product data. Includes `000` placeholders, review state, splits. |
| Accounts | ↕ two-way | Small, required offline by Capture. |
| Categories — user-created | ↕ two-way | Must be creatable offline. |
| Categories — system/seeded | ⬇ server → device | Definition is server-owned; client never creates or edits. |
| Category user-state (hide/show) | ↕ two-way | This is *user* state, not category definition. See §2.7. |
| Budgets | ↕ two-way | Small; per category per month. LWW is safe. |
| User profile (`/me`) | ⬇ server → device | Profile edits require connectivity. See §2.8. |
| Dashboard aggregates | ✗ not synced | Computed locally from Drift (already the case). |
| Sessions / tokens | ✗ local only | — |
| UI preferences, onboarding flags | ✗ local only | Low value; can be added later as one small key-value entity. |
| Sync bookkeeping (`last_seq`, dirty flags, retry queue) | ✗ local only | The sync machinery itself. |
| **Receipt images** | ✗ **never synced** | See §3.3. |

**No time-windowing.** The full business dataset syncs. Rough sizing: ~10 transactions/day
over 5 years ≈ 18,000 rows ≈ 4–5 MB; accounts, categories, and budgets are tens to hundreds
of rows each. Windowing would add range-aware tombstones, incomplete offline History search,
and hydrate-on-scroll UX for a problem the product does not have.

### 2.7 Categories are two kinds of data

A system category has a **definition** (server-owned, identical for every user) and a
**user state** (hidden or not, owned by this user). Treating them as one two-way record
causes the client to believe it may write the definition, producing duplicates at sync.

Decision: definitions arrive on hydration; user state syncs as a separate record
(`user_category_state`) keyed by `(user_id, category_id)`.

The auto-seeded per-user protected **"Uncategorized"** category must have a **deterministic
ID derived from `user_id`**, so client and server independently produce the same ID and no
duplicate can exist.

### 2.8 Server stays authoritative at ingest

The client validates for UX responsiveness only. The server re-validates every pushed record
and may return `rejected`. Consequence: the existing Go usecase layer is **not** dead code,
and business rules are not duplicated in Dart as an authority.

This requires a new UI surface: **a rejected-record state**. The user must be able to see and
resolve a transaction they entered days ago that the server refused.

Server-side dashboard endpoints (`/dashboard/home`, `/dashboard/stats`) are retained for a
future web client. They are not on the write path, so they carry no drift risk.

### 2.9 Auth and the offline window

- **Local access is gated by refresh-token validity, not access-token validity.** An expired
  access token is irrelevant offline — no API call is being made.
- The refresh-token TTL is therefore the **product-level promise of how long the app works
  offline**. Proposed: **60 days, sliding** (renewed on each successful refresh). Active users
  effectively never get logged out; genuinely abandoned devices still expire.
  → *This number needs explicit confirmation from the manager.*
- On refresh-token expiry the app enters a **"reconnect required"** state. It does **not**
  wipe local data.

**Golden rule: local unsynced data is never destroyed because of an authentication failure.**

Three login cases, which must not be collapsed:

| Case | Sync direction |
|---|---|
| Clean install / new device | server → device (hydration) |
| Same user, device still holds local data (refresh failed, session evicted) | **push first, then pull** |
| Different user on the same device | isolate/wipe previous user's data before hydrating |

The middle case is the common one in practice and is the primary data-loss risk.

Two additional hazards must be handled explicitly:

1. **Refresh-token rotation + network loss.** If the server rotates the token and the client
   loses connectivity before persisting the new one, the user is locked out while holding
   unsynced data. Mitigation: the client does not discard the old refresh token until the new
   one is durably persisted, and the server accepts the previous token within a short grace
   window.
2. **Session soft-limit (max 10, already implemented).** A long-offline device may have its
   session evicted server-side. This must land in the *push-first-then-pull* path, never the
   clean-install path.

**Logout semantics:**

- **Explicit logout** (user action): wipe local data — but block and warn first if unsynced
  records exist ("12 transactions not yet saved to the server — sync first?").
- **Token expiry / refresh failure / session eviction:** never wipe. Re-authenticate, then
  push-first-then-pull.

### 2.10 Frontend repository layer

The sync rewrite touches exactly the data paths in Budget, History, Review, Dashboard, and
Profile — the five features that currently call Drift directly, in violation of the mandatory
`Repository → Provider → Screen` pattern in `.claude/skills/frontend/coding-style.md`.

Decision: **fold the repository-layer cleanup into this work.** It will not happen otherwise.

---

## 3. LLM receipt scanning

### 3.1 The core tension

Receipt extraction requires a model call and is therefore **inherently online**, in an app
that is otherwise offline-first. Resolution: the scan is **enrichment, never a precondition**.

The flow reuses the product's existing "capture fast, resolve later" shape — the same mental
model as the `000` placeholder and the Review queue. No new concept is introduced.

1. User captures or picks a receipt image.
2. Image is stored **locally** and compressed immediately.
3. A transaction draft is created locally with `scan_pending`. The user may enter the amount
   manually at any time — the transaction is never blocked on the model.
4. **Online:** image is uploaded, the server calls the model, structured fields return,
   the draft is populated.
5. **Offline:** the draft stays `scan_pending` in a local queue and is processed on reconnect.
6. Extraction results are **suggestions**. The user always confirms. They land in the
   existing **Review** queue.

### 3.2 Never auto-commit an extracted amount

The product philosophy is "Awareness over Accuracy", but that does not license silently
recording a wrong number. Models misread totals — subtotal vs. total, tax lines, service
charge. A confirmation step is mandatory.

**Indonesian receipts specifically** must be handled explicitly in the prompt and validated
server-side:

- Thousands separator is `.` and decimal separator is `,` (`Rp 1.500,00`). This is the
  single most likely source of a 100× or 1000× error.
- PPN 11% and service charge lines are common and must not be mistaken for the total.
- Merchant names are frequently abbreviated or stylised.

The server must range-check the extracted total against the recognised line items before
returning it, and mark low-confidence extractions rather than presenting them as certain.

### 3.3 Images are never synced

**Decision: v1 extracts, then discards the original server-side.** The image is retained
**locally only** and never enters the sync protocol.

Rationale:

- Blobs are a different sync problem (resumable upload, lazy fetch, storage quota, retention
  policy) and would otherwise contaminate a clean row-sync design.
- Most users never look at a receipt photo again — the extracted data is the value.
- Eager image download on first login would otherwise mean hundreds of MB.

The transaction row carries `receipt_scanned_at` and the extraction metadata, not an image
reference. If cross-device receipt images are wanted later, they get their own transport —
they do not join this protocol.

### 3.4 The model call runs server-side

Non-negotiable: the API key must never ship in the APK, where it is trivially extractable.
Server-side placement also gives cost control, rate limiting, caching, and the ability to
change models without an app release.

**Model: `claude-opus-5`** via the official Go SDK (`github.com/anthropics/anthropic-sdk-go`),
matching the backend's language. Design points:

- **Structured output** via `output_config.format` with a JSON Schema. This constrains the
  response shape, so the backend is not parsing free-form text. Note: incompatible with
  citations (not needed here).
- **Vision input** as a base64 image block. `claude-opus-5` is in the high-resolution tier
  (2576 px long edge, up to ~4784 image tokens). Client-side downsampling is the main cost
  lever if fidelity allows — this must be measured, not assumed.
- **Thinking is on by default** on `claude-opus-5`. `max_tokens` caps thinking *and* response
  text together, so it must be sized with headroom or responses truncate mid-answer.
- **Prompt caching** on the extraction system prompt (Indonesian formatting rules, schema
  guidance). The minimum cacheable prefix on `claude-opus-5` is 512 tokens; cache reads cost
  ~0.1× input. The rules prompt is stable, so this should hit consistently.
- **Idempotency by image hash.** Reuse the existing transaction idempotency-key pattern:
  the same receipt scanned twice must not incur a second model call.
- **Synchronous first.** Extraction takes seconds; a synchronous request with a spinner is
  acceptable UX and much simpler. An async job queue is added only if measurement demands it.

### 3.5 Cost is a real product constraint

At `claude-opus-5` pricing ($5 / $25 per MTok input / output), a full-resolution receipt is
roughly **~$0.03–0.04 per scan** (image ~4.8K input tokens + ~400 output tokens). Downsampling
to the previous-tier resolution brings this to roughly **~$0.02**. Prompt caching further
reduces the fixed prompt portion.

At 100 scans/month that is roughly $2–4 per user per month — material for a personal expense
tracker. Therefore:

- A **per-user monthly scan cap** is required in v1, not deferred.
- Image-hash caching is required, not optional.
- Cheaper models exist (`claude-sonnet-5`, `claude-haiku-4-5`) as a cost lever. Choosing one
  is a deliberate quality/cost tradeoff and is the **manager's decision**, not a default.

### 3.6 Contract impact

New endpoints are required (exact shapes are `@architect`'s deliverable):

```
POST /v1/receipts/scan     multipart image → structured extraction
```

Both copies of `api-documentation/*.yaml` must be updated together, per the workspace rule.

---

## 4. Consequences

### Positive

- The 1–2 week full-rewrite scope from `task-sync.md` is no longer needed. No HLC, no version
  vectors, no `devices` table, no audit log.
- Anonymous identity and account claiming — the single largest cost item — are eliminated by
  requiring login.
- Zero migration cost: no Drift migration steps, no ID backfill, no DELETE deprecation window.
- The old sync domain is deleted rather than maintained.
- The mature Go usecase layer keeps its purpose; no dual-authority business logic.
- The frontend repository-layer debt gets paid while the code is already open.

### Negative / accepted costs

- DELETE semantics change across four domains (accepted — no production data).
- A new "rejected record" UI surface must be designed and built.
- Receipt scanning adds a hard online dependency and a recurring per-scan cost.
- Receipt images are device-local in v1; a user switching phones loses their photos (but keeps
  every extracted value).

### Risks

| Risk | Mitigation |
|---|---|
| Data loss on re-login with unsynced local writes | Push-first-then-pull path; never wipe on auth failure |
| Lockout via refresh-token rotation during network loss | Keep old token until new one is persisted; server grace window |
| Duplicate "Uncategorized" category | Deterministic ID derived from `user_id` |
| FK failure on push | Contractual push ordering (§2.4) |
| Indonesian number-format misreads by the model | Explicit prompt rules + server-side range validation + mandatory user confirmation |
| Scan cost overrun | Per-user monthly cap + image-hash cache + measured downsampling |
| Protocol becomes unchangeable after launch | `protocol_version` and pagination present from day one |

### The window closes at launch

Once a real user exists, the **sync protocol becomes the hardest thing in the system to
change** — client and server must agree, and old app versions persist on devices for months.
Everything else remains fixable later; the protocol does not. This is the moment to get it
right.

---

## 5. Open items requiring manager confirmation

1. **Refresh-token TTL.** Proposed 60 days sliding. This is the public promise of "how long
   you can stay offline".
2. **Model tier for receipt scanning.** Default is `claude-opus-5`. Stepping down to
   `claude-sonnet-5` or `claude-haiku-4-5` is a quality/cost tradeoff and is the manager's
   call, not a default.
3. **Per-user monthly scan cap.** A number is needed.

---

## 6. Supersedes

- `elora-be-go/docs/decisions/0001-defer-sync-rewrite.md` — **closed**. Option (a) full rewrite
  is rejected as over-engineered for a single-device product; this ADR adopts a scoped delta
  sync closest to option (c).
- `elora_spendos/tasks/TASK-FE-01-sync-mechanism.md` — superseded. Its retry/backoff and
  sync-status-indicator requirements are absorbed here; its UUID and checksum requirements are
  replaced by client-generated UUIDv7 plus server-assigned `server_seq`.
