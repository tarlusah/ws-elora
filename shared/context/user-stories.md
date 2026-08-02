# User stories — Online-required writes cycle

- **Source PRD:** `shared/context/PRD-online-required-writes.md` (✅ Approved 2026-08-02)
- **Decision record:** `notes/ADR-0004-online-required-writes.md`
- **Persona:** "the Spendos user" — the app's single first user/persona throughout (personal
  expense tracker, one account, already authenticated). No new persona is introduced this cycle.
- **Status:** scope is not up for renegotiation here — it was approved by the manager. This
  document translates PRD §4 (Scope) and §5 (Acceptance criteria) into testable stories for
  `@qa`. Where the PRD itself is ambiguous or internally inconsistent, that is flagged under
  "Parked / Open questions" below rather than resolved by assumption.

Every PRD §5 acceptance-criteria line is covered by at least one story — see the traceability
table at the bottom.

---

## Epic A — Online writes (network required)

### US-1 — Create a transaction while online
As a Spendos user, I want to create a new transaction while my device has a connection, so that
my spending is recorded on the server as soon as I capture it.

**Acceptance criteria**
- Given the device is online, when the user completes the Capture flow (amount + account,
  category optional via `000` placeholder), then the transaction is created server-side and
  appears in History/Review after the next cache refresh.
- The created transaction has a server-generated `id` (no client-supplied id).
- The local Drift cache reflects the new transaction after the write succeeds (via the
  post-write refresh, PRD §4.3).

### US-2 — Edit a transaction while online
As a Spendos user, I want to edit an existing transaction while online, so that I can correct
mistakes (amount, account, category, notes).

**Acceptance criteria**
- Given the device is online, when the user edits a transaction's fields and saves, then the
  server record is updated and the local cache reflects the change after refresh.
- Editing a transaction the user does not own is rejected (existing auth rules apply — no change
  this cycle).

### US-3 — Delete a transaction while online
As a Spendos user, I want to delete a transaction while online, so that I can remove entries I
don't want.

**Acceptance criteria**
- Given the device is online, when the user deletes a transaction, then the server performs a
  hard delete (no `deleted_at` tombstone — ADR-0004 §2.2) and the row disappears from the local
  cache after refresh.
- The deleted transaction's id is gone permanently — no "undo" surface exists this cycle (PRD
  §4.2: an undo UX is a separate, independent decision if wanted later).

### US-4 — Review and categorize a captured transaction while online
As a Spendos user, I want to categorize a `000`-placeholder transaction from the Review
flash-card queue while online, so that my history stays accurate without slowing down capture.

**Acceptance criteria**
- Given the device is online, when the user assigns a category to a placeholder transaction in
  Review, then the transaction's reviewed state and category are persisted server-side.
- The transaction leaves the unreviewed queue once the write succeeds, and stays reviewed after
  a cache refresh.

### US-5 — Split a transaction while online
As a Spendos user, I want to split a transaction into multiple categorized parts while online, so
that a single purchase covering multiple budget categories is recorded accurately.

**Acceptance criteria**
- Given the device is online, when the user splits a transaction into two or more parent-child
  parts via the split sheet, then all resulting parts are persisted server-side and the sum of
  the parts equals the original transaction amount.
- The split is visible correctly (parent + children) after a cache refresh.

### US-6 — Manage accounts while online
As a Spendos user, I want to create, edit, archive/restore, and set a default account while
online, so that my money sources stay organized and my default account stays current.

**Acceptance criteria**
- Given the device is online, when the user performs any of create / edit / archive / restore /
  set-default on an account, then the server accepts the write and the local cache reflects it
  after refresh.
- Exactly one account is marked default at any time after a set-default action (see US-14 for the
  underlying `default_account_id` model change).
- **Scope caveat:** per the baseline PRD (`shared/context/PRD.md` §3), account *creation* via the
  app UI ("add-account") is currently a "coming soon" placeholder, not shipped functionality.
  This story's create-account criterion can only be verified end-to-end once that placeholder is
  built — see "Parked" below. Edit / archive / restore / set-default already have UI and are
  fully testable this cycle.

### US-7 — Manage categories while online
As a Spendos user, I want to create, edit, and hide/show categories (including sub-categories)
while online, so that my transaction categorization matches how I actually think about my
spending.

**Acceptance criteria**
- Given the device is online, when the user creates, edits, or hides/shows a user-owned category,
  then the server accepts the write and the local cache reflects it after refresh.
