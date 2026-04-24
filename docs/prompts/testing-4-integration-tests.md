# BE Task: Testing Foundation — Part 4: Integration Tests

## Context

This is the final prompt (4 of 4). Integration tests exercise the full stack — HTTP request through security, controllers, services, repositories, and a real Postgres database. They prove the app works end-to-end.

**Branch:** Continue on `test/testing-foundation`.

**Reference:** PR #12 has a previous implementation. The test scenarios are correct but had a data isolation issue. This prompt specifies the correct approach.

---

## How Integration Tests Work

```java
@SpringBootTest
@AutoConfigureMockMvc
@Import(TestcontainersConfig.class)
@ActiveProfiles("test")
class SomeIntegrationTest {

    @Autowired
    private MockMvc mockMvc;
}
```

- `@SpringBootTest` — loads the full application context (all controllers, services, repos, security)
- `@AutoConfigureMockMvc` — provides `MockMvc` for sending HTTP requests without starting a real server
- `@Import(TestcontainersConfig.class)` — connects to Testcontainers Postgres
- `@ActiveProfiles("test")` — Flyway runs migrations, `ddl-auto: validate` checks entity alignment

Integration tests create data **through the API** (HTTP POST to register, create events, etc.) — not through repositories directly. They use real JWT tokens extracted from registration responses.

---

## Test Data Isolation — Critical

All `@SpringBootTest` classes share a cached Spring context and therefore the **same database**. Data persists across test methods. Flyway migrations run once at context startup — `create-drop` is NOT used.

**Every test method must generate unique usernames.** Use the `nanoTime` pattern:

```java
String username = "auth" + (System.nanoTime() % 100000000);
```

**Do NOT use hardcoded usernames** like `"integrationtest"` or `"duplicate_user"`. These create order-dependent tests — if JUnit runs them in a different order, or a future test reuses the name, you get flaky failures.

The only exception: the duplicate-username test needs to use the same username twice **within the same method**. Generate it once, use it for both registrations:

```java
@Test
void register_duplicateUsername_returns409() throws Exception {
    String username = "dup" + (System.nanoTime() % 100000000);

    // First registration — succeeds
    mockMvc.perform(post("/api/auth/register")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(/* ... username ... */))
            .andExpect(status().isOk());

    // Same username — fails with 409
    mockMvc.perform(post("/api/auth/register")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(/* ... same username ... */))
            .andExpect(status().isConflict());
}
```

---

## Integration Tests to Write

### `AuthIntegrationTest`

Tests the full auth flow — register, login, use token to access protected endpoints.

**Scenarios:**
- `fullRegisterLoginFlow` — register (unique username), extract token, login with same credentials, use token to access `GET /api/family`, verify response data
- `register_invalidData_returns400` — short username, short password, empty familyName, empty members
- `protectedEndpoint_noToken_returns401`
- `protectedEndpoint_malformedToken_returns401` — send `"Bearer not-a-valid-jwt"`
- `register_duplicateUsername_returns409` — register twice with same generated username

**How to extract tokens and IDs from responses:**

```java
MvcResult result = mockMvc.perform(post("/api/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(/* ... */))
        .andExpect(status().isOk())
        .andReturn();

String body = result.getResponse().getContentAsString();
String token = com.jayway.jsonpath.JsonPath.read(body, "$.data.token");
String memberId = com.jayway.jsonpath.JsonPath.read(body, "$.data.family.members[0].id");
```

### `CalendarEventIntegrationTest`

Tests the full CRUD lifecycle for calendar events, including cross-family access denial.

**Setup:** Use `@BeforeEach` to register a family and extract `token` + `memberId`. Use unique usernames:

```java
@BeforeEach
void setUp() throws Exception {
    String uniqueUsername = "ct" + (System.nanoTime() % 100000000);
    // register, extract token and memberId
}
```

**Scenarios:**

- `fullEventLifecycle` — create event (201), query by date range (verify it's there), get by ID, delete (204), verify deleted (404). This is the most valuable integration test — it proves the whole chain works.

- `crossFamilyAccess_denied` — register two families, create event with family A, try to access it with family B's token. Should return 404 (not 403 — no information leakage about the event's existence).

---

## What Integration Tests Should NOT Do

- **Don't duplicate unit test coverage.** Don't test every edge case of time validation or ownership checks — service unit tests handle that. Integration tests prove "the wiring works" and "the main flows complete."
- **Don't use `TestDataFactory` entity methods.** Integration tests create data through the API. `TestDataFactory` is for Mockito-based unit tests where you need in-memory objects.
- **Don't assume a clean database.** Always generate unique data. Never rely on specific row counts (e.g., `assertThat(result).hasSize(1)` is fragile if another test left data behind).

---

## Do NOT

- Add `@OneToMany` fields to any entity. Not needed for integration tests — data is created through the API, cascades are handled by the DB.
- Use hardcoded usernames.
- Modify production code.

---

## Final Verification

After all 4 prompts are complete:

1. `./mvnw verify` — all tests pass
2. No production source files were modified (only test files, `pom.xml`, CI workflow, and `application-test.yml`)
3. Entity classes `Family.java`, `FamilyMember.java`, `CalendarEvent.java` are unchanged from `main`

---

## Commits

1. `test(integration): add full-stack integration tests`
2. `ci: add GitHub Actions workflow for test gate`

Then open a PR to `main` with title: `test: add testing foundation with unit, slice, and integration tests`
