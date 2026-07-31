# PRD — Offline-first sync + LLM receipt scanning

- **Cycle:** next feature/change cycle
- **Date:** 2026-07-31
- **Baseline:** `shared/context/PRD.md` (consolidated status — read that first for where the
  project stands today)
- **Architecture decision:** `notes/ADR-0003-offline-first-sync-and-receipt-scan.md` — the
  authoritative record for *why* each choice was made. This PRD states *what* to build.
- **Status:** awaiting manager approval → then `@pm` writes `user-stories.md`, then `@architect`

---

## 1. Summary

Two changes ship in one cycle because they touch the same data paths:

1. **Offline-first sync becomes real.** Spendos already stores locally, but the sync engine was
   simplified independently on both surfaces and never reconciled. This replaces it with a
   cursor-based bidirectional delta sync, makes the local database the source of truth, and
   defines exactly what happens when auth expires or a user re-logs in on a device that still
   holds unsynced data.
2. **Receipt scanning via LLM.** The user photographs a receipt; a server-side model extracts
   merchant, total, date, and line items; the result lands in the existing Review queue as a
   suggestion the user confirms.

There is **no production data and no backward-compatibility requirement**, so breaking changes
(soft delete, client-generated IDs, schema resets) are taken now rather than worked around.

---

## 2. Goals

| # | Goal | Success looks like |
|---|---|---|
| G1 | The app is fully usable offline once logged in | Capture, Review, History, Dashboard, and Budget all work with the network off |
| G2 | No user data is ever lost | No path — including auth failure, session eviction, or re-login — destroys unsynced local writes |
| G3 | Data survives a lost phone | A fresh install + login restores every account, category, budget, and transaction |
| G4 | Sync is invisible when it works and legible when it doesn't | A clear status indicator; rejected records are visible and resolvable |
| G5 | Receipt capture is faster than typing | Photo → confirmed transaction in fewer steps than manual entry |
| G6 | Scanning never blocks capture | A receipt captured offline still produces a usable transaction immediately |

## 3. Non-goals (explicitly out of scope this cycle)

- **Multi-device concurrent use.** One account, one active device at a time.
- **Anonymous / no-account mode.** Login is required.
- **Time-windowed sync.** The full business dataset syncs.
- **Receipt images syncing across devices.** Images are device-local in v1.
- **Offline receipt extraction / on-device models.** Extraction requires connectivity.
- **Web client.** Server dashboard endpoints are retained for it, but it is not built here.

---

## 4. Scope — Offline-first sync

### 4.1 What the user experiences

- Log in once. From then on, everything works offline until the refresh token expires
  (proposed 60 days, sliding — **pending confirmation**).
- When the refresh token does expire, the app says *"reconnect to continue"*. It does **not**
  log the user out of their own data and it does **not** delete anything.
- A sync status indicator shows `syncing` / `synced` / `failed`.
- If the server rejects a record, the user sees it in a dedicated "needs attention" state with
  enough context to fix it — including for a transaction entered days earlier.
- Explicit logout warns before wiping if unsynced records exist.

### 4.2 What gets built — backend (`elora-be-go`)

| Item | Detail |
|---|---|
| Retire old sync domain | Delete `POST /v1/sync`, `GET /v1/sync/status`, `PUT /v1/sync/{id}` |
| New delta endpoints | `GET /v1/sync/changes?since=&limit=` and `POST /v1/sync/changes` |
| `protocol_version` | On every sync request and response; too-old clients rejected explicitly |
| Pagination | `limit` / `next_cursor` / `has_more` mandatory — no unbounded responses |
| Schema: 4 domains | `transactions`, `accounts`, `categories`, `budgets` gain `deleted_at` and `server_seq`; `id` becomes client-supplied UUIDv7 |
| Soft delete | Hard DELETE replaced by tombstones across all four domains. **No remaining hard-delete path** — pull reads live rows, so one hard delete is a permanent zombie record on the client (ADR §2.13) |
| Ingest sequencing | Server assigns monotonic `server_seq` per user, from a **per-user counter incremented inside the write transaction** (not a global Postgres sequence — commit-order gaps silently skip records). Device clocks never trusted. See ADR §2.12 |
| Pull read model | Each domain answers `ChangesSince` from **its own entity rows** — `WHERE user_id = ? AND server_seq > ? ORDER BY server_seq LIMIT n`, index `(user_id, server_seq)`. No changelog table, no event/projection feed. See ADR §2.12 |
| Push validation | Server re-validates every record; returns `applied` / `conflict` / `rejected` per record |
| Sync domain rewrite | Thin orchestrator over consumer-defined `SyncPushable` / `SyncPullable`; shared value types in `pkg/synccontract`; no entity, no repository. See ADR §2.11 |
| Conformance test suite | `pkg/synccontract/testing` — every domain implementation must pass it |
| `default_account_id` on user | Replaces the per-account `is_default` flag (avoids a two-record atomic update) |
| `/me` endpoint | Completes the Profile placeholders noted in the baseline PRD |
| Auth | Refresh-token grace window during rotation; session eviction must not look like a clean install |

