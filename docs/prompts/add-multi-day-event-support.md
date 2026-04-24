# Task: Add Multi-Day Event Support

## Context

This is a cross-repo change (BE + FE) that adds `endDate` to the calendar event model, enabling multi-day all-day events like vacations and trips.

**You are expected to form your own plan after exploring the codebase.** Read the existing entity, DTOs, mapper, repository, validation, and FE types/form before making changes. Push back on any part of this prompt if you disagree after seeing the code.

**Read these design docs first:**
- `docs/all-day-and-multi-day-events-design.md` — full product context, problem statement, design philosophy, field relationship table, rendering approach, and acceptance criteria
- `docs/google-calendar-integration-design.md` — the Google Calendar integration depends on this work. The `endDate` field maps directly to Google Calendar's `end.date` for multi-day all-day events. Keep this in mind but don't build for it — just don't make decisions that would conflict with it.

**Prerequisite:** The all-day rendering fix (FE PR #110) has been merged. This task builds on the all-day sections and form toggle that PR introduced. The event form already has an `isAllDay` toggle that hides time pickers — Phase B extends it with an end date picker.

---

## The Problem

The calendar event model has a single `date` field (LocalDate). There is no way to represent an event spanning multiple days. Families have multi-day events (vacations, trips, holidays, visiting relatives), and the upcoming Google Calendar integration needs to handle Google's multi-day all-day events.

## What We Want

Add an `endDate` column to the `calendar_event` table and update both BE and FE to support it.

### Data Model

- `endDate` is nullable. `NULL` = single-day event.
- `endDate` only valid when `isAllDay = true`.
- `endDate >= date` when present.
- `isAllDay = false` + `endDate != null` is **invalid** — reject with validation error.

| `isAllDay` | `endDate` | Meaning |
|---|---|---|
| `false` | `null` | Normal timed event |
| `true` | `null` | Single-day all-day event |
| `true` | `> date` | Multi-day all-day event |
| `false` | non-null | Invalid — rejected |

### Rendering Approach

Multi-day events render as **repeated all-day badges on each day in the range**. No spanning bars. This is intentional — the app is a daily at-a-glance planner, not a Gantt chart. See the design doc for details and diagrams.

---

## BE Scope

### Migration

New Flyway migration (next version number after the latest):

```sql
ALTER TABLE calendar_event ADD COLUMN end_date DATE;
```

### Entity

Add `endDate` (LocalDate, nullable) to `CalendarEvent`.

### DTOs

- `CalendarEventRequest`: add optional `endDate` field (`@JsonFormat(pattern = "yyyy-MM-dd")`)
- `CalendarEventResponse`: add `endDate` field (nullable)

### Validation

This is the important part. The cross-field rules:
- If `endDate` is provided and `isAllDay` is not true → reject ("endDate is only valid for all-day events")
- If `endDate` is provided and `endDate < date` → reject ("End date must be on or after start date")
- If `endDate` equals `date` → treat as single-day (could normalize to null, or allow it — your call)

Consider where to put this validation — DTO-level annotations, service-level checks, or a custom validator. The existing pattern in the codebase should guide you.

### Repository

The date range query needs to change. Currently (verify this yourself), events are fetched by date range. With multi-day events, a "Vacation Mar 7–9" must appear when querying March 8. The query becomes:

```sql
WHERE date <= :rangeEnd AND COALESCE(end_date, date) >= :rangeStart
```

Find the existing query and update it. This might be in the repository, service, or a specification — check the current pattern.

### Mapper

Map `endDate` between entity, request, and response. Straightforward.

### Tests

Test the behavior:
- Create a multi-day all-day event → verify it's returned when querying any day in the range
- Create a single-day all-day event (no endDate) → verify existing behavior unchanged
- Create a timed event → verify endDate is null in response
- Reject: `isAllDay = false` with `endDate` set → expect 400
- Reject: `endDate < date` → expect 400
- Reject: `endDate` set without `isAllDay = true` → expect 400
- Query a date range that overlaps the middle of a multi-day event → event should be returned
- Query a date range that doesn't overlap → event should not be returned

---

## FE Scope

### TypeScript Types

Add `endDate?: string` to `CalendarEvent`, `CalendarEventResponse`, `CreateEventRequest`, `UpdateEventRequest`.

### Zod Schema

Add `endDate` with cross-field validation:
- Optional string in `yyyy-MM-dd` format
- If present, must be >= `date`
- Only valid when `isAllDay` is checked

### Event Form

The `isAllDay` toggle already exists (PR #110) and hides time pickers when ON. Extend it:

When `isAllDay` is toggled ON:
- Time pickers already hidden (existing behavior from PR #110)
- Add an optional "End date" date picker below the start date
- If end date is not set → single-day all-day event
- If end date is set → multi-day (must be >= start date)

When `isAllDay` is toggled OFF:
- Time pickers already shown (existing behavior)
- Hide and clear the end date field

### Calendar Views

Multi-day events should appear as all-day badges on each day in the range. The all-day rendering from Phase A (the prerequisite) provides the all-day section — this task extends it to repeat the badge across multiple days.

The logic: when processing events for a given day, check if any multi-day events overlap that day (`event.date <= day <= event.endDate`). If so, include them in that day's all-day events.

### Event Detail Modal

Show the date range for multi-day events:
- Single day: "March 7, 2026"
- Multi-day: "March 7 – March 9, 2026"

### Schedule View

Decide the best UX — either:
- Show once with the full range: "March 7 – 9 · Vacation"
- Show under each day: "All day · Vacation (Day 2 of 3)"

Pick what feels right after seeing the existing schedule view code.

### Tests

Test the behavior:
- Create a multi-day event via the form → verify it appears on each day in daily/weekly/monthly views
- Edit a multi-day event → verify the update reflects correctly
- Toggle isAllDay off → verify endDate clears
- Form validation: endDate < date → expect error
- Form validation: endDate without isAllDay → expect it's hidden/disabled

---

## Starting Points

These are suggestions. Verify them yourself:

**BE:**
- Entity: `src/main/java/com/familyhub/demo/model/CalendarEvent.java`
- Request DTO: `src/main/java/com/familyhub/demo/dto/CalendarEventRequest.java`
- Response DTO: `src/main/java/com/familyhub/demo/dto/CalendarEventResponse.java`
- Mapper: `src/main/java/com/familyhub/demo/mapper/CalendarEventMapper.java`
- Repository: `src/main/java/com/familyhub/demo/repository/CalendarEventRepository.java`
- Flyway migrations: `src/main/resources/db/migration/`

**FE:**
- Types: `src/lib/types/calendar.ts`
- Zod schema: `src/lib/validations/calendar.ts`
- Event form: `src/components/calendar/components/event-form.tsx`
- Calendar views: `src/components/calendar/views/`
- Event detail modal: `src/components/calendar/components/event-detail-modal.tsx`

**Docs:**
- Design doc: `docs/all-day-and-multi-day-events-design.md`
- Google Calendar design: `docs/google-calendar-integration-design.md`
- API contract: `docs/calendar-events-api-reference.md`

---

## Workflow

1. Create a branch: `feat/multi-day-events`
2. Plan your approach — read the entity, DTOs, mapper, repository, form, and views
3. Implement in atomic commits:
   - BE: migration + entity + DTOs + mapper
   - BE: validation rules
   - BE: repository query update
   - BE: tests
   - FE: types + schema
   - FE: form changes
   - FE: view rendering
   - FE: tests
4. Update `docs/calendar-events-api-reference.md` with the new `endDate` field
5. Open a PR with a clear description

---

## Acceptance Criteria

### BE
- [ ] `end_date` column exists in DB via Flyway migration
- [ ] API accepts and returns `endDate` field
- [ ] Validation rejects `endDate` when `isAllDay` is false
- [ ] Validation rejects `endDate < date`
- [ ] Date range queries return multi-day events that overlap the range
- [ ] Existing single-day and timed events work unchanged

### FE
- [ ] Event form shows end date picker when `isAllDay` is toggled on
- [ ] Event form hides end date and clears it when `isAllDay` is toggled off
- [ ] Multi-day events appear as all-day badges on each day in the range
- [ ] Event detail modal shows date range for multi-day events
- [ ] No regressions on single-day all-day or timed events

---

## PR Review Checklist

- [ ] Migration is additive (no breaking changes to existing data)
- [ ] Cross-field validation is solid (no way to create invalid state)
- [ ] Repository query handles the overlap correctly
- [ ] FE form UX is intuitive (toggling isAllDay controls visibility of related fields)
- [ ] Tests cover behavior, not implementation
- [ ] API contract doc updated
- [ ] No unnecessary refactors outside scope
