# Recurring Events — Design Document

**Date:** 2026-03-08
**Status:** BE complete ([PR #20](https://github.com/joe-bor/family-hub-api/pull/20)), FE complete ([PR #117](https://github.com/joe-bor/FamilyHub/pull/117))
**Depends on:** [All-Day & Multi-Day Events](./all-day-and-multi-day-events-design.md) (complete)
**Depended on by:** [Google Calendar Integration](./google-calendar-integration-design.md) — native recurrence should land first so the data model is ready

---

## 1. Product Context & Key Decisions

### Why Recurring Events?

The family has real recurring events that are currently re-created manually each week:

| Event | Pattern | Duration |
|-------|---------|----------|
| Preschool | Tuesday / Thursday / Friday | School year |
| Dance class | Every Sunday | Ongoing |
| Cooking class (starting April) | Every Thursday | Ongoing from start date |
| Sports camp (summer) | Daily | Bounded date range |

These are simple, predictable weekly patterns — not complex enterprise scheduling.

### Why Native (Not Google-Dependent)?

We considered three approaches:

| Option | Approach | Verdict |
|--------|----------|---------|
| **A: Lean on Google Calendar** | Don't build recurrence. Users create recurring events in Google Calendar, sync pulls them into FamilyHub as individual instances. | **Rejected.** Creates a hard dependency on Google Calendar for a core workflow. Breaks the product premise. |
| **B: Full RRULE with all edit modes** | Full RFC 5545 support including "this and future events" edit scope. | **Rejected.** "This and future events" is the hardest calendar UX/data problem. Not worth the complexity for a family app. |
| **C: Native recurrence, scoped edit modes** | RRULE storage, BE expansion, "this event" + "all events" edit scope only. | **Chosen.** 80% of the value, ~40% of the complexity. Covers all real family use cases. |

**The product vision driving this decision:** FamilyHub is the **primary interface** for the family. Google Calendar is a data import pipe that stays connected in the background. Core interactions — including creating and managing recurring events — happen in FamilyHub. A family member should be able to connect their Google Calendar for existing events and then mostly forget Google Calendar exists.

### Build Order Decision

1. **Native recurring events** (this feature) — FamilyHub stands on its own
2. **Google Calendar read-only sync** — pulls events in, including recurring instances
3. **Google Calendar write-back** (future) — pushes native events out, including recurring ones

This order ensures FamilyHub is never dependent on Google Calendar for core functionality. When write-back lands, native RRULE can push directly to Google's `recurrence` field — same format, no translation.

### Instance Identity Decision

Expanded instances are computed on read — they don't exist in the DB and have no UUID. But the FE needs a stable identifier for React keys, TanStack Query cache, and routing to the correct API endpoint.

**Decision: `id: null` with explicit `recurringEventId` + `date` fields (Option B).**

We considered three options:

| Option | Approach | Verdict |
|--------|----------|---------|
| **A: Synthetic composite ID** | Return `id` as `"{parentId}_{date}"`. FE parses to extract routing info. | **Rejected.** FE has to know the format and split on `_`. Fragile. |
| **B: Null ID + explicit fields** | Return `id: null`, `recurringEventId: "uuid"`, `date: "2026-03-11"`. FE composes its own key. | **Chosen.** Honest contract. FE already knows about `recurringEventId`. |
| **C: Deterministic UUID** | Generate UUID v5 from `parentId + date`. Looks like a normal UUID but doesn't exist in DB. | **Rejected.** Misleading — FE could accidentally call `PUT /events/{fake-uuid}` and get a 404. |

**How the FE handles it:**

```typescript
// One-liner to get a stable key for any event (regular, parent, instance, or exception)
const eventKey = event.id ?? `${event.recurringEventId}_${formatDate(event.date)}`;
```

**Identity by event type:**

```
Regular event:     { id: "uuid",  isRecurring: false }
Recurring parent:  { id: "uuid",  isRecurring: true, recurrenceRule: "RRULE:..." }
Expanded instance: { id: null,    isRecurring: true, recurrenceRule: "RRULE:...", recurringEventId: "parent-uuid", date: "2026-03-11" }
Exception:         { id: "uuid",  isRecurring: true, recurringEventId: "parent-uuid", date: "2026-03-11" }
```

**Routing logic:**
- `event.id` exists → regular endpoints (`PUT /events/{id}`, `DELETE /events/{id}`)
- `event.id` is null → instance endpoints (`PUT /events/{recurringEventId}/instances/{date}`, `DELETE /events/{recurringEventId}/instances/{date}`)
- Once an instance is edited, it gets a real UUID (exception row). Future edits use the regular endpoint — the instance "graduates" to a real event.

### Edit Scope Decision

Only two options, not three:

| Scope | Supported | Rationale |
|-------|-----------|-----------|
| **Edit this event** | Yes | Creates an exception instance. "Preschool is cancelled this Friday." |
| **Edit all events** | Yes | Modifies the parent. "Preschool moved from 9am to 9:30am." |
| **Edit this and future** | No | The most complex option in calendar engineering. "All events" covers the same use case for a family app — if the schedule changes permanently, edit all or delete and recreate. |

### End Date Decision

**End date is optional, defaults to none (open-ended).**

Rationale: "Runs forever" has no performance or storage cost — the BE only expands instances within the query window (e.g., March 1–31), regardless of whether the series has an end date. The expansion is always bounded by the request, not the RRULE.

| Scenario | End date? |
|----------|-----------|
| Dance class every Sunday | No end date — genuinely open-ended |
| Preschool T/Th/F | Set end date to last day of school year |
| Sports camp daily | Set start + end date |
| Cooking class starting April | Set start date, no end date initially, add end when known |

The "stale event" risk (series keeps showing up after it should stop) is low stakes — user deletes the series when it's no longer needed.

---

## 2. Recurrence Model

### RRULE Format (RFC 5545)

We use RRULE strings — the same format Google Calendar uses internally. This means:
- No custom recurrence format to maintain
- Future Google Calendar write-back can push RRULE directly
- Well-documented spec with mature Java libraries for expansion

Family use cases mapped to RRULE:

```
Preschool T/Th/F:           RRULE:FREQ=WEEKLY;BYDAY=TU,TH,FR
Dance class Sunday:          RRULE:FREQ=WEEKLY;BYDAY=SU
Cooking class (from April):  RRULE:FREQ=WEEKLY;BYDAY=TH
                             (DTSTART on the event = first occurrence)
Sports camp (summer):        RRULE:FREQ=DAILY;UNTIL=20260815
Monthly bill payment:        RRULE:FREQ=MONTHLY;BYMONTHDAY=15
```

### Supported RRULE Subset

We don't need the full RFC 5545 spec. Supported properties:

| Property | Supported Values | Example |
|----------|-----------------|---------|
| `FREQ` | `DAILY`, `WEEKLY`, `MONTHLY` | `FREQ=WEEKLY` |
| `BYDAY` | `MO,TU,WE,TH,FR,SA,SU` | `BYDAY=TU,TH,FR` |
| `BYMONTHDAY` | 1–31 | `BYMONTHDAY=15` |
| `UNTIL` | Date (`yyyyMMdd`) | `UNTIL=20260615` |
| `INTERVAL` | Positive integer | `INTERVAL=2` (every 2 weeks) |

Not supported (Phase 1): `COUNT`, `BYSETPOS`, `BYMONTH`, `BYHOUR`, `WKST`, `YEARLY` frequency, complex ordinal rules ("3rd Tuesday of the month").

### Parent vs Instance Model

```
┌─────────────────────────────────────────────────────┐
│ Parent Event (stored in DB)                         │
│                                                     │
│  id: uuid-parent                                    │
│  title: "Preschool"                                 │
│  date: 2026-03-03  (first occurrence / DTSTART)     │
│  startTime: "9:00 AM"                               │
│  endTime: "12:00 PM"                                │
│  recurrenceRule: "RRULE:FREQ=WEEKLY;BYDAY=TU,TH,FR" │
│  memberId: uuid-kid                                 │
│  isAllDay: false                                    │
│  isRecurring: true                                  │
└─────────────────────────────────────────────────────┘
         │
         │  BE expands on GET /calendar/events?startDate=...&endDate=...
         ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Instance     │  │ Instance     │  │ Instance     │
│ (computed)   │  │ (computed)   │  │ (computed)   │
│              │  │              │  │              │
│ date: Mar 3  │  │ date: Mar 5  │  │ date: Mar 6  │
│ (Tuesday)    │  │ (Thursday)   │  │ (Friday)     │
└──────────────┘  └──────────────┘  └──────────────┘
```

- **Parent event:** One row in DB. Holds the RRULE, the template fields (title, time, member), and the start date.
- **Instances:** Computed on read. Not stored in DB (except exceptions). Each instance inherits all fields from the parent but with a specific `date`.
- **Exception instance:** Stored in DB when a user edits or deletes a single occurrence. Links back to parent via `recurringEventId`.

---

## 3. Data Model Changes

### Extend `calendar_event` Table

```sql
ALTER TABLE calendar_event
    ADD COLUMN recurrence_rule VARCHAR(255),        -- RRULE string, null for non-recurring
    ADD COLUMN recurring_event_id UUID,             -- points to parent for exception instances
    ADD COLUMN original_date DATE,                  -- the date this exception replaces
    ADD COLUMN is_cancelled BOOLEAN DEFAULT false;  -- true = this occurrence is deleted (EXDATE)

ALTER TABLE calendar_event
    ADD CONSTRAINT fk_recurring_parent
    FOREIGN KEY (recurring_event_id) REFERENCES calendar_event(id) ON DELETE CASCADE;

CREATE INDEX idx_calendar_event_recurring ON calendar_event(recurring_event_id);
```

### Field Relationships

| `recurrence_rule` | `recurring_event_id` | `original_date` | `is_cancelled` | Meaning |
|---|---|---|---|---|
| non-null | null | null | false | **Parent** — recurring series definition |
| null | non-null | non-null | false | **Exception** — edited single instance |
| null | non-null | non-null | true | **Cancelled** — deleted single instance (EXDATE) |
| null | null | null | false | **Regular** — non-recurring event (existing behavior) |

### Why Not EXDATE on the Parent?

Some implementations store excluded dates as a comma-separated list on the parent (`EXDATE:20260307,20260314`). We use exception rows instead because:
- An edited instance needs its own data (different title, time, etc.) — can't store that in an EXDATE string
- A cancelled instance and an edited instance use the same mechanism (exception row with `is_cancelled` flag)
- Queries are simpler — no string parsing
- Aligns with how Google Calendar represents exceptions (separate event with `recurringEventId`)

---

## 4. API Contract Changes

### Response — New Fields

```typescript
// CalendarEventResponse — added fields
{
  id: string | null;             // null for expanded instances (not in DB), UUID for everything else
  // ... existing fields ...
  recurrenceRule?: string;       // RRULE string (on parents AND expanded instances, null for regular events)
  recurringEventId?: string;     // UUID of parent (only on instances/exceptions, null for parents/regular)
  isRecurring?: boolean;         // true for parents, expanded instances, and exceptions
}
```

**Important:** Expanded instances have `id: null` because they are computed on read, not stored in DB. The FE uses `recurringEventId + date` as a composite key for these. Exception instances (edited occurrences) have a real UUID because they are stored as rows. See the [Instance Identity Decision](#instance-identity-decision) in Section 1 for the full rationale.

> **Design revision (post-FE implementation):** The original design specified `recurrenceRule: null` on expanded instances — the intent was clean normalization (RRULE "belongs" to the parent). In practice, the FE needs the RRULE on instances to display a human-readable recurrence label (e.g., "Weekly on Tue, Thu, Fri") in the event detail modal. Without it, `formatRecurrenceLabel()` falls back to a generic "Recurring event" string. Since the parent is already in memory during expansion, copying the field is trivial. `id: null` remains the sole discriminator for virtual instances. See [BE PR #21](https://github.com/joe-bor/family-hub-api/pull/21) (pending).

### Create Recurring Event

`POST /calendar/events`

```json
{
  "title": "Preschool",
  "startTime": "9:00 AM",
  "endTime": "12:00 PM",
  "date": "2026-03-03",
  "memberId": "uuid",
  "isAllDay": false,
  "recurrenceRule": "RRULE:FREQ=WEEKLY;BYDAY=TU,TH,FR"
}
```

Returns the parent event. Instances are computed on subsequent GET requests.

### Edit Single Instance

`PUT /calendar/events/{parentId}/instances/{date}`

```json
{
  "title": "Preschool (field trip)",
  "startTime": "8:00 AM",
  "endTime": "1:00 PM",
  "date": "2026-03-06",
  "memberId": "uuid",
  "isAllDay": false
}
```

Creates an exception row in DB. Future GETs return the exception data for that date instead of the computed instance.

### Edit All Events

`PUT /calendar/events/{parentId}`

Standard update — modifies the parent. All future computed instances reflect the change. Existing exceptions are preserved (they've already been explicitly customized).

### Delete Single Instance

`DELETE /calendar/events/{parentId}/instances/{date}`

Creates a cancelled exception row (`is_cancelled = true`). That date is excluded from future expansions.

### Delete All Events

`DELETE /calendar/events/{parentId}`

Deletes parent + all exception rows (cascade).

### GET /calendar/events — Expansion Behavior

The existing endpoint returns a flat list of events. Recurring events are expanded inline:

1. Query all parent events where the recurrence overlaps the date range
2. Expand each parent's RRULE within the range → virtual instances
3. Check for exceptions — replace computed instances with exception data, exclude cancelled dates
4. Merge with regular (non-recurring) events
5. Return flat list sorted by date/time

The FE doesn't paginate differently or handle recurrence — it gets a flat event list like today.

---

## 5. BE Architecture

### RRULE Expansion Library

Use **[lib-recur](https://github.com/dmfs/lib-recur)** (Java) — mature, well-tested RFC 5545 implementation. Handles `FREQ`, `BYDAY`, `UNTIL`, `INTERVAL`, etc. We don't write our own RRULE parser.

```xml
<dependency>
    <groupId>org.dmfs</groupId>
    <artifactId>lib-recur-rfc5545</artifactId>
    <version>0.17.2</version>
</dependency>
```

### Service Layer Changes

```
CalendarEventService (existing)
  ├── getEvents()        — expanded to include recurring instance expansion
  ├── createEvent()      — handles recurrenceRule field
  ├── updateEvent()      — "edit all" (modifies parent)
  ├── deleteEvent()      — "delete all" (cascade)
  │
  ├── updateInstance()   — NEW: "edit this event" (creates exception)
  ├── deleteInstance()   — NEW: "delete this event" (creates cancelled exception)
  │
  └── RecurrenceExpander — NEW: expands RRULE within date range, applies exceptions
```

### Expansion Logic (Pseudocode)

```java
List<CalendarEventResponse> getEvents(UUID familyId, LocalDate rangeStart, LocalDate rangeEnd) {
    // 1. Get all non-recurring events in range (existing query)
    List<CalendarEvent> regular = repo.findRegularEventsInRange(familyId, rangeStart, rangeEnd);

    // 2. Get recurring parents that COULD have instances in range
    List<CalendarEvent> parents = repo.findRecurringParents(familyId);

    // 3. Get all exceptions for these parents
    Map<UUID, List<CalendarEvent>> exceptions = repo.findExceptionsByParentIds(parentIds);

    // 4. Expand each parent
    List<CalendarEventResponse> expanded = new ArrayList<>();
    for (CalendarEvent parent : parents) {
        Set<LocalDate> dates = recurrenceExpander.expand(parent.getRecurrenceRule(),
            parent.getDate(), rangeStart, rangeEnd);

        List<CalendarEvent> parentExceptions = exceptions.getOrDefault(parent.getId(), List.of());
        Set<LocalDate> cancelledDates = parentExceptions.stream()
            .filter(CalendarEvent::isCancelled)
            .map(CalendarEvent::getOriginalDate)
            .collect(toSet());
        Map<LocalDate, CalendarEvent> editedInstances = parentExceptions.stream()
            .filter(e -> !e.isCancelled())
            .collect(toMap(CalendarEvent::getOriginalDate, e -> e));

        for (LocalDate date : dates) {
            if (cancelledDates.contains(date)) continue;
            if (editedInstances.containsKey(date)) {
                expanded.add(mapper.toResponse(editedInstances.get(date)));
            } else {
                expanded.add(mapper.toInstanceResponse(parent, date));
            }
        }
    }

    // 5. Merge, sort, return
    List<CalendarEventResponse> all = new ArrayList<>(regular.stream().map(mapper::toResponse).toList());
    all.addAll(expanded);
    all.sort(comparing(CalendarEventResponse::getDate).thenComparing(CalendarEventResponse::getStartTime));
    return all;
}
```

### Query for Recurring Parents

A recurring event's instances could fall in any date range, so we can't use a simple `WHERE date BETWEEN` filter. Instead:

```sql
-- Find recurring parents that might have instances in the range
SELECT * FROM calendar_event
WHERE family_id = :familyId
  AND recurrence_rule IS NOT NULL
  AND recurring_event_id IS NULL           -- is a parent, not an exception
  AND date <= :rangeEnd                     -- series started before range ends
  AND (
    -- no end date (UNTIL) → could have instances in range
    recurrence_rule NOT LIKE '%UNTIL=%'
    OR
    -- has UNTIL → extract and check overlap (done in service layer, not SQL)
    TRUE
  )
```

The UNTIL check is simpler to do in Java after fetching parents — filter out parents whose UNTIL date is before `rangeStart`.

---

## 6. FE Changes

### Recurrence Picker Component

Added to the event form when creating or editing (only for "edit all" mode):

```
┌──────────────────────────────────────┐
│ Repeat: [Does not repeat ▼]         │
│                                      │
│  Options:                            │
│  • Does not repeat                   │
│  • Daily                             │
│  • Weekly on [current day]           │
│  • Weekly on custom days...          │
│  • Monthly on the [nth]              │
│  • Custom...                         │
└──────────────────────────────────────┘

When "Weekly on custom days..." selected:
┌──────────────────────────────────────┐
│ Repeat every [1 ▼] week(s)          │
│                                      │
│ [M] [T] [W] [T] [F] [S] [S]        │  ← day-of-week toggles
│  ○   ●   ○   ●   ●   ○   ○         │  ← T/Th/F selected
│                                      │
│ Ends: ○ Never  ○ On date [____]     │
└──────────────────────────────────────┘
```

### Edit/Delete Confirmation Dialog

When a user edits or deletes a recurring instance, show a dialog:

```
┌──────────────────────────────────────┐
│ Edit recurring event                 │
│                                      │
│ ○ This event                         │
│ ○ All events                         │
│                                      │
│          [Cancel]  [OK]              │
└──────────────────────────────────────┘
```

- **This event** → `PUT /calendar/events/{parentId}/instances/{date}`
- **All events** → `PUT /calendar/events/{parentId}`
- Same pattern for delete.

### TypeScript Types

```typescript
// Extend CalendarEvent
interface CalendarEvent {
  // ... existing fields ...
  recurrenceRule?: string;
  recurringEventId?: string;
  isRecurring?: boolean;
}

// Recurrence form state (FE only, converted to RRULE before API call)
interface RecurrenceFormState {
  frequency: "none" | "daily" | "weekly" | "monthly";
  interval: number;          // default 1
  weeklyDays?: string[];     // ["TU", "TH", "FR"]
  monthDay?: number;         // 1-31
  endDate?: string;          // yyyy-MM-dd or null for "never"
}
```

The FE builds the RRULE string from `RecurrenceFormState` before sending to the API. A utility function handles this conversion:

```typescript
// recurrenceFormToRRule({ frequency: "weekly", interval: 1, weeklyDays: ["TU","TH","FR"] })
// → "RRULE:FREQ=WEEKLY;BYDAY=TU,TH,FR"
```

### Rendering

Expanded instances render exactly like regular events — no visual difference needed. Optionally, a small repeat icon (↻) on event tiles to indicate the event is part of a series. The event detail modal shows "Repeats weekly on Tue, Thu, Fri" as a human-readable label.

---

## 7. Google Calendar Compatibility

### Why RRULE Matters for Future Integration

| Scenario | How It Works |
|----------|-------------|
| **Google sync (read)** | Google events fetched with `singleEvents=true` arrive as individual instances. Stored as regular events with `source=GOOGLE`. No RRULE parsing needed. |
| **Google sync (write-back, future)** | Native recurring events push to Google with `recurrence: ["RRULE:FREQ=WEEKLY;BYDAY=TU,TH,FR"]`. Same format — no translation. |
| **Google recurring instance edited** | Google returns it as a separate event with `recurringEventId`. Maps directly to our exception model. |
| **Google recurring instance cancelled** | Google returns it with `status: "cancelled"`. Maps to our `is_cancelled` exception row. |

The data model is intentionally aligned with Google's recurring event model so that future integration is additive, not a rewrite.

---

## 8. What We Are NOT Building

- **"This and future events" edit scope** — too complex for the value. "All events" + "this event" covers family use cases.
- **Yearly frequency** — no real family use case identified (birthdays are better as all-day events)
- **Complex ordinal rules** — "3rd Tuesday of the month" etc. Not needed for the identified use cases.
- **Recurring multi-day events** — a multi-day vacation that repeats weekly doesn't make sense. Recurrence is for single-day or all-day events.
- **Drag-to-reschedule recurring instances** — edit via form only.
- **COUNT-based recurrence** — "repeat 10 times" — use end date instead.

---

## 9. Acceptance Criteria

### BE

- [ ] Migration adds `recurrence_rule`, `recurring_event_id`, `original_date`, `is_cancelled` columns
- [ ] `POST /calendar/events` accepts `recurrenceRule` field and stores parent event
- [ ] `GET /calendar/events` expands recurring events into instances within the query date range
- [ ] Expansion correctly handles `FREQ=DAILY`, `FREQ=WEEKLY;BYDAY=...`, `FREQ=MONTHLY`
- [ ] `UNTIL` end date is respected in expansion
- [ ] `INTERVAL` is respected (e.g., every 2 weeks)
- [ ] `PUT /calendar/events/{id}` updates parent (all events)
- [ ] `PUT /calendar/events/{parentId}/instances/{date}` creates exception instance
- [ ] `DELETE /calendar/events/{id}` deletes parent + cascades exceptions
- [ ] `DELETE /calendar/events/{parentId}/instances/{date}` creates cancelled exception
- [ ] Exception instances replace computed instances in GET response
- [ ] Cancelled instances are excluded from GET response
- [ ] RRULE validation rejects unsupported properties
- [ ] Tests cover expansion, exceptions, cancellations, edge cases

### FE

- [ ] Event form includes recurrence picker (none / daily / weekly / monthly)
- [ ] Weekly mode shows day-of-week toggle buttons
- [ ] Optional end date for recurrence
- [ ] Form builds RRULE string from picker state
- [ ] Edit/delete on recurring instance shows "this event" / "all events" dialog
- [ ] "This event" routes to instance endpoints
- [ ] "All events" routes to parent endpoints
- [ ] Event detail modal shows human-readable recurrence label
- [ ] Recurring event badge/icon on event tiles (optional)
- [ ] Zod schema validates recurrence form state
- [ ] Tests cover form interactions, dialog, API routing

---

## 10. Reference

### Existing Code

- BE entity: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/CalendarEvent.java`
- BE DTOs: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/CalendarEventRequest.java`, `CalendarEventResponse.java`
- BE service: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/CalendarEventService.java`
- BE mapper: `backend/family-hub-api/src/main/java/com/familyhub/demo/mapper/CalendarEventMapper.java`
- FE types: `frontend/src/lib/types/calendar.ts`
- FE form: `frontend/src/components/calendar/components/event-form.tsx`
- FE hooks: `frontend/src/api/hooks/use-calendar.ts`
- API contract: `docs/calendar-events-api-reference.md`

### Related Docs

- [All-Day & Multi-Day Events](./all-day-and-multi-day-events-design.md) — prerequisite (complete)
- [Google Calendar Integration](./google-calendar-integration-design.md) — builds on top of this
- [Google Calendar Research](./google-calendar-research.md) — recurring event section (lines 285–309)

### External References

- [RFC 5545 — iCalendar Spec](https://datatracker.ietf.org/doc/html/rfc5545#section-3.3.10) — RRULE definition
- [lib-recur (Java)](https://github.com/dmfs/lib-recur) — RRULE expansion library
- [Google Calendar Recurring Events](https://developers.google.com/calendar/api/concepts/events-calendars#recurring_events) — how Google represents recurrence
