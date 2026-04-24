# BE PR 4: Full Sync — Fetch Google Events and Store Them

## Context

This is the fourth PR in the Google Calendar integration series.
- PR 1 ([#22](https://github.com/joe-bor/family-hub-api/pull/22)): `source`/`description` columns, `EventSource` enum, Google event write protection
- PR 2 ([#23](https://github.com/joe-bor/family-hub-api/pull/23)): OAuth flow, token encryption, connect/disconnect
- PR 3 ([#24](https://github.com/joe-bor/family-hub-api/pull/24)): Calendar selection, `GoogleCredentialService`, `GoogleCalendarListService`, Google Calendar API dependency

This PR is the core of the integration: call Google's Events API, map events to our data model, and store them so `GET /calendar/events` returns both native and Google events.

Design doc: `docs/google-calendar-integration-design.md` (§6 Event Mapping Logic, §6 Sync Logic)

## Important: `singleEvents=false`

We do NOT use `singleEvents=true`. Instead:
- Google returns **recurring parents with RRULE strings** (not expanded instances)
- We store parents as rows with `recurrence_rule` set, `source=GOOGLE`
- Our existing `RecurrenceExpander` handles instance expansion on read — same code path as native recurring events
- Google exceptions (edited/cancelled instances) are stored as exception rows with `recurring_event_id` pointing to our parent's UUID

This means one DB row per recurring series instead of many rows per month. The `RecurrenceExpander` and expansion logic in `CalendarEventService.getEvents()` already handle everything — we just need to store Google events in the right shape.

## What to build

### 1. Flyway Migration: `V9__add_google_event_columns.sql`

```sql
ALTER TABLE calendar_event
    ADD COLUMN google_event_id VARCHAR(1024),
    ADD COLUMN html_link VARCHAR(2048),
    ADD COLUMN etag VARCHAR(255),
    ADD COLUMN google_updated_at TIMESTAMP;

CREATE UNIQUE INDEX idx_calendar_event_google_id
    ON calendar_event(google_event_id)
    WHERE google_event_id IS NOT NULL;
```

- `google_event_id`: Google's opaque event ID. Unique among Google events (partial unique index). Used for upsert during sync.
- `html_link`: "Open in Google Calendar" URL.
- `etag`: For future write-back conflict detection.
- `google_updated_at`: Google's last-modified timestamp.

Native events have all these columns as NULL.

### 2. Entity changes: `CalendarEvent.java`

Add the four new fields:
- `googleEventId` (String, nullable)
- `htmlLink` (String, nullable)
- `etag` (String, nullable)
- `googleUpdatedAt` (LocalDateTime or Instant, nullable)

### 3. DTO changes: `CalendarEventResponse`

Add to the response:
- `htmlLink` (String, nullable) — only non-null for Google events

`googleEventId`, `etag`, and `googleUpdatedAt` are internal — do NOT expose them in the API response.

### 4. Mapper changes: `CalendarEventMapper`

Update `toResponse()` and `toInstanceResponse()` to include `htmlLink`.

### 5. Repository changes: `CalendarEventRepository`

Add:

```java
Optional<CalendarEvent> findByGoogleEventId(String googleEventId);
void deleteByGoogleEventId(String googleEventId);
```

### 6. New service: `GoogleEventMapper`

Maps Google `Event` objects to our `CalendarEvent` entity. Three scenarios:

```java
@Component
public class GoogleEventMapper {

    // For regular (non-recurring) events and recurring parents
    public CalendarEvent toEntity(Event googleEvent, GoogleSyncedCalendar syncedCal) {
        CalendarEvent entity = new CalendarEvent();

        // Common fields
        entity.setTitle(googleEvent.getSummary());
        entity.setDescription(googleEvent.getDescription());
        entity.setLocation(googleEvent.getLocation());
        entity.setHtmlLink(googleEvent.getHtmlLink());
        entity.setGoogleEventId(googleEvent.getId());
        entity.setEtag(googleEvent.getEtag());
        entity.setGoogleUpdatedAt(/* parse googleEvent.getUpdated() */);
        entity.setSource(EventSource.GOOGLE);
        entity.setMember(syncedCal.getMember());
        entity.setFamily(syncedCal.getMember().getFamily());

        // Date/time mapping
        mapDateAndTime(googleEvent, entity);

        // Recurrence
        if (googleEvent.getRecurrence() != null && !googleEvent.getRecurrence().isEmpty()) {
            // Recurring parent — extract and store the RRULE
            // Google returns strings like "RRULE:FREQ=WEEKLY;BYDAY=MO" but iCal4j's
            // Recur constructor expects just the value part: "FREQ=WEEKLY;BYDAY=MO"
            // Strip the "RRULE:" prefix before storing.
            // Note: getRecurrence() may also contain EXDATE/EXRULE strings — find
            // the one that starts with "RRULE:" specifically.
            String rrule = googleEvent.getRecurrence().stream()
                .filter(r -> r.startsWith("RRULE:"))
                .findFirst()
                .map(r -> r.substring("RRULE:".length()))
                .orElse(null);
            entity.setRecurrenceRule(rrule);
            // recurrence_rule being non-null is what makes isRecurring=true
        }

        return entity;
    }

    // For exception instances (edited or cancelled occurrences)
    // parentEntity must already be persisted (has a UUID)
    public CalendarEvent toExceptionEntity(Event googleEvent, GoogleSyncedCalendar syncedCal,
                                           CalendarEvent parentEntity) {
        CalendarEvent entity = new CalendarEvent();

        // Same common fields as toEntity
        entity.setTitle(googleEvent.getSummary());
        entity.setDescription(googleEvent.getDescription());
        entity.setLocation(googleEvent.getLocation());
        entity.setHtmlLink(googleEvent.getHtmlLink());
        entity.setGoogleEventId(googleEvent.getId());
        entity.setEtag(googleEvent.getEtag());
        entity.setGoogleUpdatedAt(/* parse googleEvent.getUpdated() */);
        entity.setSource(EventSource.GOOGLE);
        entity.setMember(syncedCal.getMember());
        entity.setFamily(syncedCal.getMember().getFamily());

        // Date/time mapping
        mapDateAndTime(googleEvent, entity);

        // Exception fields
        entity.setRecurringEvent(parentEntity);  // FK to our parent row
        entity.setOriginalDate(parseOriginalDate(googleEvent));  // from getOriginalStartTime()
        entity.setCancelled("cancelled".equals(googleEvent.getStatus()));

        return entity;
    }

    private void mapDateAndTime(Event googleEvent, CalendarEvent entity) {
        if (googleEvent.getStart().getDate() != null) {
            // All-day event
            entity.setAllDay(true);
            entity.setDate(parseLocalDate(googleEvent.getStart().getDate()));
            entity.setStartTime(LocalTime.MIDNIGHT);  // 00:00 — matches our convention
            entity.setEndTime(LocalTime.MIDNIGHT);     // 00:00 — both are MIDNIGHT for all-day

            // Multi-day: Google end date is EXCLUSIVE, ours is INCLUSIVE
            LocalDate googleEndDate = parseLocalDate(googleEvent.getEnd().getDate());
            LocalDate inclusiveEndDate = googleEndDate.minusDays(1);
            if (inclusiveEndDate.isAfter(entity.getDate())) {
                entity.setEndDate(inclusiveEndDate);
            }
            // Single-day: endDate stays null
        } else {
            // Timed event
            entity.setAllDay(false);
            DateTime startDateTime = googleEvent.getStart().getDateTime();
            DateTime endDateTime = googleEvent.getEnd().getDateTime();
            // Parse Google DateTime to java.time types
            // startTime/endTime are LocalTime fields on the entity (NOT strings).
            // The CalendarEventMapper handles string formatting at the DTO layer using
            // DateTimeFormatter.ofPattern("h:mm a", Locale.US) — e.g., "9:00 AM", "2:30 PM"
            entity.setDate(/* LocalDate from startDateTime */);
            entity.setStartTime(/* LocalTime from startDateTime */);
            entity.setEndTime(/* LocalTime from endDateTime */);
        }
    }

    private LocalDate parseOriginalDate(Event googleEvent) {
        // event.getOriginalStartTime().getDate() for all-day
        // event.getOriginalStartTime().getDateTime() for timed events
        // Extract the LocalDate either way
    }
}
```

**Important time type note:** `startTime` and `endTime` on the `CalendarEvent` entity are `LocalTime` fields, NOT strings. The `CalendarEventMapper` converts to/from `"h:mm a"` formatted strings (e.g., `"9:00 AM"`, `"2:30 PM"`) only at the DTO boundary using `DateTimeFormatter.ofPattern("h:mm a", Locale.US)`. The `GoogleEventMapper` must set `LocalTime` values directly on the entity.

**Important all-day time note:** All-day events use `LocalTime.MIDNIGHT` (00:00) for both `startTime` and `endTime`. The existing mapper test confirms this — all-day events are created with `"12:00 AM"` / `"12:00 AM"` strings which parse to `LocalTime.MIDNIGHT`. The validation in `CalendarEventService` explicitly allows `startTime == endTime` when `isAllDay` is true.

### 7. New service: `GoogleCalendarSyncService`

Full sync only in this PR (incremental sync is PR 5).

```java
@Service
public class GoogleCalendarSyncService {

    // Triggered after calendar selection, or manually via POST /api/google/sync/{memberId}
    public void fullSync(GoogleSyncedCalendar syncedCal) {
        Credential credential = googleCredentialService.getCredential(syncedCal.getMember().getId());
        Calendar service = new Calendar.Builder(transport, jsonFactory, credential)
            .setApplicationName("FamilyHub")
            .build();

        // Fetch all events with singleEvents=false
        String pageToken = null;
        List<Event> allEvents = new ArrayList<>();

        do {
            Events response = service.events().list(syncedCal.getGoogleCalendarId())
                .setSingleEvents(false)
                .setMaxResults(250)
                .setPageToken(pageToken)
                .execute();

            if (response.getItems() != null) {
                allEvents.addAll(response.getItems());
            }
            pageToken = response.getNextPageToken();

            if (pageToken == null && response.getNextSyncToken() != null) {
                syncedCal.setSyncToken(response.getNextSyncToken());
            }
        } while (pageToken != null);

        // Delete existing Google events for this member
        // (deleteByMemberAndSource already exists in CalendarEventRepository)
        calendarEventRepository.deleteByMemberAndSource(syncedCal.getMember(), EventSource.GOOGLE);

        // Separate parents/regular from exceptions
        List<Event> parentsAndRegular = allEvents.stream()
            .filter(e -> e.getRecurringEventId() == null)
            .filter(e -> !"cancelled".equals(e.getStatus()))
            .toList();

        List<Event> exceptions = allEvents.stream()
            .filter(e -> e.getRecurringEventId() != null)
            .toList();

        // Save parents/regular first
        List<CalendarEvent> parentEntities = parentsAndRegular.stream()
            .map(e -> googleEventMapper.toEntity(e, syncedCal))
            .toList();
        calendarEventRepository.saveAll(parentEntities);
        calendarEventRepository.flush();  // ensure UUIDs are assigned before exception FK resolution

        // Save exceptions — resolve Google parent ID to our UUID
        for (Event exception : exceptions) {
            String googleParentId = exception.getRecurringEventId();
            CalendarEvent parentEntity = calendarEventRepository
                .findByGoogleEventId(googleParentId)
                .orElse(null);

            if (parentEntity == null) {
                // Orphaned exception — parent wasn't returned or was cancelled
                // Skip silently or log a warning
                continue;
            }

            CalendarEvent exceptionEntity = googleEventMapper
                .toExceptionEntity(exception, syncedCal, parentEntity);
            calendarEventRepository.save(exceptionEntity);
        }

        // Update sync metadata
        syncedCal.setLastSyncedAt(LocalDateTime.now());
        googleSyncedCalendarRepository.save(syncedCal);
    }

    // Sync all enabled calendars for a member
    public void syncMember(UUID memberId) {
        List<GoogleSyncedCalendar> calendars = googleSyncedCalendarRepository
            .findByMemberIdAndEnabledTrue(memberId);
        for (GoogleSyncedCalendar cal : calendars) {
            fullSync(cal);
        }
    }
}
```

### 8. Controller: Manual sync endpoint

Add to `GoogleCalendarController` (or `GoogleOAuthController` — wherever it fits better):

```java
// POST /api/google/sync/{memberId}
// Triggers a manual full sync for the member
// Validate: memberId belongs to authenticated family
// Validate: member is connected and has selected calendars
// Returns: 200 OK with message
```

### 9. Auto-sync on calendar selection

In the calendar selection flow (PUT /api/google/calendars/{memberId}), trigger a full sync after updating selections. This way, events appear immediately after the user selects calendars — they don't have to wait for the scheduled sync or manually trigger one.

After `GoogleCalendarSelectionService.updateCalendarSelections()` completes, call `GoogleCalendarSyncService.syncMember(memberId)` to full-sync all enabled calendars. This is simpler than tracking which calendars transitioned state, and correct because `fullSync()` does delete-then-reinsert anyway — re-syncing an already-synced calendar is idempotent.

### 10. Update `GET /calendar/events` expansion

Verify that the existing `CalendarEventService.getEvents()` expansion logic handles Google recurring parents correctly. Since Google parents are stored with `recurrence_rule` set and `source=GOOGLE`, they should already be picked up by `findRecurringParentsByFamily()` and expanded by `RecurrenceExpander`.

Things to verify:
- Google parents with RRULEs our expander doesn't support — should not crash. Wrap the `recurrenceExpander.expand()` call in `CalendarEventService.getEventsWithExpansion()` in a try/catch per parent and log a warning if iCal4j throws. Keep the `RecurrenceExpander` itself clean — error policy belongs in the caller.
- Google parents are included in the expansion loop alongside native parents
- `toInstanceResponse()` correctly copies `source` and `htmlLink` from the parent

## Key risks and what to do

**RRULE subset compatibility:** Google users may have events with `YEARLY`, `COUNT`, `BYSETPOS`, etc. iCal4j should handle these, but if it doesn't, wrap the expansion per-parent in a try/catch, log a warning, and skip that event. Do NOT let one bad RRULE crash the entire GET. Report back what you find.

**`singleEvents=false` behavior:** Verify that Google actually returns both parents AND exceptions in the same `events.list()` response. If it doesn't (e.g., exceptions are only returned with `singleEvents=true`), report back immediately — this would require a design change.

**Time types:** Our `startTime`/`endTime` are `LocalTime` fields on the entity (not strings). Google returns `DateTime` objects. Parse Google's `DateTime` to `LocalTime` directly. The `CalendarEventMapper` handles string formatting at the DTO layer — the `GoogleEventMapper` should never produce formatted time strings.

## Tests

- **Unit tests:** `GoogleEventMapper` — test all three mapping paths:
  - Regular timed event → correct date/time/source
  - All-day single-day event → `isAllDay=true`, `endDate=null`
  - All-day multi-day event → `isAllDay=true`, `endDate` set (Google exclusive → our inclusive, minus one day)
  - Recurring parent → `recurrenceRule` set, no `recurringEvent`
  - Exception instance → `recurringEvent` FK set, `originalDate` set
  - Cancelled exception → `isCancelled=true`

- **Unit tests:** `GoogleCalendarSyncService` — mock the Google Calendar API client:
  - Full sync with mix of regular + recurring + exception events
  - Verify parents saved before exceptions (FK resolution works)
  - Verify orphaned exceptions are skipped gracefully
  - Verify `deleteByMemberAndSource` called before insert

- **Integration test:** Insert Google events via sync service (mocked Google client), then call `GET /calendar/events` and verify:
  - Google regular events appear in response
  - Google recurring parents are expanded into instances
  - Google exceptions replace computed instances
  - Google cancelled exceptions are excluded
  - Native events still appear alongside Google events
  - `source` field is correct on all events
  - `htmlLink` is present on Google events

- **Controller test:** `POST /api/google/sync/{memberId}` — authorization checks, member must be connected

## What NOT to build

- No incremental sync (PR 5)
- No scheduled sync job (PR 5)
- No FE changes

## Branch & PR

- Branch: `feat/google-cal-full-sync`
- Atomic commits, open a PR when done
- Reference the design doc and prior PRs in the description

## Acceptance criteria

- [ ] V9 migration adds `google_event_id`, `html_link`, `etag`, `google_updated_at` columns with partial unique index
- [ ] `CalendarEventResponse` includes `htmlLink`
- [ ] `GoogleEventMapper` correctly maps regular, all-day, multi-day, recurring parent, exception, and cancelled events
- [ ] All-day multi-day: Google exclusive end date → our inclusive end date (minus one day)
- [ ] Recurring parents stored with `recurrence_rule` from Google's RRULE
- [ ] Exceptions stored with `recurring_event_id` FK pointing to our parent row
- [ ] `fullSync()` fetches with `singleEvents=false`, saves parents first, then exceptions
- [ ] `POST /api/google/sync/{memberId}` triggers manual sync
- [ ] Calendar selection triggers auto-sync for newly enabled calendars
- [ ] `GET /calendar/events` expands Google recurring parents alongside native ones
- [ ] RRULE expansion errors are caught per-parent (don't crash the whole GET)
- [ ] Orphaned exceptions (no matching parent) are skipped gracefully
- [ ] Tests cover the above (Google API calls mocked in tests)
