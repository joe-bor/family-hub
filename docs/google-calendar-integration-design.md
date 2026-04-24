# Google Calendar Integration — Design Document

**Date:** 2026-03-07
**Status:** Draft — All prerequisites complete. Ready for implementation planning.
**Scope:** Phase 1 — Read-only sync
**Prerequisite reading:** [google-calendar-research.md](./google-calendar-research.md), [diagrams/google-calendar-oauth-flow.mmd](./diagrams/google-calendar-oauth-flow.mmd)
**Depends on:** [all-day-and-multi-day-events-design.md](./all-day-and-multi-day-events-design.md) (complete), [recurring-events-design.md](./recurring-events-design.md) (complete — BE [PR #20](https://github.com/joe-bor/family-hub-api/pull/20)/[#21](https://github.com/joe-bor/family-hub-api/pull/21), FE [PR #117](https://github.com/joe-bor/FamilyHub/pull/117))

---

## 1. Overview

Add Google Calendar integration to FamilyHub so family members can connect their Google accounts and see their Google Calendar events rendered alongside native FamilyHub events. Events from both sources are visually unified — same colors, same calendar grid — with a small Google icon to differentiate source.

### Goals

- Each family member can connect their own Google Calendar via OAuth
- Google events are displayed in FamilyHub with the member's assigned color
- Sync runs automatically every 15 minutes using Google's sync token mechanism
- Members can choose which of their Google calendars to sync
- Disconnecting removes all Google-sourced events cleanly
- The existing calendar UI and API contract change minimally

### Non-Goals (Phase 1)

- Writing events back to Google Calendar (Phase 2+)
- Real-time push via webhooks (Phase 2, alongside WebSocket work)
- ~~Parsing or editing recurring event rules~~ **Revised.** Google recurring events are now stored as parent rows with RRULE and expanded by our `RecurrenceExpander` — same as native recurring events. See [Recurring Events design](./recurring-events-design.md) and [§6 Event Mapping](#event-mapping-logic) below.
- Google Calendar as source of truth (FamilyHub DB is source of truth)

---

## 2. Architecture

```
User → FE Settings → "Connect Google Calendar" button
  → BE builds OAuth URL → redirect to Google consent
  → Google callback → BE exchanges code for tokens
  → BE stores encrypted tokens in DB
  → BE triggers initial full sync (singleEvents=false)
  → Google recurring parents stored with RRULE (source=GOOGLE)
  → Google exceptions stored as exception rows (recurring_event_id → parent)
  → Google regular events stored as regular rows (source=GOOGLE)
  → FE fetches events as usual (GET /api/calendar/events?startDate=...&endDate=...)
  → BE expands ALL recurring events (native + Google) via RecurrenceExpander
  → FE receives flat list — native and Google events rendered on calendar

@Scheduled (every 15 min):
  → For each connected member:
    → Refresh access token if expired
    → Incremental sync via syncToken
    → Upsert/delete parents first, then exceptions
```

See [diagrams/google-calendar-oauth-flow.mmd](./diagrams/google-calendar-oauth-flow.mmd) for the full sequence diagram.

---

## 3. OAuth 2.0 Flow

### Google Cloud Console Setup

1. Create OAuth 2.0 credentials (type: **Web application**)
2. Set authorized redirect URI: `https://familyhub.joe-bor.me/api/google/callback`
3. Enable **Google Calendar API**
4. Configure OAuth consent screen (External, Testing mode — 100 user limit is fine)

### Scopes

- `https://www.googleapis.com/auth/calendar.events.readonly` — read events
- `https://www.googleapis.com/auth/calendar.calendarlist.readonly` — list available calendars

### Flow Steps

1. **FE** → `GET /api/google/auth?memberId={uuid}`
2. **BE** builds Google OAuth URL with `client_id`, `redirect_uri`, `scope`, `state=memberId`, `access_type=offline`, `prompt=consent`
3. **BE** returns the URL → **FE** redirects browser to Google
4. User consents → Google redirects to `GET /api/google/callback?code=AUTH_CODE&state=memberId`
5. **BE** exchanges code for tokens via `POST /oauth2/token` (server-to-server, `client_secret` never in browser)
6. **BE** encrypts and stores tokens in `google_oauth_token` table
7. **BE** triggers initial full sync
8. **BE** redirects to FE settings page with success indicator

### Token Management

- **Access token:** Expires in ~1 hour. Google Java client auto-refreshes using stored refresh token.
- **Refresh token:** Long-lived, only returned on first consent (or when `prompt=consent` is forced). Must be stored encrypted.
- **Encryption:** AES-256 via Spring's `TextEncryptor` or equivalent. Encryption key from environment variable.

### Critical Gotchas

- Use `access_type=offline` to get a refresh token
- Use `prompt=consent` to ALWAYS get a refresh token (Google only returns it on first consent otherwise)
- Use OAuth client type "Web application" (not "Desktop" or "Installed")
- Store refresh tokens encrypted at rest

---

## 4. Data Model Changes

### New Table: `google_oauth_token`

```sql
CREATE TABLE google_oauth_token (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    member_id UUID NOT NULL UNIQUE REFERENCES family_member(id) ON DELETE CASCADE,
    access_token TEXT NOT NULL,         -- encrypted
    refresh_token TEXT NOT NULL,        -- encrypted
    token_expiry TIMESTAMP NOT NULL,
    scope TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now(),
    updated_at TIMESTAMP NOT NULL DEFAULT now()
);
```

One row per connected member. `UNIQUE` on `member_id` — a member has at most one Google connection. `ON DELETE CASCADE` ensures cleanup when member is deleted.

### New Table: `google_synced_calendar`

```sql
CREATE TABLE google_synced_calendar (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    token_id UUID NOT NULL REFERENCES google_oauth_token(id) ON DELETE CASCADE,
    member_id UUID NOT NULL REFERENCES family_member(id) ON DELETE CASCADE,
    google_calendar_id VARCHAR(255) NOT NULL,    -- "primary" or specific ID
    calendar_name VARCHAR(255),                   -- display name for UI
    sync_token TEXT,                               -- Google's nextSyncToken
    last_synced_at TIMESTAMP,
    enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT now()
);
```

One row per synced calendar. A member can sync multiple calendars (primary, custom, shared). Each has its own sync token.

### Extend Existing Table: `calendar_event`

Add columns to the existing table:

```sql
ALTER TABLE calendar_event
    ADD COLUMN source VARCHAR(10) NOT NULL DEFAULT 'NATIVE',    -- 'NATIVE' or 'GOOGLE'
    ADD COLUMN description TEXT,
    ADD COLUMN google_event_id VARCHAR(1024),                    -- Google's opaque event ID
    ADD COLUMN html_link VARCHAR(2048),                          -- link to open in Google Calendar
    ADD COLUMN etag VARCHAR(255),                                -- for sync conflict detection
    ADD COLUMN google_updated_at TIMESTAMPTZ,                    -- Google's last-modified timestamp
    ADD COLUMN synced_calendar_id UUID REFERENCES google_synced_calendar(id) ON DELETE SET NULL,
    ADD COLUMN exdates TEXT;                                      -- comma-separated ISO dates (from EXDATE)

CREATE UNIQUE INDEX idx_calendar_event_google_id
    ON calendar_event(google_event_id)
    WHERE google_event_id IS NOT NULL;
```

- `source`: Discriminator. `NATIVE` for events created in FamilyHub, `GOOGLE` for synced events.
- `description`: Benefits both sources. Google events have descriptions; native events gain a notes field.
- `google_event_id`: Unique within Google events. Used for upsert during sync.
- `html_link`: "Open in Google Calendar" action in the event detail modal.
- `etag`: Optimistic concurrency for future write-back.
- `google_updated_at`: Change detection during sync.
- `synced_calendar_id`: FK to `google_synced_calendar` — tracks which Google calendar each event came from. Enables per-calendar deletes during sync (added in BE [PR #25](https://github.com/joe-bor/family-hub-api/pull/25)).
- `exdates`: Comma-separated ISO dates extracted from Google's `EXDATE` recurrence entries. `RecurrenceExpander` filters these dates during expansion. Separate from exception rows — EXDATE is Google's way of excluding dates without creating a full exception event (added in BE [PR #25](https://github.com/joe-bor/family-hub-api/pull/25)).

**Native events** have `source='NATIVE'`, all Google-specific columns are NULL.
**Google events** have `source='GOOGLE'`, `google_event_id` is NOT NULL.

### Flyway Migration

Spread across multiple migrations as features were delivered incrementally:

| Migration | PR | Contents |
|-----------|-----|---------|
| V4 | #22 | `source`, `description` columns on `calendar_event` |
| V5 | #23 | `CREATE TABLE google_oauth_token` |
| V6 | #23 | Standardize `family` timestamps to `TIMESTAMPTZ` |
| V7 | #24 | `CREATE TABLE google_synced_calendar` |
| V8 | #24 | Unique constraint on `(member_id, google_calendar_id)` |
| V9 | #25 | `google_event_id`, `html_link`, `etag`, `google_updated_at` columns + partial unique index |
| V10 | #25 | `synced_calendar_id` FK, `exdates` column |

> **Note:** V2 is `add_end_date_column` (Phase B) and V3 is `add_recurrence_columns` (recurring events).

---

## 5. BE API Endpoints

### New Endpoints

```
# OAuth Connection
GET    /api/google/auth?memberId={uuid}          → Returns Google OAuth redirect URL
GET    /api/google/callback?code=...&state=...   → Handles OAuth callback, stores tokens, triggers sync
DELETE /api/google/disconnect/{memberId}          → Revokes tokens, deletes Google events for member

# Connection Status
GET    /api/google/status/{memberId}             → {connected, lastSyncedAt, calendars[]}

# Calendar Selection
GET    /api/google/calendars/{memberId}          → Lists available Google calendars
PUT    /api/google/calendars/{memberId}          → Sets which calendars to sync (array of calendar IDs)

# Manual Sync
POST   /api/google/sync/{memberId}              → Triggers manual sync for one member
```

All endpoints require JWT authentication. The `memberId` must belong to the authenticated family.

### Modified Endpoints

**`GET /api/calendar/events?startDate=...&endDate=...`** — `startDate` and `endDate` are now **required** query params (changed in recurring events, BE [PR #20](https://github.com/joe-bor/family-hub-api/pull/20)). Response includes both native and Google events. Google recurring parents are expanded by our `RecurrenceExpander` within the query window — same as native recurring events. New optional fields in response:

```json
{
  "id": "uuid",                                    // null for expanded instances (native or Google)
  "title": "Doctor Appointment",
  "startTime": "2:00 PM",
  "endTime": "3:00 PM",
  "date": "2026-03-15",
  "endDate": "2026-03-15",                         // nullable, for multi-day all-day events
  "memberId": "uuid",
  "isAllDay": false,
  "location": "123 Main St",
  "isRecurring": false,                            // true for recurring parents/instances (native or Google)
  "recurrenceRule": null,                          // RRULE string on parents + expanded instances
  "recurringEventId": null,                        // parent UUID for instances/exceptions
  "source": "GOOGLE",                             // NEW — "NATIVE" or "GOOGLE"
  "description": "Annual checkup",                 // NEW — nullable
  "htmlLink": "https://calendar.google.com/..."    // NEW — nullable, Google events only
}
```

**`POST /api/calendar/events`** and **`PUT /api/calendar/events/{id}`** — Accept optional `description` field. `source` is always `NATIVE` for user-created events (not settable via API).

### Google Event Protection

Google-sourced events are **read-only** in Phase 1. The BE must reject `PUT` and `DELETE` requests for events where `source = 'GOOGLE'` with a `400 Bad Request` ("Google Calendar events cannot be modified in FamilyHub. Edit them in Google Calendar.").

---

## 6. BE Service Architecture

### New Classes

```
config/
  GoogleOAuthConfig.java              -- client_id, client_secret, redirect_uri, scopes (from env)

controller/
  GoogleOAuthController.java          -- /api/google/auth, /callback, /disconnect
  GoogleCalendarController.java       -- /api/google/status, /calendars, /sync

service/
  GoogleOAuthService.java             -- OAuth URL building, code exchange, token storage/refresh
  GoogleCalendarSyncService.java      -- Full sync, incremental sync, event mapping
  GoogleCredentialService.java        -- Builds Credential objects from stored tokens
  TokenEncryptionService.java         -- AES-256 encrypt/decrypt for token storage

mapper/
  GoogleEventMapper.java              -- Google Event <-> CalendarEvent entity mapping
                                      -- toEntity() for parents/regular, toExceptionEntity() for exceptions

model/
  GoogleOAuthToken.java               -- JPA entity
  GoogleSyncedCalendar.java           -- JPA entity
  EventSource.java                    -- Enum: NATIVE, GOOGLE

repository/
  GoogleOAuthTokenRepository.java     -- findByMemberId, findAllByFamilyId
  GoogleSyncedCalendarRepository.java -- findByTokenId, findEnabledByMemberId

scheduler/
  GoogleCalendarSyncScheduler.java    -- @Scheduled every 15 min, iterates connected members
```

### Event Mapping Logic

Google events fall into three categories: regular (non-recurring), recurring parents, and exceptions. The mapper handles each differently.

```
Google Event → CalendarEvent:

  # Common fields (all event types)
  title          ← event.getSummary()
  description    ← event.getDescription()
  location       ← event.getLocation()
  htmlLink       ← event.getHtmlLink()
  googleEventId  ← event.getId()
  etag           ← event.getEtag()
  googleUpdatedAt ← event.getUpdated()
  source         ← GOOGLE
  memberId       ← the FamilyMember who owns the synced calendar
  familyId       ← that member's family

  # Date/time mapping
  IF event.getStart().getDate() != null:      // all-day event
    isAllDay   ← true
    date       ← parseLocalDate(event.getStart().getDate())
    startTime  ← LocalTime.MIDNIGHT (or 00:00)
    endTime    ← LocalTime.MIDNIGHT
    endDate    ← parseLocalDate(event.getEnd().getDate()).minusDays(1)
                  // Google end date is EXCLUSIVE, ours is INCLUSIVE
                  // Single-day: endDate == date → store as null (same as native)
                  // Multi-day: endDate > date → store the inclusive end date
  ELSE:                                        // timed event
    isAllDay   ← false
    dateTime   ← parseZonedDateTime(event.getStart().getDateTime())
    date       ← dateTime.toLocalDate()
    startTime  ← dateTime.toLocalTime()
    endTime    ← parse(event.getEnd().getDateTime()).toLocalTime()
    endDate    ← null (timed events don't span days)

  # Recurrence mapping (singleEvents=false)
  IF event.getRecurrence() != null:           // recurring parent
    recurrenceRule    ← extract "RRULE:" entry, strip prefix
                        // Google returns "RRULE:FREQ=WEEKLY;BYDAY=MO"
                        // iCal4j expects "FREQ=WEEKLY;BYDAY=MO" (no prefix)
    exdates           ← extract "EXDATE" entries, parse to ISO dates, comma-join
                        // Google may include: "EXDATE;VALUE=DATE:20250617",
                        // "EXDATE:20250617T130000Z", "EXDATE;TZID=...:20250617T090000"
                        // All parsed to "2025-06-17" format
    isRecurring       ← true (derived from recurrenceRule being non-null)
    recurringEventId  ← null (this IS the parent)
  ELSE IF event.getRecurringEventId() != null: // exception instance
    recurrenceRule    ← null
    recurringEventId  ← look up our UUID for the parent by google_event_id
    originalDate      ← parseLocalDate(event.getOriginalStartTime())
    isCancelled       ← "cancelled".equals(event.getStatus())
    // Note: cancelled exceptions may lack start/end — fall back to parent's time fields
  ELSE:                                        // regular non-recurring event
    recurrenceRule    ← null
    recurringEventId  ← null

  # Calendar association (all event types)
  syncedCalendarId  ← syncedCal.getId()       // tracks which Google calendar this event came from
```

**Exception parent resolution:** When syncing, process parent events first, then exceptions. For each exception, look up the parent's internal UUID via `google_event_id` → set `recurring_event_id` to that UUID. This is the one extra step compared to `singleEvents=true`.

**EXDATE vs exception rows:** Google uses two mechanisms to exclude dates from recurrence: EXDATE entries on the parent (simple exclusions) and separate exception events (edited/cancelled instances with `recurringEventId`). We store both — EXDATEs in the `exdates` column, exceptions as separate rows. `RecurrenceExpander` checks both during expansion.

### Sync Logic

> **Design revision:** Original design used `singleEvents=true` which stores every expanded instance as its own DB row. Revised to use `singleEvents=false` — Google returns recurring parents with RRULE strings, and our existing `RecurrenceExpander` handles instance expansion on read. This means one DB row per recurring series instead of ~12+ rows per month, and no re-syncing unchanged data when the sync window shifts.

```java
// GoogleCalendarSyncService.java (pseudocode — reflects actual implementation)

// DD-1: Per-calendar isolation — each calendar syncs independently.
// A failure on one calendar does not affect others. This replaced the
// original abort-all pattern (which fetched all calendars before persisting
// any) because fullSync now uses per-calendar deletes, making partial
// success safe.
// **Revised (BE PR #27).** Changed from abort-all to per-calendar isolation.

@Async
void syncMember(UUID memberId) {
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
            log.error("Sync failed for calendar {} (member {})", cal.getId(), memberId);
        }
    }
}

// Fetches outside transaction, persists inside — DB connection not held during I/O
void fullSync(GoogleSyncedCalendar syncedCal, Calendar calendarClient) {
    List<Event> allEvents = fetchAllEvents(syncedCal, calendarClient);
    self.persistFullSync(syncedCal, allEvents);  // @Transactional via proxy
}

@Transactional
void persistFullSync(GoogleSyncedCalendar syncedCal, List<Event> allEvents) {
    eventRepository.deleteBySyncedCalendarAndSource(syncedCal, EventSource.GOOGLE);
    // Process parents/regular first, then exceptions (FK resolution)
    saveGoogleEvents(allEvents, Map.of(syncedCal, allEvents));
    syncedCal.setLastSyncedAt(Instant.now());
    syncedCalendarRepo.save(syncedCal);
}

void incrementalSync(GoogleSyncedCalendar syncedCal, Calendar calendarClient) {
    try {
        List<Event> changes = fetchIncrementalEvents(syncedCal, calendarClient);
        self.persistIncrementalChanges(syncedCal, changes);  // @Transactional via proxy
    } catch (GoogleJsonResponseException e) {
        if (e.getStatusCode() == 410) {
            syncedCal.setSyncToken(null);
            fullSync(syncedCal, calendarClient);  // fall back to full sync
        } else {
            throw e;
        }
    }
}

@Transactional
void persistIncrementalChanges(GoogleSyncedCalendar syncedCal, List<Event> changes) {
    // Parents first, then exceptions (FK resolution)
    for (Event event : parentChanges) {
        if ("cancelled".equals(event.getStatus())) {
            eventRepository.deleteByGoogleEventId(event.getId());  // cascades to exceptions
        } else {
            upsertEvent(event, syncedCal);
        }
    }
    for (Event event : exceptionChanges) {
        upsertException(event, syncedCal);  // cancelled exceptions stored as is_cancelled rows
    }
    syncedCal.setLastSyncedAt(Instant.now());
    syncedCalendarRepo.save(syncedCal);
}
```

### Scheduled Sync

```java
// Scheduler fires syncMember (which is @Async) for each connected member.
// Exceptions from @Async methods are handled by AsyncUncaughtExceptionHandler,
// not by the scheduler — so no try/catch needed here.
// **Revised (BE PR #27).** Removed dead try/catch; added AsyncConfig.

@Scheduled(fixedRate = 900_000)  // 15 minutes
void syncAllConnectedMembers() {
    List<GoogleOAuthToken> tokens = tokenRepository.findAll();
    for (GoogleOAuthToken token : tokens) {
        syncService.syncMember(token.getMember().getId());
    }
}
```

---

## 7. FE Changes

### TypeScript Type Updates

```typescript
// Current CalendarEvent type (after recurring events — PR #117)
interface CalendarEvent {
  id: string | null;           // null for expanded recurring instances
  title: string;
  startTime: string;
  endTime: string;
  date: Date;
  endDate?: Date;
  memberId: string;
  isAllDay: boolean;           // required (not optional)
  location?: string;
  recurrenceRule?: string;     // RRULE string (parents + expanded instances)
  recurringEventId?: string;   // parent UUID (instances/exceptions)
  isRecurring?: boolean;
  // --- NEW for Google Calendar integration ---
  source?: "NATIVE" | "GOOGLE";
  description?: string;
  htmlLink?: string;
}
```

### UI Changes

1. **Event tiles:** Add a small Google icon (e.g., Lucide `ExternalLink` or a Google 'G' SVG) in the corner when `source === "GOOGLE"`
2. **Event detail modal:** Show `description` field (read-only for Google events). Show "Open in Google Calendar" link when `htmlLink` is present. Recurrence label works automatically — Google recurring instances carry `recurrenceRule` via our expansion, same as native.
3. **Event form:** Add optional `description` textarea for native events.
4. **Edit/delete guard:** Check `source === "GOOGLE"` **before** checking `isRecurring`. Google recurring events should never show the "edit this/all events" dialog — they're read-only. Show a toast or message: "Edit this event in Google Calendar."
5. **Settings page:** Per-member Google Calendar connection section:
   - "Connect Google Calendar" button (when disconnected)
   - Connection status + last synced time (when connected)
   - Calendar picker (checkboxes for available calendars)
   - "Disconnect" button with confirmation
   - "Sync Now" button for manual trigger

### Validation Schema Update

Add `description` to the Zod schema:

```typescript
description: z.string().max(2000, "Description must be 2000 characters or less").optional(),
```

---

## 8. Configuration

### Environment Variables (BE)

```
GOOGLE_CLIENT_ID=...                    # From Google Cloud Console
GOOGLE_CLIENT_SECRET=...                # From Google Cloud Console
GOOGLE_REDIRECT_URI=https://familyhub.joe-bor.me/api/google/callback
TOKEN_ENCRYPTION_KEY=...                # AES-256 key for encrypting stored tokens
```

### application.yml

```yaml
google:
  oauth:
    client-id: ${GOOGLE_CLIENT_ID}
    client-secret: ${GOOGLE_CLIENT_SECRET}
    redirect-uri: ${GOOGLE_REDIRECT_URI}
    scopes:
      - https://www.googleapis.com/auth/calendar.events.readonly
      - https://www.googleapis.com/auth/calendar.calendarlist.readonly
  sync:
    interval-ms: 900000        # 15 minutes
    full-sync-past-days: 30
    full-sync-future-days: 90

security:
  token-encryption-key: ${TOKEN_ENCRYPTION_KEY}
```

### Local Development

For local dev, the OAuth redirect URI needs to point to localhost:
- Register a second OAuth redirect URI in Google Cloud Console: `http://localhost:8080/api/google/callback`
- Use `GOOGLE_REDIRECT_URI=http://localhost:8080/api/google/callback` in local env
- Google allows `http://localhost` without HTTPS for development

---

## 9. Disconnect & Cleanup

### Member Disconnects Google Calendar

1. `DELETE /api/google/disconnect/{memberId}`
2. BE revokes token: `POST https://oauth2.googleapis.com/revoke?token={access_token}`
3. Delete `google_synced_calendar` rows for this member (cascade from token)
4. Delete `google_oauth_token` row for this member
5. Delete all `calendar_event` rows where `source = 'GOOGLE' AND member_id = memberId`
6. Return 200 OK

### Family Member Deleted

1. `ON DELETE CASCADE` on `google_oauth_token.member_id` → token row deleted
2. `ON DELETE CASCADE` on `google_synced_calendar.member_id` → synced calendar rows deleted
3. `ON DELETE CASCADE` on `calendar_event.member_id` → all events deleted (existing behavior)
4. Best practice: revoke Google token before deletion (call in service layer)

### Family Deleted

- Cascades through: Family → FamilyMember → all related rows (tokens, calendars, events)

---

## 10. Testing Strategy

### BE Tests

- **Unit tests:** GoogleEventMapper, GoogleOAuthService (token URL building), TokenEncryptionService
- **Integration tests:** Full sync flow with mocked Google API responses (WireMock or similar)
- **Controller tests:** OAuth endpoints, sync endpoints, authorization checks
- **Edge cases:** All-day events, multi-page sync, 410 Gone handling, token refresh, cancelled events

### FE Tests

- **Unit tests:** Updated CalendarEvent type handling, description field rendering, source icon display
- **E2E tests:** Cannot test real Google OAuth in CI. Test the settings UI flow with mocked responses.

### Manual Testing

- Full OAuth flow against real Google Calendar
- Verify events appear with correct member colors
- Create/modify/delete events in Google Calendar → verify sync picks up changes
- Disconnect → verify Google events removed, native events preserved

---

## 11. Rollout Plan

### Phase 1a: BE Foundation

1. Flyway migration (new tables + alter calendar_event)
2. Token encryption service
3. OAuth flow (auth URL, callback, token storage)
4. Google Calendar list endpoint
5. Google event mapper
6. Full sync + incremental sync services
7. Scheduled sync job
8. Updated CalendarEvent response with new fields
9. Google event protection (reject PUT/DELETE on Google events)
10. Tests

### Phase 1b: FE Integration

1. Updated TypeScript types (source, description, htmlLink)
2. Settings page: Google Calendar connection UI
3. Calendar picker in Settings
4. Event detail modal: description display + "Open in Google Calendar" link
5. Event tiles: Google icon overlay
6. Event form: description textarea for native events
7. Zod schema updates
8. Tests

### Phase 1c: Deploy

1. Google Cloud Console setup (OAuth credentials, Calendar API enabled)
2. Set environment variables on production droplet
3. Deploy BE (migration runs automatically via Flyway)
4. Deploy FE
5. Connect real Google accounts and verify

---

## 12. Future Phases

| Phase | Feature | Depends On |
|-------|---------|------------|
| 2a | Write-back (create/edit events → Google Calendar) | Phase 1 + scope upgrade to `calendar.events` |
| 2b | Webhooks (near-real-time push from Google) | Phase 1 + WebSocket infra |
| 2c | ~~Recurring event display~~ Handled by [Recurring Events](./recurring-events-design.md) — complete (BE #20/#21, FE #117) | Complete |
| 3 | Bidirectional conflict resolution | Phase 2a |

---

## 13. Open Questions

1. **Timezone handling:** ~~Assume all family members are in the same timezone.~~ **Partially resolved (BE [PR #25](https://github.com/joe-bor/family-hub-api/pull/25)).** `GoogleEventMapper.toZonedDateTime()` uses the event's original timezone offset — meaning times reflect where the event was created, not the family's timezone. If a member creates an event at 3 PM EST while traveling, it appears as 3 PM for a PST-based family (should be 12 PM PST). Documented as a future improvement: add configurable family timezone.

2. **Sync window:** ~~Full sync fetches 30 days past + 90 days future.~~ **Resolved (BE [PR #25](https://github.com/joe-bor/family-hub-api/pull/25)).** With `singleEvents=false`, there's no time window on the fetch — Google returns all parents and exceptions regardless of date. Expansion is bounded by the query window on `GET /calendar/events`.

3. **Google consent screen verification:** For production with >100 users, Google requires app verification (security review). Not needed for family use, but worth noting if the app ever scales.

4. ~~**All-day event spanning multiple days:**~~ **Resolved.** The `endDate` column was added in Phase B (BE #18, FE #111). Google's exclusive `end.date` maps to our inclusive `endDate` (subtract one day). Multi-day all-day events render as repeated badges per day.

5. ~~**RRULE subset compatibility:**~~ **Resolved (BE [PR #25](https://github.com/joe-bor/family-hub-api/pull/25)).** iCal4j handles everything Google produces. Per-parent try/catch in `CalendarEventService.getEvents()` ensures one bad RRULE doesn't crash the entire response — logs a warning and skips that parent.

6. **Midnight-crossing events (new):** Events crossing midnight (e.g., 11 PM → 1 AM) have `endTime < startTime` with no `endDate` set. May confuse query/rendering. Flagged as known Phase 1 edge case (BE [PR #25](https://github.com/joe-bor/family-hub-api/pull/25)).

---

## 14. FE/BE API Contract Notes

Notes on response shapes and behavior that the FE team needs to know when building against the Google Calendar endpoints. These are implementation decisions not captured in §5 above.

### Calendar Selection Endpoints

**`GET /api/google/calendars/{memberId}`** — Returns calendars from Google merged with stored selections:

```json
{
  "data": [
    { "id": "primary", "name": "Joe's Calendar", "primary": true, "enabled": true },
    { "id": "work@group.calendar.google.com", "name": "Work", "primary": false, "enabled": false }
  ],
  "message": null
}
```

- `primary` flag comes from Google's API (which calendar is the user's default)
- `enabled` flag comes from our DB (whether the user selected it for sync)
- Makes a live Google API call — may be slower (~200-500ms)

**`PUT /api/google/calendars/{memberId}`** — Same response shape as GET:

```json
{
  "data": [
    { "id": "primary", "name": "Joe's Calendar", "primary": true, "enabled": true },
    { "id": "work@group.calendar.google.com", "name": "Work", "primary": false, "enabled": false }
  ],
  "message": "Calendar selection updated"
}
```

- Currently also makes a live Google API call to build the merged response
- **Future optimization:** The PUT could skip the Google call and return a simpler shape (`{ id, name, enabled }` without `primary`), since the FE already has the full calendar list from the preceding GET. Not done yet — flagged for when/if PUT latency becomes a concern. If this changes, the FE would need to handle the simpler PUT response shape separately from the GET response.

**`GET /api/google/status/{memberId}`** — Connection status with synced calendars:

```json
{
  "data": {
    "connected": true,
    "calendars": [
      { "id": "primary", "name": "Joe's Calendar", "enabled": true, "lastSyncedAt": null }
    ]
  },
  "message": null
}
```

- `calendars` is an empty array when `connected` is `false`
- `lastSyncedAt` will be `null` until sync is implemented (PR 4)
- This does NOT call Google — reads from DB only
