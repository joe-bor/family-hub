> Archived 2026-04-24 — implemented by joe-bor/family-hub-api#23

# BE PR 2: OAuth Flow — Google Account Connection

## Context

This is the second PR in the Google Calendar integration series. PR 1 ([#22](https://github.com/joe-bor/family-hub-api/pull/22)) added the `source`/`description` columns, `EventSource` enum, and Google event write protection.

This PR adds the OAuth 2.0 flow so a family member can connect their Google account. No sync logic yet — just authentication, token exchange, encrypted storage, and disconnect.

Design doc: `docs/google-calendar-integration-design.md` (§3 OAuth 2.0 Flow, §4 Data Model — `google_oauth_token` table, §5 Endpoints — OAuth Connection + Connection Status, §8 Configuration, §9 Disconnect & Cleanup)

## Secrets handling

**You do NOT have access to real Google credentials.** All secrets come from environment variables. Use placeholder values in test configuration. The user will plug in real values for local and production testing.

## What to build

### 1. Flyway Migration: `V5__google_oauth_token.sql`

```sql
CREATE TABLE google_oauth_token (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    member_id UUID NOT NULL UNIQUE REFERENCES family_member(id) ON DELETE CASCADE,
    access_token TEXT NOT NULL,
    refresh_token TEXT NOT NULL,
    token_expiry TIMESTAMP NOT NULL,
    scope TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now(),
    updated_at TIMESTAMP NOT NULL DEFAULT now()
);
```

Note: tokens are stored **encrypted** (see TokenEncryptionService below). The column type is TEXT because encrypted values are longer than the originals.

### 2. Configuration: `GoogleOAuthConfig`

Use `@ConfigurationProperties(prefix = "google.oauth")` or `@Value` annotations. Fields:

- `clientId` ← `GOOGLE_CLIENT_ID` env var
- `clientSecret` ← `GOOGLE_CLIENT_SECRET` env var
- `redirectUri` ← `GOOGLE_REDIRECT_URI` env var
- `scopes` — hardcode: `calendar.events.readonly`, `calendar.calendarlist.readonly`

Add to `application.yml`:

```yaml
google:
  oauth:
    client-id: ${GOOGLE_CLIENT_ID:}
    client-secret: ${GOOGLE_CLIENT_SECRET:}
    redirect-uri: ${GOOGLE_REDIRECT_URI:http://localhost:8080/api/google/callback}

security:
  token-encryption-key: ${TOKEN_ENCRYPTION_KEY:default-dev-key-change-in-production}
```

The empty defaults (`${VAR:}`) mean the app starts without Google credentials — the OAuth endpoints just won't work until configured. This is important so existing functionality isn't broken.

### 3. Entity: `GoogleOAuthToken`

JPA entity for the `google_oauth_token` table. Fields map 1:1 to the migration above.

### 4. Repository: `GoogleOAuthTokenRepository`

```java
public interface GoogleOAuthTokenRepository extends JpaRepository<GoogleOAuthToken, UUID> {
    Optional<GoogleOAuthToken> findByMemberId(UUID memberId);
    void deleteByMemberId(UUID memberId);
}
```

### 5. Service: `TokenEncryptionService`

AES-256 encrypt/decrypt for token storage. Takes the encryption key from config.

```java
@Service
public class TokenEncryptionService {
    // Constructor takes key from config
    public String encrypt(String plainText) { ... }
    public String decrypt(String cipherText) { ... }
}
```

Use Spring's `TextEncryptor` (from `spring-security-crypto`) or standard Java `javax.crypto` — your choice. The key point: tokens are never stored in plaintext.

### 6. Service: `GoogleOAuthService`

Core OAuth logic:

```java
@Service
public class GoogleOAuthService {

    // Build the Google OAuth consent URL
    // Include: client_id, redirect_uri, scope, state=memberId,
    //          access_type=offline, prompt=consent, response_type=code
    public String buildAuthorizationUrl(UUID memberId) { ... }

    // Exchange authorization code for tokens (server-to-server POST to Google)
    // Encrypt tokens before storing
    // Return the stored GoogleOAuthToken entity
    public GoogleOAuthToken exchangeCodeForTokens(String code, UUID memberId) { ... }

    // Revoke token at Google + delete from DB
    public void disconnect(UUID memberId) { ... }

    // Check if a member has a valid connection
    public boolean isConnected(UUID memberId) { ... }
}
```

For the token exchange (`exchangeCodeForTokens`):
- `POST https://oauth2.googleapis.com/token` with `code`, `client_id`, `client_secret`, `redirect_uri`, `grant_type=authorization_code`
- Use `RestTemplate` or `WebClient` — whichever the project already uses, or `RestTemplate` if neither
- Parse response for `access_token`, `refresh_token`, `expires_in`, `scope`
- Encrypt `access_token` and `refresh_token` via `TokenEncryptionService`
- Store in `google_oauth_token` table

For disconnect:
- Revoke at Google: `POST https://oauth2.googleapis.com/revoke?token={access_token}` (best effort — don't fail if Google returns an error)
- Delete the `google_oauth_token` row
- **Do NOT delete Google events yet** — that comes in a later PR when the sync tables exist

### 7. Controller: `GoogleOAuthController`

```java
@RestController
@RequestMapping("/api/google")
public class GoogleOAuthController {

    // GET /api/google/auth?memberId={uuid}
    // Returns: { "authorizationUrl": "https://accounts.google.com/o/oauth2/v2/auth?..." }
    // Validate: memberId belongs to the authenticated family

    // GET /api/google/callback?code={code}&state={memberId}
    // Exchanges code for tokens, stores them
    // Redirects to FE settings page (e.g., redirect to /settings?googleConnected=true)
    // The redirect URL can be a config property for flexibility

    // DELETE /api/google/disconnect/{memberId}
    // Revokes and deletes tokens
    // Validate: memberId belongs to the authenticated family
    // Returns: 200 OK

    // GET /api/google/status/{memberId}
    // Returns: { "connected": true/false }
    // (lastSyncedAt and calendars[] come in later PRs when sync tables exist)
    // Validate: memberId belongs to the authenticated family
}
```

**Important:** All endpoints must validate that the `memberId` belongs to the authenticated user's family. Follow the same authorization pattern used in `CalendarEventController`.

### 8. Security: Allow callback without JWT

The Google callback (`/api/google/callback`) is a redirect FROM Google — the user's browser hits this URL directly after consent. It won't have a JWT token in the header. You need to whitelist this path in the security configuration so it's accessible without authentication.

The `state` parameter (which contains the `memberId`) serves as a CSRF-like check — verify the member exists and the callback is expected.

## Tests

- **Unit tests:** `TokenEncryptionService` — encrypt then decrypt returns original. Different inputs produce different ciphertexts.
- **Unit tests:** `GoogleOAuthService` — `buildAuthorizationUrl` produces a valid URL with all required params. (Token exchange and disconnect are harder to unit test without mocking HTTP — integration tests are more valuable.)
- **Controller tests:** Auth endpoint returns a valid URL. Status endpoint returns connected/disconnected correctly. Disconnect removes the token row. Authorization checks (wrong family → 403).
- **Integration test:** Full flow as much as possible without real Google — create a token row manually, verify status is connected, disconnect, verify status is disconnected and row is deleted.

Use placeholder values for Google credentials in test configuration:

```yaml
# test/resources/application-test.yml (or @TestPropertySource)
google:
  oauth:
    client-id: test-client-id
    client-secret: test-client-secret
    redirect-uri: http://localhost:8080/api/google/callback
security:
  token-encryption-key: test-encryption-key-that-is-32-chars!!
```

## What NOT to build

- No Google Calendar API client (no `google-api-services-calendar` dependency yet)
- No `google_synced_calendar` table (PR 3)
- No sync logic (PR 4)
- No calendar list/selection endpoints (PR 3)
- No event deletion on disconnect (needs sync tables from PR 3)

## Dependencies

You will likely need:
- `spring-security-crypto` (if using Spring's `TextEncryptor`) — may already be transitive from Spring Security
- No new external dependencies should be needed for the HTTP calls to Google's token endpoint

## Branch & PR

- Branch: `feat/google-cal-oauth-flow`
- Atomic commits, open a PR when done
- Reference the design doc and PR #22 in the PR description

## Acceptance criteria

- [ ] V5 migration creates `google_oauth_token` table
- [ ] `GET /api/google/auth?memberId=...` returns a valid Google OAuth URL with all required params
- [ ] `GET /api/google/callback` exchanges code for tokens and stores them encrypted
- [ ] Callback endpoint is accessible without JWT authentication
- [ ] `GET /api/google/status/{memberId}` returns `{ connected: true/false }`
- [ ] `DELETE /api/google/disconnect/{memberId}` revokes token at Google and deletes DB row
- [ ] All endpoints validate memberId belongs to authenticated family
- [ ] Tokens are encrypted at rest (never stored in plaintext)
- [ ] App starts normally without Google credentials configured (existing functionality unaffected)
- [ ] Tests cover the above
