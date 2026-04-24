# BE PR 1: Source column, description field, and Google event protection

## Context

This is the first PR in the Google Calendar integration series. It lays the groundwork by adding the `source` discriminator, a `description` field for all events, and write-protection for future Google-sourced events. No Google API interaction yet — this is pure data model + API contract work.

Design doc: `docs/google-calendar-integration-design.md` (§4 Data Model, §5 BE API Endpoints — "Google Event Protection" and "Modified Endpoints")

## What to build

### 1. Flyway Migration: `V4__google_calendar_source_and_description.sql`

```sql
ALTER TABLE calendar_event
    ADD COLUMN source VARCHAR(10) NOT NULL DEFAULT 'NATIVE',
    ADD COLUMN description TEXT;
```

That's it for this PR. The Google-specific columns (`google_event_id`, `html_link`, `etag`, `google_updated_at`) and new tables (`google_oauth_token`, `google_synced_calendar`) come in later PRs.

### 2. Entity changes

Add to `CalendarEvent.java`:
- `source` field — `String`, defaults to `"NATIVE"`. Not nullable.
- `description` field — `String`, nullable.

Consider creating an `EventSource` enum or constants class (`NATIVE`, `GOOGLE`) so the string isn't scattered everywhere.

### 3. DTO changes

**`CalendarEventResponse`** — add:
- `source` (`String`) — always present
- `description` (`String`) — nullable

**`CalendarEventRequest`** — add:
- `description` (`String`) — optional, nullable. Add a `@Size(max = 2000)` validation.

Do NOT add `source` to the request — it's always `NATIVE` for user-created events.

### 4. Mapper changes

Update `CalendarEventMapper`:
- `toResponse()` — map `source` and `description`
- `toEntity()` / request-to-entity mapping — map `description`, hardcode `source = "NATIVE"`
- `toInstanceResponse()` — include `source` and `description` (inherit from parent)

### 5. Google event protection

In `CalendarEventService`, add a check at the top of `updateEvent()` and `deleteEvent()`:

```java
if ("GOOGLE".equals(event.getSource())) {
    throw new IllegalArgumentException(
        "Google Calendar events cannot be modified in FamilyHub. Edit them in Google Calendar.");
}
```

Also protect the instance endpoints (`updateInstance`, `deleteInstance`) — a Google recurring event's instances should be equally protected.

The exception handler should map this to a `400 Bad Request` response. Check how other validation errors are currently handled and follow the same pattern.

### 6. Tests

- **Integration test:** Create an event → verify `source: "NATIVE"` in response. Verify `description` round-trips (create with description, GET returns it, update changes it).
- **Integration test:** Directly insert a `source=GOOGLE` event in test setup → verify `PUT` and `DELETE` return 400 with the protection message.
- **Migration test:** Verify existing events get `source=NATIVE` default after migration.

## What NOT to build

- No Google OAuth, no Google API client, no sync logic
- No `google_event_id`, `html_link`, `etag`, `google_updated_at` columns (next PRs)
- No new tables (`google_oauth_token`, `google_synced_calendar`)
- No `htmlLink` or `source` in FE types (FE PRs come after all BE PRs)

## Branch & PR

- Branch: `feat/google-cal-source-and-description`
- Atomic commits, open a PR when done
- Reference the design doc in the PR description

## Acceptance criteria

- [ ] V4 migration adds `source` (NOT NULL, DEFAULT 'NATIVE') and `description` (nullable) columns
- [ ] `CalendarEventResponse` includes `source` and `description` fields
- [ ] `CalendarEventRequest` accepts optional `description` with `@Size(max = 2000)`
- [ ] Creating an event without `description` works (null/absent)
- [ ] Creating an event with `description` persists and returns it
- [ ] All existing endpoints continue to work (no breaking changes)
- [ ] `PUT` and `DELETE` on a `source=GOOGLE` event returns 400
- [ ] Instance endpoints (`PUT/DELETE /{parentId}/instances/{date}`) also reject Google events
- [ ] Tests cover the above
