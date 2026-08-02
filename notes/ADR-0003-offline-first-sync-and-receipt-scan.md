# ADR 0003 — Offline-first sync (cursor delta) + LLM receipt scanning

- **Status:** **Superseded 2026-08-01 by `notes/ADR-0004-online-required-writes.md`** for
  §2.1–§2.14 (the offline sync protocol, schema, and login-case design) — cost of implementation
  was judged too high for the product's pre-production stage. The design itself was not wrong and
  remains available to revisit in a future cycle. §3 (receipt scanning) was carried forward into
  ADR-0004 initially, but the feature was **cut from scope entirely on 2026-08-02** — not carried
  forward anywhere. This section is kept here for historical reference only. (Still supersedes and
  closes `elora-be-go/docs/decisions/0001-defer-sync-rewrite.md` — that closure is untouched.)
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
     ← { protocol_version, changes: [ { kind, id, base_seq, deleted_at, payload } ] }
     → { protocol_version, results: [ { id, status, reason, server_seq } ], next_seq }
     status ∈ applied | conflict | rejected
```

`base_seq` is the `server_seq` the client last held for that record, or `0` for a record the
server has never seen. It is what lets the server detect a write made on top of a stale version.
See §2.14 for the three statuses — note that `conflict` means the write **was** applied.

- A fresh login is the same endpoint with `since=0`. Bootstrap is not a special code path;
  it is the extreme case of ordinary sync.
- `protocol_version` is present from day one. The server rejects clients that are too old
  with an explicit error rather than failing silently.
- Pagination (`limit`, `next_cursor`, `has_more`) is mandatory from day one. No response
  may assume the whole dataset fits.

### 2.4 Push ordering is part of the contract

Foreign keys must resolve at ingest. Push and hydration need **different orders**, because
categories are pull-only (§2.7) and therefore cannot appear on the push path at all:

```
Push   (client → server):  accounts → budgets → transactions (parent before split child)
Apply  (server → client):  accounts → categories (parent before child) → budgets → transactions
```

This is a contract requirement, not an implementation detail. The server must not trust the
client's ordering — it re-orders by kind itself (§2.11).

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

**`id` is client-generated only for push-capable units** (accounts, budgets, transactions).
Categories are pull-only (§2.6), so their IDs remain server-generated. They still carry
`server_seq` and `deleted_at` — the client needs both to advance its cursor and to learn about
server-side deletions.

**DELETE becomes soft delete.** This is breaking and is accepted — no production data exists.

Because there is no data to preserve, Drift requires **no migration steps**: bump the schema
version and recreate the local database. PostgreSQL migrations may drop and recreate freely.

### 2.6 What syncs

Test applied: *would the user be upset to lose this if their phone died?*

| Data | Direction | Rationale |
|---|---|---|
| Transactions | ↕ two-way | Core product data. Includes `000` placeholders, review state, splits. |
| Accounts | ↕ two-way | Small, required offline by Capture. |
| Categories — all of them | ⬇ server → device | Server-owned entirely. Creating, editing, and hiding a category require connectivity. See §2.7. |
| Budgets | ↕ two-way | Small; per category per month. **LWW alone is not sufficient** — a budget has a natural key, so two devices can create two rows for one logical slot. See §2.14. |
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

### 2.7 Categories are pull-only

Categories sync **downward only**. The client never creates, edits, hides, or deletes a
category while offline — it uses whatever categories it already has. This follows the owner's
principle: *if there's no internet, use what's there.*

Consequences, all simplifications:

- No client-generated UUID for categories.
- No push path, no tombstone authoring, no conflict handling on the category domain.
- The separate `user_category_state` record proposed in earlier drafts is **not needed** —
  hide/show is a server-side write like any other category mutation.
- The auto-seeded per-user **"Uncategorized"** category needs **no deterministic ID**. That
  requirement only existed to stop client and server from seeding it independently; the client
  never seeds. Existing server-side seeding on register is unchanged.

**Accepted cost:** hiding or creating a category requires connectivity. Adding a push path to
the category domain for a single boolean would mean tombstones, conflict handling, and
conformance tests for an action a user performs perhaps once a year. The trade is not worth it.

Category definition and per-user state may still be stored separately server-side if the schema
already does so — that is now an internal detail, not a sync concern.

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

### 2.11 Sync domain architecture (backend)

*Settled in discussion item 1.1. Governs how §2.3–§2.6 are implemented inside
`elora-be-go`, under the rules in `.claude/skills/backend/coding-style.md`.*

**Push and pull are asymmetric.** Push carries business rules — the server is authoritative at
ingest (§2.8), so it must go through domain usecases or validation gets duplicated. Pull has no
business rules; it is pure projection of state. This asymmetry allowed a shortcut (sync reading
other domains' tables for pull) that is nevertheless **rejected**: the whole codebase is written
so domains can later be extracted into services, and a sync component that reaches across domain
tables would break that extraction. Consistency with the existing investment wins.

**Decision: `sync` is a thin orchestrator. Both directions go through domain usecases.**

#### Opt-in, per direction

Not every domain syncs, and not every syncing domain syncs both ways. Two consumer-defined
interfaces; a domain implements whichever apply:

```go
// internal/sync/usecase/ports.go
type SyncPushable interface {
    Kind() string
    ApplyBatch(ctx context.Context, changes []synccontract.Change) []synccontract.Result
}

