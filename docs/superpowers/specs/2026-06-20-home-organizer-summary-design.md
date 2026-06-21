# Home Organizer Summary — Design Spec

**Date:** 2026-06-20
**Status:** Approved (design); revised after Opus 4.8 spec review, meals cut from feed (2026-06-21); orchestration/seam hardened after Codex plan review, two passes (2026-06-21)
**Story:** `docs/product/backlog/mobile-ux/home-organizer-summary.md`
**Concept:** "The Now" + "What changed" — a calm hero plus a since-you-last-opened activity feed
**Scope:** Mobile breakpoints (≤768px), adult users. Extends the shipped home-dashboard redesign.

> **Revision note (2026-06-20):** A high-effort review verified the original §10 assumptions against the codebase and found several were false. This revision bakes the verified constraints into the design: calendar exposes only **expanded instances with nullable `id`** over a narrow query window; `ListSummary` carries **no items/timestamps**; no entity exposes `updatedAt`; FE #222's persister is **not** a reusable KV store; and the FE has **no family-local date helper** (it is device-local). The diff algorithm is also fully specified for re-edit, add-then-delete, and long-absence cases. See §4 and §10.

> **Revision note (2026-06-21, post-Codex plan review — round 2):** A second pass found that the first round's *serialization*, *baseline*, and *window* fixes were each incomplete. Now corrected: (a) cycles are serialized by a **promise-queue mutex**, not a generation guard — the guard could not stop two in-flight saves resolving out of order (§4.1, §4.5); (b) the divider baseline lives in a **cross-cycle ref re-frozen only at a meaningful open**, so an automatic cycle after an open can no longer move the divider, and `hiddenAt > 0` is required before the gap test (§4.1); (c) **reconciliation is now window-aware too** — the window-aware diff was being undone by a reconcile that dropped/demoted aged-out entries, and the "far-edge event surfaces next cycle" claim was false and is removed (§4.4); (d) `removed` rows no longer deep-link to the deleted entity (§5); (e) the member dot resolves `colorMap[member.color].hex`, not the raw color id (§6); (f) recurring collapse keys on `(kind, title, memberId, detail)` so visibly-different series edits stay separate (§4.2); (g) collapsed sub-rows are `inert`/`aria-hidden` and reduced-motion is opacity-only, not motion-off (§6).

> **Revision note (2026-06-21, post-Codex plan review):** A plan-level review surfaced orchestration and integration defects that this revision now closes in the spec (the plan carries the code). Verified against the codebase: (a) `idb-keyval.createStore` opens its database with **no version**, so a second object store cannot be added to the existing `family-hub-offline` DB — the activity store needs its **own database name** (§4.1); (b) detection must be **gated until both source queries settle** and serialized so concurrent cycles cannot lose writes, and a **meaningful open** is classified only on cold start or a real visible transition — never on a data-change cycle (§4.1, §4.5); (c) `refetchOnWindowFocus` is **off** app-wide, so return-to-visible must **explicitly refetch** before diffing (§4.5); (d) the calendar detection window **slides daily**, so window-boundary churn must be suppressed (§4.4); (e) deep-links must carry an **entity id** via the existing app-store navigation-intent pattern, not a bare module switch (§5); (f) the mobile gate must be **synchronous** because `App.tsx` renders Home before its desktop-redirect effect runs (§7); (g) `clearHomeActivity()` must clear the **localStorage markers too**, not only IndexedDB (§9). The `ActivityItem` seam is also reconciled to the built shape (§4.1).

## 1. Context

The shipped mobile home — "The Now" (`docs/superpowers/specs/2026-04-25-home-dashboard-redesign-design.md`) — is deliberately calm and calendar-only. Its §11 explicitly excluded chores/lists/meals because "none of these data sources exist." They exist now: `Chores`, `Lists`, `Recipes`, and `Meals` all ship with real data (`useChoresBoard`, `useMealsBoard`, `useLists`, `useRecipes`).

This spec evolves Home into an organizer summary **without redesigning "The Now."** It keeps the hero and agenda intact, inserts one quiet **state line** beneath the hero, and appends a **"Since you last opened" activity feed**. The feed answers the question a parent opens the app to ask when away from the always-on hub: *did anyone touch the shared plan?* — directly serving the PRD's core pain of shared visibility and "what's on the calendar?" coordination.

## 2. Anchor decisions (locked during brainstorming)

