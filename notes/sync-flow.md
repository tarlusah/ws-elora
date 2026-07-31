# Sync mechanism — flow diagrams

Companion to `ADR-0003-offline-first-sync-and-receipt-scan.md`. The ADR records *why*; this
file shows *how it runs*. Decisions shown here come from ADR §2.2–§2.9 and §2.11–§2.13.

Nothing here is a new decision. Where something is still open it is marked **OPEN**.

---

## 1. What the mechanism is made of

Three pieces of bookkeeping, and nothing else.

| Where | What | Purpose |
|---|---|---|
| Every synced row, both sides | `id` (client UUIDv7), `updated_at`, `deleted_at`, `server_seq` | Identity, tombstone, position |
| Server, one row per user | `user_sync_state.last_seq` | The sequence generator |
| Client, one value | `last_synced_seq` | The cursor — how far this device has seen |
| Client, per row | `sync_state`: `clean` / `dirty` / `flagged` / `rejected`, plus the `server_seq` this row was last known at | What still needs pushing, and what version it was edited from |

The whole protocol follows from one rule:

> Whenever the server writes anything belonging to a user, it increments that user's
> `last_seq` and stamps the new value onto the row it wrote.

So *"what changed since I last looked"* is just `WHERE server_seq > cursor`. There is no
changelog, no device table, no version history.

---

## 2. The full cycle on reconnect

Push always runs before pull. That ordering is what protects unsynced local writes.

```mermaid
flowchart TD
    A["Device reconnects"] --> B{"Any rows with<br/>sync_state = dirty?"}
    B -->|Yes| C["PUSH<br/>POST /v1/sync/changes"]
    B -->|No| D["PULL<br/>GET /v1/sync/changes?since=cursor"]
    C --> C2["Apply results per record:<br/>applied → clean<br/>rejected → rejected"]
    C2 --> D
    D --> E["UPSERT every change into Drift<br/>FK checks deferred inside the batch tx"]
    E --> F["Advance cursor to next_cursor"]
    F --> G{"has_more?"}
    G -->|Yes| D
    G -->|No| H["Synced"]

    style C fill:#e8f0fe,stroke:#4285f4
    style D fill:#e6f4ea,stroke:#34a853
    style H fill:#f1f3f4,stroke:#5f6368
```

The device re-receives its own pushed rows in the following pull. That is harmless and
deliberate: apply is an idempotent UPSERT, so a retry from the old cursor is always safe.

---

## 3. Push — client to server

Push carries business rules, so it goes through each domain's own usecase. The server
re-validates everything (ADR §2.8).

Categories and user profile are **absent by construction** — they are pull-only, so they have
no `SyncPushable` implementation and therefore no push door at all (ADR §2.11).

```mermaid
sequenceDiagram
    autonumber
    participant C as Flutter client
    participant S as sync usecase
    participant A as account usecase
    participant B as budget usecase
    participant T as transaction usecase
    participant P as PostgreSQL

    C->>S: POST /sync/changes<br/>protocol_version + changes[] with base_seq
    S->>S: Reject unsupported protocol_version
    S->>S: Group by Kind and order the kinds<br/>account → budget → transaction<br/>client ordering is NOT trusted

    S->>A: ApplyBatch(changes of kind "account")
    A->>A: Validate — server is authoritative
    A->>A: row.server_seq > base_seq ?<br/>→ stale write, mark conflict
    A->>P: UPDATE user_sync_state<br/>SET last_seq = last_seq + 1 RETURNING
    A->>P: UPSERT row with new server_seq<br/>deleted_at is never cleared
    Note over A,P: Both statements in ONE transaction
    A-->>S: Result[] — applied / conflict / rejected<br/>per record, one failure does not stop the rest

    S->>B: ApplyBatch(kind "budget")
    B-->>S: Result[]

    S->>T: ApplyBatch(kind "transaction")
    T->>T: Order internally:<br/>split parent before child
    T-->>S: Result[]

    S-->>C: results[] keyed by id, each with status + server_seq
```

**Why `base_seq` is here (ADR §2.14).** §2.9 requires push before pull, so the pushing device is
pushing *blind* — it has not yet learned what changed on the server. It does know which version
it edited, so it sends that. The server holds both halves and makes the comparison the device
could not: `row.server_seq > base_seq` means the write stood on a stale version.

Three statuses, and the middle one is the one most likely to be implemented wrongly:

| Status | Written? | Meaning |
|---|:--:|---|
| `applied` | yes | `base_seq` matched, or the server had never seen this `id` |
| `conflict` | **yes** | Applied under LWW, but it overwrote something newer. Surface it to the user |
| `rejected` | no | A rule or constraint failed. Stays `rejected` on the device |

---

## 4. Pull — server to client

Pull has no business rules; it is pure projection of current state.

```mermaid
sequenceDiagram
    autonumber
    participant C as Flutter client
    participant S as sync usecase
    participant D as each SyncPullable domain
    participant P as PostgreSQL

    C->>S: GET /sync/changes?since=1200&limit=100
    loop account, budget, transaction, category, user profile
        S->>D: ChangesSince(userID, 1200, 100)
        D->>P: WHERE user_id = ? AND server_seq > 1200<br/>ORDER BY server_seq LIMIT 100
        P-->>D: rows, tombstones included
        D-->>S: Change[] — envelope + the domain's own DTO as payload
    end
    S->>S: Merge-sort by server_seq, take the top 100
    S-->>C: changes[], next_cursor, has_more
```

