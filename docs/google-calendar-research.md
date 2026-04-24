# Google Calendar Integration — Research & Q&A

**Date:** 2026-03-07
**Purpose:** Reference document capturing research findings and design decisions for integrating Google Calendar into FamilyHub.

---

## Table of Contents

1. [Context & Goals](#context--goals)
2. [Google Calendar API v3 — Event Shape](#google-calendar-api-v3--event-shape)
3. [Events.list Endpoint](#eventslist-endpoint)
4. [OAuth Scopes](#oauth-scopes)
5. [Java Client Library](#java-client-library)
6. [Rate Limits & Quotas](#rate-limits--quotas)
7. [Recurring Events](#recurring-events)
8. [Sync Architecture — Sync Tokens](#sync-architecture--sync-tokens)
9. [Push Notifications (Webhooks)](#push-notifications-webhooks)
10. [Our Current CalendarEvent Shape](#our-current-calendarevent-shape)
11. [Field Mapping: Google vs Ours](#field-mapping-google-vs-ours)
12. [Implementation Patterns from the Community](#implementation-patterns-from-the-community)
13. [Q&A — Design Decisions](#qa--design-decisions)

---

## Context & Goals

FamilyHub has a fully functional calendar module with CRUD for native events. The family currently uses Google Calendar as their primary scheduling tool. The goal is to **read Google Calendar events and render them in FamilyHub** alongside native events, making FamilyHub a unified family calendar view.

**Phase 1 scope:** Read-only sync. Fetch events from each family member's Google Calendar and display them in FamilyHub with the correct member color mapping.

**Not in Phase 1:** Writing events back to Google Calendar, real-time push notifications (webhooks), recurring event editing.

---

## Google Calendar API v3 — Event Shape

The Event resource is the core object. Key fields relevant to FamilyHub:

### Fields We Will Use

| Field | Type | Writable? | Description |
|-------|------|-----------|-------------|
| `id` | string | read-only (after insert) | Opaque event identifier |
| `summary` | string | writable | Event title |
| `description` | string | writable | Event description (may contain HTML) |
| `location` | string | writable | Free-form geographic location text |
| `status` | string | writable | `"confirmed"`, `"tentative"`, `"cancelled"` |
| `htmlLink` | string | read-only | URL to open event in Google Calendar web UI |
| `created` | datetime (RFC 3339) | read-only | Creation timestamp |
| `updated` | datetime (RFC 3339) | read-only | Last modification timestamp |
| `etag` | string | read-only | ETag for optimistic concurrency |
| `colorId` | string | writable | Color ID ("1"–"11"), maps to Google's event color definitions |
| `recurringEventId` | string | read-only | For instances of a recurring event, the parent event's ID |

### Start/End Time Structure

`start` and `end` are nested objects, NOT simple datetime strings:

```json
{
  "date": "2026-03-07",                          // for all-day events (yyyy-MM-dd)
  "dateTime": "2026-03-07T10:00:00-05:00",       // for timed events (RFC 3339)
  "timeZone": "America/New_York"                  // IANA timezone (optional)
}
```

- Exactly ONE of `date` or `dateTime` is set (never both).
- `date` = all-day event, `dateTime` = timed event.
- `timeZone` is optional but recommended for recurring events.

### Creator & Organizer

```json
{
  "creator": {
    "email": "joe@gmail.com",
    "displayName": "Joe",
    "self": true
  },
  "organizer": {
    "email": "joe@gmail.com",
    "displayName": "Joe",
    "self": true
  }
}
```

`creator.email` is the key field for mapping events to family members.

### Attendees

```json
{
  "attendees": [
    {
      "email": "partner@gmail.com",
      "displayName": "Partner",
      "responseStatus": "accepted",
      "optional": false
    }
  ]
}
```

### Extended Properties (custom metadata)

```json
{
  "extendedProperties": {
    "private": { "familyhub_member_id": "uuid" },
    "shared": { "key": "value" }
  }
}
```

Useful for future write-back: store FamilyHub metadata on Google events without polluting the description field.

### Fields We Will Ignore (Phase 1)

| Field | Why |
|-------|-----|
| `recurrence[]` | Using `singleEvents=true` to get expanded instances instead |
| `reminders` | Not implementing notifications yet |
| `conferenceData` | Google Meet links not relevant |
| `attachments[]` | Not displaying attachments |
| `gadget` | Deprecated |
| `transparency` | Busy/free status not needed |
| `visibility` | Privacy not applicable in family context |
| `workingLocationProperties` | Enterprise feature |
| `outOfOfficeProperties` | Enterprise feature |
| `focusTimeProperties` | Enterprise feature |

---

## Events.list Endpoint

`GET https://www.googleapis.com/calendar/v3/calendars/{calendarId}/events`

### Key Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `calendarId` | string (path) | — | `"primary"` or specific calendar ID |
| `timeMin` | datetime (RFC 3339) | — | Lower bound on event end time (exclusive) |
| `timeMax` | datetime (RFC 3339) | — | Upper bound on event start time (exclusive) |
| `singleEvents` | boolean | false | Expand recurring events into instances |
| `orderBy` | string | unspecified | `"startTime"` (requires singleEvents=true) or `"updated"` |
| `maxResults` | integer | 250 | Max events per page (1–2500) |
| `pageToken` | string | — | Pagination token |
| `syncToken` | string | — | Incremental sync token (mutually exclusive with most params) |
| `showDeleted` | boolean | false | Include cancelled events |
| `q` | string | — | Free text search across summary, description, location |
| `timeZone` | string | UTC | Timezone for the response |

### Response Shape

```json
{
  "kind": "calendar#events",
  "summary": "Calendar Name",
  "timeZone": "America/Los_Angeles",
  "nextPageToken": "...",
  "nextSyncToken": "...",
  "items": [ /* Event objects */ ]
}
```

- `nextPageToken`: present if there are more pages. Call again with this token.
- `nextSyncToken`: present on the LAST page. Store this for incremental sync.

### Critical Behavior

- `timeMin`/`timeMax` filter on overlap, not containment. An event spanning the boundary is included.
- `singleEvents=true` is essential for calendar display — it expands recurring events into individual instances.
- When using `syncToken`, most other parameters are ignored (Google determines what to return).

---

## OAuth Scopes

| Scope | Level | Description |
|-------|-------|-------------|
| `calendar.events.readonly` | read-only | Read events on all calendars |
| `calendar.events` | read-write | Full event access (create/edit/delete) |
| `calendar.readonly` | read-only | Read all calendars and events |
| `calendar.calendarlist.readonly` | read-only | List available calendars |

### Our Phase 1 Scopes

- `calendar.events.readonly` — read events
- `calendar.calendarlist.readonly` — let users pick which calendar to sync

Both are classified as "sensitive" by Google. This requires an OAuth consent screen configuration but does NOT require full Google app verification when under 100 users (which is fine for a family app).

---

## Java Client Library

### Maven Dependencies

```xml
<dependency>
    <groupId>com.google.api-client</groupId>
    <artifactId>google-api-client</artifactId>
    <version>2.7.2</version>
</dependency>
<dependency>
    <groupId>com.google.oauth-client</groupId>
    <artifactId>google-oauth-client-jetty</artifactId>
    <version>1.36.0</version>
</dependency>
<dependency>
    <groupId>com.google.apis</groupId>
    <artifactId>google-api-services-calendar</artifactId>
    <version>v3-rev20241101-2.0.0</version>
</dependency>
```

Version format: `v3-revYYYYMMDD-2.0.0`. Check Maven Central for latest revision.

### Key Java Classes

**Building the service:**
```java
Calendar service = new Calendar.Builder(
    GoogleNetHttpTransport.newTrustedTransport(),
    GsonFactory.getDefaultInstance(),
    credential
)
.setApplicationName("Family Hub")
.build();
```

**The `Event` class** mirrors the JSON resource 1:1:
- `getSummary()` / `setSummary(String)` — title
- `getDescription()` / `setDescription(String)` — notes/description
- `getStart()` / `getEnd()` — returns `EventDateTime` objects
- `getLocation()` / `setLocation(String)`
- `getStatus()` — "confirmed", "tentative", "cancelled"
- `getHtmlLink()` — read-only link to Google Calendar web
- `getCreator()` — returns `Event.Creator` with `.getEmail()`
- `getAttendees()` — returns `List<EventAttendee>`
- `getId()`, `getEtag()`, `getCreated()`, `getUpdated()`
- `getRecurringEventId()` — parent ID for recurring instances

**`EventDateTime` class:**
- `getDate()` — for all-day events (returns `com.google.api.client.util.DateTime`)
- `getDateTime()` — for timed events
- `getTimeZone()` — IANA timezone string

**Listing events:**
```java
Events events = service.events().list("primary")
    .setTimeMin(new DateTime(startMillis))
    .setTimeMax(new DateTime(endMillis))
    .setSingleEvents(true)
    .setOrderBy("startTime")
    .setMaxResults(250)
    .execute();

List<Event> items = events.getItems();
String nextSyncToken = events.getNextSyncToken();
```

---

## Rate Limits & Quotas

| Quota | Limit |
|-------|-------|
| Queries per day (per project) | 1,000,000 |
| Queries per 100 seconds per user | 500 |
| Queries per 100 seconds (project total) | 5,000 |

**Practical impact for FamilyHub:** A family of 5, each syncing every 15 minutes = ~480 requests/day. Well under all limits.

**Error handling:**
- `403 rateLimitExceeded` or `429 Too Many Requests` → exponential backoff with jitter
- `401 Unauthorized` → access token expired, use refresh token
- `410 Gone` → sync token expired, do full sync

---

## Recurring Events

### How They Work

A recurring event is a single Event resource with a `recurrence` field containing RFC 5545 rules:

```json
{
  "recurrence": ["RRULE:FREQ=WEEKLY;BYDAY=MO,WE,FR"]
}
```

### `singleEvents=true` vs `false`

| Parameter | What You Get |
|-----------|-------------|
| `false` (default) | Returns the parent recurring event as one item. No individual instances. |
| `true` | Expands recurring events into individual instances within the timeMin/timeMax window. Each instance is a separate Event with `recurringEventId` pointing to parent. |

**For a calendar UI, `singleEvents=true` is the simplest option** — Google expands recurrence and returns flat instances.

### Implication for FamilyHub

> **Updated (post-recurring events implementation):** Since we now have our own `RecurrenceExpander` (iCal4j, [BE PR #20](https://github.com/joe-bor/family-hub-api/pull/20)), we use `singleEvents=false` for Google sync. Google returns recurring parents with RRULE strings, and we store them as parent rows in our DB — our expander handles instance generation on read, same as native recurring events. This avoids storing hundreds of denormalized instance rows and keeps one code path for all recurrence expansion.
>
> Google exceptions (edited/cancelled instances) are returned alongside parents and stored as exception rows with `recurring_event_id` pointing to our parent's UUID. This maps directly to our existing parent/exception model.

---

## Sync Architecture — Sync Tokens

Google's recommended sync pattern uses sync tokens for efficient incremental updates.

### Full Sync (initial, or after token expiry)

1. Call `events.list()` with `singleEvents=true`, `timeMin`, `timeMax`, NO `syncToken`
2. Page through all results using `nextPageToken`
3. Save the `nextSyncToken` from the last page
4. Persist all events to DB

### Incremental Sync (every 15 minutes)

1. Call `events.list()` with the stored `syncToken`
2. Google returns ONLY events that changed since last sync
3. For each returned event:
   - `status: "confirmed"` → upsert (insert or update) in DB
   - `status: "cancelled"` → delete from DB
4. Store the new `syncToken`

### Token Expiry

- If Google returns `410 Gone`, the sync token is stale (typically after ~7 days of inactivity)
- Must do a full sync again
- This is expected behavior, not an error

### Why This Is Efficient

An incremental sync for a calendar with no changes returns an empty `items` array almost instantly. Even with changes, only the deltas are returned. This means 15-minute polling is very cheap in terms of API quota.

---

## Push Notifications (Webhooks)

Google Calendar supports push notifications via "watch" channels.

### How It Works

1. Call `events.watch()` with your webhook URL and a channel ID
2. Google sends a `sync` verification message
3. On any event change, Google POSTs a notification to your URL
4. The notification only says "something changed" — you must call `events.list()` with your sync token to get actual changes

### Requirements

- HTTPS endpoint with valid certificate (FamilyHub has this at `familyhub.joe-bor.me`)
- Channels expire every ~7 days — must renew via `@Scheduled` task
- For local dev, need ngrok or similar tunnel

### Phase 1 Decision

**Not implementing webhooks in Phase 1.** Polling with sync tokens every 15 minutes is sufficient for a family-scale app and much simpler. Webhooks can be added in Phase 2 alongside WebSocket integration for near-real-time updates.

---

## Our Current CalendarEvent Shape

### Database Schema

```sql
CREATE TABLE calendar_event (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    date DATE NOT NULL,
    is_all_day BOOLEAN NOT NULL DEFAULT FALSE,
    location VARCHAR(255),
    member_id UUID NOT NULL REFERENCES family_member(id) ON DELETE CASCADE,
    family_id UUID NOT NULL REFERENCES family(id) ON DELETE CASCADE
);
```

### BE Entity Fields

| Field | Java Type | DB Type |
|-------|-----------|---------|
| id | UUID | UUID PK |
| title | String | VARCHAR(255) NOT NULL |
| startTime | LocalTime | TIME NOT NULL |
| endTime | LocalTime | TIME NOT NULL |
| date | LocalDate | DATE NOT NULL |
| isAllDay | boolean | BOOLEAN DEFAULT FALSE |
| location | String | VARCHAR(255) nullable |
| member | FamilyMember (FK) | UUID NOT NULL |
| family | Family (FK) | UUID NOT NULL |

### API Request/Response Format

**Request (12-hour time strings):**
```json
{
  "title": "Team Meeting",
  "startTime": "4:00 PM",
  "endTime": "5:00 PM",
  "date": "2026-03-07",
  "memberId": "uuid",
  "isAllDay": false,
  "location": "Office"
}
```

**Response:**
```json
{
  "id": "uuid",
  "title": "Team Meeting",
  "startTime": "4:00 PM",
  "endTime": "5:00 PM",
  "date": "2026-03-07",
  "memberId": "uuid",
  "isAllDay": false,
  "location": "Office"
}
```

### FE TypeScript Types

```typescript
interface CalendarEvent {
  id: string;
  title: string;
  startTime: string;     // "9:00 AM" (12-hour from API)
  endTime: string;       // "5:00 PM"
  date: Date;            // JS Date (local midnight)
  memberId: string;
  isAllDay?: boolean;
  location?: string;
}
```

### FamilyMember.email Field

```java
@Email
private String email;  // Optional, nullable, validated format
```

Stored on `FamilyMember`, not on `CalendarEvent`. This is the field used to map Google Calendar events to family members — match `creator.email` from Google to `FamilyMember.email`.

---

## Field Mapping: Google vs Ours

| Google Calendar Field | Our Field | Gap | Action |
|---|---|---|---|
| `id` | `id` (UUID) | Different ID systems | Add `googleEventId` column |
| `summary` | `title` | Name difference | Map directly |
| `description` | -- | **Missing** | Add `description` column |
| `start.dateTime` (RFC 3339) | `date` + `startTime` | Google combines, we split | Mapper splits dateTime |
| `start.date` (all-day) | `isAllDay` + `date` | Different representation | `start.date` present = allDay |
| `end.dateTime` / `end.date` | `endTime` + `date` | Same as start | Same mapping |
| `location` | `location` | Match | Direct map |
| `status` | -- | Needed for sync | Handle in sync logic |
| `htmlLink` | -- | Missing | Add `htmlLink` column |
| `etag` | -- | Missing | Add `etag` column |
| `updated` | -- | Missing | Add `googleUpdatedAt` column |
| `creator.email` | via `memberId` → `FamilyMember.email` | **Key mapping field** | Match email → member |
| `colorId` | Member color | Different systems | Ignore Google's, use member color |
| `recurrence[]` | -- | Not needed | Use `singleEvents=true` |
| `attendees` | -- | Not needed Phase 1 | Event belongs to calendar owner |
| `reminders` | -- | Not needed | Skip |

---

## Implementation Patterns from the Community

### Token Storage

The dominant pattern: store refresh tokens encrypted in a dedicated DB table. Access tokens are short-lived (1 hour) — the Google Java client handles refresh automatically if you provide a `Credential` object with the refresh token.

**Critical gotcha:** Google only returns the refresh token on the FIRST authorization. Use `access_type=offline` and `prompt=consent` in the OAuth URL to ensure you always get it.

### Service Layer Architecture

```
Controller Layer
  +-- GoogleOAuthController (handles /auth and /callback)
  +-- GoogleSyncController (manual sync trigger)
  +-- CalendarEventController (existing, now serves both sources)
      +-- CalendarSyncService (orchestrates sync)
            +-- GoogleCalendarClient (wraps Google API calls)
            |     +-- GoogleCredentialService (manages OAuth tokens)
            +-- GoogleEventMapper (Google Event <-> CalendarEvent)
            +-- CalendarEventRepository (existing DB access)
```

### Caching Strategy

**DB cache with sync tokens** is the recommended approach:
- Store mapped events in your own DB
- Serve reads from DB (fast, no API dependency)
- Sync periodically via `@Scheduled` using sync tokens (efficient)
- Full sync on first connection and on token expiry (410 Gone)

### Common Pitfalls

1. **Refresh token loss:** Only returned on first consent. Force `prompt=consent`.
2. **All-day vs timed events:** Google uses `start.date` (no time) vs `start.dateTime`. Must handle both.
3. **Timezone drift:** Google events have timezone info. Our model uses LocalTime (no TZ). Must normalize.
4. **Cancelled events in sync:** Incremental sync returns cancelled events — must delete, not ignore.
5. **Google consent screen:** Starts in "Testing" mode (100 users max). Fine for family use.
6. **Token encryption:** Store encrypted at rest. Use Spring's `TextEncryptor` or AES-256.
7. **Multiple calendars:** Users have primary + custom + shared calendars. Let them pick which to sync.

---

## Q&A — Design Decisions

### Q: Separate table for Google events vs extend existing calendar_event?

**A: Extend existing table (Option B).** Add columns: `google_event_id` (nullable), `description`, `html_link`, `etag`, `source` (enum: NATIVE/GOOGLE). Native events have `google_event_id = null`. Single query, single event model, unified rendering. The FE doesn't need to know or care where the event came from.

### Q: Can a Google account have multiple calendars?

**A: Yes.** A typical Google account has: Primary (email-based), user-created calendars (e.g., "Kids Activities"), subscribed calendars (holidays, birthdays), shared calendars from other people. The `calendarList.list()` API returns all of them. We show a picker in Settings. Default: sync "primary" only.

### Q: Does calendar selection happen in Google's consent screen?

**A: No.** Google's consent screen only asks "do you grant FamilyHub access to your calendars?" (yes/no). Calendar selection happens in our Settings UI after the OAuth connection is established.

### Q: Can we accommodate multiple calendars per member?

**A: Yes.** The data model uses a `google_synced_calendar` table (one row per synced calendar) linked to the `google_oauth_token` table (one row per connected member). Each synced calendar has its own `google_calendar_id` and `sync_token`. All events from all synced calendars map to the same `member_id`.

### Q: How do we map Google events to family members?

**A: Via email.** `creator.email` from the Google event is matched to `FamilyMember.email`. The member who connected their Google account is the owner. Events from their calendar(s) are assigned to their `member_id` and rendered with their color.

### Q: What if no email matches?

**A: Assign to the calendar owner.** The member who connected their Google account via OAuth owns all events from that calendar, regardless of the event's `creator.email`.

### Q: What happens when a user disconnects Google Calendar?

**A:** Revoke the OAuth token with Google. Delete the `google_oauth_token` row. Delete all `calendar_event` rows where `source = 'GOOGLE'` AND `member_id = that member`. Native events are untouched.

### Q: What if a family member is deleted?

**A:** Existing `ON DELETE CASCADE` on `member_id` FK handles it — all their events (native AND Google) are cascade-deleted. The `google_oauth_token` row is also cascade-deleted. We should also revoke the Google token before deletion as best practice.

### Q: Should events look different based on source?

**A: Mostly no.** Events render the same regardless of source — same color, same position, same interaction. A small Google icon on the event tile differentiates them. The event detail modal shows a "Open in Google Calendar" link for Google-sourced events.

### Q: Polling interval?

**A: 15 minutes.** Sync tokens make incremental syncs very cheap (empty response if nothing changed). 15 minutes balances freshness with simplicity. Webhooks can reduce this to near-real-time in Phase 2.

### Q: Do we need the `description`/notes field for native events too?

**A: Yes.** Adding `description` to the CalendarEvent model benefits both sources. Google events often have descriptions. Native events should support notes too (this was in the original PRD as a missing field). Add it to the model once, use it everywhere.

---

## Reference Links

- [Google Calendar API v3 Events Reference](https://developers.google.com/calendar/api/v3/reference/events)
- [Events.list Endpoint](https://developers.google.com/calendar/api/v3/reference/events/list)
- [OAuth Scopes](https://developers.google.com/calendar/api/auth)
- [Java Quickstart](https://developers.google.com/calendar/api/quickstart/java)
- [Sync Guide (sync tokens)](https://developers.google.com/calendar/api/guides/sync)
- [Push Notifications (webhooks)](https://developers.google.com/calendar/api/guides/push)
- [OAuth 2.0 for Web Server Apps](https://developers.google.com/identity/protocols/oauth2/web-server)
- [Spring Security OAuth2 Client](https://docs.spring.io/spring-security/reference/servlet/oauth2/client/index.html)
