# All-Day & Multi-Day Events — Design Document

**Date:** 2026-03-07
**Status:** Complete — Phase A (FE PR #110) and Phase B (BE #18/#19, FE #111) are merged
**Depends on:** Nothing (foundational calendar improvement)
**Depended on by:** [Recurring Events](./recurring-events-design.md), [Google Calendar Integration](./google-calendar-integration-design.md)

---

## 1. Problem Statement

### All-Day Events Are Broken

The `isAllDay` flag exists in the data model and can be set via the event form, but rendering is effectively non-functional:

- **Daily/Weekly views:** All-day events go through the same time-grid positioning as timed events. The grid displays 6AM–10PM. An all-day event with `startTime: "12:00 AM"` renders off-screen (above the visible area). All-day events are invisible in the two most-used views.
- **Monthly view:** All-day events show as regular list items. No visual distinction from timed events.
- **Schedule view:** Displays `startTime - endTime` even for all-day events. No "All day" label.
- **Event card component:** Always renders times, never shows an "All day" indicator.
- **Event detail modal:** The only place that correctly shows an "All day" badge instead of times.

### Multi-Day Events Don't Exist

The data model has a single `date` field (LocalDate). There is no way to represent an event spanning multiple days (e.g., a 4-day vacation). This matters because:

- Families have multi-day events (vacations, trips, holidays, visiting relatives)
- Google Calendar supports multi-day all-day events (`start.date` / `end.date` as a range)
- The upcoming Google Calendar integration needs to handle these events

### Why This Matters Now

The Google Calendar integration (read-only sync) is the next major feature. Google Calendar events include multi-day all-day events. Before we can sync and render those, the foundational all-day rendering must work correctly, and the data model must support date ranges.

---

## 2. Product Philosophy

The calendar module is a **daily at-a-glance planner**. Its purpose is to show the micro-schedule — what's happening at 9am, 2pm, 6pm. It is NOT a trip overview tool or a project timeline.

All-day and multi-day events serve as **contextual labels**, not primary content:

- "Birthday" on March 15 — a reminder, not a time block
- "Vacation" March 7–9 — context for those days, but the real value is the timed events within (flights, reservations, activities)
- "School closed" on a Monday — good to know, but the timed events tell you what the day actually looks like

**Design principle:** All-day events should be visible but unobtrusive. They provide context above the time grid, not compete with timed events for attention.

---

## 3. Solution Design

### Part 1: Fix All-Day Rendering (no data model changes)

Fix how `isAllDay` events render across all four calendar views using the standard calendar pattern: **a dedicated all-day section above the time grid**.

#### Daily View

```
┌─────────────────────────────────┐
│  All Day  │ Birthday (yellow)   │  ← all-day section (compact, above grid)
├───────────┼─────────────────────┤
│  6:00 AM  │                     │
│  7:00 AM  │                     │  ← existing time grid (unchanged)
│  8:00 AM  │ School drop-off     │
│  9:00 AM  │                     │
│  ...      │                     │
└─────────────────────────────────┘
```

- All-day events render as small colored badges/pills in a section above the hour grid
- Show member color (left border or background tint) + event title
- Section collapses if no all-day events for that day
- Tapping an all-day badge opens the event detail modal (existing behavior)

#### Weekly View

```
┌──────────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│          │ Mon  │ Tue  │ Wed  │ Thu  │ Fri  │ Sat  │ Sun  │
├──────────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
│ All Day  │      │ Bday │      │      │      │      │      │  ← all-day row
├──────────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
│  6:00 AM │      │      │      │      │      │      │      │
│  7:00 AM │      │      │      │      │      │      │      │  ← time grid
│  ...     │      │      │      │      │      │      │      │
└──────────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
```

- All-day events render as pills in a row above the time grid, in the correct day column
- Same visual style as daily view (colored pill with title)
- Row auto-hides if no all-day events exist in the visible week
- No spanning bars — each day cell is independent (even for multi-day events; see Part 2)

#### Monthly View

```
┌──────────┐
│    15     │
│ ● Bday   │  ← all-day event shown with an "All day" indicator or subtle styling
│ 9a Mtg   │  ← timed event with time prefix
│ +2 more   │
└──────────┘
```

- All-day events render with a subtle visual distinction from timed events
- Options: different background tint, no time prefix (timed events show "9a" or "2p" prefix), or a small dot/icon
- All-day events sort before timed events within a day cell

#### Schedule (List) View

```
Wednesday, March 15
├─ All day · Birthday                    (yellow)
├─ 9:00 AM – 10:00 AM · Team Meeting    (teal)
├─ 2:00 PM – 3:00 PM · Doctor           (coral)
```

- All-day events show "All day" label instead of time range
- All-day events sort before timed events for the day
- Same card styling, just different time label

#### Event Card Component

- When `isAllDay === true`: render "All day" label instead of `startTime - endTime`
- Applies to all card variants (large, default, compact)

#### BE Validation Change

When `isAllDay` is true, `startTime` and `endTime` become semantically meaningless. Two options:

**Option A (minimal change):** Keep requiring startTime/endTime in the API, but the FE sends sensible defaults (e.g., "12:00 AM" / "11:59 PM") and all views ignore them when `isAllDay = true`. No BE change needed.

**Option B (cleaner):** Make startTime/endTime optional when `isAllDay = true`. Requires validation logic change: "startTime and endTime are required unless isAllDay is true."

**Recommendation:** Option A for now. It's less disruptive, avoids nullable time columns, and the FE already needs to branch rendering on `isAllDay`. The times are just unused data, not harmful data. We can clean this up later if it becomes a problem.

---

### Part 2: Add Multi-Day Support (data model change)

Add an `endDate` column to `calendar_event`. Multi-day events are rendered as all-day badges on each day in the range — no spanning bars.

#### Data Model

```sql
ALTER TABLE calendar_event ADD COLUMN end_date DATE;
```

- `endDate` is nullable. `NULL` means single-day event (equivalent to `endDate = date`).
- `endDate` is only meaningful when `isAllDay = true`.
- Constraint: `endDate >= date` when not null.
- Constraint: `endDate` must be null when `isAllDay = false`.

#### Relationship Between Fields

| `isAllDay` | `endDate` | Meaning |
|---|---|---|
| `false` | `null` | Normal timed event (9am–10am on March 7) |
| `true` | `null` | Single-day all-day event (Birthday on March 7) |
| `true` | `> date` | Multi-day all-day event (Vacation March 7–9) |
| `false` | non-null | **Invalid** — rejected by validation |

#### Rendering Multi-Day Events

Multi-day events render as **repeated all-day badges on each day in the range**. No spanning bars.

Example: "Vacation" with `date = March 7`, `endDate = March 9`

**Daily view (March 8):**
```
┌─────────────────────────────────┐
│  All Day  │ Vacation (coral)    │  ← shows on each day in the range
├───────────┼─────────────────────┤
│  6:00 AM  │                     │
│  ...      │                     │
```

**Weekly view:**
```
┌──────────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ All Day  │      │      │ Vac  │ Vac  │ Vac  │      │      │
├──────────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
│  6:00 AM │      │      │      │      │      │      │      │
```

Each day shows the event independently. This is intentional — the app is a daily planner, not a Gantt chart. The repeated badge provides context without complex spanning UI.

**Monthly view:** Same as weekly — event appears in each day cell in the range.

**Schedule view:** Shows once with the date range:
```
Friday, March 7 – Sunday, March 9
├─ All day · Vacation    (coral)
```

Or shows under each day:
```
Friday, March 7
├─ All day · Vacation (Day 1 of 3)   (coral)

Saturday, March 8
├─ All day · Vacation (Day 2 of 3)   (coral)
```

Either approach works. The "Day X of Y" variant gives more context but adds rendering logic.

#### BE Query Change

The existing query pattern `WHERE date BETWEEN :startDate AND :endDate` must become a range-overlap check to catch multi-day events that started before the query window but extend into it:

```sql
WHERE date <= :rangeEnd
  AND COALESCE(end_date, date) >= :rangeStart
  AND family_id = :familyId
```

This returns:
- Single-day events where `date` falls within the range (existing behavior)
- Multi-day events where any day in their range overlaps the query range

#### API Contract Change

**Request** — add optional `endDate`:
```json
{
  "title": "Vacation",
  "date": "2026-03-07",
  "endDate": "2026-03-09",
  "isAllDay": true,
  "memberId": "uuid",
  "startTime": "12:00 AM",
  "endTime": "11:59 PM"
}
```

**Response** — add `endDate` (nullable):
```json
{
  "id": "uuid",
  "title": "Vacation",
  "date": "2026-03-07",
  "endDate": "2026-03-09",
  "isAllDay": true,
  "memberId": "uuid",
  "startTime": "12:00 AM",
  "endTime": "11:59 PM",
  "location": null
}
```

Single-day events return `endDate: null` (no breaking change for existing consumers).

#### FE Form Change

The all-day toggle already landed in Phase A (PR #110): toggling ON hides time pickers and defaults to 00:00–23:59. Phase B extends this:

When `isAllDay` is toggled ON:
- Time pickers hidden (already done)
- Show optional "End date" picker (new in Phase B)
- If end date is not set, it's a single-day all-day event
- If end date is set, it must be >= start date

When `isAllDay` is toggled OFF:
- Show time pickers (already done)
- Hide/clear end date field

---

## 4. Implementation Order

### Phase A: Fix All-Day Rendering (FE only, no data model changes) — COMPLETE

Completed in [FE PR #110](https://github.com/joe-bor/FamilyHub/pull/110).

**What landed:**
- All-day section above time grid in daily and weekly views (colored pills/bars)
- Event cards show "All day" instead of times when `isAllDay = true`
- Schedule view shows "All day" label, sorts all-day first
- Monthly view shows `●` prefix for all-day events, sorts them first
- All-day section auto-hides when no all-day events exist
- All-day toggle added to event form (hides time pickers when ON, defaults to 00:00–23:59)
- `compareEventsAllDayFirst` utility for shared sorting
- 14 new tests (446 total)

**No BE changes. No data model changes. No new fields.**

### Phase B: Add Multi-Day Support (BE + FE)

Add `endDate` to the data model and update both repos.

**BE scope:**
- Flyway migration: add `end_date` column
- Entity: add `endDate` field
- DTOs: add `endDate` to request and response
- Validation: `endDate >= date`, `endDate` only when `isAllDay = true`
- Repository: update date range queries to use overlap logic
- Mapper: handle endDate mapping
- Tests: update existing + add multi-day cases

**FE scope:**
- TypeScript types: add `endDate`
- Zod schema: add endDate validation with cross-field rules
- Event form: conditional end date picker when isAllDay is on
- All four views: render multi-day events as repeated all-day badges
- Event detail modal: show date range
- Tests: update existing + add multi-day cases

### Phase C: Google Calendar Integration

With all-day and multi-day rendering working, Google Calendar sync can map:
- Google `start.date` / `end.date` (all-day/multi-day) → our `date` + `endDate` + `isAllDay = true`
- Google `start.dateTime` / `end.dateTime` (timed) → our `date` + `startTime` + `endTime` + `isAllDay = false`

See [google-calendar-integration-design.md](./google-calendar-integration-design.md) for the full sync design.

---

## 5. What We Are NOT Building

- **Spanning bars** across day cells in month/week view. Multi-day events repeat as badges per day.
- **Timed multi-day events** (e.g., "starts 8am Friday, ends 6pm Sunday"). If `isAllDay = false`, `endDate` must be null. Multi-day is all-day only.
- **Drag-to-resize** multi-day events in the UI.
- **Recurring events** (separate feature, separate design doc when needed).
- **All-day events consuming the full time grid height.** They are badges above the grid, not blocks within it.

---

## 6. Acceptance Criteria

### Phase A (Fix All-Day Rendering) — COMPLETE (FE PR #110)

- [x] Daily view: all-day events appear as colored badges above the time grid
- [x] Weekly view: all-day events appear as colored badges in a row above the time grid, in the correct day column
- [x] Monthly view: all-day events are visually distinct from timed events (● prefix) and sort first
- [x] Schedule view: all-day events show "All day" label instead of time range
- [x] Event card: shows "All day" instead of times when `isAllDay = true`
- [x] All-day section auto-hides when no all-day events exist
- [x] Existing filter toggle ("All Day Events" checkbox) still works
- [x] Clicking an all-day badge opens the event detail modal
- [x] No regressions on timed event rendering

### Phase B (Multi-Day Support) — COMPLETE (BE #18/#19, FE #111)

- [x] BE: `end_date` column exists in DB
- [x] BE: API accepts and returns `endDate` field
- [x] BE: validation rejects `endDate` when `isAllDay = false`
- [x] BE: validation rejects `endDate < date`
- [x] BE: date range queries return multi-day events that overlap the range
- [x] FE: event form shows end date picker when `isAllDay` is toggled on
- [x] FE: multi-day events appear as all-day badges on each day in the range
- [x] FE: event detail modal shows date range for multi-day events
- [x] FE: schedule view shows date range or "Day X of Y"
- [x] No regressions on single-day all-day or timed events

---

## 7. Reference

- Current all-day rendering code:
  - `frontend/src/components/calendar/views/daily-calendar.tsx`
  - `frontend/src/components/calendar/views/weekly-calendar.tsx`
  - `frontend/src/components/calendar/views/monthly-calendar.tsx`
  - `frontend/src/components/calendar/views/schedule-calendar.tsx`
  - `frontend/src/components/calendar/components/calendar-event.tsx`
  - `frontend/src/components/calendar/components/event-detail-modal.tsx`
  - `frontend/src/components/calendar/components/calendar-filter.tsx`
- BE entity: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/CalendarEvent.java`
- BE DTOs: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/CalendarEventRequest.java`, `CalendarEventResponse.java`
- BE mapper: `backend/family-hub-api/src/main/java/com/familyhub/demo/mapper/CalendarEventMapper.java`
- API contract: `docs/calendar-events-api-reference.md`
- Google Calendar design: `docs/google-calendar-integration-design.md`
