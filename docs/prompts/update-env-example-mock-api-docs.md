# Task: Update .env.example Mock API Documentation

## Context

The `VITE_USE_MOCK_API` comment in `.env.example` says:

> IMPORTANT: Currently only calendar events go through this toggle. Family data always uses localStorage via Zustand persist.

This is outdated. All three services — auth, calendar, and family — respect the `USE_MOCK_API` toggle and call real BE endpoints when it's `false`. Family data lives in TanStack Query (not Zustand), with localStorage used only as a cache seed for instant startup.

## What Needs to Change

Update the `VITE_USE_MOCK_API` comment block in `frontend/.env.example` to accurately describe what the toggle controls.

The updated comment should explain:
- When `true` (default): all API calls go to local mock handlers with localStorage persistence
- When `false`: all API calls go to the real backend via `httpClient`
- Which services are affected: auth, calendar, and family (all of them)
- What localStorage still does when mocks are off: cache seed for instant startup + cross-tab sync (not the source of truth)

Keep it concise — this is a `.env` comment, not a README.

## Verify Before Changing

Confirm these yourself — don't take them at face value:

- `src/api/services/auth.service.ts` — has `USE_MOCK_API` toggle
- `src/api/services/calendar.service.ts` — has `USE_MOCK_API` toggle
- `src/api/services/family.service.ts` — has `USE_MOCK_API` toggle
- `src/api/mocks/index.ts` — where `USE_MOCK_API` is defined
- `src/api/hooks/use-family.ts` — uses TanStack Query, seeds from localStorage
- `src/stores/family-store.ts` — hydration gate only, not data storage

If you find any service that does NOT respect the toggle, note it in the PR description.

## Git Workflow

1. **Create a new branch** from `main`: `docs/update-env-mock-api-comment`
2. Update `.env.example`
3. **Run linting**: `npm run lint`
4. Commit: `docs(env): update VITE_USE_MOCK_API comment to reflect current architecture`
5. **Open a PR** with a description explaining:
   - What changed and why the old comment was outdated
   - The actual data flow when `VITE_USE_MOCK_API=false`: login/register → JWT stored → all API calls go through `httpClient` → responses cached in TanStack Query + written to localStorage for instant startup on reload
   - That localStorage is a cache layer, not the source of truth when mocks are off

## Acceptance Criteria

- [ ] `.env.example` comment accurately describes what `VITE_USE_MOCK_API` controls
- [ ] No mention of "only calendar events" — all services are covered
- [ ] No sensitive information exposed (no URLs, secrets, or server details)
- [ ] Linting passes
