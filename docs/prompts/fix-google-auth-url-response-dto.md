# BE Fix: Replace Map.of with Typed DTO for Google Auth URL Response

## Context

A cross-repo alignment check found that `GET /api/google/auth` returns a response the FE cannot read. This is a **breaking mismatch** — the "Connect Google Calendar" button silently fails because the field name doesn't match.

**The bug:**
- BE returns `Map.of("authorizationUrl", url)` → JSON: `{ "data": { "authorizationUrl": "https://..." } }`
- FE reads `response.data.url` → gets `undefined`
- `window.location.href = undefined` → OAuth flow never starts

**You are expected to verify this claim yourself before acting.** Read the actual controller code and confirm the field name.

## Why the BE should change

Every other endpoint in this codebase wraps a **typed DTO or record** in `ApiResponse<T>` — `AuthResponse`, `FamilyResponse`, `GoogleConnectionStatus`, `GoogleCalendarResponse`, etc. The auth URL endpoint is the **only one** using a raw `Map.of(...)`. Replacing it with a proper record aligns with the existing pattern and gives compile-time safety.

## What to fix

### 1. Create a new DTO record

Create `GoogleAuthUrlResponse.java` in the `dto/` package:

```java
public record GoogleAuthUrlResponse(String url) {}
```

Single field: `url` — matches the FE's `GoogleAuthUrl { url: string }` type.

### Starting point

- `GoogleOAuthController.java` ~line 35-38 — the `Map.of("authorizationUrl", url)` call

### 2. Update the controller

In `GoogleOAuthController.java`, replace:

```java
return ResponseEntity.ok(new ApiResponse<>(
        Map.of("authorizationUrl", url),
        "Authorization URL generated"));
```

with:

```java
return ResponseEntity.ok(new ApiResponse<>(
        new GoogleAuthUrlResponse(url),
        "Authorization URL generated"));
```

### 3. Update tests

Search for any tests that assert on the `authorizationUrl` field name and update them to expect `url` instead. Check:
- `GoogleOAuthControllerTest.java`
- `GoogleOAuthIntegrationTest.java`

The assertion likely looks like `jsonPath("$.data.authorizationUrl")` — change to `jsonPath("$.data.url")`.

## What NOT to change

- Do **not** change any FE code — the FE type is already correct
- Do **not** change any other endpoints — only the auth URL response
- Do **not** rename the controller method or change the endpoint path

## Acceptance criteria

- [ ] `GET /api/google/auth?memberId={uuid}` returns `{ "data": { "url": "https://..." }, "message": "..." }`
- [ ] No raw `Map.of(...)` used as an `ApiResponse` payload in Google controllers
- [ ] All existing tests pass with the updated field name
