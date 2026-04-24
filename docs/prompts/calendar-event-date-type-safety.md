# Task: CalendarEvent Type Safety — Date & isAllDay

## Context

During a BE/FE alignment check for the CalendarEvents API, we identified two type safety issues in the frontend. The backend now has a working CalendarEvent CRUD API (Spring Boot, PUT for updates, "h:mm a" time format, "yyyy-MM-dd" date format).

**You are expected to form your own plan after exploring the codebase.** Do not take anything below as absolute truth — verify every claim by reading the actual code. If you disagree with the severity, approach, or even the premise of a finding, push back with evidence. Reference prior work in `CLAUDE.md`, `docs/TECHNICAL-DEBT.md`, and existing patterns before proposing changes.

---

## Issue #2: `CalendarEvent.date` Type Mismatch (High Priority)

### The Claim

The `CalendarEvent` interface declares `date: Date` but the real API returns a JSON string like `"2025-02-20"`. The HTTP client (`src/api/client/http-client.ts`) does plain `response.json()` with no transformation, so at runtime `event.date` is a string — not a Date object.

### Claimed Impact

Verify each of these yourself:

1. **Runtime crash** — `event-detail-modal.tsx` calls `format(event.date, "EEEE, MMMM d, yyyy")`. The date-fns `format()` function requires a Date object and will throw `RangeError: Invalid time value` when given a string.

2. **Off-by-one day bug** — All 4 calendar views (`daily-calendar.tsx`, `weekly-calendar.tsx`, `monthly-calendar.tsx`, `schedule-calendar.tsx`) do `new Date(event.date)` which parses ISO date strings as **UTC midnight**. West of UTC, this shows the wrong day. `CLAUDE.md` explicitly warns against this pattern under "Date/Time Handling".

3. **Mocks mask the bug** — The mock handlers (`src/api/mocks/calendar.mock.ts`) call `parseLocalDate()` to convert strings to real Date objects before returning. The real API won't do this. This means the bug is invisible during development with `USE_MOCK_API=true`.

4. **Someone already noticed** — `event-form-modal.tsx` has a defensive `instanceof Date` check with a string fallback, suggesting this mismatch was recognized before but not fully addressed.

5. **Optimistic update inconsistency** — `use-calendar.ts` optimistic update calls `parseLocalDate(updatedEvent.date)` converting to a real Date for the cache, but when `onSettled` refetches, the server response overwrites with a string. Brief type mismatch between optimistic and real data.

### Starting Points (Verify These Yourself)

These are hints — **read the actual files** before acting on them:

- `CalendarEvent` type: `src/lib/types/calendar.ts` (~line 1-10)
- HTTP client (no response transformation): `src/api/client/http-client.ts` (~line 115, just `response.json()`)
- Detail modal crash site: `src/components/calendar/components/event-detail-modal.tsx` (~line 64)
- Calendar view `new Date()` usage: `src/components/calendar/views/daily-calendar.tsx` (~line 149), similar in weekly/monthly/schedule
- Mock handler date conversion: `src/api/mocks/calendar.mock.ts` (~line 125)
- Defensive instanceof check: `src/components/calendar/components/event-form-modal.tsx` (~line 29)
- Optimistic update: `src/api/hooks/use-calendar.ts` (~line 102)
- Time utils (correct pattern): `src/lib/time-utils.ts` — `parseLocalDate()` and `formatLocalDate()`

### Your Job

1. **Verify each claim** by reading the files yourself
2. **Decide on the right fix approach.** Two options were floated — pick what's best for this codebase:
   - **Option A: Response transformer** — Add a transformation step that converts `date` strings to Date objects when responses come back from the API, so the `date: Date` type remains truthful
   - **Option B: Change type to string** — Update `CalendarEvent.date` to `string` and use `parseLocalDate()` at every consumption point
   - **Option C: Something else** you discover while exploring
