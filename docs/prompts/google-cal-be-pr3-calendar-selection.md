# BE PR 3: Calendar Selection — List and Choose Google Calendars to Sync

## Context

This is the third PR in the Google Calendar integration series.
- PR 1 ([#22](https://github.com/joe-bor/family-hub-api/pull/22)): `source`/`description` columns, `EventSource` enum, Google event write protection
- PR 2: OAuth flow — token exchange, encrypted storage, connect/disconnect endpoints

This PR adds the ability for a connected member to see their available Google calendars and choose which ones to sync. No sync logic yet — just calendar discovery and selection.

Design doc: `docs/google-calendar-integration-design.md` (§4 Data Model — `google_synced_calendar` table, §5 Endpoints — Calendar Selection, §6 Service Architecture — `GoogleCredentialService`)

## What to build

### 1. Flyway Migration: `V7__google_synced_calendar.sql`

> **Note:** V5 is `google_oauth_token` and V6 is `standardize_family_timestamps` (both from PR #23).

```sql
CREATE TABLE google_synced_calendar (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    token_id UUID NOT NULL REFERENCES google_oauth_token(id) ON DELETE CASCADE,
    member_id UUID NOT NULL REFERENCES family_member(id) ON DELETE CASCADE,
    google_calendar_id VARCHAR(255) NOT NULL,
    calendar_name VARCHAR(255),
    sync_token TEXT,
    last_synced_at TIMESTAMP,
    enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT now()
);
```

One row per synced calendar. A member can sync multiple Google calendars (primary, work, shared, etc.). Each tracks its own `sync_token` for incremental sync (used in PR 4).

### 2. Entity: `GoogleSyncedCalendar`

JPA entity mapping to the table above.

### 3. Repository: `GoogleSyncedCalendarRepository`

```java
public interface GoogleSyncedCalendarRepository extends JpaRepository<GoogleSyncedCalendar, UUID> {
    List<GoogleSyncedCalendar> findByMemberId(UUID memberId);
    List<GoogleSyncedCalendar> findByMemberIdAndEnabledTrue(UUID memberId);
    Optional<GoogleSyncedCalendar> findByMemberIdAndGoogleCalendarId(UUID memberId, String googleCalendarId);
    void deleteByMemberId(UUID memberId);
}
```

### 4. Service: `GoogleCredentialService`

Builds a Google `Credential` object from stored (encrypted) tokens for a given member. This is used by both this PR (calendar listing) and future PRs (sync).

```java
@Service
public class GoogleCredentialService {

    // Build a Credential from stored tokens
    // 1. Look up GoogleOAuthToken by memberId
    // 2. Decrypt access_token and refresh_token
    // 3. Build a Credential with auto-refresh capability
    // 4. If token is expired, refresh it and update the stored token
    public Credential getCredential(UUID memberId) { ... }
}
```

Use the Google OAuth client library (`google-auth-library-oauth2-http`) or build the `Credential` manually with `GoogleCredential.Builder`. The key requirement: it should auto-refresh the access token using the stored refresh token when expired, and update the DB with the new access token.

### 5. Service: `GoogleCalendarListService`

Lists available calendars from Google for a connected member.

```java
@Service
public class GoogleCalendarListService {

    // Fetch calendar list from Google Calendar API
    // GET https://www.googleapis.com/calendar/v3/users/me/calendarList
    // Returns list of { id, summary, primary } for each calendar
    public List<GoogleCalendarInfo> listCalendars(UUID memberId) { ... }
}
```

`GoogleCalendarInfo` is a simple DTO:

```java
public record GoogleCalendarInfo(
    String id,           // Google's calendar ID (e.g., "primary", "user@gmail.com", or opaque ID)
    String name,         // Display name (summary field from Google)
    boolean primary      // Whether this is the user's primary calendar
) {}
```

### 6. Controller: `GoogleCalendarController`

```java
@RestController
@RequestMapping("/api/google/calendars")
public class GoogleCalendarController {

    // GET /api/google/calendars/{memberId}
    // Lists available Google calendars for the member
    // Returns: [{ id, name, primary, enabled }]
    // "enabled" indicates whether it's currently selected for sync
    // Requires: member is connected (has OAuth token)
    // Validate: memberId belongs to authenticated family

    // PUT /api/google/calendars/{memberId}
    // Body: { "calendarIds": ["primary", "work-calendar-id"] }
    // Sets which calendars to sync:
    //   - For each ID in the list: create a GoogleSyncedCalendar row if it doesn't exist, set enabled=true
    //   - For existing rows NOT in the list: set enabled=false (don't delete — preserves sync_token for re-enable)
    // Validate: memberId belongs to authenticated family
    // Validate: member is connected
    // Returns: updated list of synced calendars
}
```

The response for GET should merge Google's calendar list with our stored selections:

```json
[
  { "id": "primary", "name": "Joe's Calendar", "primary": true, "enabled": true },
  { "id": "family@group.calendar.google.com", "name": "Family", "primary": false, "enabled": false },
  { "id": "work@group.calendar.google.com", "name": "Work", "primary": false, "enabled": true }
]
```

### 7. Update `GoogleOAuthService.disconnect()`

Now that the `google_synced_calendar` table exists, update disconnect to also delete synced calendar rows and all Google-sourced events for the member:

```java
public void disconnect(UUID memberId) {
    // 1. Revoke token at Google (best effort)
    // 2. Delete google_synced_calendar rows (CASCADE from token would handle this, but be explicit)
    // 3. Delete all calendar_event rows where source=GOOGLE AND member_id=memberId
    // 4. Delete google_oauth_token row
}
```

### 8. Update `GET /api/google/status/{memberId}`

Now that synced calendars exist, enrich the status response:

```json
{
  "connected": true,
  "calendars": [
    { "id": "primary", "name": "Joe's Calendar", "enabled": true, "lastSyncedAt": null }
  ]
}
```

`lastSyncedAt` will be null until PR 4 (sync) populates it.

## Dependencies

New Maven dependency needed:

```xml
<dependency>
    <groupId>com.google.apis</groupId>
    <artifactId>google-api-services-calendar</artifactId>
    <version>v3-rev20241101-2.0.0</version>
</dependency>
```

This transitively pulls in `google-api-client`, `google-oauth-client`, and `google-http-client`. Use the latest stable version — check Maven Central.

This is the first PR that actually calls the Google Calendar API. PR 2 only called Google's OAuth token endpoint.

## Tests

- **Unit tests:** `GoogleCalendarListService` — mock the Google Calendar API client, verify the response is mapped correctly to `GoogleCalendarInfo` DTOs.
- **Controller tests:** GET returns calendar list, PUT updates selections, authorization checks (wrong family → 403, not connected → appropriate error).
- **Integration test:** Create OAuth token row, call PUT to select calendars, verify `google_synced_calendar` rows created. Call PUT again with different list, verify enabled/disabled toggled correctly. Disconnect, verify all rows cleaned up.
- **Integration test:** Verify disconnect now also deletes `source=GOOGLE` events for the member.

For tests that would call Google's API: mock the Google Calendar client. These are the only tests where mocking an external service is appropriate — we can't call real Google in CI.

## What NOT to build

- No sync logic (PR 4)
- No scheduler (PR 5)
- No `google_event_id`/`etag`/`html_link`/`google_updated_at` columns on `calendar_event` (PR 4)

## Branch & PR

- Branch: `feat/google-cal-calendar-selection`
- Atomic commits, open a PR when done
- Reference the design doc and prior PRs in the description

## Acceptance criteria

- [ ] V6 migration creates `google_synced_calendar` table
- [ ] `GET /api/google/calendars/{memberId}` returns available Google calendars with enabled status
- [ ] `PUT /api/google/calendars/{memberId}` creates/updates synced calendar selections
- [ ] Deselecting a calendar sets `enabled=false` (preserves `sync_token`)
- [ ] `GoogleCredentialService` builds credentials from encrypted tokens with auto-refresh
- [ ] `GET /api/google/status/{memberId}` now includes calendar list
- [ ] Disconnect cleans up synced calendars AND Google-sourced events
- [ ] All endpoints validate memberId belongs to authenticated family
- [ ] Member must be connected (have OAuth token) for calendar endpoints
- [ ] Tests cover the above (Google API calls mocked in tests)