type SyncPullable interface {
    Kind() string
    ChangesSince(ctx context.Context, userID string, seq int64, limit int) ([]synccontract.Change, error)
}
```

| Unit | Pushable | Pullable |
|---|:--:|:--:|
| account | ✓ | ✓ |
| budget | ✓ | ✓ |
| transaction | ✓ | ✓ |
| category | — | ✓ |
| user profile | — | ✓ |

Two of five units are pull-only, which is what earns the split: a merged interface would force
`category` and `user` to implement `ApplyBatch` as a no-op — code that exists only to satisfy
the compiler and fails at runtime if ever called. Split, a pull-only domain has no push door at
all. Registration in `apps/` then doubles as documentation:

```go
syncUsecase := sync.NewUsecase(
    sync.WithPushable(accountUC, budgetUC, txUC),
    sync.WithPullable(accountUC, budgetUC, txUC, categoryUC, userUC),
)
```

#### Shared value types live in `pkg/`, not `internal/sync/`

Go satisfies interfaces implicitly, so a domain never imports the *interface*. But it must
import the *parameter and return types*. If `Change` and `Result` lived in
`internal/sync/usecase`, then `internal/transaction/usecase` would import another domain —
violating the first rule of the architecture.

They therefore live in **`pkg/synccontract/`**: shared by all domains, no business logic, and
`pkg/` may never import `internal/`. ✓

#### Envelope carries sync metadata; payload is the domain's existing DTO

```go
// pkg/synccontract/
type Change struct {
    Kind      string          // ← sync metadata
    ID        string          // ← sync metadata
    DeletedAt *time.Time      // ← sync metadata
    ServerSeq int64           // ← sync metadata
    Payload   json.RawMessage // ← exactly the domain's existing DTO, unchanged
}

