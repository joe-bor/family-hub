# Fix Google OAuth PR #23 — Review Action Items

**Context:** PR #23 (`feat/google-cal-oauth-flow`) adds the Google OAuth flow for account connection. A code review identified 5 issues to fix before merging. All changes should be made on the existing `feat/google-cal-oauth-flow` branch.

**Branch:** `feat/google-cal-oauth-flow`
**PR:** https://github.com/joe-bor/family-hub-api/pull/23

---

## 1. Squash V5 + V6 migrations

**Problem:** V5 creates `google_oauth_token` with `TIMESTAMP` columns, then V6 immediately alters them to `TIMESTAMPTZ`. These are in the same PR and haven't been deployed — no reason to ship two migrations.

**Fix:**
- Edit `V5__google_oauth_token.sql` to use `TIMESTAMPTZ` for `token_expiry`, `created_at`, and `updated_at` from the start
- Delete `V6__fix_google_oauth_token_timestamps.sql`
- Verify: tests still pass with the single migration

## 2. Standardize `family` table timestamps to `TIMESTAMPTZ`

**Problem:** `V1__initial_schema.sql` uses `TIMESTAMP(6)` for `family.created_at` and `family.updated_at`. The new `google_oauth_token` table uses `TIMESTAMPTZ`. This is an inconsistency — `TIMESTAMPTZ` is the correct type for Java `Instant` columns.

**Fix:**
- Since V6 is now free (after squashing above), create `V6__standardize_family_timestamps.sql`:
  ```sql
  ALTER TABLE family ALTER COLUMN created_at TYPE TIMESTAMPTZ;
  ALTER TABLE family ALTER COLUMN updated_at TYPE TIMESTAMPTZ;
  ```
- Verify: existing tests still pass

## 3. Handle Google consent denial in callback

**Problem:** When a user denies consent, Google redirects to `/api/google/callback?error=access_denied&state=...` — no `code` parameter. The current `@RequestParam String code` is required, so Spring throws `MissingServletRequestParameterException` and the user sees a raw 400 error page.

**Fix in `GoogleOAuthController.handleCallback`:**
- Make `code` optional: `@RequestParam(required = false) String code`
- Add `@RequestParam(required = false) String error`
- If `error` is present (or `code` is null), redirect to the frontend with an error indicator (e.g., `frontendRedirectUrl + "?error=consent_denied"` or similar)
- Add a unit test for the consent-denied case
- Add an integration test that the callback with `error` param redirects gracefully

## 4. Add try/catch for token exchange failures in callback

**Problem:** If the `exchangeCodeForTokens` call fails (Google is down, invalid code, network error), `RestClient` throws an unhandled exception. The user sees a raw error page instead of being redirected back to the frontend.

**Fix in `GoogleOAuthController.handleCallback`:**
- Wrap `googleOAuthService.exchangeCodeForTokens(code, memberId)` in a try/catch
- On failure, log the error and redirect to the frontend with an error indicator (e.g., `?error=token_exchange_failed`)
- Add a unit test: mock `exchangeCodeForTokens` to throw, verify the response is a redirect with error param (not a 500)

## 5. Reorder validation in `exchangeCodeForTokens`

**Problem:** The current order is: validate `accessToken` → delete existing token → validate `refreshToken`. If `refreshToken` is null, `@Transactional` rollback saves you, but the code is "accidentally correct." A future change removing the transaction or calling this from a different context would expose the bug — the user's existing token gets deleted before we confirm we have a valid replacement.

**Fix in `GoogleOAuthService.exchangeCodeForTokens`:**
- Move the `refreshToken == null` check up, immediately after the `accessToken == null` check — before the delete block
- The final order should be: validate response → validate accessToken → validate refreshToken → delete old token → save new token
- Existing tests should still pass (this is a pure reorder, no behavior change for the happy path)

---

## General Instructions

- Make atomic commits for each fix using conventional format: `fix(oauth): ...` or `refactor(oauth): ...`
- Run the full test suite after all changes: `JWT_SECRET="dGVzdC1zZWNyZXQta2V5LXRoYXQtaXMtbG9uZy1lbm91Z2gtZm9yLWhtYWMtc2hhLTI1Ng==" ./mvnw test`
- All 169+ existing tests must continue to pass
- Push to the existing `feat/google-cal-oauth-flow` branch
- Do NOT open a new PR — these fixes go into the existing PR #23
