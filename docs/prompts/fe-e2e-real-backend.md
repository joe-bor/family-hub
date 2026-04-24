# Task: Run FE E2E Tests Against Real Backend in CI

## Context

We're eliminating the FE's dependency on the runtime mock layer (`src/api/mocks/`) for E2E tests. Currently, Playwright E2E tests run against the mock handlers (via `USE_MOCK_API=true`) and seed test state by writing directly to localStorage. This mock-first approach has caused alignment issues between FE and BE in the past.

The BE now publishes a Docker image to GHCR (`ghcr.io/joe-bor/family-hub-api`) on every merge to `main`. This image runs the full Spring Boot API with H2 in-memory database — no external dependencies needed.

**You are expected to form your own plan after exploring the codebase.** Read the existing CI workflow, E2E helpers, Playwright config, Vite config, and mock layer before making changes. Push back if you disagree with the approach.

**IMPORTANT:** This task depends on the GHCR image being available. Verify `ghcr.io/joe-bor/family-hub-api` exists and is pullable before starting implementation. If it doesn't exist yet, stop and flag it.

---

## What We Want

Three things change:

### 1. CI Workflow — Start Real BE Before E2E

Update `.github/workflows/ci.yml` so that before E2E tests run, CI:
- Pulls the BE Docker image from GHCR
- Starts it with appropriate env vars (JWT_SECRET, dev profile for H2)
- Waits for the health check to pass (`/api/health`)
- Runs E2E tests with `VITE_USE_MOCK_API=false`
- Tears down the container after tests complete

You'll need to decide the cleanest way to run the container in CI — a `docker-compose.ci.yml` that references the GHCR image, a plain `docker run`, or another approach. The root repo has a `docker-compose.yml` that builds from source, but CI should pull the pre-built image instead.

#### Things to Consider

- The Vite dev server (started by Playwright) needs to proxy `/api` requests to the BE container. Check `vite.config.ts` for an existing proxy config, or add one.
- The BE uses H2 by default in the `dev` profile — no database setup needed.
- JWT_SECRET must be a valid base64-encoded string. The test secret from `application-test.yml` works: `dGVzdC1zZWNyZXQta2V5LXRoYXQtaXMtbG9uZy1lbm91Z2gtZm9yLWhtYWMtc2hhLTI1Ng==`
- Consider whether GHCR pull needs authentication in CI (public repos may not need it).

### 2. E2E Test Helpers — Seed via API Calls Instead of localStorage

The existing helpers in `e2e/helpers/test-helpers.ts` write directly to localStorage to fake auth state and test data. These need to change to make real API calls:

| Current Helper | What It Does Now | What It Should Do |
|---------------|-----------------|-------------------|
| `seedAuth()` | Writes a fake JWT to localStorage | Call `POST /api/auth/register` to create a test family, store the real JWT |
| `seedFamily()` | Writes family data to Zustand persist key | Merged into seedAuth — registration creates the family |
| `seedEmptyCalendar()` | Writes a sentinel event to mock storage key | Not needed — real DB starts empty |
| (new) | — | Helper to create calendar events via `POST /api/calendar/events` for tests that need pre-existing events |

Each E2E test should get a **fresh, isolated state**. Since the BE uses H2 in-memory, the simplest approach is to register a unique family per test (e.g., `testuser_${Date.now()}`). This avoids cross-test contamination without needing to reset the database.

### 3. Playwright/Vite Config

- `playwright.config.ts` — the webServer env should include `VITE_USE_MOCK_API: "false"` so the app uses `httpClient` instead of mock handlers
- `vite.config.ts` — may need a dev server proxy to forward `/api` to `http://localhost:8080`
- Verify the app works end-to-end: Playwright → Vite → httpClient → real BE → H2

### Starting Points

- `.github/workflows/ci.yml` — current FE CI workflow
- `e2e/helpers/test-helpers.ts` — current localStorage-based seeders
- `e2e/calendar-crud.spec.ts` — example E2E test to see how seeders are used
- `playwright.config.ts` — webServer config with `VITE_E2E: "true"`
- `vite.config.ts` — check for existing proxy config
- `src/api/mocks/` — the mock layer being replaced (for understanding, not modification)
- Root `docker-compose.yml` at `../../docker-compose.yml` — reference for how the BE container is configured (env vars, healthcheck, ports)

---

## What NOT to Change

- Do **not** remove the mock layer (`src/api/mocks/`) in this PR — that's a separate task after this lands and is verified
- Do **not** modify BE code
- Do **not** change unit tests — they use MSW (`src/test/mocks/`), which is separate and stays
- Do **not** change the Lighthouse or build steps in CI

---

## Git Workflow

1. **Create a new branch** from `main`: `feat/e2e-real-backend`
2. **Make atomic commits** using conventional format. Suggested breakdown:
   - `ci: add real backend container to e2e pipeline`
   - `test(e2e): convert seeders from localStorage to API calls`
   - `test(e2e): update playwright config for real backend`
3. **Run E2E tests locally** against the real BE (via `docker compose up api` from root) to verify before pushing
4. **Open a PR** when complete

## Acceptance Criteria

- [ ] CI starts the real BE container from GHCR before E2E tests
- [ ] CI waits for BE health check before running Playwright
- [ ] E2E test helpers seed data via real API calls (register, login, create events)
- [ ] `VITE_USE_MOCK_API=false` is set for E2E runs
- [ ] Vite proxies `/api` to the BE container
- [ ] All existing E2E tests pass against the real backend
- [ ] CI tears down the BE container after tests
- [ ] Unit tests are unaffected (still use MSW)
- [ ] The mock layer (`src/api/mocks/`) is untouched

## PR Review Checklist

- [ ] No hardcoded test data that assumes mock behavior (e.g., `event-0-1` style IDs)
- [ ] Each test gets isolated state (unique family per test, or DB reset between tests)
- [ ] CI doesn't leak secrets (JWT_SECRET can be hardcoded for H2 dev — it's not a real secret)
- [ ] Failure mode is clean — if BE container fails to start, CI fails fast with a clear error
