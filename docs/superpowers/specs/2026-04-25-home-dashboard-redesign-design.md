# Home Dashboard Redesign — Design Spec

**Date:** 2026-04-25
**Status:** Approved (design)
**Story:** `docs/product/backlog/mobile-ux/home-dashboard-redesign.md`
**Concept:** "The Now" — moment-aware mobile home surface
**Scope:** Mobile breakpoints (≤768px), adult users, calendar-only data sources

## 1. Context

The mobile home screen today is a 2-column grid of colored module buttons. It works but reads as a prototype. This spec replaces it with a calm, layered, mobile-first home surface that answers four questions Family Hub users actually ask: *what matters now?*, *what's next today?*, *what's coming up?*, *what is happening across my family at a glance?*

The dashboard is the **home surface for adult users on mobile**. It is not a module launcher, not a calendar replacement, and not a child-facing view. Those are deliberately separate concerns and, where they require their own design, separate stories.

## 2. Anchor decisions (locked during brainstorming)

| # | Decision | Rationale |
|---|---|---|
| 1 | **Adult-first.** Joe + Partner are the primary audience. | The current 2-col color grid is already well suited to the 4-year-old; replacing it with an adult-first dashboard means the child gets a dedicated surface in a new story rather than a regression here. |
| 2 | **Pure home surface — no module launcher tiles.** | The persistent bottom nav handles cross-module navigation. A dashboard that doubles as a launcher cannot earn the "premium calm" target. |
| 3 | **Today as hero + small horizon peek (next 1–3 events in the next 48h).** | Pure today is the calmest scope; the morning question "anything early tomorrow?" is high-frequency enough to justify a deliberately subordinate horizon region. |
| 4 | **Family-merged by default, with member chips for one-tap focus.** Single-focus only. | Multi-member filtering is a calendar-tab concern. The dashboard's job is glanceable family awareness with a fast escape hatch into one person. |
| 5 | **Mobile-only redesign.** Touchscreen and tablet keep the current home until a dedicated follow-up story. | One design cannot serve mobile glance + at-home ambient + child accessibility without compromising all three. |

## 3. Concept summary — "The Now"

A moment-aware home: the **hero** announces the most relevant moment (right now / up next / all clear), and everything below is supporting context. The dashboard adapts its hero state as time passes.

This concept was selected over two alternatives:
- **"The List"** — agenda as typeset list. Calmest but indistinguishable from a generic agenda app.
- **"The Spine"** — vertical time spine with events docked at hours. Most distinctive but reads as "calendar lite" and looks empty most of the time.

"The Now" wins because it cannot be confused with the calendar tab in a screenshot, scales gracefully to sparse data (most days), and maps directly to family-coordination questions.

## 4. Information architecture

### 4.1 Regions, top to bottom

1. **Header line** — greeting (time-of-day-aware) + date. Single line.
2. **Member-chip row** — circular avatars, single-line, horizontally scrollable if needed. Single-focus filter.
3. **Hero card** — the announcement. Five possible states (§4.2).
4. **Rest of today** — compressed list of today's remaining events excluding the hero's subject. All-day events pin at the top of this list.
5. **Coming up** — up to 3 events in `(endOfToday, endOfToday+2)`, light single-line entries with date label ("Tomorrow", "Sun"). Region omitted entirely if empty.
6. **Floating "+"** — bottom-right, 16px above the bottom-nav inset. Opens existing event-create sheet.

### 4.2 Hero card state machine

```
let now        = currentTime
let today      = filteredTodayEventsSortedByStart   // post member-chip filter
let inProgress = today.find(e => e.start <= now < e.end && !e.allDay)
let next       = today.find(e => e.start > now && !e.allDay)
let allDay     = today.filter(e => e.allDay)

if (inProgress)              → RIGHT_NOW(inProgress)
else if (next)               → UP_NEXT(next)
else if (allDay.length)      → ALL_DAY_ONLY(allDay[0])      // only when no timed events at all
else if (had earlier today)  → REST_OF_DAY_CLEAR
else                         → ALL_CLEAR_TODAY
```

Recompute triggers: data change (TanStack Query), 30-second interval, `visibilitychange → visible`.

