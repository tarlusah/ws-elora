# PRD — Online-required writes + LLM receipt scanning

- **Cycle:** next feature/change cycle
- **Date:** 2026-08-01 (revised — see revision note below)
- **Baseline:** `shared/context/PRD.md` (consolidated status — read that first for where the
  project stands today)
- **Architecture decision:** `notes/ADR-0004-online-required-writes-and-receipt-scan.md` — the
  authoritative record for *why* each choice was made. This PRD states *what* to build.
- **Status:** awaiting manager approval → then `@pm` writes `user-stories.md`, then `@architect`

**Revision note:** this PRD originally described full bidirectional offline-first sync (see
`notes/ADR-0003-offline-first-sync-and-receipt-scan.md`). On 2026-08-01 the manager stepped that
back to an online-required-write model before any production user exists — implementation cost
was judged too high for the product's current stage. This revision reflects that decision. The
receipt-scanning half is essentially unchanged.

---

## 1. Summary

Two changes ship in one cycle because they touch the same data paths:

1. **Writes require a live connection; reads work offline from a local cache.** Spendos already
   stores locally (Drift). That cache keeps its role for fast, offline-capable *reading* —
   accounts, transactions, budgets, categories, dashboard aggregates all display the last data
   fetched from the server. Creating, editing, or deleting anything requires connectivity at that
   moment. There is no local write queue, no sync protocol, no conflict resolution.
2. **Receipt scanning via LLM.** The user photographs a receipt; a server-side model extracts
   merchant, total, date, and line items; the result lands in the existing Review queue as a
   suggestion the user confirms. Like every other write, this requires connectivity.

There is **no production data and no backward-compatibility requirement**, so this cycle uses
plain, server-authoritative CRUD rather than the sync protocol previously scoped.

---

## 2. Goals

| # | Goal | Success looks like |
|---|---|---|
| G1 | The app is usable offline for **reading** | Viewing Capture history, Review, History, Dashboard, and Budget all work with the network off, showing the last-fetched data |
| G2 | Writing is honest about connectivity | Every create/edit/delete action fails immediately and explicitly ("needs connection") when offline — never silently queued, never silently lost |
| G3 | Data survives a lost phone | A fresh install + login restores every account, category, budget, and transaction from the server |
| G4 | Receipt capture is faster than typing | Photo → confirmed transaction in fewer steps than manual entry |
| G5 | Scanning never blocks the user on the model | The user can always type the amount manually instead of waiting for extraction |

**Dropped from the previous cycle's goals** (see ADR-0004 §4): "fully usable offline including
writes," "no user data ever lost via a local write queue," and "sync status indicator / rejected
records visible" — these depended on the offline-write mechanism this cycle no longer builds.

## 3. Non-goals (explicitly out of scope this cycle)

- **Offline writes of any kind.** Creating, editing, or deleting anything requires connectivity.
  This is the one that changed — see ADR-0004.
- **Multi-device concurrent use.** One account, sequential device use, as before.
- **Anonymous / no-account mode.** Login is required.
- **Receipt images syncing across devices.** Images are device-local, extracted then discarded
  server-side.
- **Offline receipt extraction / on-device models.** Extraction requires connectivity — no longer
  a special case, since everything does now.
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
  offline-window promise. Value pending manager confirmation (§8).
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
| Auth | Refresh-token TTL is still a decision to make (§8); rotation + grace window is a nice-to-have against spurious logouts, not a correctness requirement anymore |

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
locally from the cache, as before. Not cached: sessions/tokens, receipt images.

---

## 5. Scope — LLM receipt scanning

### 5.1 User flow

1. From Capture, the user taps *scan receipt* and takes or picks a photo.
2. **This requires connectivity**, the same as any other write now — no offline fallback queue.
3. The image uploads; the server extracts merchant, total, date, currency, line items, and a
   suggested category.
4. A draft transaction is created **with those fields already populated**, landing in the
   existing Review queue as a suggestion.
5. The user can always type the amount manually instead of waiting for extraction — the
   transaction is never blocked on the model, only on connectivity, same as everything else.
6. **The user always confirms. Nothing is auto-committed.**

