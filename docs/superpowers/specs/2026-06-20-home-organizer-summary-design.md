# Home Organizer Summary — Design Spec

**Date:** 2026-06-20
**Status:** Approved (design); revised after Opus 4.8 spec review, meals cut from feed (2026-06-21)
**Story:** `docs/product/backlog/mobile-ux/home-organizer-summary.md`
**Concept:** "The Now" + "What changed" — a calm hero plus a since-you-last-opened activity feed
**Scope:** Mobile breakpoints (≤768px), adult users. Extends the shipped home-dashboard redesign.

> **Revision note (2026-06-20):** A high-effort review verified the original §10 assumptions against the codebase and found several were false. This revision bakes the verified constraints into the design: calendar exposes only **expanded instances with nullable `id`** over a narrow query window; `ListSummary` carries **no items/timestamps**; no entity exposes `updatedAt`; FE #222's persister is **not** a reusable KV store; and the FE has **no family-local date helper** (it is device-local). The diff algorithm is also fully specified for re-edit, add-then-delete, and long-absence cases. See §4 and §10.

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

- A **new dedicated `idb-keyval` store** (e.g. `home-activity`) holding:
  - `snapshot` — last-seen state of feed entities, keyed by a per-module composite key (§4.2), storing the fields needed to detect change + the title/detail needed to render a later "removed".
  - `changeLog: ActivityItem[]` — accumulated, coalesced changes within the window.
  - `snapshotSavedAt` — when `snapshot` was last written (drives stale-reseed, §4.4).
- localStorage markers: `lastSeen` (last *meaningful* open → divider), `hiddenAt` (last hide → classify next open).

```
on app load (ANY load, incl. quick peeks):
  fresh = feed-window queries (events[today..+W], all list summaries)   // meals not in feed (state line only)
  if (now - snapshotSavedAt > STALE_RESEED):     // long absence — see §4.4(d)
      snapshot = fresh; snapshotSavedAt = now; return   // reseed silently, no dump
  deltas = diff(fresh, snapshot)                  // added / edited / removed, per composite key
  changeLog = reconcile(changeLog, fresh)         // drop phantom adds; demote vanished edits → removed (§4.4)
  changeLog = mergeCoalesced(changeLog, deltas, detectedAt = now)   // §4.4 merge rule
  changeLog = prune(changeLog, olderThan 48h)
  snapshot = fresh; snapshotSavedAt = now         // never miss a change

on visibilitychange→hidden:  hiddenAt = now

on becoming visible / cold start:
  meaningful = coldStart || (now - hiddenAt > MEANINGFUL_GAP) || dayChanged(lastSeen)
  render feed from changeLog; draw divider at lastSeen (§5)
  if meaningful: lastSeen = now                    // advance divider AFTER render
```

Key property: detection runs on **every** load (nothing is lost), but the **divider advances only on a meaningful open** — a brief peek captures-and-keeps without marking anything seen.

`ActivityItem` is the integration seam — a future backend `GET /api/activity` (the AI phase) replaces the source with no UI change:

```
ActivityItem {
  key: string              // per-module composite key (§4.2) — stable across loads
  module: "calendar" | "lists"
  kind: "added" | "edited" | "removed"
  title: string            // from fresh, or from snapshot for removals
  detail?: string          // "+3 items", "moved to 5:00 PM", "Tue 9:00 AM"
  memberColor?: string     // assigned member's color, if any (assignment, not authorship)
  occurredAt: number       // detectedAt (no entity updatedAt exists today — §10.1)
  deepLink: { module, entityRef, date? }
}
```

### 4.2 What counts as a change, per module (with composite keys)

- **Calendar** — added / edited / removed. Source data is **expanded instances with `id: string | null`** over a query window (§4.5), so:
  - **Composite key** = `id` when non-null, else `` `${recurringEventId}_${formatLocalDate(date)}` `` (per the existing `getEventKey` fallback).
  - "Edited" = a meaningful field moved: `title`, `startTime`/`endTime`, `date`/`endDate`, `isAllDay`, `location`, `memberId`. Ignore `htmlLink`/`source`/sync noise.
  - **Recurring:** there is **no source-event accessor** on the FE. We diff at the occurrence level, then **coalesce within the calendar group by `recurringEventId`**: identical edits across instances of one series render as a single sub-row ("Soccer (weekly) · moved to 5:00 PM"). The original "series = one item" guarantee is **dropped**; practically, coalescing yields one row per distinct series-change.