### 4.3 Data scope

- **Source:** existing events query — same one the calendar tab consumes. No new endpoint.
- **Today range:** `[startOfDay(localTZ), endOfDay(localTZ))`. Recurring events use the calendar tab's existing expansion.
- **Coming-up range:** events with `start ∈ (endOfToday, endOfToday+2)`, sorted by start, capped at 3.
- **Multi-day events:** appear in today's list when today is in `[start, end]`; rendered with a span affix (`→ ends Sat` / `from Mon →`). Same in Coming-up if start falls in range.
- **Member-chip focus:** filters Hero, Today list, and Coming-up consistently.
- **No new data source orchestration:** the dashboard consumes the existing events query only. It does not own Google connection management, sync triggering, or global sync-state aggregation.

### 4.4 Empty and loading states

- **No events today, no events in next 48h:** Hero = "All clear today" with a soft glyph. Rest-of-today empty. Coming-up omitted. Empty space is part of the calm — do not collapse the layout.
- **Initial load:** skeleton hero + chip row + 2 rows. No full-screen spinner.

### 4.5 Google integration semantics

- **Event-first, not Google-first.** Native events and Google-synced events render through the same dashboard states. A valid schedule must never be replaced by a "connect Google" prompt.
- **No dashboard-specific onboarding in MVP.** If the visible scope has no events, the dashboard shows the normal calm empty state (`ALL_CLEAR_TODAY` or `REST_OF_DAY_CLEAR`) regardless of Google connection status.
- **Google connection is member-scoped today.** The current FE app models Google connection/sync per member, not globally. This story does not invent a family-level aggregate state.
- **Connection and manual sync stay in settings.** Connect, disconnect, calendar selection, and "Sync Now" remain in the existing member settings surfaces for this story.
- **No dashboard-level sync pill or pull-to-refresh in MVP.** Those require a separately defined member/family sync model and are intentionally deferred.

## 5. Visual vocabulary

The whole surface sits inside existing tokens — cream/purple, Nunito, member colors. No new color, spacing, type, or shadow tokens are introduced. "Premium calm" comes from restraint and rhythm, not new ingredients.

### 5.1 Surface & layering

- **Background:** existing cream surface, full bleed, untouched.
- **Hero card:** elevated via **a hairline border at low opacity + a soft, vertical-only drop shadow.** No `backdrop-filter`, no frosted glass, no translucent material.
- **Today list rows:** no card chrome. Rows sit on the background, separated by airy spacing rather than dividers.
- **Coming-up rows:** same shape as today rows, smaller and dimmer — subordination via type and opacity, not via a different container.
- **One structural divider on the entire screen:** a single low-opacity hairline between Rest-of-today and Coming-up.
- **Three layers, no more:** one elevated thing (hero), one quiet field (agenda), one subordinate region (Coming-up).

### 5.2 Hero card specifics

- 24–28px interior padding.
- **Member color appears as a 4px leading accent bar only.** Card body stays neutral. This is the design's single ambient-color concession.
- Time/relative line ("in 38 min", "ends in 22 min", "at 3:00 PM"): above title, ~14–15px, ~70% opacity.
- Title: 24–28px Nunito semibold (the largest type on the screen).
- Location/notes: below title, ~14–15px, ~60% opacity.
- **"Right now" indicator:** small filled member-color dot next to the time, slow 2-second breathing pulse. Exactly one pulse, never a strobe.

### 5.3 Type & rhythm

| Element | Size | Weight | Opacity |
|---|---|---|---|
| Greeting | 14 | Regular | 60% |
| Date | 14 | Regular | 80% |
| Hero meta | 14–15 | Regular | 60–70% |
| Hero title | 24–28 | Semibold | 100% |
| Today row title | 16 | Semibold | 100% |
| Today row time | 14 | Regular | 70% |
| Coming-up row | 14 | Regular | 60% |

Spacing on the existing 4/8 system. **Hero gets ~2× the breathing room of any other region** — this single rule sells the layering more than anything else.

### 5.4 Color discipline

Member colors appear in exactly three places:
- Chip avatars (full)
- Hero accent bar (full, 4px)
- Today row leading dot (full)