### 4.3 What gets built — frontend (`elora_spendos`)

| Item | Detail |
|---|---|
| Rewrite `lib/core/sync/` | Replace, do not refactor. Cursor delta sync with `last_synced_seq` |
| Client-generated IDs | UUIDv7 at record creation |
| Dirty tracking | `sync_state`: `clean` / `dirty` / `rejected` |
| Retry with backoff | Exponential, max 3 attempts, debounced — specified in TASK-FE-01 but never implemented |
| Sync status UI | `syncing` / `synced` / `failed` — specified but never implemented |
| Rejected-record UI | **New surface.** Visible, explains why, lets the user fix and re-push |
| Login flow branching | Clean install → hydrate. Same user with local data → **push first, then pull**. Different user → isolate/wipe |
| Logout flow | Warn-if-unsynced, then wipe. Distinct from token expiry, which never wipes |
| Drift schema reset | Bump version and recreate — no migration steps needed |
| **Repository layer cleanup** | Budget, History, Review, Dashboard, Profile currently call Drift directly. Restore the mandatory `Repository → Provider → Screen` pattern while these paths are open |
| Budget feature | Still a 9-line stub. Design HTML shipped 2026-05-15; build it on the new data layer rather than building it twice |

### 4.4 Push ordering (contract requirement)

```
accounts → categories (parent before child) → budgets → transactions (parent before split child)
```

Applies to both client push and server hydration. A transaction created offline that references
a category also created offline must not fail its foreign key.

### 4.5 What syncs

Two-way: transactions (incl. `000` placeholders, review state, splits), accounts, budgets.
Server → device only: categories (all of them — creating, editing, and hiding a category
requires connectivity), user profile.
Not synced: dashboard aggregates (computed locally), sessions/tokens, UI preferences,
sync bookkeeping, **receipt images**.

---

## 5. Scope — LLM receipt scanning

### 5.1 User flow

1. From Capture, the user taps *scan receipt* and takes or picks a photo.
2. The image is compressed and stored **locally**; a draft transaction is created immediately
   with `scan_pending`.
3. The user can type the amount manually at any point — **the transaction is never blocked on
   the model**.
4. Online: the image uploads, the server extracts, the draft is populated with merchant, total,
   date, currency, line items, and a suggested category.
5. Offline: the draft stays queued and processes on reconnect.
6. The extraction appears in the existing **Review** queue as a suggestion. **The user always
   confirms.** Nothing is auto-committed.

This deliberately reuses the product's existing "capture fast, resolve later" shape — the same
mental model as the `000` placeholder. No new concept is introduced for the user to learn.

### 5.2 Accuracy requirements (Indonesian receipts)

These are correctness requirements, not nice-to-haves:

- **Number format.** Thousands separator `.`, decimal separator `,` (`Rp 1.500,00`). This is the
  most likely source of a 100× or 1000× error and must be handled explicitly in the prompt and
  validated server-side.
- **PPN 11% and service charge** lines must not be mistaken for the total.
- The server **range-checks** the extracted total against recognised line items before returning.
- Low-confidence extractions are marked as such, not presented as certain.

### 5.3 What gets built — backend

| Item | Detail |
|---|---|
| `POST /v1/receipts/scan` | Multipart image in, structured extraction out |
| Model call | `claude-opus-5` via `github.com/anthropics/anthropic-sdk-go`, server-side only |
| Structured output | JSON Schema via `output_config.format` — the backend does not parse free-form text |
| Prompt caching | Extraction rules prompt is stable and exceeds the 512-token minimum; cache reads cost ~0.1× |
| Idempotency | Keyed by image hash — the same receipt scanned twice must not incur a second model call |
| Rate limiting | **Per-user monthly scan cap, required in v1** — number pending manager decision |
| Image retention | Extract, then discard server-side. Nothing persisted. |
| Sizing | `max_tokens` must account for thinking + response together (thinking is on by default on `claude-opus-5`) |

**API key never ships in the app.** This is why extraction is server-side, without exception.

### 5.4 What gets built — frontend

