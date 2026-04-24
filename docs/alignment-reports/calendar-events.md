# Alignment Report: CalendarEvents

**Date:** 2026-02-20 (updated 2026-02-25)
**Status:** ALL RESOLVED

Compared FE (`frontend/`) and BE (`backend/family-hub-api/`) implementations of the CalendarEvents API. See `docs/calendar-events-api-reference.md` for the shared contract.

---

## Breaking Mismatches

### 1. Time Format: FE sends 24-hour, BE expects 12-hour

| Side | Detail | File |
|------|--------|------|
| FE | Zod schema validates 24h format with regex `/^([01]?[0-9]\|2[0-3]):[0-5][0-9]$/` (e.g., `"14:00"`) | `frontend/src/lib/validations/calendar.ts` |
| BE | Parses with `DateTimeFormatter.ofPattern("h:mm a")` (e.g., `"2:00 PM"`) | `backend/.../mapper/CalendarEventMapper.java` |
| Doc | States API uses 12-hour and FE should convert before sending | `docs/calendar-events-api-reference.md:122-126` |

**Impact:** Every create/update request will throw `DateTimeParseException` on the BE, resulting in a 500 error.

**Fix:** FE should convert 24h → 12h before sending requests. The `format24hTo12h()` utility already exists in `src/lib/time-utils.ts`.

