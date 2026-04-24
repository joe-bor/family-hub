# Task: Remove `id` from Update Request Body Types

## Context

Our frontend API layer has an anti-pattern where `UpdateEventRequest` and `UpdateMemberRequest` include `id` as a field in the request body, even though the `id` is already in the URL path parameter (`PATCH /calendar/events/:id`). This is redundant and violates REST conventions — the URL identifies the resource, the body describes the mutation.

The same pattern exists in **both** the calendar and family modules. Both should be cleaned up.

## Starting Points (Verify These Yourself)

These are hints to help you get oriented — **do not take them at face value**. Read the actual code, trace the usage, and confirm before making changes.

### Calendar Module

- `UpdateEventRequest` type in `src/lib/types/calendar.ts` (~line 30) — has `id: string` as a required field
- `calendarService.updateEvent()` in `src/api/services/calendar.service.ts` (~line 42) — uses `request.id` to build the URL path AND sends the full `request` as the body
- `calendarMockHandlers.updateEvent()` in `src/api/mocks/calendar.mock.ts` (~line 240) — uses `request.id` to find the event and in error messages
- `useUpdateEvent` optimistic update in `src/api/hooks/use-calendar.ts` (~line 82) — `onMutate` callback receives the full mutation variables and uses `updatedEvent.id` to match events in the cache
- `handleUpdateEvent` in `src/components/calendar/calendar-module.tsx` (~line 175) — constructs the request with `id: editingEvent.id`
- MSW test handler in `src/test/mocks/handlers.ts` (~line 185) — **already** uses `Omit<UpdateEventRequest, "id">` for the body and gets `id` from URL params. This is a clue that someone already recognized the issue here.

### Family Module

- `UpdateMemberRequest` type in `src/lib/types/family.ts` (~line 129) — same pattern, `id: string` required
- `familyService.updateMember()` in `src/api/services/family.service.ts` (~line 49) — same pattern as calendar
- Mock handler and hooks likely follow the same pattern — verify this yourself

## What Needs to Change

The core idea: separate the `id` (used for routing) from the body (used for the mutation payload). The mutation variables wrapper still needs access to `id` for optimistic cache matching.

A common pattern is:

```typescript
// Type: body no longer has id
interface UpdateEventRequest {
  title?: string;
  startTime?: string;
  // ... no id
}

// Service: accepts id separately
async updateEvent(id: string, request: UpdateEventRequest): Promise<...> {
  return httpClient.patch(`/calendar/events/${id}`, request);
}

// Hook: mutation variables include id alongside body
// The onMutate callback still has access to id for cache matching
```

**However**, don't blindly follow this suggestion. Research the actual usage, understand how TanStack Query's `mutationFn` receives variables, and determine the cleanest approach. Consider how `onMutate`, `onError`, and `onSettled` callbacks access the mutation variables.

## Important: Do Your Own Research

- **Search exhaustively** for all usages of `UpdateEventRequest` and `UpdateMemberRequest` across the entire codebase. Use grep/search — don't rely only on the files listed above.
- **Check tests** — unit tests, integration tests, E2E tests, test fixtures, and MSW handlers may all reference these types or construct objects matching their shape.
- **Check barrel exports** — these types are likely re-exported through index files.
- **Understand the optimistic update flow** before changing it. Read through the full `onMutate` → `onError` → `onSettled` lifecycle in the hooks.
- **Look for other update request types** that might have the same pattern beyond calendar and family.
- **Check if components destructure or spread these types** in ways that would break.

## Git Workflow

1. **Create a new branch** from `main` using the format: `refactor/remove-id-from-update-request-bodies`
2. **Plan your implementation** using plan mode before writing any code. The plan should cover:
   - All files that need to change (discovered through your own research)
   - The order of changes (type changes first, then consumers)
   - How tests need to be updated
   - Any risks or edge cases
3. **Make atomic commits** at natural breakpoints using conventional commit format:
   - `refactor(types): remove id from UpdateEventRequest and UpdateMemberRequest`
   - `refactor(api): update service methods to accept id as separate parameter`
   - `refactor(hooks): update mutation hooks for separated id parameter`
   - etc. — use your judgment on commit boundaries
4. **Run linting and tests after each commit** to ensure nothing is broken:
   - `npm run lint`
   - `npm run test -- --run` (non-watch mode)
   - Fix any issues before moving to the next commit