This still reuses the product's existing "capture fast, resolve later" shape for the *model's*
role — the user is never blocked waiting on the model — but capture itself now requires
connectivity like every other write, so there is no separate offline-queue behavior to design for
scanning specifically.

### 5.2 Accuracy requirements (Indonesian receipts) — unchanged

- **Number format.** Thousands separator `.`, decimal separator `,` (`Rp 1.500,00`). Handled
  explicitly in the prompt and validated server-side.
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
| Rate limiting | Per-user monthly scan cap, required in v1 — number pending manager decision |
| Image retention | Extract, then discard server-side. Nothing persisted. |
| Sizing | `max_tokens` must account for thinking + response together (thinking is on by default on `claude-opus-5`) |

**API key never ships in the app.** This is why extraction is server-side, without exception.

### 5.4 What gets built — frontend

- Camera / gallery capture, client-side compression.
- Local image store, never synced.
- Explicit "needs connection" state when offline — the scan action is simply unavailable, not
  queued.
- Populated-draft confirmation UI inside the existing Review flow.

### 5.5 Cost — unchanged

At `claude-opus-5` rates, roughly **$0.02–0.04 per scan** depending on image resolution. At 100
scans/month that is roughly $2–4 per user per month. The monthly cap and image-hash cache are v1
scope, not deferred. Stepping down to `claude-sonnet-5` or `claude-haiku-4-5` is the manager's
call — see §8.

---

## 6. Acceptance criteria

Online-required writes:

- [ ] With the network enabled, a user can create, edit, delete, review, split, and categorise
      transactions; manage accounts, categories, and budgets.
- [ ] With the network disabled, the same actions fail immediately with an explicit "needs
      connection" message — nothing is silently queued or silently lost.
- [ ] With the network disabled, the user can still view Capture history, Review, History,
      Dashboard, and Budget using the last-fetched data.
- [ ] A fresh install + login restores the complete dataset from the server.
- [ ] Logging in as a different user on the same device replaces the cache — no data mixing.
- [ ] Refresh-token expiry produces a "log in again" state without corrupting the local cache.

Receipt scanning:

- [ ] Attempting to scan while offline shows an explicit "needs connection" state.
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
3. `@architect` updates **both** copies of `api-documentation/*.yaml` — the receipt-scan endpoint,
   plus whatever schema changes fall out of dropping client-generated ids and tombstones. The two
   copies must not drift.
4. `@backend` and `@frontend` run in parallel against the agreed contract, each in its own
   worktree. The frontend uses contract mocks until the backend endpoint lands.
5. `@qa` tests both surfaces: the online-write happy path, the offline "needs connection" gate,
   and the receipt-scan flow. No conflict/rejected-record matrix — that mechanism doesn't exist.
6. Manager review → `@devops` deploy.

**Sequencing note:** Budget is a clean slate on the frontend and should be built once — not now
and rewritten again later, unaffected by this pivot.

---

## 8. Open decisions blocking `@architect`

| # | Decision | Default if unanswered |
|---|---|---|
| 1 | Refresh-token / session TTL | 60 days, sliding (carried over from ADR-0003; now lower-stakes) |
| 2 | Model tier for extraction | `claude-opus-5` |
| 3 | Per-user monthly scan cap | — none; a number is required |
| 4 | Hard delete vs. soft delete | Hard delete (no longer forced by sync; free choice — see ADR-0004 §5) |

---

## 9. Notes for whoever picks this up

- The remote GitHub copies of both repos are roughly 1–2 weeks behind what the baseline PRD
  describes (backend last pushed 2026-05-09, frontend 2026-05-02, versus status documents dated
  2026-05-15/16). Push local work before relying on the remote as the source of truth.
- `notes/ADR-0003-offline-first-sync-and-receipt-scan.md` remains as historical record of the
  fuller offline-first design. It is not wrong — it was superseded for cost reasons at the
  product's current pre-production stage, and remains available to revisit in a future cycle.
- Field names and details (e.g. whether system categories can be renamed by users) were derived
  from the domain list in `shared/context/PRD.md`, not from the live schema. They must be
  verified against the actual code before entering the API contract.
