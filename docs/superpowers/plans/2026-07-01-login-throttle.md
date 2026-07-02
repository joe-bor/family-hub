# Login Brute-Force Throttle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Lock a username out of `POST /api/auth/login` for 15 minutes after 5 consecutive failed attempts, returning HTTP 429.

**Architecture:** A new in-memory `LoginThrottleService` (ConcurrentHashMap + `@Scheduled` sweep, modeled on the existing `OAuthStateStore`) is consulted by `AuthService.login` before any credential work. A new `TooManyLoginAttemptsException` maps to 429 in `GlobalExceptionHandler`. No new dependencies, no config surface, no persistence.

**Tech Stack:** Java 21, Spring Boot, JUnit 5 + Mockito + AssertJ, Maven (`./mvnw`).

**Spec:** `docs/superpowers/specs/2026-07-01-login-throttle-design.md` (root workspace repo `joe-bor/family-hub`)

**Repo:** All work happens in `backend/family-hub-api/` (GitHub: `joe-bor/family-hub-api`). Branch: `feat/login-throttle`. Commit messages use conventional format with no AI attribution.

**Test environment note:** Unit and `@WebMvcTest` tests run without Docker. The full suite includes TestContainers integration tests, which need Docker running and `JWT_SECRET` exported:

```bash
export JWT_SECRET=dGVzdC1zZWNyZXQta2V5LXRoYXQtaXMtbG9uZy1lbm91Z2gtZm9yLWhtYWMtc2hhLTI1Ng==
```

---

### Task 0: Branch

- [ ] **Step 1: Create the feature branch**

```bash
cd backend/family-hub-api
git checkout main && git pull
git checkout -b feat/login-throttle
```

---

### Task 1: `TooManyLoginAttemptsException` + `LoginThrottleService`

**Files:**
- Create: `src/main/java/com/familyhub/demo/exception/TooManyLoginAttemptsException.java`
- Create: `src/main/java/com/familyhub/demo/service/LoginThrottleService.java`
- Test: `src/test/java/com/familyhub/demo/service/LoginThrottleServiceTest.java`

Design decisions locked by the spec:
- Keyed by normalized username (trim + lowercase). No IP tracking.
- 5th consecutive failure locks for a fixed 15 minutes from that failure. Attempts during the lock are rejected up front (via `assertNotLocked`) and do not extend the window.
- An expired lock means a fresh start: the entry is cleared; the next failure counts as 1.
- "Consecutive" is time-bounded: a non-locked entry whose last failure is older than the 15-minute lock duration is stale and resets to a fresh count (this is also what lets the sweep keep the map bounded against bogus-username spam).
- Successful login clears the entry.
- `Clock` is injected (bean already exists in `TimeConfig`) so tests control time.

- [ ] **Step 1: Write the failing test**

Create `src/test/java/com/familyhub/demo/service/LoginThrottleServiceTest.java`:

