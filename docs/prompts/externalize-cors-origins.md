# Task: Externalize CORS Allowed Origins for Production Deployment

## Context

The Family Hub API is about to be deployed to a cloud platform (likely Railway). The frontend will be redeployed pointing at the live backend instead of mocks. Right now, CORS origins are hardcoded to localhost in `SecurityConfig.java`, which means the deployed frontend will be blocked by the browser's same-origin policy.

This is a **deployment blocker** — the only code change needed before the BE can go live.

## Starting Points (Verify These Yourself)

These are hints to help you get oriented — **do not take them at face value**. Read the actual code and confirm before making changes.

### CORS Configuration

- `SecurityConfig.java` (~line 51-65) — `corsConfigurationSource()` bean hardcodes `http://localhost:5173` and `http://localhost:3000`
- No existing property or env var for allowed origins in `application.yml`, `application-dev.yml`, or `application-prod.yml`

### Existing Env Var Pattern

The project already externalizes config via env vars with profile-specific defaults:

- `JWT_SECRET` → `${JWT_SECRET}` in `application.yml`
- `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` → `${DB_URL}` etc. in `application-prod.yml`

Follow this same pattern for CORS origins.

## What Needs to Change

Add a `cors.allowed-origins` property (or similar) that:

1. **Dev profile** — defaults to `http://localhost:5173,http://localhost:3000` (current behavior, no breakage)
2. **Prod profile** — reads from an environment variable (e.g., `CORS_ALLOWED_ORIGINS`)
3. **SecurityConfig** — reads the property instead of hardcoding, splits on comma if multiple origins

### Suggested approach (push back if you have a better idea):

```yaml
# application-dev.yml
cors:
  allowed-origins: http://localhost:5173,http://localhost:3000

# application-prod.yml
cors:
  allowed-origins: ${CORS_ALLOWED_ORIGINS}
```

```java
// SecurityConfig.java
@Value("${cors.allowed-origins}")
private String allowedOrigins;

// In corsConfigurationSource():
configuration.setAllowedOrigins(List.of(allowedOrigins.split(",")));
```

**Important considerations:**
- `setAllowCredentials(true)` is already set — this means `allowedOrigins` **cannot** be `*`. Each origin must be explicitly listed. Make sure the implementation doesn't accidentally allow wildcard.
- Keep the existing allowed methods, headers, and credentials settings unchanged.
- Trim whitespace from split origins (e.g., `"http://a.com, http://b.com"` should work).

## Important: Do Your Own Research

- **Check if Spring Boot has a built-in property** for CORS origins that you can leverage instead of a custom property. If there's a standard `spring.web.cors.*` namespace, prefer it.
- **Check if any other beans or filters** reference the CORS configuration. Don't break anything.
- **Verify the existing tests** — if there are integration tests that rely on CORS behavior, update them.
- **Check if `@CrossOrigin` annotations** exist on any controllers. If so, those would override the global config and need the same treatment.

## Git Workflow

1. **Create a new branch** from `main`: `feat/externalize-cors-origins`
2. **Plan your implementation** using plan mode before writing any code
3. **Make atomic commits** using conventional commit format:
   - `feat(config): externalize CORS allowed origins via env var`
   - `test: update integration tests for CORS configuration` (if applicable)
   - etc.
4. **Run tests after each commit**:
   - `./mvnw test`
   - Fix any issues before moving on
5. **Open a PR** when complete using `gh pr create`

## Acceptance Criteria

- [ ] CORS origins are read from configuration, not hardcoded in Java
- [ ] Dev profile defaults to `localhost:5173` and `localhost:3000` (zero-config local dev)
- [ ] Prod profile reads from an environment variable
- [ ] Multiple origins work (comma-separated)
- [ ] `allowCredentials(true)` is preserved — wildcard `*` is not allowed
- [ ] Existing dev workflow is unchanged (no new env vars required for local dev)
- [ ] All existing tests pass
- [ ] CLAUDE.md is updated if the CORS section or env var docs need changes