3. **Fix all affected files** — not just the ones listed above; search exhaustively
4. **Align mock handlers** so they match real API behavior (return strings, not Date objects). Tests should catch bugs, not hide them.
5. **Fix or add tests** that would have caught this

---

## Issue #3: `isAllDay` Optionality (Low Priority — Assess If Worth Fixing)

### The Claim

FE types declare `isAllDay?: boolean` (optional, could be `undefined`) while the BE uses `boolean isAllDay` (Java primitive, defaults to `false`). At runtime this works because JSON omission + Java primitive default = `false`. But the types tell different stories.

### Your Job

1. **Search the codebase** for any code that relies on `isAllDay` being `undefined` vs `false` — patterns like `if (event.isAllDay !== undefined)`, `event.isAllDay ?? defaultValue`, optional chaining on isAllDay, etc.
2. **If nothing depends on the distinction**, this might just be a type alignment tweak (remove the `?`) or not worth changing at all
3. **Make a judgment call** — fix it, document it as tech debt, or explicitly leave it alone with reasoning in the PR body

---

## Git Workflow

1. **Create a new branch** from `main` using format: `fix/calendar-event-type-safety`
2. **Plan before coding** — explore all affected files, read existing patterns in `CLAUDE.md` and `docs/TECHNICAL-DEBT.md`, then form your approach
3. **Make atomic commits** at natural breakpoints using conventional commit format: `type(scope): description`
   - e.g., `fix(types): align CalendarEvent.date type with API response`
   - e.g., `fix(calendar): use parseLocalDate instead of new Date for event dates`
   - e.g., `fix(mocks): return date strings to match real API behavior`
4. **Continue working after each commit** — do not block for review
5. **Run tests** (`npm test -- --run`) and **lint** (`npm run lint`) before opening the PR. Fix any failures.
6. **Run the build** (`npm run build`) to catch type errors

---

## Before Finishing

1. **Update docs if warranted:**
   - `CLAUDE.md` — If your fix introduces or reinforces a pattern future agents/devs should follow (e.g., guidance on how API response dates should be handled, or updating existing Date/Time Handling section)
   - `docs/TECHNICAL-DEBT.md` — If you identify related debt worth tracking, or if this fix resolves something that belongs in the Completed Items table. Follow the existing format (move, don't copy; include PR number).
2. **Do NOT add unnecessary docs** — only update if there's genuine value for future contributors

---

## Pull Request

Open a PR when implementation is complete. The PR must include:

- **Title:** Short, under 70 characters
- **Body structure:**

```
## Summary
- What was fixed and why (1-3 bullets)

## Findings
- What you discovered during exploration
- Confirm or contradict the claims from this prompt
- If you chose a different approach than suggested, explain why

## Gotchas
- Anything surprising or non-obvious about the changes
- Things that could confuse someone reading the diff
  (e.g., "The mocks were converting strings to Dates which masked the bug — now they return strings to match real API behavior, which is why N test files changed")
- Timezone-related subtleties worth knowing

## Testing
- What was tested and how
- Commands run (test, lint, build)

## Test Plan
- [ ] Checklist items for manual/automated verification
```

---

## Acceptance Criteria

- [ ] `CalendarEvent.date` type matches what the real API actually returns
- [ ] No `new Date(event.date)` usage for date-only strings (use `parseLocalDate()` per CLAUDE.md)
- [ ] `event-detail-modal.tsx` handles the date correctly without crashing
- [ ] All calendar views handle dates timezone-safely
- [ ] Mock handlers return data matching real API shape (don't mask type issues)
- [ ] `isAllDay` optionality assessed and addressed (fix, document, or justified no-op)
- [ ] All tests pass (`npm test -- --run`)
- [ ] Lint passes (`npm run lint`)
- [ ] Build passes (`npm run build`)
- [ ] Docs updated where warranted
- [ ] PR opened with findings, gotchas, and test plan
