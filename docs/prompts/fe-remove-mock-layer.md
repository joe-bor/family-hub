# Task: Remove the Runtime Mock Layer

## Context

The FE was originally built mock-first — a runtime mock layer in `src/api/mocks/` let the app run without a backend. This caused alignment issues between FE and BE, and the mock layer became tech debt once the real backend was wired up.

As of PR #105, FE CI runs E2E tests against the real Spring Boot backend via Docker. The mock layer is no longer used in CI or production. It's dead code. Time to remove it.

**You are expected to form your own plan after exploring the codebase.** Verify every claim below by reading the actual code. Push back if you find something unexpected.

---

## What Gets Removed

### 1. The mock layer itself (`src/api/mocks/`)

The entire directory:
- `src/api/mocks/index.ts` — `USE_MOCK_API` toggle + barrel exports
- `src/api/mocks/auth.mock.ts` — mock auth handlers
- `src/api/mocks/calendar.mock.ts` — mock calendar handlers
- `src/api/mocks/family.mock.ts` — mock family handlers
- `src/api/mocks/delay.ts` — simulated network delays

Verify no unit tests import from this directory before deleting. A pre-deletion check found zero test files importing from `src/api/mocks/` — the unit tests use MSW (`src/test/mocks/`), which is completely separate and stays.

### 2. `USE_MOCK_API` branching in service files

Three service files have `if (USE_MOCK_API) { return mockHandler() }` in every method:
- `src/api/services/auth.service.ts`
- `src/api/services/calendar.service.ts`
- `src/api/services/family.service.ts`

Remove the mock imports and conditional branches. Each method becomes just the `httpClient` call. The `calendar.service.ts` has a `toCalendarEvent()` / `mapEventsResponse()` transformation that applies to both paths — keep that, it's real logic.

### 3. `VITE_USE_MOCK_API` env var — all references

- `frontend/.env` — remove the variable
- `frontend/.env.example` — remove the variable and any comments about mock mode
- `src/api/mocks/index.ts` — already deleted with the mock directory
- `src/api/index.ts` — exports `USE_MOCK_API`, remove that export
- `playwright.config.ts` — forwards `VITE_USE_MOCK_API` to webServer env, remove it
- `.github/workflows/ci.yml` — sets `VITE_USE_MOCK_API: "false"` on the E2E step, remove it

### 4. `E2E_REAL_API` / `USE_REAL_API` dual-mode toggle

This was a migration bridge so specs could run in mock or real mode. With mocks gone, there's only one mode.

- `e2e/helpers/test-helpers.ts` — remove `USE_REAL_API` export
- `e2e/helpers/test-helpers.ts` — the mock-seeding helpers (`seedAuth`, `seedFamily`, `seedEmptyCalendar`) become unused. Remove them and any supporting types/interfaces they use (e.g., `FamilyStorageData`). Keep helpers that are still used (e.g., `clearStorage`, `waitForHydration`, `waitForCalendarReady`, `createTestMember`, etc.) — verify before deleting.
- All spec files (`e2e/*.spec.ts`) — collapse `if (USE_REAL_API) { ... } else { ... }` blocks to just the real-API path
- `.github/workflows/ci.yml` — remove `E2E_REAL_API: "true"` from the E2E step env
- `package.json` — remove the `test:e2e:real` script (it becomes identical to `test:e2e`)

### 5. Docs and config cleanup

- `frontend/.env.example` — update comments to reflect that a running backend is required
- Add a note near the `test:e2e` script or in a README that E2E tests require `docker compose -f docker-compose.e2e.yml up -d` first

---

## What Stays

- **MSW test mocks** (`src/test/mocks/`) — used by Vitest unit tests, completely separate system
- **`httpClient`** (`src/api/client/`) — the real API client, untouched
- **Service files** — they stay, just without the mock branching
- **`docker-compose.e2e.yml`** — still needed to start the BE for E2E
- **`e2e/helpers/api-helpers.ts`** — the real-API seeders (`registerFamily`, `seedBrowserAuth`)
- **All unit tests and E2E test logic** — only the seeding/setup changes, not the test assertions

---

## What NOT to Do

- Do **not** touch MSW mocks (`src/test/mocks/`)
- Do **not** modify the backend or Docker config
- Do **not** refactor service files beyond removing the mock branching
- Do **not** add new features or change test assertions

---

## Git Workflow

1. **Create a new branch** from `main`: `chore/remove-mock-layer`
2. **Make atomic commits** using conventional format. Suggested breakdown:
   - `chore: remove runtime mock layer` — delete `src/api/mocks/`
   - `refactor(services): remove USE_MOCK_API branching` — clean up the 3 service files
   - `chore: remove VITE_USE_MOCK_API env var references` — env files, playwright config, CI workflow
   - `test(e2e): remove dual-mode toggle and mock seeders` — collapse specs to real-only, remove unused helpers
   - `docs: update env example and add E2E prerequisite note`
3. **Run unit tests** (`npm test -- --run`) — must pass, should be unaffected
4. **Run E2E tests locally** (`docker compose -f docker-compose.e2e.yml up -d` then `npm run test:e2e`) — must pass
5. **Run lint** (`npm run lint`) and **build** (`npm run build`)
6. **Open a PR** when complete

## Acceptance Criteria

- [ ] `src/api/mocks/` directory deleted
- [ ] No `USE_MOCK_API` references remain in service files
- [ ] No `VITE_USE_MOCK_API` references remain in env files, config, or CI
- [ ] No `USE_REAL_API` / `E2E_REAL_API` references remain in specs, helpers, or CI
- [ ] E2E specs have no `if/else` dual-mode blocks
- [ ] Mock-only helpers (`seedAuth`, `seedFamily`, `seedEmptyCalendar`) removed from `test-helpers.ts`
- [ ] `test:e2e:real` script removed from `package.json`
- [ ] Unit tests pass (`npm test -- --run`)
- [ ] E2E tests pass against real backend
- [ ] Lint passes (`npm run lint`)
- [ ] Build passes (`npm run build`)