```java
package com.familyhub.demo.service;

import com.familyhub.demo.exception.TooManyLoginAttemptsException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;
import java.time.ZoneOffset;

import static org.assertj.core.api.Assertions.assertThatCode;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class LoginThrottleServiceTest {

    private MutableClock clock;
    private LoginThrottleService throttle;

    @BeforeEach
    void setUp() {
        clock = new MutableClock();
        throttle = new LoginThrottleService(clock);
    }

    private void failTimes(String username, int times) {
        for (int i = 0; i < times; i++) {
            throttle.recordFailure(username);
        }
    }

    @Test
    void assertNotLocked_unknownUsername_passes() {
        assertThatCode(() -> throttle.assertNotLocked("nobody"))
                .doesNotThrowAnyException();
    }

    @Test
    void assertNotLocked_belowThreshold_passes() {
        failTimes("smith", 4);

        assertThatCode(() -> throttle.assertNotLocked("smith"))
                .doesNotThrowAnyException();
    }

    @Test
    void assertNotLocked_atThreshold_throws() {
        failTimes("smith", 5);

        assertThatThrownBy(() -> throttle.assertNotLocked("smith"))
                .isInstanceOf(TooManyLoginAttemptsException.class);
    }

    @Test
    void assertNotLocked_lockExpired_passesAndStartsFresh() {
        failTimes("smith", 5);
        clock.advance(Duration.ofMinutes(15).plusSeconds(1));

        assertThatCode(() -> throttle.assertNotLocked("smith"))
                .doesNotThrowAnyException();

        // Fresh start: one new failure must NOT re-lock
        throttle.recordFailure("smith");
        assertThatCode(() -> throttle.assertNotLocked("smith"))
                .doesNotThrowAnyException();
    }

    @Test
    void recordSuccess_resetsCounter() {
        failTimes("smith", 4);
        throttle.recordSuccess("smith");
        failTimes("smith", 4);

        assertThatCode(() -> throttle.assertNotLocked("smith"))
                .doesNotThrowAnyException();
    }

    @Test
    void staleFailures_resetToFreshCount() {
        failTimes("smith", 4);
        clock.advance(Duration.ofMinutes(15).plusSeconds(1));
        failTimes("smith", 4); // stale entry resets: this is counts 1-4, not 5-8

        assertThatCode(() -> throttle.assertNotLocked("smith"))
                .doesNotThrowAnyException();
    }

    @Test
    void usernames_areNormalized() {
        failTimes("  Smith ", 5);

        assertThatThrownBy(() -> throttle.assertNotLocked("smith"))
                .isInstanceOf(TooManyLoginAttemptsException.class);
    }

    @Test
    void differentUsernames_trackedIndependently() {
        failTimes("smith", 5);

        assertThatCode(() -> throttle.assertNotLocked("jones"))
                .doesNotThrowAnyException();
    }

    @Test
    void cleanupExpired_removesExpiredAndStaleEntries_keepsActiveOnes() {
        failTimes("locked-expired", 5);
        failTimes("stale", 2);
        clock.advance(Duration.ofMinutes(15).plusSeconds(1));
        failTimes("active", 1);
        failTimes("locked-active", 5);

        throttle.cleanupExpired();

        assertThat(throttle.size()).isEqualTo(2);
    }

    /** Minimal controllable clock for lock-expiry tests. */
    private static final class MutableClock extends Clock {
        private Instant instant = Instant.parse("2026-07-01T00:00:00Z");

        void advance(Duration duration) {
            instant = instant.plus(duration);
        }

        @Override
        public ZoneId getZone() {
            return ZoneOffset.UTC;
        }

        @Override
        public Clock withZone(ZoneId zone) {
            return this;
        }

        @Override
        public Instant instant() {
            return instant;
        }
    }
}
```