**Prompt:** `docs/prompts/fix-fe-be-wiring.md` (Issue #4)

---

### 2. URL Path: FE omits `/api` prefix

| Side | Detail | File |
|------|--------|------|
| FE | Service calls `httpClient.get("/calendar/events", ...)` — leading `/` makes `new URL()` discard the base URL path | `frontend/src/api/services/calendar.service.ts` |
| FE | `new URL("/calendar/events", "http://localhost:8080/api")` → `http://localhost:8080/calendar/events` | `frontend/src/api/client/http-client.ts:72` |
| BE | Controller mapped to `/api/calendar/events` | `backend/.../controller/CalendarEventController.java` |

**Impact:** Every FE request will 404.

**Fix:** Either add trailing `/` to base URL and remove leading `/` from service paths, or change base URL to `http://localhost:8080` and prefix all service paths with `/api/`.

**Prompt:** `docs/prompts/fix-fe-be-wiring.md` (Issue #2)

---

### 3. Validation Error Status Code: FE expects 422, BE returns 400

| Side | Detail | File |
|------|--------|------|
| FE | `mapStatusToErrorCode()` maps 422 → `VALIDATION_ERROR`. 400 falls to default → `NETWORK_ERROR` | `frontend/src/api/client/api-error.ts:49-66` |
| BE | `GlobalExceptionHandler` returns 400 for `MethodArgumentNotValidException` | `backend/.../exception/GlobalExceptionHandler.java` |

**Impact:** Validation errors won't trigger the FE's validation error handling. Field-level error messages from `errors[]` won't display.

**Fix:** Add `case 400` to `mapStatusToErrorCode` switch statement.

**Prompt:** `docs/prompts/fix-fe-be-wiring.md` (Issue #3)

---

### 4. Infinite Reload Loop: useFamily fires before auth

| Side | Detail | File |
|------|--------|------|
| FE | `useSetupComplete()` calls `useFamily()` unconditionally at top of `App.tsx` | `frontend/src/App.tsx:96` |
| FE | `useFamily()` fires `GET /family` immediately with `initialDataUpdatedAt: 0` forcing refetch | `frontend/src/api/hooks/use-family.ts:85-107` |
| FE | `onUnauthorized` does `window.location.reload()` on 401 | `frontend/src/api/client/http-client.ts:175-183` |
| BE | All endpoints except `/api/auth/**` require JWT | `backend/.../config/SecurityConfig.java:42-46` |

**Impact:** App enters infinite reload loop when `VITE_USE_MOCK_API=false` and no auth token exists. App is completely unusable.

**Fix:** Prevent `useFamily` query from firing when unauthenticated.

**Prompt:** `docs/prompts/fix-fe-be-wiring.md` (Issue #1)

---

## Risky Mismatches

### 5. `CalendarEvent.date` type: FE uses `Date`, API returns `string`

| Side | Detail | File |
|------|--------|------|
| FE | `date: Date` (JavaScript Date object) | `frontend/src/lib/types/calendar.ts` |
| BE | `date` is a `String` formatted as `"yyyy-MM-dd"` | `backend/.../dto/CalendarEventResponse.java` |
| FE | Mock handlers call `parseLocalDate()` converting strings to Date objects — masks the bug | `frontend/src/api/mocks/calendar.mock.ts` |

**Impact:** When mocks are off, `event.date` will be a string at runtime. Code calling `date.getDay()`, `format(event.date, ...)` etc. will throw. Calendar views using `new Date(event.date)` will show wrong dates west of UTC.

**Prompt:** `docs/prompts/calendar-event-date-type-safety.md` (existing)

---

### 6. `UpdateEventRequest` includes `id` in body

| Side | Detail | File |
|------|--------|------|
| FE | `UpdateEventRequest` has `id: string` as a required field | `frontend/src/lib/types/calendar.ts` |
| BE | Uses same `CalendarEventRequest` for create and update — no `id` field; comes from path param | `backend/.../dto/CalendarEventRequest.java` |

**Impact:** Extra `id` in body is silently ignored by Jackson. Not breaking, but creates a false contract and redundant data on the wire.

**Prompt:** `docs/prompts/remove-id-from-update-request-bodies.md` (existing, deferred)

---

### 7. `isAllDay` optionality inconsistency

| Side | Detail | File |
|------|--------|------|
| FE | `isAllDay?: boolean` (optional, could be `undefined`) | `frontend/src/lib/types/calendar.ts` |
| BE | `boolean isAllDay` (primitive, defaults `false` if omitted from JSON) | `backend/.../dto/CalendarEventRequest.java` |
| BE | Response: `boolean isAllDay` (always present, never null) | `backend/.../dto/CalendarEventResponse.java` |

**Impact:** Minor. FE may treat `undefined` differently from `false` in conditional checks. Responses will always have `isAllDay: false` where FE expects it could be `undefined`.

**Prompt:** `docs/prompts/calendar-event-date-type-safety.md` (bundled with Issue #5)

---

## Cosmetic Mismatches

### 8. FE `title` max length (100) has no BE counterpart

| Side | Detail | File |
|------|--------|------|
| FE | `max(100, "Event name must be 100 characters or less")` | `frontend/src/lib/validations/calendar.ts` |
| BE | Only `@NotEmpty` — no `@Size(max=100)` or column length | `backend/.../dto/CalendarEventRequest.java` |

**Impact:** Direct API calls bypassing the FE could insert titles > 100 chars that the FE can't edit (Zod rejects on form load).

**Fix:** Add `@Size(max = 100)` to the BE `title` field.

---

### 9. BE lacks `@Pattern` on `startTime`/`endTime`

| Side | Detail | File |
|------|--------|------|
| BE | Only `@NotEmpty` — no format validation annotation | `backend/.../dto/CalendarEventRequest.java` |
| BE | Invalid format causes `DateTimeParseException` in mapper → generic 500 | `backend/.../mapper/CalendarEventMapper.java` |

**Impact:** Bad time strings pass Bean Validation and produce 500 instead of 400 with a helpful message.

**Fix:** Add `@Pattern(regexp = "^(1[0-2]|0?[1-9]):[0-5][0-9] (AM|PM)$")` to `startTime` and `endTime`.

---

## Observations

1. **Mock-first development**: FE defaults to `USE_MOCK_API=true`, so the real HTTP path is never exercised in development. All breaking mismatches surface only when mocks are turned off.

2. **In-memory filtering**: BE loads all events via `findByFamily()` and filters with Java streams. Fine for family-sized datasets, won't scale to large volumes.

3. **No `familyId` in response**: `CalendarEventResponse` only includes `memberId`, not `familyId`. The FE doesn't need it since the authenticated user implies the family.

4. **Single DTO for create and update**: BE uses `CalendarEventRequest` for both POST and PUT. Clean, but means no partial update (PATCH) support.

5. **Error response shape compatible**: FE's `parseErrorResponse` reads `body.message` and `body.errors` — both fields exist on the BE's `ErrorResponse` record. The shape mismatch is less severe than initially assessed; the main issue is the status code mapping (Issue #3).

6. **BE build tool mismatch**: `backend/CLAUDE.md` says Gradle, but the project uses Maven (`mvnw` + `pom.xml`).

---

## Prompt Tracker

| Issue | Severity | Prompt | Status | PR |
|-------|----------|--------|--------|----|
| #1 Infinite reload loop | Breaking | `docs/prompts/fix-fe-be-wiring.md` | Resolved | [FE #83](https://github.com/joe-bor/FamilyHub/pull/83) |
| #2 URL path resolution | Breaking | `docs/prompts/fix-fe-be-wiring.md` | Resolved | [FE #83](https://github.com/joe-bor/FamilyHub/pull/83) |
| #3 Validation status code | Breaking | `docs/prompts/fix-fe-be-wiring.md` | Resolved | [FE #83](https://github.com/joe-bor/FamilyHub/pull/83) |
| #4 Time format mismatch | Breaking | `docs/prompts/fix-fe-be-wiring.md` | Resolved | [FE #83](https://github.com/joe-bor/FamilyHub/pull/83) |
| #5 `date` type mismatch | Risky | `docs/prompts/calendar-event-date-type-safety.md` | Resolved | [FE #87](https://github.com/joe-bor/FamilyHub/pull/87) |
| #6 `id` in update body | Risky | `docs/prompts/remove-id-from-update-request-bodies.md` | Resolved | [FE #93](https://github.com/joe-bor/FamilyHub/pull/93) |
| #7 `isAllDay` optionality | Risky | `docs/prompts/calendar-event-date-type-safety.md` | Resolved | [FE #87](https://github.com/joe-bor/FamilyHub/pull/87) |
| #8 `title` max length | Cosmetic | No prompt — BE fix | Resolved | [BE #8](https://github.com/joe-bor/family-hub-api/pull/8) |
| #9 Time format validation | Cosmetic | No prompt — BE fix | Resolved | [BE #8](https://github.com/joe-bor/family-hub-api/pull/8) |
