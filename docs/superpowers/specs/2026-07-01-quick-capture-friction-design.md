# Quick-Capture Friction Fixes

**Date:** 2026-07-01
**Status:** Shipped in `family-hub` `0.3.21` (FE PR #276)
**Owner:** Family Hub product / planning
**Origin:** Mobile UX clunkiness review (2026-07-01), theme selected by Joe: quick-capture friction

## Summary

The three highest-frequency capture flows on the phone each carry a repeat-tax that makes daily use annoying. This spec fixes the three that have no overlap with in-flight work:

1. **Lists rapid multi-add** — adding N grocery items currently costs N full open/type/save/close cycles.
2. **Event time picker** — a scroll wheel with 1-minute granularity plus an OK step, and end time does not follow start time (duration silently shrinks).
3. **Missing Location field** — the event form has no location input even though the API contract (`CreateEventRequest.location`), Zod schema, and detail view all support it. No user can ever enter a location from the UI.

Frontend-only. No backend changes, no new endpoints, no schema changes.

## Explicitly out of scope

- **Meals plan-sheet redesign** (form-first layout, Image URL field, recipe search) — adjacent to the shipped [Focused meal planning sessions](../../product/backlog/module-foundations/focused-meal-planning-sessions.md) story (FE #262, BE `v1.8.0`, FE `0.3.21`). Still a separate follow-up, not part of this quick-capture release.
- **Tap-empty-slot / drag-to-create event creation** — covered by the planned [drag-to-create](../../product/backlog/mobile-ux/drag-to-create.md) story.
- Inline always-visible list composer (AnyList-style) — rejected in favor of the smaller keep-open-sheet change.
- Native `input[type=time]` and time-slot-list pickers — rejected; keep the app's wheel component.
- Any change to backend validation (times remain 12-hour strings on the wire).

## 1. Lists — rapid multi-add (keep-open sheet)

**Component:** `src/components/lists/list-item-sheet.tsx` (+ its tests), possibly `list-detail-view.tsx` for session wiring.

Behavior in **add mode** only:

- On successful save, the sheet **stays open**: item text clears, the text input **refocuses** (keyboard stays up), and the **category selection persists** for the next item.
- Enter/Go on the software keyboard submits, same as the Save button.
- After the first successful add in a session, the header dismiss label changes from `Cancel` to **`Done`** — items already saved cannot be "cancelled". Swipe-down dismiss keeps working throughout and is equivalent to Done.
- Each added item appears in the list behind the sheet via the existing optimistic mutation; the new row animating into the list plus the cleared input **is** the confirmation. No toast, no extra flash inside the sheet.
- Save with empty text: existing validation behavior unchanged.
- **Edit mode is unchanged**: saving an edit still closes the sheet.

Failure handling: if the create mutation fails, the sheet stays open with the typed text intact and the existing error surface (whatever `list-item-sheet` does today) shows; nothing is cleared.

## 2. Event form — time picker granularity + duration preservation

**Components:** `src/components/ui/time-picker.tsx`, `src/components/calendar/components/event-form.tsx` (+ tests).

### 5-minute wheel steps

- `MINUTES` becomes `0, 5, 10, … 55` (12 positions instead of 60).
- **Off-grid values are never silently rounded.** If the picker opens on a value not on the 5-minute grid (e.g. a 9:47 event synced from Google), that exact value is rendered as an extra wheel position, selected, until the user scrolls away from it. Saving without touching the wheel keeps 9:47.

### Duration preservation

- In both create and edit forms, changing **start time shifts end time by the same delta**, preserving the current duration (create default: 1 hour).
- Clamp: the form models a single-day time range, so if the shifted end would pass midnight, clamp end to 11:59 PM. The existing end-after-start validation still applies after clamping.
- Editing **end time only** changes duration and does not move start. Existing validation (end after start) unchanged.

### Quick nudges

- Add small **−15 / +15 minute** nudge buttons beside each time field (44px min targets) so common adjustments never open the wheel.
- Nudging start time also shifts end time (duration preservation applies to any start change, wheel or nudge).
- Fixed-time preset chips were considered and rejected: they assume a specific family schedule; relative nudges are universal.

## 3. Event form — Location field

**Components:** `src/components/calendar/components/event-form.tsx`, `event-form-modal.tsx` (+ tests).

- The `Add description` progressive-disclosure expander is renamed **`Add details`** and reveals two fields: **Location** (single-line text input, max 255 chars — matching the existing Zod schema and BE `@Size(max=255)`, placeholder "Where?") above **Description** (unchanged).
- `location` wires into the existing create/update payloads — `CreateEventRequest.location` / `UpdateEventRequest.location` already exist in the types and Zod schema; this adds only the missing input and form plumbing.
- When **editing** an event that already has a location or description, the details section starts **expanded** so existing data is never hidden behind the expander.
- Detail views already render location; no change there.

## Testing

Component tests (Vitest):

- Multi-add: save keeps sheet open, clears text, refocuses input, persists category; header shows Done after first add; edit mode still closes on save; failed save preserves typed text.
- Time picker: minute wheel exposes 12 positions; off-grid initial value shown and preserved when untouched.
- Duration preservation: start change shifts end by delta (create and edit); midnight clamp to 11:59 PM; end-only edit leaves start alone; nudge buttons shift correctly.
- Location: renders inside details expander; round-trips through create and edit payloads; edit form starts expanded when location or description present.

E2E (Playwright, real BE):

- Grocery multi-add: add 3 items in one sheet session, assert all 3 persist and sheet closed via Done.
- Create event with location, reopen detail, assert location displayed.

## Success criteria

- Adding 5 list items requires opening the sheet once, not five times.
- Setting an event to a common time (e.g. 6:15 PM) requires scrolling at most 12 minute positions or two nudge taps, and never re-adjusting end time to keep a 1-hour event 1 hour.
- A parent can enter "Where?" for an event from the phone form, and synced/legacy events with odd minutes are never silently rounded.
