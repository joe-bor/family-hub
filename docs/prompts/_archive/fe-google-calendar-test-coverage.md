> Archived 2026-04-25 — implemented by joe-bor/FamilyHub#128

# FE Cleanup: Google Calendar Service Test Coverage

## Context

A cross-repo alignment check found two cosmetic issues in the Google Calendar FE code. Neither causes runtime bugs, but both reduce confidence in the integration layer.

**You are expected to verify these claims yourself before acting.** Read the actual test files and confirm the gaps.

## Issue 1: Service test covers only 3 of 6 endpoints

### The claim

`src/api/services/google-calendar.service.test.ts` has tests for `getAuthUrl`, `getConnectionStatus`, and `disconnect` — but is missing tests for `getCalendars`, `updateCalendars`, and `syncCalendar`.

### Starting points

- `src/api/services/google-calendar.service.test.ts` — the existing test file
- `src/api/services/google-calendar.service.ts` — the service with all 6 methods

### What to fix

Add test cases for the three missing service methods, following the same pattern as the existing tests (inline `server.use()` with MSW):

1. **`getCalendars`** — `GET /google/calendars/{memberId}` returns `ApiResponse<GoogleCalendarInfo[]>`
2. **`updateCalendars`** — `PUT /google/calendars/{memberId}` with `{ calendarIds: string[] }` body, returns `ApiResponse<GoogleCalendarInfo[]>`
3. **`syncCalendar`** — `POST /google/sync/{memberId}` returns 202

Verify the request shape (especially that `updateCalendars` sends the correct body) and response typing.

## Issue 2: No shared MSW handlers for Google endpoints

### The claim

The shared MSW handler file at `src/test/mocks/handlers.ts` has no default handlers for any `/api/google/*` endpoints. All Google Calendar tests use inline `server.use()` overrides. This is fine functionally, but means there's no centralized mock for components that render Google Calendar state as part of a larger test.

### Starting point

- `src/test/mocks/handlers.ts` — the shared handler file

### What to fix

Add default happy-path handlers for the most commonly queried Google endpoints:

1. **`GET /api/google/status/:memberId`** — return `{ connected: false, calendars: [] }` (disconnected by default, tests that need connected state override)
2. **`GET /api/google/calendars/:memberId`** — return empty array

These two are the query hooks that fire automatically when components mount. The mutation endpoints (`PUT`, `POST`, `DELETE`) don't need defaults — they're only called on user action and are better mocked inline per-test.

Follow the existing handler patterns in the file (URL structure, `HttpResponse.json` usage, `ApiResponse` envelope).

## What NOT to change

- Do **not** touch any BE code
- Do **not** refactor existing inline mocks in component tests to use the shared handlers — that's optional follow-up work
- Do **not** add tests for the TanStack Query hooks — those already have their own test file