| # | Decision | Rationale |
|---|---|---|
| 1 | **Blend, not redesign.** Keep "The Now" hero + agenda untouched; add a state line + activity feed. | The calm hero is the proven glance surface; additions are subordinate, not a competing dashboard. |
| 2 | **Two lenses: state vs. activity.** State line = "what's true now" (chores left, tonight's dinner). Feed = "what changed since I last looked." | They answer different questions and may intentionally overlap (tonight's dinner can show in both). |
| 3 | **"Since you last opened" = hybrid** (rolling 48h window + a "new" divider). Baseline advances only on a *meaningful* open. | A mobile PWA returns to the foreground constantly; a naive "since last open" reads empty. The hybrid keeps "what did I miss" without erasing on quick peeks. |
| 4 | **Client-side diff now, behind a normalized `ActivityItem[]` interface (Approach 3).** Backend activity log deferred to the AI phase. | Delivers value with no backend. The feed is a *derived view*, not a new data domain, so "BE-first" does not apply yet. |
| 5 | **Single shared account → no edit attribution.** | One login per family; "members" are assignment targets/colors, not users. The feed is event-centric ("Event added · Dentist"), never actor-centric. Rows may tint by an entity's *assigned* member color. |
| 6 | **Mobile-only, family-wide additions.** | Same gating as the existing dashboard. The always-on tablet never renders this surface. State line + feed ignore member-chip focus. |
| 7 | **Feed sources = calendar + lists.** Chores and meals → state line only. Recipes excluded. | Chores change daily by design; recipes are a library; meals already surface as "tonight's dinner" in the state line, so meals-as-activity was cut from v1 (decided 2026-06-21) — the lowest-value, most-awkward source (no timestamp, null-able id, week-keyed). |

## 3. Information architecture

Top to bottom on the mobile home surface. **Bold = new; the rest is the existing "The Now," unchanged.**

```
 Good evening · Fri, Jun 20            header        (existing)
 (Mom)(Dad)(Mia)                       member chips  (existing, scopes hero+agenda only)
 ┌────────────────────────────────┐
 │ UP NEXT · in 40 min            │   HERO — "The Now", unchanged
 │ Soccer practice       6:00 PM  │
 └────────────────────────────────┘
 3 chores left today · Tacos tonight   ◄ NEW: state line (family-wide)

 Rest of today                         (existing)
   7:30 PM  Bath + bedtime
 Coming up                             (existing)
   Tomorrow 9:00 AM  Dentist

 Since you last opened                 ◄ NEW: activity feed (family-wide)
   Calendar · 2 added, 1 changed   ▸
   Groceries · +3 items
   ──────────── earlier ────────────
   Camping list · 2 checked off
                                   (+)  floating create (existing)
```

### 3.1 State line (new)

- **Content:** family-wide standing facts, priority order: chores remaining today, then tonight's dinner. One quiet line/strip under the hero.
- **Sources & derivation:**
  - Chores — `useChoresBoard().data.data.today.summary.remaining` (clean; already used by `chores-scope-column.tsx`).
  - Tonight's dinner — `useMealsBoard(weekStart)` where `weekStart = formatLocalDate(getWeekStartSunday(now))`; find the day whose `date === formatLocalDate(now)`, then the slot with `mealType === "dinner"`, read `primary?.title`. (Multi-step; spelled out so the plan doesn't reinvent it.)
- **Graceful segments:** each segment is independent and omitted when empty (all chores done → omit or "All done"; no dinner planned → omit). If both empty, the whole line is omitted (no empty chrome).
- **Device-local** "today" (the FE has no family-local helper — see §4.3). **Not chip-filtered** in v1. Chip-scoped chores is a noted later enhancement.

### 3.2 Activity feed (new)

Coalesced, expandable, reverse-chronological recent changes with a "new since you last looked" divider. Mechanics in §4, presentation in §5.

## 4. Data & mechanics

### 4.1 Sourcing (Approach 3 — client-side diff)

