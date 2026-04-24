# Recurring Calendar Events — BE Implementation Prompt

**For:** Backend agent (`backend/family-hub-api/`)
**Design doc:** `docs/recurring-events-design.md`
**Date:** 2026-03-09

---

## What you're building

Native recurring event support using RRULE (RFC 5545). Events are stored once as a parent row with an RRULE string and expanded into virtual instances on read. Users can edit or delete individual occurrences (creating exception rows). The design doc (`docs/recurring-events-design.md`) has the full product context, data model, API contract, and acceptance criteria — read it first.

This prompt doesn't repeat the design doc. It corrects things we got wrong, fills in gaps we discovered during a prior implementation attempt, and gives you a head start on technical decisions that cost us time.

---

## Critical correction: RRULE expansion library

The design doc recommends **lib-recur** (`org.dmfs:lib-recur-rfc5545:0.17.2`). This is wrong on two counts:

1. **The artifact ID is wrong.** The real artifact is `org.dmfs:lib-recur` (not `lib-recur-rfc5545`). Version 0.17.1 is the latest on Maven Central.

2. **lib-recur is incompatible with Java 21.** The `RecurrenceRule$Part` enum uses `EnumSet.of()` inside its own static initializer, which triggers a `ClassCastException` at class load time on Java 21. This is a known JDK issue ([JDK-8211749](https://bugs.openjdk.org/browse/JDK-8211749)). The lib-recur repo targets Java 8, has no Java 21 CI, and nobody has reported the issue upstream. It's a dead end — don't waste time trying older versions; the `Part` enum design has been the same for years.

**Use ical4j instead.** We researched three alternatives:

| Library | Verdict |
|---------|---------|
| **ical4j** (`org.mnode.ical4j:ical4j:4.2.3`) | **Use this.** Actively maintained, Java 21 tested in CI, `Recur<LocalDate>` API maps directly to our use case. |
| google-rfc-2445 (`org.jresearch:org.jresearch.google-rfc-2445:1.0.11`) | Viable but older codebase, RFC 2445 (not 5545), less modern API. |
| lib-recur (`org.dmfs:lib-recur:0.17.1`) | Broken on Java 21. No fix available. |

### ical4j key API

```java
// Parse an RRULE string
Recur<LocalDate> recur = new Recur<>("FREQ=WEEKLY;BYDAY=TU,TH,FR;INTERVAL=2");

// Expand within a date range
LocalDate seed = parent.getDate();  // DTSTART
List<LocalDate> dates = recur.getDates(seed, rangeStart, rangeEnd);

// Inspect parsed properties for validation
recur.getFrequency()    // Frequency enum
recur.getDayList()      // List<WeekDay> for BYDAY
recur.getMonthDayList() // List<Integer> for BYMONTHDAY
recur.getCount()        // -1 if not set
recur.getUntil()        // null if not set
recur.getInterval()     // -1 if not set
recur.getSetPosList()   // List<Integer> for BYSETPOS
```

ical4j handles RFC 5545 parsing and date expansion. You still write your own validation layer on top (reject YEARLY, COUNT, BYSETPOS, etc.) — but you validate the parsed `Recur` object's getters, not raw strings.

**Important caveat:** We haven't actually run ical4j in this project yet. The research says it works on Java 21, but verify it yourself — add the dependency, write a small spike test that parses an RRULE and expands dates, and confirm it works before building on top of it. If it doesn't work, `google-rfc-2445` is your fallback, and manual parsing is the last resort (it works, we proved that, but it's tech debt).

---

## Things the design doc gets right (don't change these)

- **Data model** (Section 3) — the four columns, the self-referencing FK with CASCADE, the field relationship table. All correct and tested.
- **API contract** (Section 4) — the endpoints, the instance identity contract (`id: null` for virtual instances), the edit scope (this event / all events). All correct.
- **Expansion logic** (Section 5 pseudocode) — the overall flow (regular events + expanded recurring + exceptions merged and sorted). Correct.
- **Acceptance criteria** (Section 9 BE checklist) — use this as your test plan.

---

## Things the design doc gets wrong or is vague about

### 1. RRULE prefix inconsistency

The design doc sometimes includes the `RRULE:` prefix (e.g., `"RRULE:FREQ=WEEKLY;BYDAY=TU,TH,FR"`) and sometimes doesn't. Pick one and be consistent. We stored **without the prefix** (just `"FREQ=WEEKLY;BYDAY=TU,TH,FR"`) because:
- ical4j's `Recur` constructor expects the string without the prefix
- It's less noisy in the DB
- The prefix is an iCalendar property name, not part of the RRULE value itself

If you go with ical4j, confirm what format its parser expects and store accordingly.

### 2. Flyway migration column constraint

The design doc shows `ADD COLUMN is_cancelled BOOLEAN DEFAULT false`. Our migration used `BOOLEAN NOT NULL DEFAULT FALSE` — the `NOT NULL` matters because Hibernate maps `boolean` (primitive) fields and will complain without it. Make sure the entity field and migration agree.

### 3. GET behavior without date range

The design doc doesn't specify what happens when `GET /calendar/events` is called without `startDate`/`endDate`. You need to decide:
- We chose: **skip expansion entirely, return parent records as-is** alongside regular events. This way the FE knows recurring series exist without the BE expanding infinite instances.
- The alternative is to require a date range for recurring expansion and only return regular events when no range is given.
- Either works, but be explicit about it and test it.

### 4. Recurring + endDate mutual exclusion

The design doc says "Recurring multi-day events" are not supported (Section 8) but doesn't say how to enforce it. You need a validation rule: if `recurrenceRule` is non-null and `endDate` is non-null, reject with a clear error message. The `endDate` field is for multi-day all-day events; recurring events use `UNTIL` in the RRULE for series end dates.

### 5. Member filter on recurring events

The design doc doesn't mention how member filtering interacts with recurring expansion. When `memberId` is passed as a query param, you need to filter recurring parents by member **before** expanding — don't expand all parents and filter after, that's wasteful.

### 6. Sorting

The design doc says "sorted by date/time" but the response uses string-formatted times (`"9:00 AM"`). Sorting these lexicographically works for `h:mm AM/PM` format only if you're careful (10:00 AM sorts after 9:00 AM as strings, which is wrong). Consider sorting by the entity's `LocalTime` before mapping to DTOs, or sort the responses aware of AM/PM. We sorted by `date` then `startTime` (string) — this has the 10 vs 9 bug but nobody caught it yet. Your call on whether to fix it now.

---

## Implementation structure

The design doc doesn't prescribe a PR structure. Here's a natural breakdown based on what we learned:

1. **Data model + validation** — Migration, entity fields, DTO changes, RRULE validator, mapper updates. This is the foundation; nothing works without it.
2. **Recurrence expansion** — RecurrenceExpander, service refactor for GET, new repository queries. This is where ical4j does the heavy lifting.
3. **Instance edit/delete** — Two new endpoints, two new service methods, one new repository query. Small and focused.

Each step should be independently testable. Run the full test suite after each step — existing tests must still pass (backward compatibility).

---

## Testing notes from our implementation

- **Unit tests don't need Docker.** Only integration tests use Testcontainers. Structure your tests so you can iterate fast on unit tests (`./mvnw test -Dtest='YourTestClass'`).
- **The service test uses `@InjectMocks` with Mockito.** When you add new dependencies to `CalendarEventService` (like the expander), you need corresponding `@Mock` fields or `@InjectMocks` won't wire them.
- **Existing tests construct `CalendarEventRequest` inline.** When you add the `recurrenceRule` field to the record, every inline construction breaks. You'll need to update them all — including in `CalendarEventServiceTest`, `CalendarEventMapperTest`, and `TestDataFactory`. The `CalendarEventResponse` builder is more forgiving (new fields default to null/false).
- **Lenient mocking helps.** Most existing service tests deal with regular events and don't care about the recurring parent query. A `lenient().when(repo.findByFamilyAndRecurrenceRuleIsNotNull(...)).thenReturn(List.of())` in `@BeforeEach` saves you from updating every test individually.
- **Integration tests validate the full stack.** After all three steps, write an integration test that: creates a recurring event, GETs with a date range (verifies expansion), edits one instance, deletes another, GETs again (verifies exception replaces computed, cancelled is excluded), then deletes the parent (verifies cascade).

---

## Files you'll touch

| File | What changes |
|------|-------------|
| `pom.xml` | Add ical4j dependency |
| `src/main/resources/db/migration/V3__add_recurrence_columns.sql` | New migration |
| `.../model/CalendarEvent.java` | 4 new fields (recurrenceRule, recurringEvent, originalDate, isCancelled) |
| `.../dto/CalendarEventRequest.java` | Add optional recurrenceRule field |
| `.../dto/CalendarEventResponse.java` | Add recurrenceRule, recurringEventId, isRecurring |
| `.../mapper/CalendarEventMapper.java` | Map new fields + new `toInstanceResponse()` method |
| `.../service/RecurrenceRuleValidator.java` | **New** — validates parsed RRULE against allowed subset |
| `.../service/RecurrenceExpander.java` | **New** — expands RRULE within date range, applies exceptions |
| `.../service/CalendarEventService.java` | Validation wiring, GET refactor, updateInstance(), deleteInstance() |
| `.../repository/CalendarEventRepository.java` | New query methods |
| `.../controller/CalendarEventController.java` | 2 new instance endpoints |
| `TestDataFactory.java` | New factory methods, update existing ones for new field |

---

## What to verify yourself

Don't take this prompt or the design doc as gospel. Specifically:

- **Verify ical4j works on Java 21 with Spring Boot 4.** Add the dependency, write a test, run it. If it fails, try google-rfc-2445 before falling back to manual parsing.
- **Verify ical4j's `Recur` constructor format.** Does it want `"FREQ=WEEKLY;BYDAY=MO"` or `"RRULE:FREQ=WEEKLY;BYDAY=MO"`? This determines what you store in the DB.
- **Read the existing code.** The service, mapper, controller, and tests have patterns and conventions. Follow them — don't introduce new patterns.
- **Run the full test suite** (`JWT_SECRET="dGVzdC1zZWNyZXQta2V5LXRoYXQtaXMtbG9uZy1lbm91Z2gtZm9yLWhtYWMtc2hhLTI1Ng==" ./mvnw test`) after each step. Integration tests need Docker running.
- **Push back on this prompt** if something doesn't make sense or you find a better approach. This is guidance, not a spec.
