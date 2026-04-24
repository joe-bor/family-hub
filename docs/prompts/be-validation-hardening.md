# BE Task: Validation Hardening

## Context

A cross-repo alignment check compared FE Zod schemas against BE Jakarta annotations. The BE is missing several constraints that the FE enforces — meaning direct API calls (or a future different client) could bypass validation and store bad data.

The principle: **BE should not rely on FE for validation.** Every constraint the FE enforces should have a BE counterpart.

---

## Findings — Sorted by Priority

### 1. Time Format Validation (High)

**Current state:** `startTime` and `endTime` in `CalendarEventRequest.java` only have `@NotEmpty`. No format validation at the annotation level.

**Problem:** The mapper (`CalendarEventMapper.java` line 53) parses times with `DateTimeFormatter.ofPattern("h:mm a")`. If someone sends `"14:00"` or `"garbage"`, the `DateTimeParseException` is unhandled and bubbles up as a **500 Internal Server Error** instead of a **400 Bad Request**.

**Fix:** Add `@Pattern` on both time fields to validate 12-hour format before the mapper runs:

```java
@NotEmpty
@Pattern(regexp = "^(1[0-2]|[1-9]):[0-5][0-9] (AM|PM)$", message = "Time must be in 12-hour format (e.g., 2:00 PM)")
String startTime,

@NotEmpty
@Pattern(regexp = "^(1[0-2]|[1-9]):[0-5][0-9] (AM|PM)$", message = "Time must be in 12-hour format (e.g., 2:00 PM)")
String endTime,
```

**Verify:** Test with `"14:00"`, `"2:00PM"` (no space), `"13:00 PM"`, and `"2:00 pm"` (lowercase). The FE sends uppercase AM/PM — decide if you want case-insensitive matching.

**Also consider:** Even with the `@Pattern`, a defensive try-catch around `LocalTime.parse()` in the mapper is good practice — the regex and the formatter should agree, but belt-and-suspenders.

---

### 2. endTime Must Be After startTime (High)

**Current state:** No validation. The BE will happily save an event where endTime is before startTime.

**Problem:** The FE has a Zod `.refine()` that prevents this, but a direct API call could create invalid data.

**Fix:** Add a programmatic check in `CalendarEventService` (in both `addCalendarEvent` and `updateCalendarEvent`), after parsing the times:

```java
LocalTime start = CalendarEventMapper.parseTime(request.startTime());
LocalTime end = CalendarEventMapper.parseTime(request.endTime());
if (!end.isAfter(start)) {
    throw new IllegalArgumentException("End time must be after start time");
}
```

You'll need to handle `IllegalArgumentException` in `GlobalExceptionHandler` to return a 400. Or use a custom exception that's already mapped.

**Decision needed:** What about midnight-crossing events (e.g., 11 PM → 1 AM)? The FE doesn't support this — Zod just checks minutes-since-midnight. If the BE doesn't need it either, a simple `isAfter` check is fine.

---

### 3. Max Length Constraints (Medium)

**Current state:** Several string fields have no `@Size` annotation. The DB uses varchar(255) defaults, but no application-level enforcement.

| Field | DTO | FE Limit | BE Limit | Suggested |
|-------|-----|----------|----------|-----------|
| `title` | `CalendarEventRequest` | 100 | none | `@Size(max = 100)` |
| `location` | `CalendarEventRequest` | 255 (being added) | none | `@Size(max = 255)` |
| `name` | `FamilyMemberRequest` | 30 | none | `@Size(max = 30)` |
| `familyName` | `RegisterRequest` | 50 | none | `@Size(max = 50)` |
| `password` | `LoginRequest` + `RegisterRequest` | 100 | none (only min 8) | `@Size(max = 100)` |
| `name` | `FamilyRequest` | 50 | min 1 only | `@Size(min = 1, max = 50)` |

**Fix:** Add `@Size` annotations to each field. Example for `CalendarEventRequest`:

```java
@NotEmpty
@Size(max = 100, message = "Title must be 100 characters or less")
String title,
```

---

### 4. Username Format Validation (Low)

**Current state:** `LoginRequest` and `RegisterRequest` validate username with `@Size(min = 3, max = 20)` but no format constraint.

**Problem:** The FE enforces `[a-z0-9_]+` (lowercase alphanumeric + underscores). A direct API call could register `"ADMIN"`, `"user name"`, or `"user@name"`. These would work in the DB but could break FE assumptions (FE lowercases before sending).

**Fix (optional):** Add a `@Pattern` if you want BE parity:

```java
@Pattern(regexp = "^[a-z0-9_]+$", message = "Username can only contain lowercase letters, numbers, and underscores")
```

**Your call** — this is low risk since the FE gates it, and existing users were created through the FE. But it's the right thing for API self-sufficiency.

---

## Not a Code Task — Manual Verification

### Time Format Contract

The FE converts to 12-hour format before sending and converts back to 24-hour for display. The alignment doc (`docs/calendar-events-api-reference.md`) says the API transports times as `"h:mm a"`.

Quick verification against the running BE:

1. **Create** an event with `"2:00 PM"` / `"3:30 PM"` — does the response return those exact strings?
2. **GET** the event back — still `"2:00 PM"` / `"3:30 PM"`?
3. **Send `"14:00"`** — does it 400 (after adding `@Pattern`) or 500 (before)?
4. **Update** the event, change only the title — do times survive the round-trip in the same format?

This just confirms the mapper's `DateTimeFormatter.ofPattern("h:mm a", Locale.US)` is consistent across all operations. No code change needed if it passes.

---

## Summary Checklist

- [ ] Add `@Pattern` on `startTime` and `endTime` for 12-hour format
- [ ] Add endTime > startTime validation in service layer
- [ ] Handle the new validation error in `GlobalExceptionHandler` (return 400)
- [ ] Add `@Size(max = ...)` on: title (100), location (255), member name (30), familyName (50), password (100), family request name (50)
- [ ] Optional: Add `@Pattern` on username for `[a-z0-9_]+`
- [ ] Manually verify time format round-trip against running BE