Derived on-device from data the app fetches. No new backend endpoint. **New** persistent state (the FE #222 persister is a single-key TanStack persister and cannot host this — see §10.3):

- A **new dedicated `idb-keyval` store in its own database** (e.g. database `family-hub-home-activity`, store `home-activity`). It must **not** reuse the offline cache's `family-hub-offline` database: `idb-keyval.createStore` calls `indexedDB.open(dbName)` with no version number, so `onupgradeneeded` fires only when the database is first created. The offline cache already created `family-hub-offline` at version 1 with the `query-cache` store; opening it again without a version bump will **not** create a second object store, and every transaction against the missing store throws (silently swallowed by the persistence guards). A distinct database name sidesteps versioned-upgrade plumbing entirely. The store holds:
  - `snapshot` — last-seen state of feed entities, keyed by a per-module composite key (§4.2), storing the fields needed to detect change + the title/detail needed to render a later "removed".
  - `changeLog: ActivityItem[]` — accumulated, coalesced changes within the window.
  - `snapshotSavedAt` — when `snapshot` was last written (drives stale-reseed, §4.4).
- localStorage markers: `lastSeen` (last *meaningful* open → divider), `hiddenAt` (last hide → classify next open).

```
detect(now, isOpenEvent):                          // runs inside the serialization mutex (below)
  if (events query or lists query not yet settled): // §4.5 — never diff partial data
      render feed from persisted changeLog; return  // show what we have; wait for data
  fresh = feed-window queries (events[today..+W], all list summaries)   // meals not in feed
  if (no prior snapshot) OR (now - snapshotSavedAt > STALE_RESEED):     // first run / long absence §4.4
      snapshot = fresh; snapshotSavedAt = now; window = currentWindow
      lastSeen = now; displayBaseline = now        // nothing is "new" yet
      render empty feed; return                     // reseed silently, no dump
  deltas = diff(fresh, snapshot, overlap(prevWindow, currentWindow))  // window-bounds-aware §4.4
  changeLog = mergeCoalesced(changeLog, deltas, detectedAt = now)   // merge FIRST (§4.4)
  changeLog = reconcile(changeLog, fresh, currentWindow)   // WINDOW-AWARE: keep aged-out entries (§4.4)
  changeLog = prune(changeLog, olderThan 48h)
  snapshot = fresh; snapshotSavedAt = now; window = currentWindow   // never miss a change
  // Classify the OPEN, not the data change: only an open can advance the divider.
  meaningful = isOpenEvent AND (coldStart || (hiddenAt > 0 AND now - hiddenAt > MEANINGFUL_GAP) || dayChanged(lastSeen))
  if (meaningful) displayBaseline = lastSeen        // re-freeze the divider ONLY at a real open
  render feed from changeLog; draw divider at displayBaseline (§5)
  if (meaningful) lastSeen = now                     // persist advance AFTER capturing the baseline

triggers (all enqueued behind one mutex):
  cold start                  → detect(now, isOpenEvent = true)
  source data changed         → detect(now, isOpenEvent = false)   // refresh log; never advance divider
  visibilitychange→hidden     → hiddenAt = now
  becoming visible            → refetch(events, lists); then detect(now, isOpenEvent = true)   // §4.5
```

Key properties: (1) detection runs on **every** settled load (nothing is lost), but the **divider advances only on a meaningful *open*** — a data-change cycle refreshes the log without marking anything seen, and a brief peek captures-and-keeps; (2) cycles are **serialized through a promise-queue mutex** — each cycle's `load→diff→save` runs to completion before the next begins, so a slow earlier save can never land after a newer one (a generation guard alone does **not** serialize: two saves already in flight can still resolve out of order); (3) the divider baseline is held **across cycles** and re-frozen **only at a meaningful open** — automatic cycles and quick peeks reuse it, so they cannot move the divider; (4) `hiddenAt > 0` is required before the gap test, so the default `hiddenAt = 0` (never hidden) does not read as "hidden for years."

`ActivityItem` is the integration seam — a future backend `GET /api/activity` (the AI phase) replaces the source with no UI change. The fields are **flat** (the `{module, entityId, date}` triple *is* the deep-link payload; there is no nested `deepLink` object) so the same record drives both diffing and navigation without a second mapping:

```
ActivityItem {
  storeKey: string         // `${module}:${entityKey}` per-module composite key (§4.2) — stable across loads
  module: "calendar" | "lists"
  kind: "added" | "edited" | "removed"
  title: string            // from fresh, or from snapshot for removals
  detail?: string          // "+3 items", "moved to 5:00 PM", "Tue 9:00 AM"
  memberId?: string        // assigned member's id (assignment, not authorship); color resolved at
                           //   render via useFamilyMemberMap — store the stable id, not a color string
  recurringEventId?: string// series id, for calendar coalescing (§4.2)
  date?: string            // formatLocalDate — part of the deep-link target
  entityId?: string | null // raw calendar id (may be null) / list id — the deep-link target
  detectedAt: number       // = occurredAt; no entity updatedAt exists today (§10.1)
}
```

> Seam note: the original draft carried a `memberColor` and a nested `deepLink`. We store **`memberId`** instead (member colors can change; the id is stable, and `useFamilyMemberMap()` already resolves id→color at render — see §6), and keep the deep-link fields flat. A future server `GET /api/activity` returns this same flat shape, so the UI is unchanged.

### 4.2 What counts as a change, per module (with composite keys)

- **Calendar** — added / edited / removed. Source data is **expanded instances with `id: string | null`** over a query window (§4.5), so:
  - **Composite key** = `id` when non-null, else `` `${recurringEventId}_${formatLocalDate(date)}` `` (per the existing `getEventKey` fallback).
  - "Edited" = a meaningful field moved: `title`, `startTime`/`endTime`, `date`/`endDate`, `isAllDay`, `location`, `memberId`. Ignore `htmlLink`/`source`/sync noise.
  - **Recurring:** there is **no source-event accessor** on the FE. We diff at the occurrence level, then **coalesce within the calendar group by `(recurringEventId, change-signature)`** — where the signature is everything the sub-row *renders*: `(kind, title, memberId, detail)`. Two instances collapse to one sub-row **only if they would display identically**; any visible difference — a retitle, a reassignment (dot color), a time move — keeps them **separate**. A `(kind, detail)`-only signature would wrongly collapse, e.g., a location change and a member change that happen to share the same displayed time; including title + memberId fixes that. The original "series = one item" guarantee is **dropped**; practically, coalescing yields one row per distinct, visibly-different series-change.
- **Lists** — source is `useLists()` → `ListSummary[]` = `{id,name,kind,totalItems,completedItems}` (**no items, no timestamps**; per-item data needs `useList(id)` per list, only cached for opened lists). So the feed detects, **per list (key = list `id`)**: list **created** (new id, rendered "<name> · New list" per §5), **renamed** (name changed), **removed** (id gone), and **item count deltas** surfaced as `+N items` (totalItems rose) / `N checked off` (completedItems rose) / `N removed` (totalItems fell). **No per-item identity diffing in v1.** This matches the §5 mockups and needs no per-item data.
  - **Decreasing `completedItems` (an item un-checked), with no other field change, is *not* surfaced** — it is a low-signal correction, and "since you last opened" is about forward progress and additions, not undo. It must **not** fall through to a generic "updated" row (there is no such signal). Concretely: if the only change is `completedItems` decreasing, the list produces **no delta**. A simultaneous `totalItems` change still surfaces via the `+N items` / `N removed` signals above.
- **Meals** — excluded from the feed (state line only; meals-as-activity cut from v1 — §7).
- **Chores** — excluded from the feed (state line only).
- **Recipes** — excluded entirely.

### 4.3 Timing & locale

- `MEANINGFUL_GAP` default ~4h; `STALE_RESEED` = the 48h window; both tuned in the plan.
- **Dates are device-local.** The FE has no family-local date helper (`family.timezone` is backend-only and never used for FE date math; all of `time-utils.ts` is device-local). `dayChanged`, "today," and `weekStart` all use device-local time. Acceptable for the single-home personas; family-tz correctness is a noted future enhancement, not v1.
- Window = 48h on `detectedAt`. Two independent caps, both **post-coalesce**, each with an "and N more" affix when exceeded: (1) **top-level feed entries** (the calendar group counts as one entry; each changed list is one entry) cap at ~20 — this is what bounds vertical length; (2) **calendar sub-rows inside the expanded group** cap separately (~10) so a busy week cannot produce an unbounded expansion. Capping only the *entry* count (without the inner sub-row cap) would let one calendar group hold arbitrarily many rows while counting as a single entry — hence both caps.

### 4.4 Algorithm edge cases (fully specified)

- **(merge / coalescing rule)** Per `(module, key)` keep exactly one `changeLog` entry. On a new delta for an existing key: `kind` resolves by precedence **added > removed > edited** (added-then-edited stays "added"; anything-then-removed becomes "removed"); `occurredAt = now` (latest detection); `detail`/`title` recomputed from the latest `fresh`/`snapshot`. A pure no-op (no watched field changed) produces no delta.
- **(re-edit before a meaningful open)** Handled by the merge rule — the entry updates in place; the divider hasn't moved, so it stays "new."
- **(add-then-delete across a peek)** `reconcile()` runs each load: any `changeLog` entry with `kind: "added"` whose key is **absent from `fresh`** is **dropped** (no phantom row, no 404 deep-link). An `edited` entry whose key vanished is **demoted to `removed`** (title from snapshot). "Deletions are free" via the snapshot only covers entities still present in the snapshot; `reconcile()` covers entities that were only ever in the log.
- **(long absence / stale snapshot)** If `now - snapshotSavedAt > STALE_RESEED`, the next load **reseeds silently** (refresh snapshot, emit no deltas) so a returning user is **not** shown a 20-item dump of changes they can't act on; the feed reads "all caught up" and populates from the next real change. (Trade-off: changes made during a multi-day absence are not retro-listed — acceptable, and the correct call given no server history.)
- **(first run / no snapshot)** Seed `snapshot` silently; feed shows the calm empty state.
- **(null/duplicate keys)** Composite keys (§4.2) make expanded event instances safely keyable despite `id: null`.
- **(sliding detection window)** The calendar query window is `[today, today+W]`, so it **slides forward every device-local day**. A naive snapshot diff would then fabricate changes at both edges on each rollover: events that drop below the new `today` vanish from `fresh` and read as **removed**; events newly inside the top edge read as **added** — neither was actually edited. Two suppressions are needed, and **both** must be window-aware or the fix is undone:
  - **Diff** only compares calendar entities within the **overlap** of the previous and current windows (`[max(prevStart,freshStart), min(prevEnd,freshEnd)]`); entities outside the overlap are pre-existing context, not deltas.
  - **Reconcile** must likewise preserve a logged calendar entry whose entity is **outside the current window** — it merely aged out of the fetch, it was not deleted. A non-window-aware reconcile would re-introduce the bug: it would **drop** an aged-out logged `added` and **falsely demote** an aged-out `edited` to `removed`, since both are absent from `fresh`. Out-of-window log entries are kept as-is until normal 48h pruning. (Lists are unwindowed and unaffected by either step.)
  - **Honest limitation (corrected):** a genuinely new event created for a date **beyond the previous far edge** does **not** surface as "added" when the window later slides over it. Once the slide brings it into `fresh`, the diff suppresses it as a top-edge entrant and writes it silently into the snapshot; the next cycle then sees it as unchanged. We cannot distinguish such an event from one that merely aged into range (there is no `createdAt`). This is accepted for v1 — the event is still visible in the calendar; the feed simply does not flag far-future additions (within-window additions surface normally). The earlier draft's claim that it "surfaces on the next cycle" was wrong and is removed.

### 4.5 Data dependencies & performance

- Home today fetches only a **3-day** events query (`use-dashboard-events.ts`). The feed needs more, so Home will **mount additional queries**: a **dedicated wider events query** for detection (e.g. `[startOfToday, +28d]` — tunable; this is what lets "Dentist next Tuesday" surface, which a 3-day window would miss) and `useLists()` for the feed, plus `useMealsBoard(thisWeek)` and `useChoresBoard()` for the state line.
- Honest cost: these are **real first-load fetches** on Home (deduped by TanStack Query and shared with the module tabs / offline cache, but not free). There is **no new backend endpoint** and no duplicate of an existing in-flight query; "reuse" means reusing the existing query definitions, not avoiding fetches.
- The diff/feed compute runs over in-memory data **after** the hero paints; it must not block first paint.
- **Detection is gated until both source queries have settled with data.** While `useCalendarEvents`/`useLists` are pending or errored, the hook renders the persisted `changeLog` and does **not** snapshot or diff. Treating a not-yet-loaded query as an empty array would seed an empty first snapshot (then report everything as "added" when data arrives) or read a pending module as fully "removed." The source arrays must also be **stable identities** when empty (a shared `EMPTY` constant, not a fresh `?? []` each render) so the detection effect does not re-fire on every render.
- **Detection cycles are serialized through a mutex.** A cold start, a data-change, and a return-to-visible can each trigger a cycle; because each cycle is `load → diff → save`, overlapping cycles can persist out of order and lose writes. A **promise-queue mutex** chains cycles so each one's `load→diff→save` completes before the next begins — the newest-queued cycle therefore writes last and wins. A generation guard alone is **insufficient**: it can drop a stale cycle's *render*, but once two `saveState` calls are already in flight (a newer cycle started after the older passed its final check) the older write can still resolve last and clobber the newer state. Every cycle runs (no gen-skip), so a meaningful-open cycle is never dropped because a data-change cycle queued right after it.
- **Return-to-visible must explicitly refetch.** `refetchOnWindowFocus` is **off** app-wide (`providers/query-client.ts`), so simply re-running detection on visible would re-diff the *same stale cache* and surface nothing changed on another device. The feed's visibility handler `await`s `events.refetch()` + `lists.refetch()` **before** detecting. (Cross-device freshness is still pull-only/no-websockets per §7; this just makes "open the app and see what changed" actually refetch.)
- The wider events query's range is **memoized by a local-date string** (`formatLocalDate(today)`), not by a `Date` object — `startOfDay(new Date())` is a new reference every render and would churn the range (and thus the query key / detection effect) needlessly. The injected `nowProvider` is likewise stabilized (a ref or `useCallback`) so the visibility listener is not torn down and re-registered on every Home render.
- The feed owns a single, dedicated visibility hook (records `hiddenAt` on hide; refetches + re-runs detection on return-to-visible). It must **not** duplicate the hero's `now`-recompute logic (`use-hero-state.ts`); a separate single-purpose listener for the hidden-timestamp is acceptable (relaxed 2026-06-21 — two passive single-purpose listeners are cleaner than coupling the hero hook to the feed). The plan's Tech-Stack "reuse" list should name this as a *new* feed-owned listener, not as reuse of `useDashboardNow`. The divider baseline lives in a **ref that persists across cycles** and is re-frozen **only at a meaningful open** (to the pre-advance `lastSeen`); every render — open, data-change, or peek — draws the divider at that ref. Order on a meaningful open: re-freeze `displayBaseline = lastSeen`, render at it, **then** advance `lastSeen`. Automatic cycles never touch the baseline, so they cannot move the divider.

## 5. Feed presentation & interaction

Rule: **group only when grouping reduces noise.** A module with ≥2 changes collapses into one summary line with an expander; a single change renders directly. Lists are inherently per-list.

- **Calendar** — coalesces all event changes into one expandable group; expand reveals each change with a marker (`+` added · `~` changed · `−` removed) and a time/affix; recurring series collapse to one sub-row per distinct rendered signature `(kind, title, memberId, detail)` (§4.2). Each sub-row deep-links to that event/date.
- **Lists** — one row per changed list, **name first** for scannability: "Groceries · +3 items", "Camping · 2 checked off", "Party · New list" (a created list reads `<name> · New list`, consistent with the others — not "New list · Party"). Tap opens the list. No item-level expansion in v1.

```
 Since you last opened                       Calendar expanded:
   Calendar · 2 added, 1 changed   ▸           Calendar · 2 added, 1 changed   ▾
   Groceries · +3 items                          + Dentist            Tue 9:00 AM
   ──────────── earlier ────────────            + Swim lesson        Sat 10:00 AM
   Camping list · 2 checked off                  ~ Soccer (weekly)  → 5:00 PM
```

- **Ordering:** newest-first by `occurredAt`; a group sits at its most-recent change. Tiebreak within a same-`detectedAt` batch: module priority (calendar > lists > meals), then title — so the order is deterministic and doesn't reshuffle between renders.
- **Divider:** drawn at `lastSeen`; shown **only** when both a non-empty "new" and a non-empty "earlier" section exist (otherwise omitted). This means the divider appears only right after a meaningful open that had prior context — intended, not a bug.
- **Empty state:** a calm "You're all caught up" line mirroring the hero's all-clear tone.
- **Expansion is ephemeral** — collapses on the next meaningful open; no persisted toggle. Because `HomeDashboard` stays mounted across background→foreground, the expanded/collapsed state must be **reset on the meaningful-open epoch** (e.g. key the group list on a `meaningfulOpenId` that the orchestration hook bumps on a meaningful open), not left to persist in component state.
- **Deep links carry an entity id, not just a module.** A row opens the relevant **entity**: a list row opens that list's detail; an in-window calendar row opens its event sheet; an out-of-window calendar row opens the Calendar module focused on that event/date. A bare `setActiveModule("lists")` cannot do this — `selectedListId` is component-local in `ListsView`. Use the existing **app-store navigation-intent pattern** (the same `start*Draft` / `consume*Draft` shape already used for meal-placement and recipe-creation): the feed sets a `listDetailIntent` / `calendarFocusIntent` alongside the module switch, and `ListsView` / `CalendarModule` consume it on mount to open the target. In-window calendar rows still open directly via the dashboard's existing `handleEventClick` (no module switch), consistent with "rows open detail, don't switch tabs."
  - **`removed` rows must not deep-link to the deleted entity** (it would 404). A removed list lands on the **Lists module** (no detail); a removed calendar event **focuses its day** in the Calendar (the day still exists), not a dead event sheet.

## 6. Visual & motion vocabulary

- **No new tokens.** Reuse cream/purple, Nunito, member colors, and the 4/8 spacing system.
- **Motion** reuses the shipped system: easing `cubic-bezier(0.32, 0.72, 0, 1)`, durations 150/250/400ms; expand/collapse and divider use these. `prefers-reduced-motion` is **opacity-only, not motion-off**: the expand/collapse height animation is gated behind `motion-safe`, while an opacity fade stays on — reduced-motion users get a cross-fade, not an abrupt snap and not a height slide.
- **Collapsed sub-rows are inert.** When a calendar group is collapsed the sub-rows remain mounted (so collapse can animate) but the sub-list is `inert` + `aria-hidden` — fully non-interactive and out of the accessibility tree, not merely `tabindex=-1`.
- **Press feedback & haptics** come free via shipped seams — `usePressable` (FE #230) and optional-haptics (FE #236, behind its per-device toggle). No new interaction infra. **Every interactive row is pressable** — the collapsed group/list rows *and* the expanded calendar sub-rows; the sub-rows must not be plain buttons missing the press seam.
- Feed rows use quiet field styling (no per-row card chrome), consistent with the agenda; member color appears only as a small leading dot where an item has an assignment. The dot is **part of v1, not deferred**: `useFamilyMemberMap()` (`api/hooks/use-family.ts`) already resolves `memberId → color`, so the dot is a small, available wiring — pass the member map into the feed and tint the leading dot from `row.memberId`.
- **Touch targets:** every row holds a **≥44px** target (`min-h-11`); the compact `py-2` on expanded sub-rows is below that and must be raised.

## 7. Scope

### In scope
- Mobile (≤768px) home surface only, where the existing dashboard renders (`activeModule === null`).
- State line (chores-left-today + tonight's dinner), family-wide, device-local.
- "Since you last opened" feed for calendar + lists, with coalescing, expand, divider, empty state, 48h/max-row window, and the §4.4 edge handling.
- New `idb-keyval` activity store + localStorage markers, cleared on logout/401 (§9).

### Out of scope / non-goals
- **AI summary / overload warnings** — future phase; this spec only leaves the `ActivityItem` seam.
- **Backend activity log** — deferred to the AI phase (Approach 2).
- **Chores/recipes in the feed; list item-level identity diffing/expansion; a manual "mark all seen" button; chip-filtering of the state line/feed.**
- **Push notifications for changes** — owned by the Notifications story; this feed is in-app/pull only, surfacing on next refetch (no websockets).
- **Touchscreen/tablet home and child-mode home** — their own deferred stories.
- **Real-time cross-device sync; family-timezone-correct dates.**
- **Meals in the activity feed** — cut from v1 (decided 2026-06-21); meals surfaces only via the state line.

## 8. Dependencies

| Type | Item | Effect |
|---|---|---|
| Base | Shipped home-dashboard redesign ("The Now") | Extended; hero/agenda/chips/"+" reused unchanged. |
| Data | `useLists`, `useMealsBoard`, `useChoresBoard`, calendar events query | All shipped. Feed + state line are derived views; Home mounts them (§4.5). |
| New code | A new `idb-keyval` activity store + localStorage markers | **Not** a reuse of FE #222; clearing must be wired into logout + 401 (§9). |
| Future | AI day-summary story; backend activity log | Successors that consume the `ActivityItem` seam. Not prerequisites. |

## 9. Acceptance criteria

### Functional
- [ ] On the mobile home surface (≤768px), the hero, member chips, "rest of today," "coming up," and floating "+" are unchanged from the shipped dashboard.
- [ ] A state line under the hero shows chores remaining today and tonight's dinner; each segment omits when empty; the whole line omits when both are empty.
- [ ] State line and feed are family-wide and unaffected by member-chip focus; chips still filter the hero + agenda.
- [ ] The feed shows recent changes for calendar and lists only (chores, meals, and recipes never appear in the feed).
- [ ] Changes are detected on every load and accumulated; a brief background→foreground (quick peek) neither advances the divider nor drops unseen changes.
- [ ] The "new since you last looked" divider advances only on a meaningful **open** (cold start, visible after `MEANINGFUL_GAP`, or device-local day rollover) — **never on a background data-change/refetch cycle**, and never spuriously before the first hide (the `hiddenAt = 0` default must not read as "hidden 4h+"). It is shown only when both "new" and "earlier" sections are non-empty.
- [ ] Detection never runs against partial data: while either source query is pending/errored the persisted log still renders, but no new snapshot is taken and nothing is mis-reported as added/removed. The first real data arrival does not dump every entity as "added." Window-edge churn from the daily-sliding calendar window is suppressed (§4.4).
- [ ] Calendar changes coalesce into one expandable group when ≥2; a single change renders directly; recurring instances collapse only when `(recurringEventId, kind, title, memberId, detail)` matches, yielding one sub-row per distinct rendered series-change (§4.2).
- [ ] Lists changes render one row per changed list using summary signals only (created/renamed/removed + count deltas) — no per-item data required.
- [ ] An entity added then removed before a meaningful open produces **no** lingering "added" row (reconcile against `fresh`); a removed entity present in the snapshot shows as "removed" with its prior title.
- [ ] After a long absence (> `STALE_RESEED`), the feed reseeds silently and shows "all caught up" rather than a bulk dump.
- [ ] Each feed row deep-links to the relevant **entity** (list detail by id; calendar event/date), via the app-store navigation-intent pattern — not a bare module switch. A row-click test asserts the entity target (list id / event), not merely that `activeModule` changed. Expansion is ephemeral and resets on the meaningful-open epoch.

### Quality
- [ ] No new design tokens; spacing/type/color/motion from the existing system; `prefers-reduced-motion` respected.
- [ ] Feed rows inherit shipped press-feedback + optional-haptics seams; no new interaction infra. Visibility handling is consolidated into single-purpose hooks with no duplicated `now`-recompute logic (a dedicated feed listener for the hidden-timestamp is acceptable).
- [ ] Layout holds 320px–768px without horizontal overflow; rows ≥44px touch target.
- [ ] Gated to mobile **synchronously** (via `useIsMobile`), so the tablet/touchscreen surface never mounts the feed logic — *not* relying on the desktop→Calendar redirect, which is a post-render `useEffect`: `App.tsx` renders `HomeDashboard` for `activeModule === null` on the first render before that effect runs, so a redirect-only gate would still fire the wider queries and persist a snapshot once on desktop. A desktop-width test asserts the feed/state-line do **not** mount.

### Performance / data
- [ ] No new backend endpoint. The feed/state line derive from existing query definitions; Home mounting the wider events query + lists/meals/chores is acknowledged as real first-load fetching (deduped by TanStack Query).
- [ ] Diff/feed compute runs after the hero paints and does not block first paint.
- [ ] The new `idb-keyval` store and `lastSeen`/`hiddenAt` markers are per-device and **explicitly cleared on logout and on the 401 handler** (alongside `clearOfflineReadCache`), preventing cross-account leakage on a shared device. `clearHomeActivity()` clears **both** the IndexedDB store **and** the two localStorage markers — neither logout nor the 401 handler does a blanket `localStorage.clear()`, so a store-only clear would leave one account's divider baseline for the next. **Both** the logout path and the 401 path have a test asserting the clear is awaited.

## 10. Resolved constraints (verified against code) + plan-time tunings

Verified during the spec review (file:line evidence in the review record):

1. **No `updatedAt`/`createdAt`** on calendar (`CalendarEventResponse`) or meals (`MealSlot`); `ListItem`/`ListDetail` have them but `ListSummary` (what `useLists()` returns) does **not**. → `occurredAt = detectedAt`. The snapshot must store all watched fields so "edited" is detectable without timestamps.
2. **Calendar returns expanded instances with `id: string | null`** (recurrence expanded server-side), via a range-bounded query. → composite key + `recurringEventId` coalescing (§4.2); a dedicated wider detection window (§4.5).
3. **FE #222 persister is a single-key TanStack `PersistedClient` persister with an allowlist** — not a KV store; `clearOfflineReadCache()` clears only its store. → new `idb-keyval` store + explicit clearing wired into both `useLogout()` and the 401 handler.
4. **No family-local date helper; FE is device-local** (`time-utils.ts`; `family.timezone` is BE-only). → device-local dates in v1.
5. **Chores expose `today.summary.remaining` cleanly; meals require weekStart+dayIndex+mealType derivation** for tonight's dinner (§3.1).
6. **`idb-keyval.createStore` opens its DB with no version** (`indexedDB.open(dbName)`); `onupgradeneeded` fires only at DB creation, so a second object store cannot be added to the already-created `family-hub-offline` DB. → the activity store uses its **own database** (§4.1).
7. **`refetchOnWindowFocus` is `false` app-wide** (`providers/query-client.ts`). → return-to-visible must explicitly `refetch()` before diffing (§4.5); it cannot rely on focus refetch.
8. **`App.tsx` renders `HomeDashboard` for `activeModule === null` before** its desktop→Calendar redirect `useEffect` runs. → the mobile gate must be **synchronous** (`useIsMobile`), not redirect-based (§7, §9).
9. **Neither `useLogout()` nor `handleUnauthorized()` blanket-clears `localStorage`** (they remove only specific auth/family keys). → `clearHomeActivity()` must clear the `lastSeen`/`hiddenAt` markers itself (§9).
10. **The app-store already has a navigation-intent pattern** (`startMealPlacementFromRecipe`/`consumeMealPlacementDraft`, `startRecipeCreationFromMealSlot`/`consumeRecipeCreationDraft`). → reuse it for feed deep-links carrying a list id / calendar focus (§5).

Plan-time tunings: exact `MEANINGFUL_GAP`, `STALE_RESEED`, the detection-window length (`+W` days), the post-coalesce entry cap, and the calendar sub-row cap.

## 11. Story-file & roadmap updates required

- Create `docs/product/backlog/mobile-ux/home-organizer-summary.md` (status: planned), linking this spec + the plan once written, and cross-linking the shipped home-dashboard-redesign story as its base.
- Roadmap: promote "Home organizer summary" from the "Recommended additions to shape" note into a captured story under the Mobile UX polish backlog; note future siblings (AI day-summary story; backend activity log).

## 12. Future directions (not in this story)

- **AI day-summary / overload warnings** — consumes aggregated day state + `ActivityItem[]` to summarize the day or flag conflicts (e.g. a long recipe on an overloaded day). Wants durable server-side history → motivates the backend log below.
- **Backend activity log (Approach 2)** — append-only server record exposed via `GET /api/activity?since=…`; drop-in for the client diff via the `ActivityItem` seam; the right substrate once cross-device consistency, family-tz correctness, deletions-history, or AI history is required.