- **Scope caveat:** per the baseline PRD, sub-category *creation* via the app UI ("add-sub-category")
  is currently a "coming soon" placeholder — see "Parked" below. Whether system (non-user)
  categories can be renamed by the user is an unverified fact (see `notes/discussion-backlog.md`,
  "Verification still owed") — this story's acceptance criteria apply to user-owned categories
  only until that's confirmed.

### US-8 — Manage budgets while online
As a Spendos user, I want to create, edit, delete, and copy a monthly per-category budget while
online, so that I can plan and track spending limits.

**Acceptance criteria**
- Given the device is online, when the user (or a direct API call) performs create / edit /
  delete / copy on a budget, then the server accepts the write.
- **Scope caveat — flagged, not assumed:** the frontend Budget screen is an unbuilt 9-line stub
  as of the baseline PRD, and PRD §4.3 explicitly calls it "its own open item," unaffected by
  this cycle. There is no app UI to drive this story end-to-end yet. Recommend `@qa` verify this
  story's acceptance criteria against the backend API directly this cycle (budget CRUD is already
  complete server-side per baseline PRD §2); full UI-level verification is blocked on the
  separately-scheduled Budget frontend build. See "Parked" below.

---

## Epic B — Connectivity honesty (writes never silently queue or vanish)

### US-9 — Blocked write shows an explicit "needs connection" message
As a Spendos user, I want to be told immediately and clearly when a create/edit/delete action
can't go through because I'm offline, so that I never wonder whether my change was saved.

**Acceptance criteria**
- Given the device is offline (airplane mode / no network), when the user attempts to create,
  edit, or delete a transaction, account, category, or budget, then the action fails immediately
  with an explicit "needs connection" message.
- No local draft, queued write, or dirty record is created as a result of the failed attempt —
  confirmed by inspecting the local Drift store after the failed attempt (nothing new/changed).
- After reconnecting, the same action, retried by the user, succeeds normally (US-1 through US-8
  apply again) — the earlier failed attempt does not need to be "resolved," retried
  automatically, or referenced again by the app.
- This applies uniformly to every write surface in Epic A — one connectivity gate, not a
  per-feature reimplementation (PRD §4.3).

---

## Epic C — Offline reading

### US-10 — View cached data with no network connection
As a Spendos user, I want to view my Capture history, Review queue, History, Dashboard, and
Budget screens while offline, so that I can still check my finances without a connection.

**Acceptance criteria**
- Given the device is offline and the user has previously fetched data while online, when the
  user opens Capture history, Review, History, Dashboard, or Budget, then each screen renders
  using the last-fetched (cached) data, with no network error blocking the view.
- Dashboard aggregates (current + previous month, sparkbar, top-3 categories) compute correctly
  from the cached data alone, offline.
- **Scope caveat:** the Budget screen itself is not built this cycle (see US-8's caveat) — this
  story's Budget bullet is not testable end-to-end until that screen exists. All other screens
  listed are testable now.

---

## Epic D — Login, cache lifecycle, and sessions

### US-11 — Fresh install + login restores the complete dataset
As a Spendos user, I want a fresh install of the app, after I log in, to show all my accounts,
categories, budgets, and transactions exactly as they exist on the server, so that losing my
phone doesn't mean losing my data.

**Acceptance criteria**
- Given a user with existing server-side data, when they install the app fresh (empty local
  Drift DB) and log in, then every account, category, budget, and transaction from the server
  appears in the app after the initial fetch completes.
- No manual "sync" action is required — the fetch-and-populate happens as part of login (ADR-0004
  §2.3).

### US-12 — Switching users on the same device replaces the cache
As a Spendos user sharing a device with someone else (or switching my own second account), I want
logging in as a different user to completely replace what was cached before, so that I never see
someone else's data mixed with mine.

**Acceptance criteria**
- Given user A is logged in with cached data, when user B logs in on the same device, then all of
  user A's cached rows (accounts, categories, budgets, transactions) are cleared before user B's
  data is fetched and populated.
- After user B's login completes, no row attributable to user A is visible anywhere in the app
  (Capture history, Review, History, Dashboard, Budget, account/category pickers).

### US-13 — Refresh-token expiry prompts re-login without corrupting the cache
As a Spendos user, I want an expired session to simply ask me to log in again, without losing or
corrupting the data I can already see offline, so that a routine session expiry doesn't feel like
data loss.

**Acceptance criteria**
- Given the refresh token has expired (or the session was evicted by the 10-session soft limit),
  when the user next opens the app or attempts an action, then they are prompted to log in again.
- The local cache is left intact (not wiped) at the moment of expiry — the user can still view
  cached data offline before re-authenticating, per US-10.
- After successful re-login, the cache is refreshed/replaced normally per US-11/US-12 (same user
  re-authenticating does not need to wipe first; a different user logging in still follows US-12).
- Refresh-token TTL is 60 days, sliding (confirmed) — `@qa` can test the "log in again" prompt
  path directly (e.g. by forcing token expiry/eviction) without waiting out the full 60 days.

---

## Epic E — Supporting backend/profile items (PRD §4.2)

### US-14 — Exactly one default account per user
As a Spendos user, I want my default account to be unambiguous, so that Capture and other
account-defaulting flows never have to guess between two "default" accounts.

**Acceptance criteria**
- The system enforces a single default account per user via `user.default_account_id` (replacing
  the old per-account `is_default` flag — PRD §4.2).
- Setting a new default account is a single atomic operation from the user's point of view (no
  observable intermediate state with zero or two defaults).
