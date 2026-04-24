# Fix `isAllDay` optionality in CalendarEvent types

## Context

The BE returns `isAllDay` as a **primitive boolean** (always present, defaults `false`). The FE types incorrectly mark it as optional in the response type, which flows through to the domain model.

This was flagged during alignment checks and previously noted in `docs/prompts/calendar-event-date-type-safety.md` (Issue #3). It's cosmetic — `undefined` is falsy like `false` so nothing breaks at runtime — but it's a type inaccuracy worth cleaning up before we add more features.

## What to change

**File:** `src/lib/types/calendar.ts`

1. In `CalendarEvent` interface: change `isAllDay?: boolean` → `isAllDay: boolean`
2. The `CalendarEventResponse` type derives from `CalendarEvent` via `Omit`, so it inherits this fix automatically.
3. `CreateEventRequest` and `UpdateEventRequest` can keep `isAllDay?: boolean` since it's optional on the *request* side (BE defaults to `false` if omitted).

## Verify

- `npm run build` — ensure no type errors from the stricter type
- `npm run test` — all tests should pass; some test fixtures may need `isAllDay: false` added where they previously omitted it
- Check `src/test/fixtures/calendar.ts` — the `createTestEvent` factory and any hardcoded test events may need updating

## Branch & PR

- Branch: `fix/isallday-optionality`
- Commit type: `fix(types)`
- This is a small, self-contained change — one commit is fine