They never wash a card and never appear as backgrounds. Saturated member colors become noise the moment they take more than ~5% of pixel area.

### 5.5 Member chips

- 36px circular visual, 44px hit area.
- Member-color fill, white initials.
- Selected: 2px outer ring (member color, 80% opacity) + 1.04 scale.
- When one is focused, the others fade to 60% opacity (still tappable).

## 6. Motion vocabulary

One easing curve: **`cubic-bezier(0.32, 0.72, 0, 1)`**. Three durations: **150 / 250 / 400ms**.

| Transition | Duration | Behavior |
|---|---|---|
| Hero state change (e.g., UP_NEXT → RIGHT_NOW) | 400ms | Cross-fade + 4px upward translate on entering content |
| Chip focus change | 250ms | Selected scales 1.04 + ring fade-in; others dim to 60% |
| Today list re-filter | 200ms | Filtered-out rows fade out; remaining rows reflow |
| Row press | 100ms | Scale to 0.98, no ripple |
| Pull-to-refresh | native | Sync thread pulses while syncing |
| Empty-state glyph | 4-second cycle | Slow breathing fade — the only ambient-life signal on truly empty days |

### 6.1 What we deliberately don't do

- No `backdrop-filter` blur anywhere. (The line between "premium calm" and "macOS cosplay.")
- No parallax, scroll-driven hero shrinking, animated gradients.
- No segmented-control imitations or system-font mimicry. Stays on Nunito.
- No haptics (PWA limits).
- No page-transition slides between dashboard and other tabs.

### 6.2 Reduced motion

Under `prefers-reduced-motion`:
- All transitions degrade to opacity-only (no translates).
- Breathing pulse and breathing fade are disabled.
- Chip selection scale is removed; ring still appears.

## 7. Interaction model

| Element | Tap | Notes |
|---|---|---|
| Header | — | Non-interactive |
| Member chip (unselected) | Set focus | Single-focus only |
| Member chip (currently focused) | Clear focus | Returns to "everyone" |
| Hero card (event states) | Open event-detail sheet | Same sheet as calendar tab |
| Hero card (empty states) | No-op | "All clear" is a destination, not a doorway |
| Today list row | Open event-detail sheet | — |
| All-day pinned strip row | Open event-detail sheet | — |
| Coming-up row | Open event-detail sheet | Does *not* navigate to calendar tab — consistent with today rows |
| Floating "+" | Open event-create sheet | Pre-fill: today, current focused member if any, next 30-min slot |

## 8. App-shell integration

- In the current FE app, the dashboard is the **mobile home surface** rendered when `activeModule === null`. This story does not introduce route-based navigation.
- On desktop / non-mobile breakpoints, the current app-shell behavior remains unchanged: `activeModule === null` still redirects to Calendar.
- Persistent bottom nav is a separate story and a real prerequisite. This dashboard assumes nav-based module switching already exists in the mobile shell; it does not render the nav itself.
- Event detail continues to open through the existing in-app modal / sheet patterns. No deep-link or router changes are in scope here.

## 9. Accessibility

- Member chips: `aria-label="Focus on <Name>'s events"`, `aria-pressed` for selected state.
- Hero card: `<section>` with `aria-label` reflecting state ("Right now: Soccer practice, ends in 22 minutes").
- Reduced motion respected as specified in §6.2.
- All touch targets ≥44px hit area; chips 36px visual + 8px halo. All rows ≥44px tall.
- Member identity is never carried by color alone — chip avatars carry initials; today rows expose the member name via the dot's accessible label.

## 10. How this differs from the calendar tab

- Calendar tab is a **time grid** (hour lines, fixed slots, visible empty time). Dashboard has none of these — events float free, empty hours simply absent.
- Calendar tab is dense by purpose; dashboard is sparse by purpose.
- Calendar tab supports horizontal date navigation; dashboard does not navigate dates at all. There is no swipe-to-yesterday and no week strip — *today is the only frame*.
- **Test:** dashboard and calendar tab side-by-side in screenshots must not be confusable.

## 11. Out of scope

