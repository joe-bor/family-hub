# Calendar Events API Reference

Backend implementation reference extracted from the frontend codebase.

---

## Endpoints

| Method   | Path                     | Description         |
| -------- | ------------------------ | ------------------- |
| `GET`    | `/calendar/events`       | List events         |
| `GET`    | `/calendar/events/:id`   | Get single event    |
| `POST`   | `/calendar/events`       | Create event        |
| `PUT`    | `/calendar/events/:id`   | Full update         |
| `DELETE` | `/calendar/events/:id`   | Delete event        |

---

## Response Wrapper

All responses (except DELETE) use this envelope:

```json
{
  "data": <T>,
  "message": "optional string"
}
```

DELETE returns **204 No Content** with no body.

---

## Data Shapes

### CalendarEvent (response)

```typescript
{
  id: string;            // UUID
  title: string;
  startTime: string;     // "9:00 AM" — 12-hour format
  endTime: string;       // "5:00 PM" — 12-hour format
  date: string;          // "yyyy-MM-dd" (e.g. "2026-02-15")
  memberId: string;      // UUID — foreign key to family member
  isAllDay: boolean;     // always present, defaults false
  location?: string;
  endDate?: string;      // "yyyy-MM-dd" — only present for multi-day all-day events
}
```

### CreateEventRequest (POST body)

```typescript
{
  title: string;
  startTime: string;     // "2:00 PM" — 12-hour
  endTime: string;       // "3:00 PM" — 12-hour
  date: string;          // "yyyy-MM-dd"
  memberId: string;
  isAllDay?: boolean;
  location?: string;
  endDate?: string;      // "yyyy-MM-dd" — only valid when isAllDay=true, must be >= date
}
```

### UpdateEventRequest (PUT body)

Full replacement — send the complete object.

```typescript
{
  title: string;
  startTime: string;     // 12-hour
  endTime: string;       // 12-hour
  date: string;          // "yyyy-MM-dd"
  memberId: string;
  isAllDay: boolean;
  location?: string;     // optional — omit or null to clear
  endDate?: string;      // "yyyy-MM-dd" — only valid when isAllDay=true, must be >= date
}
```

> `id` is not included in the body — it's only used in the URL path param and for optimistic cache matching on the FE.

### GET /calendar/events — Query Parameters

| Param       | Type   | Required | Description                          |
| ----------- | ------ | -------- | ------------------------------------ |
| `startDate` | string | No       | Inclusive lower bound (`yyyy-MM-dd`) |
| `endDate`   | string | No       | Inclusive upper bound (`yyyy-MM-dd`) |
| `memberId`  | string | No       | Filter by family member UUID         |

The FE typically fetches by date range and does additional member filtering client-side,
so `memberId` may not be used heavily — but it should work.

---

## Error Responses

The FE maps HTTP status codes to internal error codes:

| Status | FE Error Code      | When to return                    |
| ------ | ------------------ | --------------------------------- |
| 401    | `UNAUTHORIZED`     | Missing or invalid auth token     |
| 403    | `FORBIDDEN`        | Event belongs to different family |
| 404    | `NOT_FOUND`        | Event ID doesn't exist            |
| 409    | `CONFLICT`         | Duplicate or scheduling collision |
| 422    | `VALIDATION_ERROR` | Invalid input fields              |
| 5xx    | `SERVER_ERROR`     | Unexpected server failure         |

Error response body should include a `message` field:

```json
{
  "message": "Event not found"
}
```

---

## Gotchas & Implementation Notes

### 1. Time format is 12-hour

The API transports times as **12-hour strings** (`"2:00 PM"`, `"9:00 AM"`).
The FE form uses 24-hour internally but converts before sending requests.
Store and return 12-hour format.

### 2. Dates must be yyyy-MM-dd strings — NOT full ISO timestamps

The FE uses `parseLocalDate()` to avoid timezone drift.

- Return: `"2026-02-15"`
- Do NOT return: `"2026-02-15T00:00:00Z"` (will shift dates across timezones)

### 3. DELETE returns void

Every other mutation returns `{ data, message? }`. DELETE returns just 204.

### 4. PUT is a full replacement

Send all fields in the request body. The backend replaces the entire entity — omitted fields are not preserved.

### 5. Date range filtering is inclusive (with multi-day overlap)

`startDate=2026-02-01&endDate=2026-02-28` returns events where
`event.date <= rangeEnd AND coalesce(event.endDate, event.date) >= rangeStart`.

This ensures multi-day events that overlap the range are included even if their start date is before the range.

### 6. memberId is a foreign key

