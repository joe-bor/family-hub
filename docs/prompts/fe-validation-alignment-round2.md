# Task: Close Remaining FE Validation Gaps + Document Pattern

## Context

A previous alignment pass (`fe-validation-alignment.md`) fixed the username max length and location max length gaps. A second audit found two remaining minor gaps where FE Zod schemas don't enforce limits that the BE does. This prompt also asks you to document the validation alignment pattern so future modules (chores, meals, etc.) don't repeat the mock-first drift problem.

**Verify these claims yourself before acting.** Read the actual FE schemas and compare against the BE DTOs listed below.

---

## Gap 1: Member Email Missing Max Length

### The Claim

`createMemberProfileSchema` in `src/lib/validations/family.ts` validates email with `.email()` but no `.max(254)`. The BE's `FamilyMemberRequest.java` has `@Email @Size(max = 254)`. Extremely unlikely to hit in practice, but it's a gap.

Also check `familyMemberSchema` (same file, used for localStorage validation) — it has the same missing max length on email.

### What to Fix

Add `.max(254)` to the email field in both schemas, before the `.email()` check. Verify the correct Zod chaining order — `.max()` should come before `.email()` or after, whichever makes the error message clearest.

### Starting Points

- `src/lib/validations/family.ts` — `createMemberProfileSchema` email field
- `src/lib/validations/family.ts` — `familyMemberSchema` email field
- BE reference: `FamilyMemberRequest.java` — `@Email @Size(max = 254)`

---

## Gap 2: Member AvatarUrl Missing Max Length

### The Claim

The `familyMemberSchema` in `src/lib/validations/family.ts` has `avatarUrl: z.string().optional()` with no max length. The BE's `FamilyMemberRequest.java` has `@Size(max = 254)` on avatarUrl.

This field likely isn't user-editable via a form today, but the localStorage validation schema should still enforce it to catch corrupt data.

### What to Fix

Add `.max(254)` to the `avatarUrl` field in `familyMemberSchema`.

### Starting Points

- `src/lib/validations/family.ts` — `familyMemberSchema` avatarUrl field
- BE reference: `FamilyMemberRequest.java` — `@Size(max = 254)` on avatarUrl

---

## Task 3: Add a Validation Alignment Reference Comment

Add a comment block at the top of each validation file documenting which BE DTO it mirrors. This makes it easy for future developers (or agents) to verify alignment.

Example format (adapt to fit existing style):

```typescript
/**
 * Validation schemas for family forms.
 *
 * These constraints mirror the BE's Jakarta validation annotations:
 * - FamilyRequest.java: name @Size(1-50)
 * - FamilyMemberRequest.java: name @Size(max 30), color @NotNull,
 *   email @Email @Size(max 254), avatarUrl @Size(max 254)
 *
 * When adding new fields, check the corresponding BE DTO first.
 */
```

Add similar comments to:
- `src/lib/validations/auth.ts` — mirrors `LoginRequest.java`, `RegisterRequest.java`
- `src/lib/validations/family.ts` — mirrors `FamilyRequest.java`, `FamilyMemberRequest.java`
- `src/lib/validations/calendar.ts` — mirrors `CalendarEventRequest.java`

Keep comments concise — just the DTO name and constraint summary, not a full copy of the annotations.

---

## What NOT to Change

- Do **not** touch BE code
- Do **not** refactor or reorganize existing schemas
- Do **not** add validations beyond what the BE enforces

---

## Git Workflow

1. **Create a new branch** from `main`: `fix/validation-gaps-round2`
2. **Make atomic commits** using conventional format:
   - `fix(validations): add email max length to member schemas`
   - `fix(validations): add avatarUrl max length to member schema`
   - `docs(validations): add BE DTO alignment references`
3. **Run tests** (`npm test -- --run`) and **lint** (`npm run lint`) — fix any failures
4. **Run the build** (`npm run build`) to catch type errors
5. **Open a PR** when complete

## Acceptance Criteria

- [ ] `createMemberProfileSchema` email has `.max(254)`
- [ ] `familyMemberSchema` email has `.max(254)`
- [ ] `familyMemberSchema` avatarUrl has `.max(254)`
- [ ] All three validation files have a comment referencing their BE DTO counterpart
- [ ] All tests pass (`npm test -- --run`)
- [ ] Lint passes (`npm run lint`)
- [ ] Build passes (`npm run build`)