These belong to other stories or are explicitly deferred:

- **Persistent bottom nav** — separate story, *hard prerequisite*.
- **Touchscreen / tablet home** — explicit follow-up story (to be written). Existing 2-col color grid stays on those surfaces.
- **Child-mode home** — separate follow-up story (to be written) for the 4-year-old's home surface.
- **Expandable bottom sheet** — separate story; the "+" opens the existing full-screen create sheet today, will inherit the new sheet for free when that ships.
- **Drag-to-create on dashboard** — that pattern lives in the calendar tab.
- **Tasks / chores / meals / weather** — none of these data sources exist; dashboard does not stub them.
- **Notifications / status badges / overdue UI** — dashboard is moment-aware, not status-aware.
- **Dashboard-owned Google connect / sync UI** — stays in settings/member profile for MVP.
- **Full onboarding redesign** — broader onboarding lives in `sidebar-settings-onboarding-mobile`.

## 12. Dependencies

| Type | Item | Effect |
|---|---|---|
| **Hard prerequisite** | `mobile-ux/persistent-bottom-nav.md` | Must ship before this story is merged to production. The dashboard removes launcher tiles and assumes nav-based module switching already exists. |
| **Soft dependency** | `mobile-ux/visual-identity-refinement.md` | If running in parallel, dashboard inherits any spacing/type tightening. No coordination needed. |
| **Soft dependency** | Google Calendar incremental sync (in-flight epic) | Improves data freshness over time, but the dashboard can ship against the existing events query without adding sync UI. |

## 13. Acceptance criteria

### Functional

- [ ] Replaces the 2-column module grid as the mobile home surface on breakpoints ≤768px.
- [ ] Integrates with the current FE app shell as the mobile `activeModule === null` surface; desktop/non-mobile behavior stays unchanged.
- [ ] Hero renders the correct state across all five cases (RIGHT_NOW, UP_NEXT, ALL_DAY_ONLY, REST_OF_DAY_CLEAR, ALL_CLEAR_TODAY).
- [ ] Hero transitions live as time crosses event boundaries, verified by 30-second poll + `visibilitychange` recompute.
- [ ] Member-chip focus filters Hero, Today list, and Coming-up consistently. Single-focus only.
- [ ] All-day events pin at the top of Today list. ALL_DAY_ONLY hero state is reached only when there are no timed events today.
- [ ] Coming-up renders 0–3 events in `(endOfToday, endOfToday+2)`; region omitted when empty.
- [ ] "+" opens the event-create sheet pre-filled with today's date and the currently focused member, if any.
- [ ] Native and Google-synced events are rendered through the same dashboard states; a missing Google connection never replaces a valid schedule or calm empty state.
- [ ] The dashboard does not add dashboard-owned Google connect/sync UI in MVP; connection management remains in settings/member profile.

### Quality

- [ ] No new design tokens introduced; spacing, type, and color all sourced from the existing system.
- [ ] All motion uses the three-duration / single-easing system specified in §6.
- [ ] `prefers-reduced-motion` respected as specified in §6.2.
- [ ] All rows ≥44px touch target; chips ≥44px hit area.
- [ ] Layout holds at 320px (iPhone SE) through 768px without horizontal overflow.
- [ ] **Visual diff:** dashboard and calendar tab are not confusable in screenshots.

### Performance

- [ ] Dashboard reuses the existing events query; no new BE endpoint and no duplicate fetch.
- [ ] Hero state recompute does not trigger today-list re-render when the subject hasn't changed.

## 14. Story-file updates required

Update `docs/product/backlog/mobile-ux/home-dashboard-redesign.md` to:

- Tighten scope: adult-first, mobile-only, no module launchers.
- Replace acceptance criteria with §13.
- Link to this spec.
- Note the new sibling stories that should be created later: child-mode home, touchscreen home.
- Bump `updated` to 2026-04-25.

No roadmap changes required — the story remains listed under the Mobile UX polish epic.

## 15. Open questions

None. All anchor decisions were resolved during brainstorming. Implementation-level decisions (specific token references, exact px values within the documented ranges, file/component layout in the FE repo) are deferred to the implementation plan.