- **Lists** — source is `useLists()` → `ListSummary[]` = `{id,name,kind,totalItems,completedItems}` (**no items, no timestamps**; per-item data needs `useList(id)` per list, only cached for opened lists). So the feed detects, **per list (key = list `id`)**: list **created** (new id), **renamed** (name changed), **removed** (id gone), and **item count deltas** surfaced as `+N items` (totalItems rose) / `N checked off` (completedItems rose) / `N removed` (totalItems fell). **No per-item identity diffing in v1.** This matches the §5 mockups and needs no per-item data.
- **Meals** — excluded from the feed (state line only; meals-as-activity cut from v1 — §7).
- **Chores** — excluded from the feed (state line only).
- **Recipes** — excluded entirely.

### 4.3 Timing & locale

- `MEANINGFUL_GAP` default ~4h; `STALE_RESEED` = the 48h window; both tuned in the plan.
- **Dates are device-local.** The FE has no family-local date helper (`family.timezone` is backend-only and never used for FE date math; all of `time-utils.ts` is device-local). `dayChanged`, "today," and `weekStart` all use device-local time. Acceptable for the single-home personas; family-tz correctness is a noted future enhancement, not v1.
- Window = 48h on `detectedAt`; cap to ~20 rows **post-coalesce**, with an "and N more" affix if exceeded.

### 4.4 Algorithm edge cases (fully specified)

- **(merge / coalescing rule)** Per `(module, key)` keep exactly one `changeLog` entry. On a new delta for an existing key: `kind` resolves by precedence **added > removed > edited** (added-then-edited stays "added"; anything-then-removed becomes "removed"); `occurredAt = now` (latest detection); `detail`/`title` recomputed from the latest `fresh`/`snapshot`. A pure no-op (no watched field changed) produces no delta.
- **(re-edit before a meaningful open)** Handled by the merge rule — the entry updates in place; the divider hasn't moved, so it stays "new."
- **(add-then-delete across a peek)** `reconcile()` runs each load: any `changeLog` entry with `kind: "added"` whose key is **absent from `fresh`** is **dropped** (no phantom row, no 404 deep-link). An `edited` entry whose key vanished is **demoted to `removed`** (title from snapshot). "Deletions are free" via the snapshot only covers entities still present in the snapshot; `reconcile()` covers entities that were only ever in the log.
- **(long absence / stale snapshot)** If `now - snapshotSavedAt > STALE_RESEED`, the next load **reseeds silently** (refresh snapshot, emit no deltas) so a returning user is **not** shown a 20-item dump of changes they can't act on; the feed reads "all caught up" and populates from the next real change. (Trade-off: changes made during a multi-day absence are not retro-listed — acceptable, and the correct call given no server history.)
- **(first run / no snapshot)** Seed `snapshot` silently; feed shows the calm empty state.
- **(null/duplicate keys)** Composite keys (§4.2) make expanded event instances safely keyable despite `id: null`.

### 4.5 Data dependencies & performance

- Home today fetches only a **3-day** events query (`use-dashboard-events.ts`). The feed needs more, so Home will **mount additional queries**: a **dedicated wider events query** for detection (e.g. `[startOfToday, +28d]` — tunable; this is what lets "Dentist next Tuesday" surface, which a 3-day window would miss) and `useLists()` for the feed, plus `useMealsBoard(thisWeek)` and `useChoresBoard()` for the state line.
- Honest cost: these are **real first-load fetches** on Home (deduped by TanStack Query and shared with the module tabs / offline cache, but not free). There is **no new backend endpoint** and no duplicate of an existing in-flight query; "reuse" means reusing the existing query definitions, not avoiding fetches.
- The diff/feed compute runs over in-memory data **after** the hero paints; it must not block first paint.
- The feed owns a single, dedicated visibility hook (records `hiddenAt` on hide; re-runs detection on return-to-visible). It must **not** duplicate the hero's `now`-recompute logic (`use-hero-state.ts`); a separate single-purpose listener for the hidden-timestamp is acceptable (relaxed 2026-06-21 — two passive single-purpose listeners are cleaner than coupling the hero hook to the feed). Order on a meaningful open: render feed from `changeLog` at the current `lastSeen`, then advance `lastSeen`.

## 5. Feed presentation & interaction

Rule: **group only when grouping reduces noise.** A module with ≥2 changes collapses into one summary line with an expander; a single change renders directly. Lists are inherently per-list.

- **Calendar** — coalesces all event changes into one expandable group; expand reveals each change with a marker (`+` added · `~` changed · `−` removed) and a time/affix; recurring series collapse to one sub-row (§4.2). Each sub-row deep-links to that event/date.
- **Lists** — one row per changed list ("Groceries · +3 items", "Camping · 2 checked off", "New list · Party"); tap opens the list. No item-level expansion in v1.

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
- **Expansion is ephemeral** — collapses on the next open; no persisted toggle.
- **Deep links:** a row opens the relevant module/entity (calendar event sheet at its date; the list detail; meals at the day), consistent with the dashboard's "rows open detail, don't switch tabs."

## 6. Visual & motion vocabulary

