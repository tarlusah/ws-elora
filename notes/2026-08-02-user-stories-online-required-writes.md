# Notes — writing user-stories.md for the online-required-writes cycle

**Date:** 2026-08-02
**Author:** `@pm`
**Output:** `shared/context/user-stories.md`

## Context

This was not a fresh planning pass. `shared/context/PRD-online-required-writes.md` was already
written and approved by the manager (2026-08-02) before I was engaged — my job was purely to
translate its approved §4 (Scope) and §5 (Acceptance criteria) into testable user stories for
`@qa` and `@architect`'s pre-flight gap review. Scope itself was not up for renegotiation, and I
did not attempt to trim or re-litigate it.

## What I read, in order

1. `shared/context/PRD.md` — baseline consolidated status (what's actually built today, both
   repos).
2. `notes/ADR-0004-online-required-writes.md` — the *why*.
3. `shared/context/PRD-online-required-writes.md` — the *what*, approved.
4. `notes/discussion-backlog.md` — settled-decision detail/reasoning trail.

## Structure chosen

Five epics, mapped directly to PRD structure so traceability is obvious:
- **Epic A** (US-1..US-8): the online-write happy path, one story per write action named in §5's
  first checklist bullet (create/edit/delete/review/split transactions; manage accounts,
  categories, budgets). Splitting these into 8 stories instead of 1 combined story exists because
  `@qa` needs independently testable, independently failable units — a single "manage
  everything" story would be untestable as one pass/fail unit.
- **Epic B** (US-9): the connectivity gate, written as *one* cross-cutting story rather than one
  per write type — the PRD is explicit this is a single mechanism (one gate in front of every
  write action, not a per-feature reimplementation), so the story mirrors that.
- **Epic C** (US-10): offline reading, one cross-cutting story across the five listed screens.
- **Epic D** (US-11, US-12, US-13): login/cache-lifecycle/session stories, matching PRD §5's
  remaining three checklist bullets 1:1.
- **Epic E** (US-14, US-15): the two backend/profile items from PRD §4.2 that aren't phrased as a
  §5 checklist bullet but are explicitly in scope (`default_account_id`, `/me` endpoint) — I
  included them because the task brief said to translate §4 *and* §5, and leaving them out would
  have meant `@architect`'s contract work for those two items had no corresponding story.

Every PRD §5 checklist line maps to at least one story — verified explicitly in the traceability
table at the bottom of `user-stories.md`, per the task requirement.

## Gaps I found and did not paper over

While cross-referencing PRD §5's acceptance criteria against the *actual current state* of the
frontend (`shared/context/PRD.md` §3), I found real tension between what §5 says must be testable
and what's actually built:

1. **Account creation ("add-account") and sub-category creation ("add-sub-category")** are listed
   in the baseline PRD as "coming soon" placeholders in the frontend — not shipped. PRD §5's AC1
   says "manage accounts... categories" without carving these out, and §4.3 (this cycle's
   frontend scope) doesn't mention building them. This is a genuine contradiction between what
   §5 claims is verifiable and what §4.3 commits to building — I flagged it under "Parked" rather
   than guessing whether finishing these placeholders is silently in scope.
2. **The Budget screen is an unbuilt 9-line stub**, and §4.3 itself calls it "its own open item,
   unaffected by this pivot" — yet Budget shows up in Goal G1 and in AC1/AC3 of §5. Same shape of
   gap as #1, specific to Budget. I recommended (not decided) that `@qa` verify budget-related
   acceptance criteria against the backend API directly this cycle, since the backend budget
   domain is already complete per the baseline PRD, and flagged the UI gap for the manager.
3. **Cache freshness indicator** is explicitly optional in the PRD ("if wanted") — no decision was
   made, so I wrote no acceptance criteria for it rather than inventing a threshold.
4. **Refresh-token rotation grace window** — confirmed to be built, but no specific duration was
   set anywhere (PRD, ADR-0004, or the backlog). I wrote the observable "log in again" behavior
   as testable (US-13) but flagged that a precise "survives a blip of N seconds" criterion needs
   a number from the manager before `@qa` can test it precisely.
5. **Whether system (non-user) categories can be renamed by the user** is listed as an unresolved
   *fact* (not a decision) in `notes/discussion-backlog.md`'s "Verification still owed" section.
   I scoped US-7's acceptance criteria to user-owned categories only, rather than assume system
   categories behave the same way.

None of these are scope-creep asks — they're places where the already-approved PRD's own
sections don't fully agree with each other or with the documented state of the frontend. I judged
guessing here would produce untestable or wrong acceptance criteria for `@qa`, which is worse
than surfacing the gap explicitly.

## What I deliberately did not do

- Did not touch the API contract, data model, or any technical decision — that's `@architect`'s
  job per my mandate.
- Did not re-open or re-trim PRD scope — it was approved before I was engaged, for a reason.
- Did not invent acceptance-criteria numbers (grace-window duration, freshness-label wording)
  where the PRD left them genuinely open — see "Parked" above.
