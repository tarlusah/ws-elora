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
| Client, per row | `sync_state`: `clean` / `dirty` / `rejected` | What still needs pushing |

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

    C->>S: POST /sync/changes<br/>protocol_version + changes[]
    S->>S: Reject unsupported protocol_version
    S->>S: Group by Kind and order the kinds<br/>account → budget → transaction<br/>client ordering is NOT trusted

    S->>A: ApplyBatch(changes of kind "account")
    A->>A: Validate — server is authoritative
    A->>P: UPDATE user_sync_state<br/>SET last_seq = last_seq + 1 RETURNING
    A->>P: UPSERT row with new server_seq
    Note over A,P: Both statements in ONE transaction
    A-->>S: Result[] — applied / rejected

    S->>B: ApplyBatch(kind "budget")
    B-->>S: Result[]

    S->>T: ApplyBatch(kind "transaction")
    T->>T: Order internally:<br/>split parent before child
    T-->>S: Result[]

    S-->>C: results[] keyed by id, each with status + server_seq
```

**OPEN (item 1.5):** what happens when one record inside a batch is rejected — whole batch
rolled back, or the rest still applied — and whether `conflict` means anything at all on a
single-device product. The diagram shows only `applied` / `rejected`.

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
    dirty --> rejected: push returned rejected
    rejected --> dirty: user fixes it in the needs-attention screen
    clean --> clean: pulled again — UPSERT is idempotent

    note right of rejected
        A new UI surface.
        The record is never discarded
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

## 8. Note on ADR §2.4

§2.4 states the required order as
`accounts → categories (parent before child) → budgets → transactions`
and applies it to both push and hydration. Since §2.7 made categories pull-only, categories
cannot appear on the push path at all. The two orders are different:

- **Push (client → server):** accounts → budgets → transactions
- **Apply on pull (server → client):** accounts → categories → budgets → transactions

The diagrams above follow that split. §2.4 should be reworded to match — flagged, not yet
changed.