- **No new tokens.** Reuse cream/purple, Nunito, member colors, and the 4/8 spacing system.
- **Motion** reuses the shipped system: easing `cubic-bezier(0.32, 0.72, 0, 1)`, durations 150/250/400ms; expand/collapse and divider use these; respect `prefers-reduced-motion` (opacity-only).
- **Press feedback & haptics** come free via shipped seams — `usePressable` (FE #230) and optional-haptics (FE #236, behind its per-device toggle). No new interaction infra.
- Feed rows use quiet field styling (no per-row card chrome), consistent with the agenda; member color appears only as a small leading dot where an item has an assignment.

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
- [ ] The "new since you last looked" divider advances only on a meaningful open (cold start, visible after `MEANINGFUL_GAP`, or device-local day rollover), and is shown only when both "new" and "earlier" sections are non-empty.
- [ ] Calendar changes coalesce into one expandable group when ≥2; a single change renders directly; recurring series collapse to one sub-row via `recurringEventId`.
- [ ] Lists changes render one row per changed list using summary signals only (created/renamed/removed + count deltas) — no per-item data required.
- [ ] An entity added then removed before a meaningful open produces **no** lingering "added" row (reconcile against `fresh`); a removed entity present in the snapshot shows as "removed" with its prior title.
- [ ] After a long absence (> `STALE_RESEED`), the feed reseeds silently and shows "all caught up" rather than a bulk dump.
- [ ] Each feed row deep-links to the relevant module/entity; expansion is ephemeral.

### Quality
- [ ] No new design tokens; spacing/type/color/motion from the existing system; `prefers-reduced-motion` respected.
- [ ] Feed rows inherit shipped press-feedback + optional-haptics seams; no new interaction infra. Visibility handling is consolidated into single-purpose hooks with no duplicated `now`-recompute logic (a dedicated feed listener for the hidden-timestamp is acceptable).
- [ ] Layout holds 320px–768px without horizontal overflow; rows ≥44px touch target.
- [ ] Gated to mobile; the tablet/touchscreen surface never mounts the feed logic.

### Performance / data
- [ ] No new backend endpoint. The feed/state line derive from existing query definitions; Home mounting the wider events query + lists/meals/chores is acknowledged as real first-load fetching (deduped by TanStack Query).
- [ ] Diff/feed compute runs after the hero paints and does not block first paint.
- [ ] The new `idb-keyval` store and `lastSeen`/`hiddenAt` markers are per-device and **explicitly cleared on logout and on the 401 handler** (alongside `clearOfflineReadCache`), preventing cross-account leakage on a shared device.

## 10. Resolved constraints (verified against code) + plan-time tunings

Verified during the spec review (file:line evidence in the review record):

1. **No `updatedAt`/`createdAt`** on calendar (`CalendarEventResponse`) or meals (`MealSlot`); `ListItem`/`ListDetail` have them but `ListSummary` (what `useLists()` returns) does **not**. → `occurredAt = detectedAt`. The snapshot must store all watched fields so "edited" is detectable without timestamps.
2. **Calendar returns expanded instances with `id: string | null`** (recurrence expanded server-side), via a range-bounded query. → composite key + `recurringEventId` coalescing (§4.2); a dedicated wider detection window (§4.5).
3. **FE #222 persister is a single-key TanStack `PersistedClient` persister with an allowlist** — not a KV store; `clearOfflineReadCache()` clears only its store. → new `idb-keyval` store + explicit clearing wired into both `useLogout()` and the 401 handler.
4. **No family-local date helper; FE is device-local** (`time-utils.ts`; `family.timezone` is BE-only). → device-local dates in v1.
5. **Chores expose `today.summary.remaining` cleanly; meals require weekStart+dayIndex+mealType derivation** for tonight's dinner (§3.1).

Plan-time tunings: exact `MEANINGFUL_GAP`, `STALE_RESEED`, the detection-window length (`+W` days), and the post-coalesce row cap.

## 11. Story-file & roadmap updates required

- Create `docs/product/backlog/mobile-ux/home-organizer-summary.md` (status: planned), linking this spec + the plan once written, and cross-linking the shipped home-dashboard-redesign story as its base.
- Roadmap: promote "Home organizer summary" from the "Recommended additions to shape" note into a captured story under the Mobile UX polish backlog; note future siblings (AI day-summary story; backend activity log).

## 12. Future directions (not in this story)

- **AI day-summary / overload warnings** — consumes aggregated day state + `ActivityItem[]` to summarize the day or flag conflicts (e.g. a long recipe on an overloaded day). Wants durable server-side history → motivates the backend log below.
- **Backend activity log (Approach 2)** — append-only server record exposed via `GET /api/activity?since=…`; drop-in for the client diff via the `ActivityItem` seam; the right substrate once cross-device consistency, family-tz correctness, deletions-history, or AI history is required.