- Camera / gallery capture, client-side compression.
- Local image store, never synced.
- Offline scan queue with retry.
- Populated-draft confirmation UI inside the existing Review flow.
- Explicit "scan needs connection" state when offline.

### 5.5 Cost

At `claude-opus-5` rates, roughly **$0.02–0.04 per scan** depending on image resolution
(~4.8K image tokens at full resolution + ~400 output tokens). At 100 scans/month that is
roughly $2–4 per user per month — material for a personal expense tracker.

Consequences already reflected above: the monthly cap and image-hash cache are **v1 scope, not
deferred**. Downsampling is the main additional lever and must be measured against extraction
accuracy rather than assumed.

Stepping down to `claude-sonnet-5` or `claude-haiku-4-5` is a deliberate quality/cost tradeoff
and is the **manager's decision** — see §8.

---

## 6. Acceptance criteria

Sync:

- [ ] With the network disabled, a user can create, edit, delete, review, split, and categorise
      transactions; manage accounts, categories, and budgets; and view Dashboard and History.
- [ ] Reconnecting pushes every local change and pulls every remote change, in the contractual
      order, with no foreign-key failure.
- [ ] A fresh install + login restores the complete dataset.
- [ ] **Re-login on a device holding unsynced writes pushes them before pulling — nothing is lost.**
- [ ] Refresh-token expiry produces a "reconnect required" state and destroys nothing.
- [ ] Session eviction (soft-limit) follows the push-first path, not the clean-install path.
- [ ] Refresh-token rotation interrupted by network loss does not lock the user out.
- [ ] Logging in as a different user on the same device does not mix data.
- [ ] Explicit logout warns when unsynced records exist.
- [ ] A server-rejected record is visible, explained, and resolvable in the UI.
- [ ] Deleting a record on one device does not resurrect it after sync.
- [ ] Concurrent writes to the same user never cause a pull to skip a record — a client that
      paginates while writes are landing ends up with every change.
- [ ] No write path on the four synced tables hard-deletes: not the API, not a cleanup job, not
      a cascading foreign key.
- [ ] A change made through a non-sync path (the REST CRUD endpoints, if retained) is picked up
      by the next pull — `server_seq` is bumped there too.
- [ ] Offline, the user can categorise transactions using existing categories; attempting to
      create or hide a category shows a clear "needs connection" state rather than failing.
- [ ] Every domain implementing `SyncPushable` / `SyncPullable` passes the shared conformance
      test suite.
- [ ] A client sending an unsupported `protocol_version` receives an explicit error.
- [ ] Budget/History/Review/Dashboard/Profile go through a repository layer.

Receipt scanning:

- [ ] A receipt captured offline yields a usable transaction immediately and extracts on reconnect.
- [ ] Extraction never auto-commits; the user confirms every value.
- [ ] `Rp 1.500,00` is read as 1500, not 150000 or 1.5.
- [ ] PPN and service charge are not mistaken for the total.
- [ ] The same image scanned twice triggers only one model call.
- [ ] Exceeding the monthly cap produces a clear message, not a silent failure.
- [ ] No API key is present in the built app artifact.

---

## 7. Dependencies and sequencing

1. Manager approves this PRD and answers §8.
2. `@pm` writes `user-stories.md`.
3. `@architect` updates **both** copies of `api-documentation/*.yaml` (sync endpoints, receipt
   scan endpoint, schema changes) and the data model. The two copies must not drift.
4. `@backend` and `@frontend` run in parallel against the agreed contract, each in its own
   worktree. The frontend uses contract mocks until the backend endpoint lands.
5. `@qa` tests both surfaces, with a specific matrix for the offline → sync → conflict →
   resolve paths and the three login cases.
6. Manager review → `@devops` deploy.

**Sequencing note:** Budget is a clean slate on the frontend and should be built once, on the
new data layer — not built now and rewritten after the sync work.

---

## 8. Open decisions blocking `@architect`

| # | Decision | Default if unanswered |
|---|---|---|
| 1 | Refresh-token TTL — the public promise of "how long you can stay offline" | 60 days, sliding |
| 2 | Model tier for extraction | `claude-opus-5` |
| 3 | Per-user monthly scan cap | — none; a number is required |

---

## 9. Notes for whoever picks this up

- The remote GitHub copies of both repos are roughly 1–2 weeks behind what the baseline PRD
  describes (backend last pushed 2026-05-09, frontend 2026-05-02, versus status documents dated
  2026-05-15/16). Push local work before relying on the remote as the source of truth.
- Field names and details in the ADR (e.g. whether system categories can be renamed by users)
  were derived from the domain list in `shared/context/PRD.md`, not from the live schema. They
  must be verified against the actual code before entering the API contract.
