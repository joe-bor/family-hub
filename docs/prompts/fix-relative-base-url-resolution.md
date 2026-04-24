# Task: Fix Relative Base URL Resolution in HTTP Client

## Context

The `httpClient` in `src/api/client/http-client.ts` uses `new URL(endpoint, baseUrl)` to build request URLs. This breaks when `baseUrl` is a relative path like `/api` because `new URL()` requires an absolute base URL.

This was introduced in the URL path resolution fix (commit `bcba8f4`, PR #83), which correctly normalized trailing slashes and stripped leading slashes — but only tested with absolute URLs like `http://localhost:8080/api`. Production uses `VITE_API_BASE_URL=/api` (relative, same-origin via nginx proxy), which causes `new URL("auth/register", "/api/")` to throw `TypeError: Invalid base URL`.

The error is thrown at line 79, **before** the try/catch that wraps the fetch call (line 92), so it propagates as an unhandled `TypeError` instead of an `ApiException`. This means every API call silently fails in production.

## Starting Points (Verify These Yourself)

- `src/api/client/http-client.ts` line 67 — `normalizedBaseUrl` is built from the raw `baseUrl`
- `src/api/client/http-client.ts` line 79 — `new URL(cleanEndpoint, normalizedBaseUrl)` throws when base is relative
- `src/api/client/http-client.ts` line 181 — singleton uses `import.meta.env.VITE_API_BASE_URL || "/api"`
- `src/api/client/http-client.test.ts` — existing URL resolution tests (all use absolute URLs)
- `.env` — currently `VITE_API_BASE_URL=/api`
- `.env.example` — documents `/api` as the default for same-origin

## What Needs to Change

Resolve relative base URLs against `window.location.origin` so `new URL()` always gets an absolute base. The fix should be in the `createHttpClient` function, near where `normalizedBaseUrl` is computed (line 67 area).

Something like:

```typescript
const normalizedBaseUrl = baseUrl.endsWith("/") ? baseUrl : `${baseUrl}/`;
const resolvedBaseUrl = normalizedBaseUrl.startsWith("http")
  ? normalizedBaseUrl
  : new URL(normalizedBaseUrl, window.location.origin).toString();
```

Then use `resolvedBaseUrl` instead of `normalizedBaseUrl` on line 79.

**However**, don't blindly copy this. Consider:
- What if `baseUrl` starts with `//` (protocol-relative)?
- Does this break the existing test suite that uses absolute URLs?
- Is `window.location.origin` available in the test environment (jsdom)?

## Tests

Add test cases for relative base URLs:

- `createHttpClient({ baseUrl: "/api" })` — should resolve against origin
- `createHttpClient({ baseUrl: "/api/" })` — trailing slash already present
- Existing absolute URL tests must still pass
- Consider the test environment — `window.location.origin` in jsdom is typically `http://localhost`

## Git Workflow

1. **Create a new branch** from `main`: `fix/relative-base-url-resolution`
2. Make the fix and add tests
3. **Run linting and tests**:
   - `npm run lint`
   - `npm run test -- --run`
4. Atomic commits:
   - `fix(api): resolve relative base URLs against window.location.origin`
   - `test(api): add relative base URL resolution tests`
5. **Open a PR** when complete

## Acceptance Criteria

- [ ] `createHttpClient({ baseUrl: "/api" })` resolves URLs correctly
- [ ] `createHttpClient({ baseUrl: "http://localhost:8080/api" })` still works (no regression)
- [ ] All existing tests pass
- [ ] New tests cover relative base URL scenarios
- [ ] Linting passes