5. **Update documentation** where relevant:
   - `frontend/CLAUDE.md` — if the API layer patterns or service signatures change in a way that affects developer guidance
   - `frontend/docs/TECHNICAL-DEBT.md` — add this as a completed item with the PR number once the PR is created. Follow the existing format in that doc (move, don't copy; include PR number)
6. **Open a PR** when implementation is complete using `gh pr create`

## Acceptance Criteria

- [ ] `id` is no longer a field in `UpdateEventRequest` or `UpdateMemberRequest` types
- [ ] Service methods accept `id` as a separate parameter from the request body
- [ ] The HTTP request body sent to the server does NOT include `id`
- [ ] Optimistic updates still work correctly (cache matching by id)
- [ ] All existing tests pass (update test code as needed)
- [ ] MSW handlers continue to work correctly
- [ ] Mock handlers continue to work correctly
- [ ] Linting passes with no new warnings
- [ ] TECHNICAL-DEBT.md is updated with the completed item
- [ ] CLAUDE.md is updated if service method signatures in the documentation changed

---

## Analysis: Optimistic Update Data Flow

This section documents the full TanStack Query mutation lifecycle for both modules. Use this to understand the impact of the refactor and what to watch for during PR review.

### How TanStack Query Mutations Pass Data

`useMutation` has one type for the **variables** — the single argument passed to `.mutate()`. Every callback (`onMutate`, `onError`, `onSuccess`, `onSettled`) receives those same variables.

### Current Flow: Calendar (`useUpdateEvent`)

**Step 1 — Component calls `.mutate()`** (`calendar-module.tsx:175-185`):
```typescript
const request: UpdateEventRequest = {
  id: editingEvent.id,        // ← id is here
  title: formData.title,
  startTime: format24hTo12h(formData.startTime),
  // ...
};
updateEvent.mutate(request);   // ← entire object = mutation variables
```

**Step 2 — `mutationFn` sends HTTP request** (`use-calendar.ts:80`):
```typescript
mutationFn: calendarService.updateEvent,
// which does: httpClient.patch(`/calendar/events/${request.id}`, request)
// id in URL AND body
```

**Step 3 — `onMutate` runs BEFORE the HTTP request** (optimistic update, `use-calendar.ts:82-110`):
```typescript
onMutate: async (updatedEvent) => {   // ← same object from .mutate()
  // snapshots current cache for rollback
  // then:
  old.data.map((event) =>
    event.id === updatedEvent.id      // ← uses id to FIND the event
      ? {
          ...event,
          ...updatedEvent,            // ← spreads ALL fields including id
          date: updatedEvent.date
            ? parseLocalDate(updatedEvent.date)
            : event.date,
        }
      : event,
  );
```

**Step 4 — `onError` rolls back** (`use-calendar.ts:114-121`):
```typescript
onError: (error, _updatedEvent, context) => {
  // restores cache from snapshot, doesn't need id
}
```

**Step 5 — `onSettled` refetches** (`use-calendar.ts:123-125`):
```typescript
onSettled: () => {
  queryClient.invalidateQueries({ queryKey: calendarKeys.events() });
  // replaces optimistic data with real server response
}
```

### Current Flow: Family (`useUpdateMember`)

The family hook (`use-family.ts:321-389`) is **more involved** — it uses `request.id` in three places:

1. **`onMutate`** (line 339): `m.id === request.id` — cache matching to find the member
2. **`onSuccess`** (line 377): `m.id === request.id ? response.data : m` — replaces optimistic version with real server response
3. **`onError`** (line 361): rollback from snapshot (doesn't need id)

Note that `onSuccess` in the family hook does a **second cache update** using `request.id` — the calendar hook does not do this. This is an extra place where `id` must remain accessible to the hook.

### Refactor Approaches

**Option A: Wrapper type**
```typescript
type UpdateEventVariables = {
  id: string;
  body: UpdateEventRequest;  // id removed from this type
};

mutationFn: ({ id, body }) => calendarService.updateEvent(id, body),

onMutate: async ({ id, body }) => {
  // event.id === id for matching
  // ...body for field updates
}
```

**Option B: Flat destructure (less disruptive)**
```typescript
// mutation variables are still a flat object with id + fields
mutationFn: ({ id, ...body }) => calendarService.updateEvent(id, body),

onMutate: async (variables) => {
  // variables.id for matching, spread rest for fields
}
```

Option B is less disruptive — the call site in `calendar-module.tsx` barely changes since it already constructs this exact shape. The key difference is the **service method** now takes `(id, body)` separately and only sends `body` over the wire.

### PR Review Checklist

When reviewing the PR, verify these specifically:

1. **Mutation variables type** — does `onMutate` still have access to `id`? It needs it for cache matching in both calendar and family hooks.
2. **Service method signature** — does it accept `(id, body)` separately and only send `body` in the HTTP request? The `id` should only appear in the URL path.
3. **`onMutate` spread** — when it does `{ ...event, ...updatedFields }`, does `id` leak into the spread? Harmless if it does (same value), but cleaner if destructured out.
4. **`onSuccess` in `useUpdateMember`** — line 377 uses `request.id` to replace optimistic data with server response. This must keep working after the refactor.
5. **Call sites in components** — do `handleUpdateEvent` and the family equivalent still construct the object naturally without awkward reshaping?
6. **MSW test handlers** — `handlers.ts:185` already uses `Omit<UpdateEventRequest, "id">`. After the refactor, the `Omit` wrapper should no longer be needed since `id` won't be in the type. Verify the handler was simplified.
7. **Mock handlers** — `calendar.mock.ts` and `family.mock.ts` handlers currently receive the full request with `id`. After the refactor, the mock service method signature should match the real service (separate `id` param).

### Decision Status

**Status: Deferred** — The Spring Boot backend can safely ignore the extra `id` field in the request body. This refactor is a code quality improvement, not a bug fix. Pursue it when there's bandwidth, not as a blocker for backend integration.
