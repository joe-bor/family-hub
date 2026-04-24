# PR #20 Review — Recurring Events

Good work overall. The architecture is solid — 3-query expansion, exception model, integration test lifecycle. These are required changes before merge.

---

## Blockers

### 1. Virtual instance ID must be `null`, not parent ID

`toInstanceResponse` (CalendarEventMapper.java:123) sets `.id(parent.getId())`. This means every expanded instance shares the same UUID. **This violates our Instance Identity Decision** (see `docs/recurring-events-design.md`, Section 1).

Fix:
- `.id(null)` — virtual instances are not in the DB and must not pretend to be
- `.recurringEventId(parent.getId())` — currently set to `null`, must be the parent's UUID. The FE needs this to route to `PUT /{parentId}/instances/{date}`
- `CalendarEventResponse.id` must be nullable — change from `UUID` to `UUID` that serializes as `null` in JSON (use `@JsonInclude(Include.ALWAYS)` or just let Jackson serialize null)

The contract:
```
Regular event:     { id: "uuid",  isRecurring: false }
Recurring parent:  { id: "uuid",  isRecurring: true, recurrenceRule: "RRULE:..." }
Expanded instance: { id: null,    isRecurring: true, recurringEventId: "parent-uuid" }
Exception:         { id: "uuid",  isRecurring: true, recurringEventId: "parent-uuid" }
```

### 2. Lexicographic startTime sort is broken

CalendarEventService.java:320 sorts by `startTime` as a string. `"9:00 AM"` sorts after `"10:00 AM"` because `'9' > '1'`. This is a real bug — events at 9am will appear after events at 10am.

Fix: Parse to `LocalTime` for comparison. The mapper already has `stringToTime()` — use it or add a comparator that parses the 12h string.

### 3. No validation that instance date falls on a recurrence

`editRecurringInstance` and `deleteRecurringInstance` accept any date. A client could `PUT /{parentId}/instances/2025-06-04` (a Wednesday) for a `BYDAY=TU,TH,FR` rule. This silently creates an orphaned exception row for a date that never appears in expansion.

Fix: After finding the parent, expand the RRULE and verify the requested date is in the set. Return 400 if not: `"Date 2025-06-04 is not an occurrence of this recurring event"`.

### 4. editRecurringInstance missing endDate

The apply-edits block doesn't copy `endDate` to the exception row. If someone edits an instance to be multi-day all-day, the endDate is lost.

Fix: Add `exception.setEndDate(update.getEndDate())` alongside the other setters.

### 5. No date-range guard on expansion

If a client sends `?startDate=2020-01-01&endDate=2030-12-31`, ical4j expands 10 years of daily events (~3,650 dates per parent).

Fix: Cap the query range at 1 year max. If `rangeEnd - rangeStart > 366 days`, return 400: `"Date range must not exceed one year"`. Add this check at the top of `getEventsWithExpansion`.

### 6. Error message for recurring + endDate

Current message: `"Recurring events must not have an end date"` — ambiguous because RRULE has UNTIL.

Fix: Change to `"Recurring events cannot span multiple days (endDate). To set when the series ends, use UNTIL in the recurrence rule."`

---

## Required Cleanup

### 7. Add CHECK constraint to migration

Add defense in depth for the recurrenceRule + endDate mutual exclusion:

```sql
ALTER TABLE calendar_event
    ADD CONSTRAINT chk_no_recurring_multiday
    CHECK (recurrence_rule IS NULL OR end_date IS NULL);
```

### 8. Wildcard import

CalendarEventService.java:201 uses `java.util.*`. The rest of the codebase uses explicit imports. Fix to match.

### 9. Unnecessary local variable

`UUID mid = memberId` at line 254 exists only for the lambda. Use a method reference or extract the filter as a named `Predicate<CalendarEvent>`.

### 10. Missing newline at EOF

CalendarEventRequest.java — add trailing newline.

### 11. Remove Ical4jSpikeTest

`Ical4jSpikeTest.java` served its purpose during exploration. It doesn't belong in the test suite — the real expansion behavior is covered by `RecurrenceExpanderTest`.

### 12. deleteRecurringInstance family ownership pattern

`deleteRecurringInstance` uses `findByFamilyAndId` which scopes to family, so it's safe. But `editRecurringInstance` does an explicit family check on the member while delete doesn't validate the member at all. Make the pattern consistent — both methods should validate the same way.

---

## After fixing, verify:

1. All existing tests pass (backward compat)
2. Integration test covers: expanded instance has `id: null` and `recurringEventId: parent-uuid`
3. Integration test covers: parent delete cascades exception rows (already present — just verify it still passes)
4. Sort order test: event at 9:00 AM sorts before event at 10:00 AM on the same date