Events reference family members by `memberId`. The backend should validate that
the member exists and belongs to the authenticated user's family.

### 8. endDate validation rules

- `endDate` is only valid when `isAllDay` is `true` — BE returns 400 if sent with a timed event
- `endDate` must be `>= date` — BE returns 400 if end date is before start date
- If `endDate == date` (single-day all-day), BE normalizes to `null` for storage
- FE Zod schema enforces the same rules client-side

### 7. Success messages on mutations

The mock returns these — they're optional but part of the `ApiResponse` type:

- POST: `"Event created successfully"`
- PUT: `"Event updated successfully"`

---

## FE Files for Cross-Reference

| Purpose              | Path                                                 |
| -------------------- | ---------------------------------------------------- |
| Types                | `frontend/src/lib/types/calendar.ts`                 |
| API response type    | `frontend/src/lib/types/api-response.ts`             |
| Service layer        | `frontend/src/api/services/calendar.service.ts`      |
| TanStack Query hooks | `frontend/src/api/hooks/use-calendar.ts`             |
| Mock handlers        | `frontend/src/api/mocks/calendar.mock.ts`            |
| Validation schema    | `frontend/src/lib/validations/calendar.ts`           |
| Time utilities       | `frontend/src/lib/time-utils.ts`                     |
| HTTP client          | `frontend/src/api/client/http-client.ts`             |
| Error handling       | `frontend/src/api/client/api-error.ts`               |
| Calendar module      | `frontend/src/components/calendar/calendar-module.tsx`|

---

## Status

**Fully aligned and deployed** (FE + BE live at https://familyhub.joe-bor.me as of 2026-02-28).
No open items — all identified alignment issues are resolved.

---

## Resolved

Items that have been verified or fixed, with links to the PRs for traceability.

| Issue | PR | Details |
|-------|-----|---------|
| FE uses PUT (not PATCH) for updates | [FE #81](https://github.com/joe-bor/FamilyHub/pull/81) | `httpClient.put()` aligns with full-replacement contract |
| PUT sends full object, mock handlers use true PUT semantics | [FE #86](https://github.com/joe-bor/FamilyHub/pull/86) | Fixed mock `??` fallback (was PATCH semantics), added 4 PUT verification tests + UI integration test |
| CalendarEvent.date type safety | [FE #87](https://github.com/joe-bor/FamilyHub/pull/87) | Service-layer mapper converts string → Date via `parseLocalDate()`. Mocks return strings to match real API. |
| FE validation alignment | [FE #89](https://github.com/joe-bor/FamilyHub/pull/89) | Username max 30→20, location max 255, boundary tests added |
| BE validation hardening | [BE #8](https://github.com/joe-bor/family-hub-api/pull/8) | @Pattern on times, @Size on 6 DTOs, startTime>endTime guard, `String`→`LocalDate` for date fields |
| BE 12h time format verified | Manual check | Mapper uses `DateTimeFormatter.ofPattern("h:mm a")` — confirmed consistent |
| URL path resolution | [FE #83](https://github.com/joe-bor/FamilyHub/pull/83) | Base URL normalized, leading `/` stripped |
| HTTP 400 mapped to VALIDATION_ERROR | [FE #83](https://github.com/joe-bor/FamilyHub/pull/83) | `mapStatusToErrorCode` handles 400 |
| Infinite reload loop in useFamily | [FE #83](https://github.com/joe-bor/FamilyHub/pull/83) | Auth guard added, only reloads on session expiry |
| Remove `id` from UpdateEventRequest body | [FE #93](https://github.com/joe-bor/FamilyHub/pull/93) | `id` removed from PUT body types; still used for URL path + optimistic cache |
| Externalize CORS origins | [BE #9](https://github.com/joe-bor/family-hub-api/pull/9) | CORS origins now read from config/env var; dev defaults to localhost |
| Phase A: Fix all-day event rendering | [FE #110](https://github.com/joe-bor/FamilyHub/pull/110) | Dedicated all-day sections in daily/weekly views, visual distinction in monthly/schedule, "All day" labels |
| Phase B: Add endDate + multi-day support (BE) | [BE #18](https://github.com/joe-bor/family-hub-api/pull/18), [BE #19](https://github.com/joe-bor/family-hub-api/pull/19) | V2 migration, entity/DTO/mapper/service updates, cross-field validation (endDate requires isAllDay, must be >= date), date range overlap filtering |
| Phase B: Add endDate + multi-day support (FE) | [FE #111](https://github.com/joe-bor/FamilyHub/pull/111) | Types, Zod schema, end date picker in form, multi-day rendering across all views, service layer mapping |
