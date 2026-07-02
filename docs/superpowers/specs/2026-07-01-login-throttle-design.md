# Login Brute-Force Throttle

**Date:** 2026-07-01
**Status:** Approved design, ready for implementation plan
**Owner:** Family Hub backend
**Origin:** 2026-07-01 whole-product security review (finding: no throttle/lockout on `POST /api/auth/login`)

## Summary

Add a per-username lockout to the backend login endpoint. After 5 consecutive failed login attempts for a username, further login attempts for that username are rejected with HTTP 429 for 15 minutes. A successful login resets the counter.

This protects the single shared family password — the whole security perimeter of the product — from online brute-force guessing, which today runs uncontested against the public API at `familyhub.joe-bor.me`.

Registration gating was considered in the same session and deliberately deferred; this spec covers only the login throttle.

## Design

### Component: `LoginThrottleService`

A new Spring `@Component` in the backend (`backend/family-hub-api`), following the existing `OAuthStateStore` pattern: an in-memory `ConcurrentHashMap` plus a `@Scheduled` cleanup sweep.

- **Key:** normalized username (trimmed, lowercased). Tracking is per-username, not per-IP, because the asset being protected is the family password — a per-username cap bounds total guesses regardless of attacker IP count. No `X-Forwarded-For` parsing.
- **State per key:** consecutive-failure count and lock expiry timestamp.
- **Policy:**
  - Each failed login (unknown username or wrong password) increments the counter.
  - On the 5th consecutive failure, the username locks for a fixed 15-minute window measured from that 5th failure. Attempts during the lock are rejected without touching the counter or extending the window.
  - When the lock expires, the entry is cleared entirely — the next failure starts a fresh count of 1, it does not re-lock immediately.
  - A successful login clears the entry.
  - Threshold and lock duration are constants in the service (no new config surface). If tuning is ever needed, promote to properties then.
- **API:**
  - `assertNotLocked(username)` — throws when locked; called before credential checking.
  - `recordFailure(username)` — called when credentials are rejected.
  - `recordSuccess(username)` — called after successful authentication.
- **Cleanup:** scheduled sweep removes expired entries (same cadence style as `OAuthStateStore`) so the map cannot grow unbounded from bogus usernames.
- **Persistence:** none. Single-instance deployment; a restart clearing counters is acceptable. Attackers cannot trigger restarts.

### Integration point: `AuthService.login`

1. `assertNotLocked(username)` — before the repository lookup, so locked attempts do no BCrypt work and leak nothing about username existence.
2. Existing credential check unchanged.
3. On `InvalidCredentialException` path: `recordFailure(username)` then rethrow.
4. On success: `recordSuccess(username)` before returning the token.

Only `login` is throttled. `register` and `check-username` are out of scope.

### Error handling

- New exception (e.g. `TooManyLoginAttemptsException`) with a generic message: `"Too many failed login attempts. Try again in a few minutes."` The message must not reveal whether the username exists and must not echo the username back.
- New handler in `GlobalExceptionHandler` mapping it to **HTTP 429** using the existing `ErrorResponse` shape, logging method + path at `warn` (consistent with the other handlers — no password, no username in the response body).

### Frontend / compatibility impact

- **Existing users:** unaffected. The throttle only fires after 5 consecutive failures on one username.
- **FE:** no changes required. The login form surfaces the server's `ErrorResponse` message; worst case a locked-out user sees the new message text. (Verify during implementation that the FE error mapping doesn't swallow 429 into a silent state; if it does, that is a small FE follow-up, not a blocker.)
- **E2E/CI:** unaffected. Specs register unique families and never fail logins, so the throttle cannot trip. No compose or helper changes.
- **API contract:** additive only — a new 429 response on `POST /api/auth/login`. Update `docs/` API notes if login responses are documented.

### Testing

Unit tests on `LoginThrottleService` (thresholds, lock expiry via injectable clock, reset-on-success, cleanup sweep) and `AuthService` integration-style tests: 5 failures lock the account, locked attempts return 429 even with the correct password, success resets the counter, and lock expiry restores access. Follow the repo's existing test conventions (TestContainers only where the existing tests already require it; this feature is unit-testable without a DB).

## Non-goals

- Registration gating / invite codes (deferred by explicit decision).
- Per-IP tracking, proxy-header parsing, fail2ban/nginx layers.
- Persistent or distributed rate limiting, new dependencies (bucket4j etc.).
- CAPTCHA, progressive backoff, or notification on lockout.

## Accepted tradeoff

Someone who knows the family username can deliberately lock it for 15 minutes at a time by spamming wrong passwords (griefing/DoS vector). Accepted at family scale in exchange for a design with no IP-tracking complexity.

## Rollout

Standard BE flow: feature branch + PR in `family-hub-api`, merge to `main`, release via release-please. No env vars, no migration, no droplet changes. Effective as soon as the released image is deployed.
