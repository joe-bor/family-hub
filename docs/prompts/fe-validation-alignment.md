# Task: Align FE Zod Validation Schemas with BE Constraints

## Context

A cross-repo alignment check compared FE Zod schemas against BE Jakarta annotations field-by-field. Two FE-side issues were found that need fixing. The BE side is handled separately.

**You are expected to verify these claims yourself before acting.** Read the actual code, confirm the gaps, and push back if you disagree.

---

## Issue 1: Username Max Length — FE allows 30, BE allows 20

### The Claim

`usernameSchema` in `src/lib/validations/auth.ts` (line 16) sets `.max(30)`. The BE's `LoginRequest.java` and `RegisterRequest.java` both use `@Size(max = 20)`. A user could type a 25-character username, pass FE validation, submit, and get a 400 from the BE.

### What to Fix

Change `.max(30, ...)` to `.max(20, ...)` in `usernameSchema`. Update the error message accordingly. This schema is shared by both the login and register forms via `loginFormSchema` and `credentialsFormSchema`, so a single change fixes both.

### Starting Points

- `src/lib/validations/auth.ts` ~line 16 — the `usernameSchema` with `.max(30)`
- Check for any tests that assert on the max length or error message

---

## Issue 2: Location Has No Max Length

### The Claim

`eventFormSchema` in `src/lib/validations/calendar.ts` (line 44) defines `location` as `z.string().optional()` with no length constraint. The BE also has no constraint (varchar 255 default). A user could paste a very long string into the location field.

### What to Fix

Add `.max(255, "Location must be 255 characters or less")` to the `location` field in `eventFormSchema`. This matches the DB column's varchar(255) default.

The chain should be: `z.string().max(255, "Location must be 255 characters or less").optional()` — verify the correct Zod ordering for optional + max.

### Starting Points

- `src/lib/validations/calendar.ts` ~line 44 — `location: z.string().optional()`
- Check if the location field has any other consumers beyond the event form

---

## What NOT to Change

- Do **not** touch BE code — BE validation fixes are handled separately
- Do **not** change any other validation rules — only the two issues above
- Do **not** add validations that don't correspond to a real BE constraint

---

## Git Workflow

1. **Create a new branch** from `main`: `fix/validation-alignment`
2. **Make atomic commits** using conventional format:
   - `fix(validations): align username max length with backend (30 → 20)`
   - `fix(validations): add location max length to event form schema`
3. **Run tests** (`npm test -- --run`) and **lint** (`npm run lint`) — fix any failures
4. **Run the build** (`npm run build`) to catch type errors
5. **Update tests** if any assert on the old max length or error message
6. **Open a PR** when complete

## Acceptance Criteria

- [ ] `usernameSchema` max changed from 30 to 20
- [ ] Error message updated to reflect 20-char limit
- [ ] `location` field has `.max(255)` constraint
- [ ] All tests pass (`npm test -- --run`)
- [ ] Lint passes (`npm run lint`)
- [ ] Build passes (`npm run build`)
