> Archived 2026-04-24 — implemented by joe-bor/FamilyHub#110

# Task: Fix All-Day Event Rendering

## Context

The `isAllDay` flag exists in the data model and can be toggled in the event form, but all-day events are effectively broken across all calendar views. This is a FE-only fix — no data model or BE changes needed.

**You are expected to form your own plan after exploring the codebase.** Read the relevant view components, event card, detail modal, and filter logic before making changes. Push back on any part of this prompt if you disagree after seeing the code.

**Read the design doc first:** `docs/all-day-and-multi-day-events-design.md` — it covers the full product context, the problem, the design philosophy, and acceptance criteria. This prompt focuses on Phase A from that doc.

---

## The Problem

All-day events don't render properly in any calendar view:

- **Daily/Weekly views:** All-day events go through time-grid positioning. An all-day event with `startTime: "12:00 AM"` renders off-screen because the visible grid is 6AM–10PM. They are invisible.
- **Monthly view:** All-day events look identical to timed events. No visual distinction.
- **Schedule view:** Shows `startTime - endTime` instead of an "All day" label.
- **Event card component:** Always renders times, even when `isAllDay = true`.
- **Event detail modal:** The only place that correctly shows an "All day" badge. This is fine as-is.

## What We Want

Follow the standard calendar pattern (Google Calendar, Apple Calendar, Outlook): **a dedicated all-day section above the time grid** in daily and weekly views, and appropriate labeling in monthly and schedule views.

### Daily View

- Add an all-day section above the hour grid
- All-day events render as small colored badges/pills (member color + title)
- Section auto-hides when no all-day events exist for that day
- Tapping a badge opens the event detail modal (existing behavior)
- Timed events in the grid below are unchanged

### Weekly View

- Add an all-day row above the time grid, spanning all 7 day columns
- All-day events appear as badges in the correct day column
- Row auto-hides when no all-day events exist in the visible week
- Same interaction behavior as daily view

### Monthly View

- All-day events should be visually distinct from timed events
- Possibilities: no time prefix (timed events could show "9a" or "2p"), different background, or a subtle indicator — explore what looks right
- All-day events sort before timed events within a day cell

### Schedule (List) View

- All-day events show "All day" label instead of the `startTime - endTime` text
- All-day events sort before timed events for each day

### Event Card Component

- When `isAllDay === true`: render "All day" instead of `startTime - endTime`
- Applies to all card size variants

### What NOT to Change

- Event form — no changes needed (isAllDay toggle already works)
- Event detail modal — already shows "All day" badge correctly
- BE / API — no backend changes
- Data model — no new fields
- The existing "All Day Events" filter toggle should continue to work (hides/shows all-day events)

---

## Starting Points

These are suggestions, not gospel. Verify them yourself:

- `src/components/calendar/views/daily-calendar.tsx` — daily view with time-grid positioning
- `src/components/calendar/views/weekly-calendar.tsx` — weekly view
- `src/components/calendar/views/monthly-calendar.tsx` — monthly view
- `src/components/calendar/views/schedule-calendar.tsx` — schedule/list view
- `src/components/calendar/components/calendar-event.tsx` — event card rendering
- `src/components/calendar/components/event-detail-modal.tsx` — detail modal (reference for existing "All day" badge)
- `src/components/calendar/components/calendar-filter.tsx` — filter toggle

---

## Workflow

1. Create a branch: `fix/all-day-event-rendering`
2. Plan your approach — read the components, understand the current rendering pipeline
3. Implement in atomic commits (e.g., daily view, weekly view, monthly view, schedule view, event card — each could be its own commit)
4. Test behavior, not implementation:
   - Create an all-day event → verify it appears correctly in each view
   - Create a timed event → verify it still renders correctly (no regressions)
   - Toggle the "All Day Events" filter → verify all-day events hide/show
   - Verify the all-day section hides when no all-day events exist
   - Test on mobile viewport — all-day section should work on small screens
5. Update any relevant docs if the rendering behavior is documented anywhere
6. Open a PR with a clear description of what changed and screenshots/recordings of the before/after

---

## Acceptance Criteria

- [ ] Daily view: all-day events appear as colored badges above the time grid
- [ ] Weekly view: all-day events appear in a row above the time grid, in the correct day column
- [ ] Monthly view: all-day events are visually distinct from timed events and sort first
- [ ] Schedule view: all-day events show "All day" label instead of time range
- [ ] Event card: shows "All day" instead of times when `isAllDay = true`
- [ ] All-day section auto-hides when no all-day events exist
- [ ] Existing "All Day Events" filter toggle still works
- [ ] Tapping an all-day badge opens the event detail modal
- [ ] No regressions on timed event rendering
- [ ] Works on mobile viewports

---

## PR Review Checklist

- [ ] All four views handle all-day events correctly
- [ ] No timed event rendering regressions
- [ ] All-day section doesn't take excessive vertical space
- [ ] Mobile responsive
- [ ] Tests cover the new behavior (not just implementation details)
- [ ] No unnecessary refactors outside the scope of this task