type Result struct {
    ID        string
    Status    string // applied | rejected
    Reason    string
    ServerSeq int64
}
```

This is what lets sync stay generic. Sync orders and paginates by `server_seq` and reports
results by `id` **without knowing whether it holds a transaction or a budget**. Had those fields
lived inside the payload, sync would have to either import concrete domain types (rule
violation) or unmarshal the payload to find them — which re-creates the envelope implicitly.

The second consequence matters as much: because sync metadata is not in the payload, **there is
exactly one representation of each entity**. The payload is the same DTO the REST endpoint uses.
Adding a field to a transaction means adding it once, and both the online and offline paths pick
it up. Two separate DTOs would silently drop new fields on the sync path only — the path
developers exercise least, on a failure mode that produces no error and is unreproducible for
anyone with a working connection.

A flat wire format (metadata merged into the payload, sync unmarshalling twice) also works.
The envelope was preferred because it can be described once in OpenAPI while the payload varies
per kind, and because it makes the "every payload carries id and seq" contract structural rather
than conventional.

Follow-on: **`id` is supplied by the client on every write path**, so the CRUD endpoints (if
retained — see item 1.8) must accept a client-supplied ID rather than generating one. Two ID
policies in one system would be worse than either.

#### `ApplyBatch`, not per-record `Apply`

Sync enforces ordering *between* kinds (`accounts → categories → budgets → transactions`, and
it must **not** trust the client's ordering). But ordering also matters *within* a kind: a split
transaction's parent must land before its child; a parent category before its child. Deciding
that requires knowing about parent references — domain knowledge sync must not have.

So a domain receives its whole slice at once and orders it internally. This also puts
transaction granularity in the domain's hands rather than sync's, which loosens the coupling to
item 1.5.

#### `sync` has no entity and no repository

`server_seq` lives on each domain's own rows; the cursor lives on the client; idempotency reuses
the existing per-domain pattern. The server needs to remember nothing about sync, so the domain
is **stateless** — the same shape as `dashboard`, which is aggregation-only with no entity.

#### Where the adapters live

Each domain needs a small amount of code to unmarshal `Payload` into its own type and call its
own usecase. This goes in an additional file under the domain's existing `usecase/` folder
(`{domain}_sync.go`). No change to the house folder structure.

#### Sync is not anemic

Its own logic: ordering enforcement between kinds, dispatch by kind, cursor and pagination
semantics, `protocol_version` negotiation, merge-sort across pull streams, and result
aggregation.

The merge-sort is cheap because the sequence is global and therefore a total order: query each
pullable unit with `WHERE user_id = ? AND server_seq > cursor ORDER BY server_seq LIMIT n`,
merge, take the top `n`. **The cursor stays a single `int64`** — no compound cursor. The cost is
over-fetching (`n` per unit to return `n` overall), which is immaterial at this data volume.

#### Correctness is enforced by tests, not discipline

Opt-in per domain means the mechanical parts — tombstones, `server_seq` assignment,
`deleted_at` filtering — are written once per pushable unit rather than once overall. That is
the honest cost of this design versus a centralised sync repository: more places to get the same
thing subtly wrong, on a failure mode that loses user data.

Mitigations, both required:

1. Mechanical helpers in `pkg/` so each domain writes only what is genuinely domain-specific.
2. **A conformance test suite in `pkg/synccontract/testing` that every implementation must
   pass.** This is the only thing that keeps four implementations behaving identically.

### 2.12 Where pull-side changes come from

*Settled in discussion item 1.2. Answers how a domain implements `ChangesSince` (§2.11).*

**Decision: `ChangesSince` reads the domain's own entity rows. `server_seq` is authoritative on
the row. There is no changelog table and no event/projection feed for sync.**

```sql
SELECT ... FROM {table}
WHERE user_id = $1 AND server_seq > $2
ORDER BY server_seq
LIMIT $3
```

#### The usual reasons for a changelog are already gone

| Reason a changelog normally exists | Status here |
|---|---|
| Hard delete leaves no trace | Removed — soft delete (§2.5); the tombstone *is* the row |
| History of intermediate versions is needed | Not needed — LWW per record (§2.2). A client 30 days behind wants the latest state, not twelve superseded versions. Reading rows compacts this for free |
| One table to paginate across domains | Removed — §2.11 already chose merge-sort over per-domain streams with a single `int64` cursor |
| Isolate sync read load from OLTP tables | Immaterial at ~18,000 rows per user |

One honest argument survives: a changelog outlives a row that is hard-deleted out of band (a
migration, a cleanup job, a manual fix). That risk is accepted, and it is what makes §2.13
mandatory rather than hygienic.

#### Event + projection is not merely more expensive — it contradicts §2.11

If sync were fed by a changelog populated from events, the changelog table has to belong
somewhere, and both answers are already ruled out:

- **Owned by `sync`** → sync gains an entity and a repository, contradicting §2.11, and the
  table holds rows from five domains — precisely the cross-domain construct that would block
  later extraction into services, which is the reason the pull shortcut was rejected in 1.1.
- **Owned per domain** (`transaction_changelog`, …) → five extra tables duplicating data that
  already exists on the row, each with its own consistency guarantee to maintain, and the
  single-table pagination benefit disappears anyway.

There is no third placement.

Durability is a second, independent objection, and the in-memory bus is not the core of it: the
entity write and the projection write are **in different transactions**. A crash between them
loses that change from the changelog permanently — the client's cursor moves past it with no
error, no retry, and no way to detect the divergence. Moving to Kafka in phase 2 does not fix
this; it would still require a transactional outbox. "Wait until Kafka" is not an answer.

#### `server_seq` is assigned from a per-user counter, inside the write transaction

A global Postgres sequence (`nextval`) is **rejected**, because it silently loses records:
transaction A takes seq 100, transaction B takes 101, B commits first. A client pulling in that
window sees 101, stores cursor = 101, and never sees 100. No error is raised. This hazard is
about *when the sequence is assigned versus when the commit becomes visible*, so a changelog
would have suffered it identically.

The counter is therefore incremented in the same transaction as the write:

```sql
UPDATE user_sync_state SET last_seq = last_seq + 1 WHERE user_id = $1 RETURNING last_seq
```

The row lock serialises writes per user, so a gap can never be visible mid-flight. On a
single-device product the contention is effectively zero. Side benefit: `last_seq` per user is a
cheap "is this client current?" check.

*This settles the generator question that item 1.3 was to decide. Where in the repository layer
the assignment happens, so that non-sync write paths cannot bypass it, is settled in §2.15.*

With gaps closed, mutable `server_seq` is safe under pagination: the sequence only ever
increases and the client pages in ascending order, so a row updated mid-pull always moves
*ahead* of the cursor, never behind it. Nothing is skipped. The changelog's usual advantage — an
immutable log with stable pagination — produces no correctness difference here.

#### The unit of sync is the shape the client sees, not the physical table

If a domain composes its client-facing record from more than one table — §2.7 permits category
definition and per-user hide/show state to be stored separately — then **every constituent table's
write must bump the `server_seq` of the row the client is keyed on**. Otherwise a user hides a
category and their device never learns. `ChangesSince` must consider all constituent tables.

#### Consequences

- **Item 1.4 collapses.** Both of its candidate read models (a `UNION ALL` view over five tables,
  or a changelog table) are cross-domain constructs already excluded by §2.11 — a UNION view over
  five domain tables *is* sync reading other domains' tables. What remains of 1.4 is an index,
  `(user_id, server_seq)` per table, and the pagination already specified in §2.11.
- **Item 1.6 becomes correctness-critical, not hygienic.** See §2.13, settled in §2.16.
- No change history. An audit trail, if ever wanted, is built separately — consistent with §4.
- Five near-identical `ListChangedSince` queries, one per pullable unit. This is the per-domain
  repetition cost accepted in §2.11; the conformance suite must specifically cover: tombstones
  are returned, `server_seq` is strictly increasing, `limit` is honoured, `has_more` is true when
  any unit still has rows beyond the window, and no record is skipped under concurrent writes.
- One hot `user_sync_state` row per user, locked on every write.

#### This decision is reversible; that is part of why it was taken

The `Change` envelope on the wire is identical whether the data comes from a row or a changelog.
Adding a changelog later is therefore a purely server-internal change — no app release, no
protocol negotiation. Unlike §2.3 and item 1.5, this is not locked at launch, which argues for
the design with fewer moving parts now.

For the same reason, publishing sync events "just in case" while sync does not consume them is
**rejected**: unconsumed events are speculative work, and once they exist someone will build on
them and inherit the durability problem above. Adding a publish call later is trivial.

### 2.13 Hard delete is banned on synced tables

*Raised by 1.2; the implementation shape is settled in §2.16 (item 1.6).*

Because pull reads live rows, a row removed by any path other than a tombstone becomes a
**permanent zombie on every client** — never deleted, never re-pulled, with no error on either
side. Soft-delete enforcement is therefore load-bearing for sync correctness, not just for read
correctness, on `transactions`, `accounts`, `categories`, and `budgets`.

### 2.14 Push semantics

*Settled in discussion item 1.5.*

#### Conflict is real, and it is a staleness problem — not a concurrency problem

"Single device" is a product intention, not an enforced constraint: the session soft-limit is
10, so logging in on a second device does not end the first device's session. §2.1 already scopes
the sequential-device window. Three concrete cases were worked through, and none of them involves
two writes happening at the same moment:

| # | Case | What LWW alone does |
|---|---|---|
| **A** | Device A edits transaction X offline on Monday. Device B edits the same X on Wednesday and syncs. A reconnects Thursday. | A's Monday content wins on ingest order. B's newer correction is silently reverted. |
| **B** | Device B deletes X on Tuesday and syncs. A pushes an edit to X on Thursday, carrying `deleted_at = null`. | The update clears the tombstone. **A deleted record comes back to life with stale content.** |
| **C** | Both devices create a budget for Food/August, each with its own client UUID. | LWW cannot see it — the `id`s differ. Result is two rows for one logical slot and a double-counted dashboard. |

Because the gap between the two writes is *days*, no ordering or locking mechanism removes any
of them. This was tested against three proposals in discussion and none moved the needle:

- **Keying sync by device** — detects staleness at best, and worse than `base_seq` does (a device
  cursor is a global watermark, so a device's own unpulled writes look stale). Per-device
  sequences are version vectors under another name, rejected by §2.2. Reintroduces the `devices`
  table §4 counts as removed, on an identity that reinstalls do not preserve.
- **Server ingest timestamp instead of an integer** — orders by arrival exactly as the counter
  does, so it changes no outcome, while adding three silent failure modes: ties (a batch push in
  one transaction stamps every record identically, so paginating past them skips records),
  the same commit-visibility gap that disqualified `nextval` in §2.12, and a clock that NTP can
  step backwards. Counter monotonicity is structural; clock monotonicity is a hope.
- **Advisory lock per `user_id` + entity** — serialises concurrent syncs, but the conflicting
  writes are days apart, so the lock is uncontended exactly when it would be needed. Separating
  the push and pull lock keys is correct as far as it goes, and following it through shows why
  it is unnecessary: a pull writes nothing, so a pull lock guards nothing, leaving "serialise
  writes per user" — which the `user_sync_state` row lock in §2.12 already does, on the write
  path only, released at commit, with no connection-pool leak risk.

The conclusion is structural: **conflict is semantic, not mechanical.** Two human intentions, at
two different times, about one record. No sequence, clock, lock, or table can say which intention
was right, because that information is not in the system. Only three levers exist — prevent a
second writer (which would mean disabling the product), detect that a write stood on a stale
version, or set a policy for what happens then. The third is unavoidable; the only real choice is
whether the user is told.

#### `base_seq` is what makes push-first safe

§2.9 requires **push before pull**, so that unsynced local writes are transmitted before anything
overwrites them. That ordering is right — pulling first would destroy the local edit before it
was ever sent — but it means **the pushing device is pushing blind**: it cannot know what changed
on the server, because it has not pulled yet.

The device does, however, know which version it edited. So every pushed record carries
`base_seq`, and the server — which holds both halves — makes the comparison the device could not:

```
row.server_seq > base_seq   →   this write stood on a stale version
```

This is optimistic concurrency control riding on a sequence that already exists. No new table, no
clock, no device identity, and it is per record rather than per device, so it does not produce
false positives on the device's own unpulled writes.

#### The three statuses

| Status | Written? | Meaning |
|---|:--:|---|
| `applied` | yes | `base_seq` matched the row, or the server had never seen this `id` |
| `conflict` | **yes** | Applied under LWW, **but** it stood on a stale version. The client must surface it |
| `rejected` | no | A business rule or constraint failed. The record stays `dirty`/`rejected` on the device |

`conflict` meaning *applied* is the part most likely to be implemented wrongly. It is not a
failure — the record landed, and the client's copy is now authoritative. What the user is being
told is that their write overwrote a newer one.

**Chosen policy: detect, apply, and flag** — not "reject and make the user merge". A merge UI
showing old and new values side by side is real design and build work for an event that may occur
once a year on a product that does not promise multi-device use. The already-planned
needs-attention surface (§2.8) is reused instead.

This is deliberately the cheap half of a reversible decision. **`base_seq` on the wire is what
locks at launch**; the reaction to it is server-side logic that can change at any time without an
app release. Adding the field later would mean negotiating with app versions that persist on
devices for months.

#### Batch granularity: per record, grouped by the domain

**A rejected record must never block the rest of the batch.** A device pushing three days of
offline work cannot be stopped by one transaction the server refuses — that would leave the
device unable to make progress at all, which is the failure this whole design exists to prevent.

So the batch is **not** atomic as a whole, and not atomic per kind. Each record succeeds or fails
on its own, except where a domain knows several records must land together (a split parent and its
children). That grouping is the domain's call, consistent with §2.11 putting transaction
granularity in the domain's hands.

Consequence: a partially applied batch is the normal case, not an error path. A concurrent pull
from another device can therefore observe a torn batch — a transaction whose account has not
landed yet. This is already handled: item 4.2 defers foreign-key checks inside the client's apply
transaction, and the remainder arrives on the next pull. It self-heals.

#### Two rules that close cases B and C

These are defects, not preferences, and hold regardless of the conflict policy.

1. **Tombstones are sticky.** An update may never clear `deleted_at`. Once deleted, a record stays
   deleted; undelete is an explicit, separate action, not a side effect of a stale edit landing.
2. **Budgets have a natural key.** `(user_id, category_id, period)` is unique. The second device
   to create the same slot gets `rejected` with a reason the client can act on, rather than a
   duplicate row. The client-generated UUID remains the row's identity; the natural key is a
   constraint on top of it.

#### Sync log and `ingested_at` — operational only

Two additions for debugging and support, deliberately off the correctness path:

- A **sync log**: who synced, when, from what cursor, how many records pushed, and the
  applied/conflict/rejected counts. Without it, a "my data disappeared" report is uninvestigable.
- An **`ingested_at`** column alongside `server_seq` on each synced table, so a human reading the
  database sees "3 Aug 14:22" rather than "seq 1450".

Neither is ever read by the pull path. That is what keeps them compatible with §2.12: what was
rejected there was a changelog as the *source* of changes, not the existence of a log.

### 2.15 `server_seq` assignment lives in the repository layer

*Settled in discussion item 1.3. §2.12 already decided the generator (per-user counter,
incremented inside the write transaction); this decides where that call happens so no write
path can bypass it.*

**Decision: the repository assigns `server_seq` internally, inside its own `Create` / `Update` /
soft-delete methods. The usecase never sets it and cannot pass it in.**

A shared helper does the mechanical part:

```go
// pkg/syncseq/syncseq.go — infra only, no business logic, per pkg/ rules
func NextSeq(ctx context.Context, userID string) (int64, error) {
    // UPDATE user_sync_state SET last_seq = last_seq + 1 WHERE user_id = $1 RETURNING last_seq
    // uses the tx already present on ctx, per the existing repo convention
}
```

Each pushable domain's repository (`account`, `budget`, `transaction` — not `category`, which is
pull-only) calls `syncseq.NextSeq` as the first step inside its own write method, before building
the row. `entity.ServerSeq` is not a field the caller populates.

**Why not in the usecase.** If assignment were an explicit call the usecase makes before invoking
`repo.Save`, every write usecase in every domain — including ones written later (an admin tool, a
bulk import, a cleanup job) — would have to remember to make it. That is exactly the "discipline"
§2.11 already rejected as the enforcement mechanism for sync correctness. Putting it inside the
repository's write methods instead of behind a parameter makes it structurally impossible to
write a row through the repo without a `server_seq` — there is no code path that skips it, not a
convention that can be forgotten.

**Cost.** The call is duplicated across three repositories rather than centralised once. This is
the same per-domain repetition already priced into §2.11; the conformance test suite required
there should assert, per domain, that repeated writes for one user produce a strictly increasing
`server_seq`.

**Boundary.** This closes "a usecase forgot to call it." It does not close a write that bypasses
the repository entirely — raw SQL, a migration, a manual `psql` fix. That is a different class of
risk, the same one §2.13/§2.16 addresses for hard deletes below.

### 2.16 Soft-delete enforcement (item 1.6)

*Raised to 🔴 by §2.13: pull reads live rows, so anything that removes a row without a tombstone
is a permanent, silent zombie on every client.* Two distinct concerns, two different answers:

#### (a) Read-time filter — same shape as §2.15: default-safe, explicit escape hatch

A domain's normal read methods (`List`, `Get`) always filter `deleted_at IS NULL` internally —
this is not a parameter the caller can omit. The one caller that legitimately needs deleted rows
is sync's own `ChangesSince` (§2.12), which uses a separately named method,
`ListAllIncludingDeleted`, so any call site reading tombstones is visible as such in review rather
than hidden behind a boolean flag.

#### (b) Hard-delete ban — enforced in PostgreSQL, not only in Go

A `BEFORE DELETE` trigger on each of `transactions`, `accounts`, `categories`, `budgets` raises an
exception:

```sql
CREATE FUNCTION reject_hard_delete() RETURNS trigger AS $$
BEGIN
  RAISE EXCEPTION 'hard delete banned on %; use soft delete (deleted_at)', TG_TABLE_NAME;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_hard_delete BEFORE DELETE ON transactions
  FOR EACH ROW EXECUTE FUNCTION reject_hard_delete();
