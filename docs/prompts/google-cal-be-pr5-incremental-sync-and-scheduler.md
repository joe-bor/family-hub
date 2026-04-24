# BE PR 5: Incremental Sync + Scheduled Sync Job

## Context

This is the fifth and final BE PR in the Google Calendar integration series.
- PR 1 ([#22](https://github.com/joe-bor/family-hub-api/pull/22)): `source`/`description` columns, Google event protection
- PR 2 ([#23](https://github.com/joe-bor/family-hub-api/pull/23)): OAuth flow, token encryption
- PR 3 ([#24](https://github.com/joe-bor/family-hub-api/pull/24)): Calendar selection, credential service
- PR 4 ([#25](https://github.com/joe-bor/family-hub-api/pull/25)): Full sync, event mapping, RRULE/EXDATE storage

PR 4 gave us `fullSync` — delete-then-reinsert all events. This PR adds:
1. **Incremental sync** — use Google's `syncToken` to fetch only changes since last sync
2. **Scheduled job** — run incremental sync every 15 minutes for all connected members

Design doc: `docs/google-calendar-integration-design.md` (§6 Sync Logic — incremental sync pseudocode, §6 Scheduled Sync)

## What exists already

Look at `GoogleCalendarSyncService.java` before starting. Key things already in place:
- `fetchAllEvents()` already stores `syncToken` on the `GoogleSyncedCalendar` entity during full sync
- `syncMember(UUID memberId)` orchestrates fetch + persist with abort-on-failure pattern
- `persistSyncedEvents()` is `@Transactional`, separated from network I/O
- `buildCalendarClient()` builds a Google Calendar API client for a member
- `GoogleSyncedCalendarRepository` has `findByMemberIdAndEnabledTrue()`
- `CalendarEventRepository` has `findByGoogleEventId()` and `deleteByGoogleEventId()`

## What to build

### 1. Incremental sync method in `GoogleCalendarSyncService`

Add an `incrementalSync` method that uses the stored `syncToken`:

```java
// Fetches only changes since last sync for a single calendar
List<Event> fetchIncrementalEvents(GoogleSyncedCalendar syncedCal, Calendar calendarClient) throws IOException {
    List<Event> changedEvents = new ArrayList<>();
    String pageToken = null;

    do {
        Events response = calendarClient.events().list(syncedCal.getGoogleCalendarId())
            .setSyncToken(syncedCal.getSyncToken())  // key difference from fullSync
            .setPageToken(pageToken)
            .execute();

        if (response.getItems() != null) {
            changedEvents.addAll(response.getItems());
        }
        pageToken = response.getNextPageToken();

        if (pageToken == null && response.getNextSyncToken() != null) {
            syncedCal.setSyncToken(response.getNextSyncToken());
        }
    } while (pageToken != null);

    return changedEvents;
}
```

**Key difference from `fetchAllEvents`:** Uses `syncToken` instead of `setSingleEvents(false)`. Google returns only events that changed since the last sync — new, updated, or cancelled.

### 2. Persist incremental changes

Incremental sync does NOT delete-then-reinsert. It upserts individual events:

```java
@Transactional
void persistIncrementalChanges(GoogleSyncedCalendar syncedCal, List<Event> changedEvents) {
    // Separate parents/regular from exceptions — process parents first
    List<Event> parentChanges = changedEvents.stream()
        .filter(e -> e.getRecurringEventId() == null)
        .toList();
    List<Event> exceptionChanges = changedEvents.stream()
        .filter(e -> e.getRecurringEventId() != null)
        .toList();

    // Process parent/regular changes
    for (Event event : parentChanges) {
        if ("cancelled".equals(event.getStatus())) {
            // Delete parent — FK CASCADE deletes its exceptions too
            calendarEventRepository.deleteByGoogleEventId(event.getId());
        } else {
            upsertEvent(event, syncedCal);
        }
    }

    // Process exception changes
    for (Event event : exceptionChanges) {
        if ("cancelled".equals(event.getStatus())) {
            // Option A: Store as is_cancelled exception row (matches full sync behavior)
            // Option B: Delete the exception row if it exists
            // Go with Option A — consistent with how full sync handles it,
            // and RecurrenceExpander already skips cancelled exceptions
            upsertException(event, syncedCal);
        } else {
            upsertException(event, syncedCal);
        }
    }

    syncedCal.setLastSyncedAt(Instant.now());
    syncedCalendarRepository.save(syncedCal);
}

private void upsertEvent(Event googleEvent, GoogleSyncedCalendar syncedCal) {
    CalendarEvent entity = googleEventMapper.toEntity(googleEvent, syncedCal);
    calendarEventRepository.findByGoogleEventId(googleEvent.getId())
        .ifPresentOrElse(
            existing -> updateExistingEvent(existing, entity),
            () -> calendarEventRepository.save(entity)
        );
}

private void upsertException(Event googleEvent, GoogleSyncedCalendar syncedCal) {
    String googleParentId = googleEvent.getRecurringEventId();
    CalendarEvent parent = calendarEventRepository
        .findByGoogleEventId(googleParentId)
        .orElse(null);
    if (parent == null) {
        log.warn("Orphaned exception {} — parent {} not found, skipping",
            googleEvent.getId(), googleParentId);
        return;
    }
    CalendarEvent entity = googleEventMapper.toExceptionEntity(googleEvent, syncedCal, parent);
    calendarEventRepository.findByGoogleEventId(googleEvent.getId())
        .ifPresentOrElse(
            existing -> updateExistingEvent(existing, entity),
            () -> calendarEventRepository.save(entity)
        );
}

private void updateExistingEvent(CalendarEvent existing, CalendarEvent updated) {
    // Copy all mutable fields from updated to existing
    // (existing has the correct ID — we're updating in place)
    existing.setTitle(updated.getTitle());
    existing.setDescription(updated.getDescription());
    existing.setLocation(updated.getLocation());
    existing.setDate(updated.getDate());
    existing.setEndDate(updated.getEndDate());
    existing.setStartTime(updated.getStartTime());
    existing.setEndTime(updated.getEndTime());
    existing.setAllDay(updated.isAllDay());
    existing.setHtmlLink(updated.getHtmlLink());
    existing.setEtag(updated.getEtag());
    existing.setGoogleUpdatedAt(updated.getGoogleUpdatedAt());
    existing.setRecurrenceRule(updated.getRecurrenceRule());
    existing.setExdates(updated.getExdates());
    existing.setCancelled(updated.isCancelled());
    // Don't update: id, googleEventId, source, member, family, syncedCalendar, recurringEvent
    calendarEventRepository.save(existing);
}
```

### 3. Handle 410 Gone (sync token expired)

When Google returns `410 Gone`, the sync token is stale. Fall back to full sync:

```java
try {
    List<Event> changes = fetchIncrementalEvents(syncedCal, calendarClient);
    persistIncrementalChanges(syncedCal, changes);
} catch (GoogleJsonResponseException e) {
    if (e.getStatusCode() == 410) {
        log.warn("Sync token expired for calendar {} — falling back to full sync",
            syncedCal.getGoogleCalendarId());
        syncedCal.setSyncToken(null);  // clear stale token
        fullSync(syncedCal, calendarClient);  // existing method
    } else {
        throw e;
    }
}
```

### 4. Update `syncMember` to use incremental when possible

The current `syncMember` always does a full sync. Update the logic:
- If a calendar has a `syncToken` → try incremental sync
- If a calendar has no `syncToken` (first sync, or token cleared after 410) → full sync

The abort-on-partial-failure pattern from PR 4 applies to full sync (where we delete-then-reinsert). For incremental sync, per-calendar errors can be handled independently — a failed incremental sync for one calendar doesn't affect another since we're upserting, not deleting.

```java
public void syncMember(UUID memberId) {
    List<GoogleSyncedCalendar> calendars = findEnabledCalendars(memberId);
    Calendar calendarClient = buildCalendarClient(memberId);

    for (GoogleSyncedCalendar cal : calendars) {
        try {
            if (cal.getSyncToken() != null) {
                incrementalSync(cal, calendarClient);
            } else {
                fullSync(cal, calendarClient);
            }
        } catch (Exception e) {
            log.error("Sync failed for calendar {}: {}", cal.getGoogleCalendarId(), e.getMessage());
            // Continue syncing other calendars — don't abort
        }
    }
}
```

**Note:** This changes the abort-on-failure behavior from PR 4. With incremental sync, per-calendar isolation is safe because we're upserting, not deleting. For full sync (which does delete-then-reinsert), the existing `fullSync` method already handles it per-calendar via `deleteBySyncedCalendarAndSource`. Think about whether the abort pattern still makes sense or if per-calendar error handling is better for both modes. Use your judgment — the key requirement is: never lose events due to a partial failure.

### 5. Scheduled sync job: `GoogleCalendarSyncScheduler`

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class GoogleCalendarSyncScheduler {

    private final GoogleOAuthTokenRepository tokenRepository;
    private final GoogleCalendarSyncService syncService;

    @Scheduled(fixedRate = 900000)  // 15 minutes
    public void syncAllConnectedMembers() {
        List<GoogleOAuthToken> tokens = tokenRepository.findAll();
        log.info("Scheduled sync: {} connected members", tokens.size());

        for (GoogleOAuthToken token : tokens) {
            try {
                syncService.syncMember(token.getMember().getId());
            } catch (Exception e) {
                log.error("Scheduled sync failed for member {}: {}",
                    token.getMember().getId(), e.getMessage());
                // Continue to next member
            }
        }
    }
}
```

**Important:** Make sure `@EnableScheduling` is present on the application class or a config class. Check if it's already there — it may have been added for another feature.

**Important:** `syncMember` is `@Async` — the scheduler fires it and moves on to the next member. Multiple members sync in parallel. Verify the async executor has enough threads (Spring's default `SimpleAsyncTaskExecutor` creates a new thread per call, which is fine for a few members but worth noting).

### 6. Conditional scheduling

The scheduler should not run if Google Calendar is not configured (no `GOOGLE_CLIENT_ID`). Use `@ConditionalOnProperty` or check in the method:

```java
@Scheduled(fixedRate = 900000)
public void syncAllConnectedMembers() {
    if (!googleOAuthConfig.isConfigured()) {
        return;  // Google Calendar not set up — skip
    }
    // ...
}
```

Or add an `isConfigured()` method to `GoogleOAuthConfig` that returns `true` when `clientId` is non-empty. This prevents noisy "0 connected members" logs on every dev machine that doesn't have Google credentials.

### 7. Update disconnect to handle incremental sync state

When a member disconnects (`DELETE /api/google/disconnect/{memberId}`), verify that:
- Synced calendar rows are deleted (already handled via CASCADE)
- Google events are deleted (already handled in `GoogleOAuthService.disconnect()`)
- The scheduler gracefully handles a member disappearing mid-sync (the `try/catch` per member in the scheduler handles this)

No new code needed — just verify the existing disconnect flow still works with the scheduler running.

## Tests

- **Unit tests:** `GoogleCalendarSyncService`
  - Incremental sync: upsert new event, update existing event, delete cancelled event
  - Incremental sync: parent changes processed before exceptions
  - 410 Gone: falls back to full sync, clears sync token
  - Orphaned exceptions in incremental sync: skipped gracefully

- **Unit tests:** `GoogleCalendarSyncScheduler`
  - Iterates all connected members and calls `syncMember`
  - One member's failure doesn't stop others
  - Skips when Google is not configured

- **Integration test:**
  - Full sync → verify sync token stored → incremental sync with changes → verify upserts applied correctly
  - Incremental sync deletes a parent → verify exceptions cascade-deleted

## What NOT to build

- No FE changes (FE PRs come after all BE work)
- No webhook/push notifications (Phase 2b)
- No write-back (Phase 2a)

## Branch & PR

- Branch: `feat/google-cal-incremental-sync`
- Atomic commits, open a PR when done
- Reference the design doc and prior PRs in the description

## Acceptance criteria

- [ ] Incremental sync uses stored `syncToken` to fetch only changes
- [ ] New events are inserted, updated events are upserted, cancelled events are deleted/marked
- [ ] Parent changes are processed before exceptions (FK resolution)
- [ ] 410 Gone falls back to full sync and clears stale sync token
- [ ] Orphaned exceptions are skipped gracefully
- [ ] `@Scheduled` job runs every 15 minutes for all connected members
- [ ] One member's sync failure doesn't block others
- [ ] Scheduler is silent when Google credentials are not configured
- [ ] `syncMember` uses incremental sync when sync token exists, full sync otherwise
- [ ] Disconnect still works correctly with scheduler running
- [ ] Tests cover the above
