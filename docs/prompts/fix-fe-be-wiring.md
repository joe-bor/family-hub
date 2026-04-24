# Task: Fix FE ↔ BE Wiring Issues

## Context

We're wiring the frontend to the real Spring Boot backend for the first time. With `VITE_USE_MOCK_API=false`, the app is completely broken — it enters an infinite reload loop and no API calls reach the correct endpoints. There are four wiring issues that need to be fixed together so the FE can talk to the running BE.

**You are expected to verify every claim below by reading the actual code.** If you disagree with the analysis or find a better approach, push back with evidence.

---

## Issue #1: Infinite Reload Loop (Critical — App Won't Load)

### The Claim

When `VITE_USE_MOCK_API=false`, the app enters an infinite `window.location.reload()` loop and never renders.

**Root cause chain:**

1. `App.tsx` calls `useSetupComplete()` at the top of the component — before the `!isAuthenticated` guard can short-circuit rendering
2. `useSetupComplete()` calls `useFamily()` which fires `familyService.getFamily()` immediately
3. `useFamily()` uses `initialDataUpdatedAt: 0` which marks localStorage seed data as stale, triggering an immediate background refetch
4. With mocks off, this sends `GET /family` to the backend, which returns 401 (no auth token)
5. The `httpClient`'s `onUnauthorized` callback (`src/api/client/http-client.ts` ~line 175) does `localStorage.removeItem(token)` → `window.location.reload()`
6. On reload, step 1 repeats → infinite loop

### Starting Points (Verify Yourself)

- `App.tsx` (~line 90-96): `useSetupComplete()` called unconditionally at component top
- `useFamily()` in `src/api/hooks/use-family.ts` (~line 85-107): `queryFn: familyService.getFamily` fires immediately, `initialDataUpdatedAt: 0` forces refetch
- `familyService.getFamily()` in `src/api/services/family.service.ts` (~line 17-22): sends `GET /family` when mocks are off
- `httpClient` `onUnauthorized` in `src/api/client/http-client.ts` (~line 175-183): `window.location.reload()` on 401
- `useSetupComplete` in `src/api/hooks/use-family.ts`: derives from `useFamily()` — trace this

### Suggested Fix Direction

The `useFamily` query should not fire when the user isn't authenticated. Consider adding `enabled: isAuthenticated` to the query options — but think carefully about **where** that guard lives. Options include:

- Inside `useFamily()` itself (requires it to read auth state — creates a coupling)
- In `useSetupComplete()` or other derived hooks
- In `App.tsx` by restructuring so unauthenticated users never mount components that call `useFamily()`
- Making `onUnauthorized` smarter — e.g., don't reload if there was no token to begin with

Explore the tradeoffs yourself. The fix should handle:
- Fresh visit with no token (should show login, not loop)
- Expired token (should clear and show login, not loop)
- Legitimate authenticated requests that get 401 (session expired mid-use)

---

## Issue #2: URL Path Resolution (Critical — All Requests 404)

### The Claim

The FE service layer uses absolute paths (leading `/`) like `/calendar/events` with `new URL(endpoint, baseUrl)`. When `baseUrl` is `http://localhost:8080/api`, the leading `/` makes `new URL()` treat the endpoint as an absolute path from the origin, discarding the `/api` segment.

```
new URL("/calendar/events", "http://localhost:8080/api")
→ "http://localhost:8080/calendar/events"    // WRONG — /api dropped

new URL("calendar/events", "http://localhost:8080/api/")
→ "http://localhost:8080/api/calendar/events"  // Correct — needs trailing / AND no leading /
```

The BE controller is mapped to `@RequestMapping("/api/calendar/events")`. So every FE request will 404.

### Starting Points (Verify Yourself)

- `httpClient` URL construction: `src/api/client/http-client.ts` (~line 72): `new URL(endpoint, baseUrl)`
- `VITE_API_BASE_URL` in `frontend/.env`: `http://localhost:8080/api` (no trailing slash)
- Calendar service paths: `src/api/services/calendar.service.ts` — all start with `/calendar/events`
- Family service paths: `src/api/services/family.service.ts` — starts with `/family`
- Auth service paths: `src/api/services/auth.service.ts` — likely starts with `/auth`

### Suggested Fix Direction

Two reasonable approaches — pick what's cleanest:

**Option A: Fix the base URL + remove leading slashes**
- Change `VITE_API_BASE_URL` to `http://localhost:8080/api/` (add trailing slash)
- Remove leading `/` from all service endpoint paths (`calendar/events` instead of `/calendar/events`)
- This is the `new URL()` intended behavior — trailing slash makes the base a "directory"

**Option B: Prefix endpoints with `/api`**
- Change `VITE_API_BASE_URL` to `http://localhost:8080` (remove `/api`)
- Prefix all service endpoint paths with `/api/` (`/api/calendar/events`)
- Leading `/` works correctly here since the base URL has no path to lose

**Important:** Whichever option you pick, search for ALL service files that construct URLs. Don't fix calendar and miss family/auth. Also check if mock handlers or tests hardcode URLs that need updating.

---

## Issue #3: Validation Error Status Code Mapping (Medium — Validation Errors Misclassified)

### The Claim