-- repeated for accounts, categories, budgets
```

**Why not just remove `Delete()` from the repo interface.** The risk named in §2.13 is explicitly
a migration, a cleanup job, or a manual fix — none of which go through the Go repository layer at
all. Only something enforced in Postgres itself closes that gap.

**Why a trigger and not `REVOKE DELETE`.** Migrations run automatically on startup through the
same connection the app uses (`database-conventions.md`), which means the app's DB role is very
likely the owner of these tables. Postgres table owners bypass `GRANT`/`REVOKE` entirely — revoking
`DELETE` from the owning role has no effect. Achieving the same guarantee via privileges would need
either role separation (a migrator role vs. a query-only app role) or `FORCE ROW LEVEL SECURITY`
with a always-false `DELETE` policy — both heavier, and role separation is a deploy-level change
for `@devops`, not a decision this item needs to force. A trigger needs neither: it works
regardless of who owns the table or which role issues the statement.

**Cost.** One more thing that lives in the database rather than in Go — someone reading only
`internal/{domain}/repository/` won't see why `DELETE` fails. Document it in the migration file
and in `.claude/skills/backend/database-conventions.md` once this repo is checked out.

**Boundary.** A trigger stops accidental hard deletes from application code, migrations, and
scripts running under normal privileges. It does not stop a superuser or someone with migration
access from dropping the trigger itself — that is a different trust boundary than this item is
meant to cover.

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
| Duplicate "Uncategorized" category | Server-side seeding on register only; the client never seeds (§2.7) |
| **Client silently skips a record** because a lower `server_seq` committed after a higher one | Per-user counter incremented inside the write transaction; global Postgres sequence rejected (§2.12) |
| **Zombie record** on the client after an out-of-band hard delete | Hard delete banned on all four synced tables via DB trigger; enforcement is item 1.6 (§2.13, §2.16) |
| **A change invisible to sync** because a write path forgot to bump `server_seq` | Assignment lives in the repository layer, not the sync usecase (item 1.3, §2.15); conformance suite (§2.11) |
| **A stale device silently reverts a newer edit** | `base_seq` on every pushed record; server returns `conflict` and the client surfaces it (§2.14) |
| **A deleted record resurrected** by a stale edit landing later | Tombstones are sticky — an update never clears `deleted_at` (§2.14) |
| **Two budgets for one category-month**, invisible to LWW because the `id`s differ | Unique natural key `(user_id, category_id, period)`; the loser is `rejected` (§2.14) |
| One bad record blocks a device's entire backlog | Per-record batch granularity; a `rejected` record never stops the rest (§2.14) |
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
