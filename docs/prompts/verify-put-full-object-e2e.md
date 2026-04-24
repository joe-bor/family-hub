# Task: Verify PUT Full-Object Semantics End-to-End

## Context

PR #81 changed calendar event updates from PATCH to PUT on the FE, aligning with the API contract (`docs/calendar-events-api-reference.md`). The contract says:

> PUT is a full replacement — send all fields in the request body. The backend replaces the entire entity — omitted fields are not preserved.

The FE constructs a complete `UpdateEventRequest` with all fields before sending. **But nobody has verified the full round-trip against the real BE.** If the FE omits a field or the BE nulls out something unexpected, data gets silently wiped.

This is a verification + hardening task, not a rewrite.

---

## What You're Verifying

The question is simple: **when a user edits one field (e.g., title), does every other field survive the round-trip?**

Specifically:
1. Does the FE send ALL fields in the PUT body, including ones the user didn't change?
2. Does the BE `update` method set all fields from the request, or does it skip/null-out optional ones?
3. Do optional fields (`location`, `isAllDay`) survive when present? When absent/null?

---

## Starting Points (Verify These Yourself)

### FE — Request Construction

- `calendar-module.tsx` `handleUpdateEvent()` — constructs `UpdateEventRequest` from `EventFormData`. Check: does it populate every field from the existing event + form changes?
- `event-form-modal.tsx` `eventToFormData()` — pre-populates the edit form from the existing event. Check: does it carry over ALL fields, including optional ones like `location`?
- `calendar.service.ts` `updateEvent()` — sends `httpClient.put(..., request)`. Check: does it pass the entire object, or does it strip/transform anything?
- `UpdateEventRequest` type in `src/lib/types/calendar.ts` — compare field list against `CalendarEventRequest` DTO on the BE

### BE — Update Implementation

- `CalendarEventService.java` update method — does it set every field from the request? Pay attention to `isAllDay` and `location` handling when they're null vs absent
- `CalendarEventMapper.java` — does the mapper handle null optional fields correctly, or does it NPE?
- `CalendarEventRequest.java` — `isAllDay` is `Boolean` (nullable wrapper) and `location` is `String` (nullable). What happens when the FE sends `isAllDay: false` vs omitting it entirely?

---

## Scenarios to Verify

### Scenario 1: Edit title only, preserve everything else
1. Create event with: title, startTime, endTime, date, memberId, location="Office", isAllDay=false
2. Edit only the title
3. After save: confirm location is still "Office" and isAllDay is still false

### Scenario 2: Optional fields — null vs omit
1. Create event WITH location
2. Edit event, clear the location field
3. After save: confirm location is null/empty, not the old value

### Scenario 3: isAllDay round-trip
1. Create event with isAllDay=true
2. Edit something else (e.g., title)
3. After save: confirm isAllDay is still true, not reset to false/null

### Scenario 4: Time format survives the round-trip
1. Create event with startTime="2:00 PM", endTime="3:30 PM"
2. Edit title only
3. After save: confirm times are still "2:00 PM" and "3:30 PM", not converted to 24h or mangled

---

## What To Do With Findings

### If everything works:
- Add a focused integration test (or E2E test if the project has that infra) covering scenario 1 at minimum
- Mark this item as resolved in `docs/calendar-events-api-reference.md` TODO list

### If something breaks:
- Document exactly what broke and in which layer (FE construction, BE update logic, mapper)
- Fix it, with tests proving the fix
- If the fix is on the BE side (human-managed), create a clear writeup of the bug and expected behavior — do NOT modify BE code

---

## Git Workflow

1. **Create a new branch** from `main`: `test/verify-put-full-object-e2e`
2. **Plan before coding** — trace the data flow yourself before writing tests
3. **Make atomic commits**: `test(calendar): add PUT full-replacement verification tests`
4. **Run tests** (`npm test -- --run`) and **lint** (`npm run lint`) before opening the PR
5. **Open a PR** with findings documented in the body — what was verified, what passed, what (if anything) broke

## Acceptance Criteria

- [ ] All 4 scenarios traced through FE code and verified (code-level, not just manual)
- [ ] At least one test confirms PUT sends all fields and they survive the round-trip
- [ ] Optional field handling (null vs omit) is documented or tested
- [ ] If bugs found: fixed (FE side) or documented (BE side)
- [ ] All tests pass, lint passes, build passes