- [ ] **Step 2: Run the test to verify it fails to compile (classes don't exist)**

```bash
./mvnw test -Dtest=LoginThrottleServiceTest
```

Expected: BUILD FAILURE — compilation error, `LoginThrottleService` and `TooManyLoginAttemptsException` do not exist.

- [ ] **Step 3: Create the exception**

Create `src/main/java/com/familyhub/demo/exception/TooManyLoginAttemptsException.java`:

```java
package com.familyhub.demo.exception;

public class TooManyLoginAttemptsException extends RuntimeException {
    public TooManyLoginAttemptsException() {
        super("Too many failed login attempts. Try again in a few minutes.");
    }
}
```

The message is intentionally generic: it must not reveal whether the username exists and must not echo the username back.

- [ ] **Step 4: Create the service**

Create `src/main/java/com/familyhub/demo/service/LoginThrottleService.java`:

```java
package com.familyhub.demo.service;

import com.familyhub.demo.exception.TooManyLoginAttemptsException;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.util.Locale;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class LoginThrottleService {

    static final int MAX_FAILURES = 5;
    static final Duration LOCK_DURATION = Duration.ofMinutes(15);

    private final Clock clock;
    private final Map<String, Attempts> store = new ConcurrentHashMap<>();

    public LoginThrottleService(Clock clock) {
        this.clock = clock;
    }

    /**
     * Rejects the attempt if this username is currently locked out.
     * An expired lock is cleared here so the username gets a fresh start.
     */
    public void assertNotLocked(String username) {
        String key = normalize(username);
        Attempts attempts = store.get(key);
        if (attempts == null || attempts.lockedUntil() == null) {
            return;
        }
        if (attempts.lockedUntil().isAfter(now())) {
            throw new TooManyLoginAttemptsException();
        }
        store.remove(key);
    }

    public void recordFailure(String username) {
        store.compute(normalize(username), (key, existing) -> {
            Instant now = now();
            int failures = (existing == null || isExpired(existing, now))
                    ? 1
                    : existing.failures() + 1;
            Instant lockedUntil = failures >= MAX_FAILURES ? now.plus(LOCK_DURATION) : null;
            return new Attempts(failures, now, lockedUntil);
        });
    }

    public void recordSuccess(String username) {
        store.remove(normalize(username));
    }

    @Scheduled(fixedRate = 60_000)
    void cleanupExpired() {
        Instant now = now();
        store.entrySet().removeIf(entry -> isExpired(entry.getValue(), now));
    }

    // Visible for testing
    int size() {
        return store.size();
    }

    /**
     * An entry no longer counts against the username when its lock has lapsed,
     * or — for non-locked entries — when the last failure is older than the
     * lock duration (failures that old are no longer "consecutive").
     */
    private boolean isExpired(Attempts attempts, Instant now) {
        Instant cutoff = attempts.lockedUntil() != null
                ? attempts.lockedUntil()
                : attempts.lastFailureAt().plus(LOCK_DURATION);
        return cutoff.isBefore(now);
    }

    private Instant now() {
        return Instant.now(clock);
    }

    private static String normalize(String username) {
        return username == null ? "" : username.trim().toLowerCase(Locale.ROOT);
    }

    private record Attempts(int failures, Instant lastFailureAt, Instant lockedUntil) {}
}
```

- [ ] **Step 5: Run the test to verify it passes**

```bash
./mvnw test -Dtest=LoginThrottleServiceTest
```

Expected: PASS (10 tests).

- [ ] **Step 6: Commit**

```bash
git add src/main/java/com/familyhub/demo/exception/TooManyLoginAttemptsException.java \
        src/main/java/com/familyhub/demo/service/LoginThrottleService.java \
        src/test/java/com/familyhub/demo/service/LoginThrottleServiceTest.java
git commit -m "feat(auth): add in-memory login throttle service"
```

---

### Task 2: Map `TooManyLoginAttemptsException` to HTTP 429

**Files:**
- Modify: `src/main/java/com/familyhub/demo/exception/GlobalExceptionHandler.java` (add one handler alongside the existing `@ExceptionHandler` methods)
- Test: `src/test/java/com/familyhub/demo/controller/AuthControllerTest.java` (add one test)

- [ ] **Step 1: Write the failing test**

Add to `AuthControllerTest` (it already uses `@WebMvcTest(AuthController.class)` with `@MockitoBean AuthService authService` — follow the existing `login`-test style in that file). Add the import `com.familyhub.demo.exception.TooManyLoginAttemptsException`, then:

```java
@Test
void login_throttled_returns429WithGenericMessage() throws Exception {
    given(authService.login(any())).willThrow(new TooManyLoginAttemptsException());

    mockMvc.perform(post("/api/auth/login")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            {
                                "username": "testfamily",
                                "password": "password123"
                            }
                            """))
            .andExpect(status().isTooManyRequests())
            .andExpect(jsonPath("$.message")
                    .value("Too many failed login attempts. Try again in a few minutes."));
}
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
./mvnw test -Dtest=AuthControllerTest#login_throttled_returns429WithGenericMessage
```

Expected: FAIL — status 500 instead of 429 (unmapped exception falls through to the generic handler).

- [ ] **Step 3: Add the handler**

In `GlobalExceptionHandler` (same package as the exception, so no import is needed), add this method after `handleInvalidCredential`:

```java
@ExceptionHandler(TooManyLoginAttemptsException.class)
public ResponseEntity<ErrorResponse> handleTooManyLoginAttempts(TooManyLoginAttemptsException ex, HttpServletRequest request) {
    log.warn("Login throttled: {} {}", request.getMethod(), request.getRequestURI());
    return buildErrorResponse(HttpStatus.TOO_MANY_REQUESTS, request, ex.getMessage());
}
```

Note the log line deliberately includes neither username nor password — consistent with `handleInvalidCredential`.

- [ ] **Step 4: Run the test to verify it passes**

```bash
./mvnw test -Dtest=AuthControllerTest
```

Expected: PASS (all tests in the class, including the new one).

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/familyhub/demo/exception/GlobalExceptionHandler.java \
        src/test/java/com/familyhub/demo/controller/AuthControllerTest.java
git commit -m "feat(auth): return 429 for throttled login attempts"
```

---

### Task 3: Wire the throttle into `AuthService.login`

**Files:**
- Modify: `src/main/java/com/familyhub/demo/service/AuthService.java`
- Test: `src/test/java/com/familyhub/demo/service/AuthServiceTest.java`

Ordering contract from the spec: `assertNotLocked` runs **before** the repository lookup, so locked attempts do no BCrypt work and leak nothing about username existence.

- [ ] **Step 1: Write the failing tests**

In `AuthServiceTest`, add a mock alongside the existing ones (`@InjectMocks` wires it through the Lombok-generated constructor):

```java
@Mock
private LoginThrottleService loginThrottleService;
```

Add imports:

```java
import com.familyhub.demo.exception.TooManyLoginAttemptsException;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verifyNoInteractions;
```

Add these tests after the existing `login_*` tests:

```java
@Test
void login_locked_throws429AndSkipsCredentialCheck() {
    LoginRequest request = createLoginRequest();
    doThrow(new TooManyLoginAttemptsException())
            .when(loginThrottleService).assertNotLocked(request.username());

    assertThatThrownBy(() -> authService.login(request))
            .isInstanceOf(TooManyLoginAttemptsException.class);

    verifyNoInteractions(familyRepository, passwordEncoder, jwtService);
}

@Test
void login_invalidPassword_recordsFailure() {
    LoginRequest request = createLoginRequest();
    when(familyRepository.findByUsername(request.username())).thenReturn(Optional.of(family));
    when(passwordEncoder.matches(request.password(), family.getPassword())).thenReturn(false);

    assertThatThrownBy(() -> authService.login(request))
            .isInstanceOf(InvalidCredentialException.class);

    verify(loginThrottleService).recordFailure(request.username());
    verify(loginThrottleService, never()).recordSuccess(anyString());
}

@Test
void login_unknownUsername_recordsFailure() {
    LoginRequest request = createLoginRequest();
    when(familyRepository.findByUsername(request.username())).thenReturn(Optional.empty());

    assertThatThrownBy(() -> authService.login(request))
            .isInstanceOf(InvalidCredentialException.class);

    verify(loginThrottleService).recordFailure(request.username());
}

@Test
void login_success_recordsSuccess() {
    LoginRequest request = createLoginRequest();
    when(familyRepository.findByUsername(request.username())).thenReturn(Optional.of(family));
    when(passwordEncoder.matches(request.password(), family.getPassword())).thenReturn(true);
    when(jwtService.generateToken(family)).thenReturn("jwt-token");

    authService.login(request);

    verify(loginThrottleService).recordSuccess(request.username());
    verify(loginThrottleService, never()).recordFailure(anyString());
}
```

- [ ] **Step 2: Run the tests to verify the new ones fail**

```bash
./mvnw test -Dtest=AuthServiceTest
```

Expected: the 4 new tests FAIL (`loginThrottleService` never invoked); pre-existing tests still PASS.

- [ ] **Step 3: Wire the throttle into `AuthService`**

In `AuthService`, add the dependency (Lombok `@RequiredArgsConstructor` handles the constructor):

```java
private final LoginThrottleService loginThrottleService;
```

Replace the `login` method body:

```java
@Transactional
public AuthResponse login(LoginRequest loginRequest) {
    loginThrottleService.assertNotLocked(loginRequest.username());

    Family family = familyRepository.findByUsername(loginRequest.username())
            .orElse(null);

    if (family == null || !passwordEncoder.matches(loginRequest.password(), family.getPassword())) {
        loginThrottleService.recordFailure(loginRequest.username());
        throw new InvalidCredentialException("Invalid username or password.");
    }

    loginThrottleService.recordSuccess(loginRequest.username());
    String token = jwtService.generateToken(family);
    log.info("Family logged in, familyId={}, username={}", family.getId(), family.getUsername());
    return new AuthResponse(token, FamilyMapper.toDto(family));
}
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
./mvnw test -Dtest=AuthServiceTest
```

Expected: PASS (all tests, including the 4 new ones).

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/familyhub/demo/service/AuthService.java \
        src/test/java/com/familyhub/demo/service/AuthServiceTest.java
git commit -m "feat(auth): throttle repeated failed login attempts"
```

---

### Task 4: Full-suite verification + FE 429 display check

- [ ] **Step 1: Run the full backend test suite** (Docker must be running; TestContainers integration tests included)

```bash
export JWT_SECRET=dGVzdC1zZWNyZXQta2V5LXRoYXQtaXMtbG9uZy1lbm91Z2gtZm9yLWhtYWMtc2hhLTI1Ng==
./mvnw test
```

Expected: BUILD SUCCESS, no failures. If `AuthIntegrationTest` has a test that performs 5+ failed logins for the same username within one test run, it will now hit the throttle — if that happens, use distinct usernames per assertion in that test rather than weakening the throttle.

- [ ] **Step 2: Verify the FE surfaces the 429 message (read-only check, no FE changes expected)**

Read `frontend/src/api/client/api-error.ts` (`mapStatusToErrorCode`) and the login form's error rendering (`frontend/src/components` login/onboarding form). Confirm that a non-2xx response's `ErrorResponse.message` is shown to the user (the FE already displays server messages for 401 on bad credentials — confirm 429 follows the same path and isn't swallowed). If 429 is swallowed into a silent/generic state, do NOT fix it in this branch — note it in the PR description as a small FE follow-up issue.

- [ ] **Step 3: Push and open the PR**

```bash
git push -u origin feat/login-throttle
gh pr create --repo joe-bor/family-hub-api \
  --title "feat(auth): throttle repeated failed login attempts" \
  --body "Closes #<ISSUE_NUMBER>

Per-username login lockout: 5 consecutive failures lock the username for 15 minutes; locked attempts get HTTP 429 before any credential work. In-memory only, no new dependencies, no config.

Spec: joe-bor/family-hub docs/superpowers/specs/2026-07-01-login-throttle-design.md
Plan: joe-bor/family-hub docs/superpowers/plans/2026-07-01-login-throttle.md

Contract checklist:
- [ ] 5 consecutive failures per normalized username -> 15-min lock (LoginThrottleServiceTest)
- [ ] Locked attempts return 429 with generic message, no username echo (AuthControllerTest)
- [ ] Lock check runs before repository/BCrypt work (AuthServiceTest.login_locked_throws429AndSkipsCredentialCheck)
- [ ] Success resets the counter; expired lock = fresh start (LoginThrottleServiceTest)
- [ ] No new dependencies, env vars, or migrations"
```

Replace `<ISSUE_NUMBER>` with the implementation Issue number.

---

## Execution contract (non-negotiables)

1. Throttle is per normalized username (trim + lowercase); no IP tracking.
2. 5 consecutive failures → fixed 15-minute lock; attempts during the lock return 429 and do not extend it; expired lock = fresh start (next failure counts as 1).
3. `assertNotLocked` runs before the repository lookup in `AuthService.login`.
4. 429 body uses the existing `ErrorResponse` shape with the exact message "Too many failed login attempts. Try again in a few minutes." — never reveals whether the username exists.
5. Successful login clears the counter.
6. In-memory only: no new dependencies, no new env vars, no migrations, no changes to register/check-username.