Each unit is queried for `n` rows to return `n` overall. The over-fetch is immaterial at this
data volume, and it is what keeps the cursor a single `int64` instead of a compound cursor
(ADR §2.11).

A tombstone is not a special case: a deleted row has `deleted_at` set and a bumped
`server_seq`, so it flows through the same query as any other change.

---

## 5. Why the sequence comes from a per-user counter

The generator choice is not cosmetic. A global Postgres sequence loses records silently,
because the number is taken *before* the commit becomes visible.

```mermaid
sequenceDiagram
    participant TA as Write A
    participant TB as Write B
    participant C as Client pulling

    Note over TA,C: REJECTED — global sequence
    TA->>TA: nextval → 100
    TB->>TB: nextval → 101
    TB->>TB: COMMIT — 101 is now visible
    C->>C: pulls, sees 101, stores cursor = 101
    TA->>TA: COMMIT — 100 is now visible
    Note over C: 100 sits behind the cursor.<br/>The client never sees it. No error is raised.
```

```mermaid
sequenceDiagram
    participant TA as Write A
    participant TB as Write B
    participant C as Client pulling

    Note over TA,C: ADOPTED — per-user counter inside the tx
    TA->>TA: UPDATE user_sync_state ... RETURNING → 100<br/>row lock held
    TB--xTB: waits on the row lock
    TA->>TA: COMMIT — 100 visible, lock released
    TB->>TB: RETURNING → 101
    TB->>TB: COMMIT — 101 visible
    C->>C: pulls, sees 100 then 101
    Note over C: Numbers become visible in order.<br/>Nothing can be skipped.
```

Contention is one row lock per write per user — effectively zero on a single-device product.

---

## 6. Record lifecycle on the client

```mermaid
stateDiagram-v2
    [*] --> clean: pulled from server
    [*] --> dirty: created locally, UUIDv7 assigned

    clean --> dirty: edited or deleted offline
    dirty --> clean: push returned applied
    dirty --> flagged: push returned conflict — the write DID land
    flagged --> clean: user acknowledges it overwrote something newer
    dirty --> rejected: push returned rejected — the write did not land
    rejected --> dirty: user fixes it in the needs-attention screen
    clean --> clean: pulled again — UPSERT is idempotent

    note right of rejected
        One needs-attention surface
        serves both flagged and rejected.
        A record is never discarded
        and never silently dropped.
    end note
```

Deletion is not a state here. A deleted record sets `deleted_at` and goes `clean → dirty` like
any other edit; the tombstone then syncs as ordinary data.

---

## 7. Login — the three cases that must not collapse

This is where data is actually at risk (ADR §2.9).

```mermaid
flowchart TD
    L["Login succeeds"] --> Q1{"Local database<br/>holds data?"}
    Q1 -->|No| H["Clean install<br/>PULL since = 0"]
    Q1 -->|Yes| Q2{"Same user as<br/>the local data?"}
    Q2 -->|No| W["Different user<br/>isolate or wipe first,<br/>then PULL since = 0"]
    Q2 -->|Yes| PF["Same user, device still holds writes<br/>PUSH FIRST, then PULL from the stored cursor"]
    PF --> N["Never wipe.<br/>Refresh failure and session eviction<br/>both land here — not in clean install"]

    style PF fill:#fce8e6,stroke:#d93025
    style N fill:#fce8e6,stroke:#d93025
```

The middle branch is the common one in practice and the primary data-loss risk. An expired
refresh token or a session evicted by the max-10 soft limit must never be mistaken for a clean
install.

---

## 8. Push and apply use different orders

Foreign keys must resolve at ingest, but the two directions do not carry the same kinds —
categories are pull-only (§2.7), so they cannot appear on the push path at all.

- **Push (client → server):** accounts → budgets → transactions (parent before split child)
- **Apply on pull (server → client):** accounts → categories (parent before child) → budgets → transactions

ADR §2.4 originally gave one list for both; it has been corrected to match.

---

## 9. What the server decides, per record

Each record is decided on its own. One `rejected` never stops the rest of the batch — a device
pushing three days of offline work cannot be blocked by a single record the server refuses.

```mermaid
flowchart TD
    R["Record arrives in ApplyBatch"] --> V{"Business rules and<br/>constraints pass?"}
    V -->|No| REJ["rejected<br/>not written, stays on the device"]
    V -->|Yes| B{"row.server_seq > base_seq?"}
    B -->|No| APP["applied<br/>written, server_seq bumped"]
    B -->|Yes| CON["conflict<br/>WRITTEN under LWW,<br/>then flagged to the user"]
    APP --> T["In both write paths:<br/>deleted_at is never cleared"]
    CON --> T
```

The three cases this exists for (ADR §2.14): a stale device reverting a newer edit, a stale edit
resurrecting a deleted record, and two devices creating one logical budget slot. None of them is
a concurrency race — the writes are days apart — which is why no lock, timestamp, or device key
addresses them.