- This is a data-model change with no new end-user-visible flow beyond US-6's set-default action —
  `@qa` verifies via US-6's acceptance criteria plus a direct check that only one account ever
  reports as default.

### US-15 — Profile screen shows real identity and stats
As a Spendos user, I want my Profile screen to show my actual name, email, and account stats
instead of placeholder values, so that the screen feels finished and trustworthy.

**Acceptance criteria**
- Given the device is online and the `/me` endpoint is available, when the user opens Profile,
  then their real identity fields (name, email) and available stat numbers render instead of
  placeholder text.
- This story covers the read-only identity/stats display only. Full profile *editing* and
  *preferences* remain "coming soon" placeholders per the baseline PRD and are **not** in scope
  this cycle — see "Parked" below.

---

## Parked / Open questions

Not scope renegotiation — the PRD's core scope stands. These are genuine gaps or contradictions
between PRD sections (or unresolved facts) that I did not want to paper over by guessing.

1. **Account creation ("add-account") and sub-category creation ("add-sub-category") UI are
   still "coming soon" placeholders** per the baseline PRD (`shared/context/PRD.md` §3), and
   `PRD-online-required-writes.md` §4.3 does not mention building them out this cycle. Yet §5
   AC1 lists "manage accounts... categories" as an acceptance-criteria bullet without carving
   these out. I did not assume either reading — flagging for the manager/architect to pick:
   (a) AC1's account/category bullets are verified against whatever CRUD already has UI, with
   the rest checked via direct API calls, or (b) finishing these two placeholders is silently
   pulled into this cycle's frontend scope. Affects US-6 and US-7.

2. **Budget frontend screen is an unbuilt stub, explicitly called "its own open item" unaffected
   by this pivot (PRD §4.3), yet Budget appears in Goal G1 and in AC1/AC3 of §5.** Same kind of
   gap as #1, specifically for Budget: recommend backend-API-only verification of budget-related
   acceptance criteria this cycle (US-8, and the Budget bullet of US-10), pending a manager call
   on whether the Budget UI build gets pulled forward. Not assumed either way.

3. **Cache freshness indicator** ("data as of [time]") is explicitly optional in PRD §4.3 ("if
   wanted") — no decision was made on whether to build it. No measurable acceptance criteria are
   written for it in this document; if the manager decides to build it, a follow-up story is
   needed once the decision is made.

4. **Refresh-token rotation + grace window** is confirmed to be built (ADR-0004 §2.4, backlog
   2.2 — "worth doing, not urgent"), but no specific grace-window duration was set. US-13 tests
   the observable "log in again" behavior at expiry, but a precise "survives a network blip of
   up to N seconds during rotation" criterion cannot be written until a number is chosen.

5. **Whether system (non-user) categories can be renamed/hidden by the user** is an open,
   unverified fact per `notes/discussion-backlog.md` ("Verification still owed"), not a decision.
   US-7's acceptance criteria are scoped to user-owned categories only until this is confirmed
   against the actual backend behavior.

---

## Traceability — PRD §5 acceptance criteria → user stories

| PRD §5 checklist item | Covered by |
|---|---|
| Online: create/edit/delete/review/split/categorize transactions; manage accounts, categories, budgets | US-1, US-2, US-3, US-4, US-5, US-6, US-7, US-8 |
| Offline: same actions fail immediately with "needs connection", nothing queued/lost | US-9 |
| Offline: view Capture history, Review, History, Dashboard, Budget from last-fetched data | US-10 |
| Fresh install + login restores the complete dataset | US-11 |
| Logging in as a different user replaces the cache, no data mixing | US-12 |
| Refresh-token expiry produces "log in again" without corrupting the local cache | US-13 |

Supporting backend/profile items from PRD §4.2 not phrased as a §5 checklist bullet but in scope:
US-14 (`default_account_id`), US-15 (`/me` endpoint).