The BE returns HTTP 400 for validation errors (`MethodArgumentNotValidException`). The FE `mapStatusToErrorCode()` only maps 422 to `VALIDATION_ERROR`. HTTP 400 falls through the default case and gets classified as `NETWORK_ERROR` (since `400 < 500`).

This means validation errors from the BE won't trigger the FE's validation error handling path — field-level errors from the `errors[]` array won't display properly.

### Starting Points (Verify Yourself)

- `mapStatusToErrorCode` in `src/api/client/api-error.ts` (~line 49-66): switch statement, no `case 400`
- `parseErrorResponse` in `src/api/client/http-client.ts` (~line 37-60): reads `body.errors` — this part is compatible with the BE's `ErrorResponse` shape

### Suggested Fix

Add `case 400` to the switch in `mapStatusToErrorCode` to map it to `VALIDATION_ERROR`. Alternatively, map both 400 and 422 — the FE should handle either.

Consider: is there any FE code that checks for `VALIDATION_ERROR` code specifically and shows field-level errors? If so, verify it will work correctly when triggered by a 400 response with the BE's error shape.

---

## Issue #4: Time Format Mismatch (Medium — Create/Update Will Fail)

### The Claim

The FE Zod schema validates `startTime`/`endTime` as 24-hour format (`"14:00"`). The BE parses them with `DateTimeFormatter.ofPattern("h:mm a")` expecting 12-hour format (`"2:00 PM"`). When the FE sends `"14:00"`, the BE throws `DateTimeParseException` which falls through to the generic 500 handler.

The API reference doc (`docs/calendar-events-api-reference.md`) states the API uses 12-hour format and the FE should convert before sending.

### Starting Points (Verify Yourself)

- Zod schema: `src/lib/validations/calendar.ts` — time regex validates 24h format
- Conversion utils: `src/lib/time-utils.ts` — `format24hTo12h()` and `format12hTo24h()` already exist
- Calendar service: `src/api/services/calendar.service.ts` — sends request body as-is, no conversion
- Create/update call site: `src/components/calendar/calendar-module.tsx` or similar — trace where the form data becomes a request
- Mock handler: `src/api/mocks/calendar.mock.ts` — check what format it expects/stores

### Suggested Fix Direction

Convert times at the service layer boundary — the service methods should `format24hTo12h()` on outbound requests and `format12hTo24h()` on inbound responses. This keeps the form and UI working in 24h internally while the API uses 12h.

**But verify first:** check if `calendar-module.tsx` already does this conversion before calling the mutation. The alignment check analysis found a reference to `format24hTo12h` in the component — read the actual code to see if conversion is already happening at the call site.

If conversion already happens at the call site, you may just need to ensure it's consistent and covers all paths (create AND update). If it's not happening, add it at the service layer.

---

## Git Workflow

1. **Create a new branch** from `main`: `fix/fe-be-wiring`
2. **Explore and plan** before coding — read all the files mentioned, trace the flows, verify the claims
3. **Make atomic commits** at natural breakpoints:
   - `fix(api): prevent useFamily query from firing before auth`
   - `fix(api): fix URL path resolution for real backend`
   - `fix(api): map HTTP 400 to VALIDATION_ERROR`
   - `fix(api): convert time format at service boundary`
   - Use your judgment on ordering and boundaries
4. **Run lint and tests after each commit:**
   - `npm run lint`
   - `npm run test -- --run`
5. **Verify the fix works end-to-end:** start the BE (`cd backend/family-hub-api && JWT_SECRET=dev-secret-key-for-local-testing ./mvnw spring-boot:run`), set `VITE_USE_MOCK_API=false` in `.env`, start the FE (`npm run dev`), and confirm:
   - App loads without infinite loop
   - Login/register screen appears
   - After registering, API calls reach the BE at correct URLs
   - Creating/updating events works with time format conversion
6. **Open a PR** when complete

---

## Acceptance Criteria

- [ ] App loads without infinite reload loop when `VITE_USE_MOCK_API=false` and no auth token exists
- [ ] Unauthenticated users see the login screen
- [ ] API requests hit the correct BE URLs (e.g., `http://localhost:8080/api/calendar/events`)
- [ ] HTTP 400 from BE is classified as `VALIDATION_ERROR` in the FE
- [ ] Times are sent as 12-hour format (`"2:00 PM"`) to the BE
- [ ] Times from BE responses are converted to 24-hour format for FE internal use
- [ ] All existing tests pass (update as needed)
- [ ] Lint passes
- [ ] Build passes (`npm run build`)
- [ ] Mock API mode (`VITE_USE_MOCK_API=true`) still works correctly

---

## PR Checklist

```
## Summary
- What was fixed and why

## Findings
- Confirm or contradict each claim from this prompt
- Any additional issues discovered during exploration

## Testing
- Commands run (test, lint, build)
- Manual verification with real BE running

## Test Plan
- [ ] Fresh visit (no token) → login screen, no reload loop
- [ ] Register new family → requests hit correct BE URLs
- [ ] Create event → time sent as 12h, saved correctly
- [ ] Update event → time round-trips correctly (24h form → 12h API → 24h display)
- [ ] Validation error → shows as VALIDATION_ERROR, not NETWORK_ERROR
- [ ] Mock mode still works (`VITE_USE_MOCK_API=true`)
- [ ] All automated tests pass
```
