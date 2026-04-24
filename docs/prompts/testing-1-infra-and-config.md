# BE Task: Testing Foundation — Part 1: Infrastructure & Configuration

## Context

We're building a test suite from scratch for the Family Hub API. This is the first of 4 prompts — it sets up the dependencies, test configuration, shared utilities, and CI workflow that all subsequent test prompts depend on.

**Branch:** Create `test/testing-foundation` from `main`. All 4 prompts build on this branch sequentially.

**Reference:** PR #12 (`test/testing-foundation`) has a previous implementation. Use it for reference but do NOT copy blindly — several decisions in that PR were incorrect. This prompt specifies what we actually want.

---

## 1. Dependencies

Add these test dependencies to `pom.xml`:

```xml
<!-- Testcontainers — spins up real Postgres for tests -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>

<!-- JsonPath — for extracting values from JSON responses in integration tests -->
<dependency>
    <groupId>com.jayway.jsonpath</groupId>
    <artifactId>json-path</artifactId>
    <scope>test</scope>
</dependency>
```

**Do NOT add H2 test dependencies.** Tests must run against real Postgres via Testcontainers to match production.

---

## 2. Test Configuration — `application-test.yml`

Create `src/test/resources/application-test.yml`:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  flyway:
    enabled: true
  sql:
    init:
      mode: never

jwt:
  secret: dGVzdC1zZWNyZXQta2V5LXRoYXQtaXMtbG9uZy1lbm91Z2gtZm9yLWhtYWMtc2hhLTI1Ng==
  expiration: 86400000
  issuer: family-hub-api

cors:
  allowed-origins: http://localhost
```

### Critical: Why `validate` + Flyway, not `create-drop`

Production uses Flyway migrations as the source of truth for the database schema (`V1__initial_schema.sql`). Tests must use the **same schema source** — otherwise the test DB and production DB can drift apart silently.

- `flyway.enabled: true` — Flyway runs migrations against the Testcontainers Postgres, producing the exact production schema
- `ddl-auto: validate` — Hibernate checks that entity annotations match the Flyway-generated schema at startup. If they drift, the test fails immediately with a clear error
- **Do NOT use `create-drop`** — that generates the schema from JPA annotations, creating a separate source of truth

---

## 3. Testcontainers Config

Create `src/test/java/com/familyhub/demo/config/TestcontainersConfig.java`:

```java
@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfig {

    private static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return POSTGRES;
    }
}
```

**How this works:**
- `static final` — one Postgres container for the entire test run (singleton per classloader). All test contexts share it.
- `@ServiceConnection` — Spring Boot auto-configures `spring.datasource.*` from the running container. No manual URL/username/password config needed.
- The container starts automatically when the first test context that imports this config is created.

---

## 4. Custom Security Annotation — `@WithMockFamily`

The app uses `Family` as the `UserDetails` implementation. Most endpoints require authentication. Create a custom annotation so tests can simulate an authenticated family without needing a real JWT.

Create `src/test/java/com/familyhub/demo/security/WithMockFamily.java`:

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@WithSecurityContext(factory = WithMockFamilySecurityContextFactory.class)
public @interface WithMockFamily {
    String id() default "00000000-0000-0000-0000-000000000001";
    String name() default "Test Family";
    String username() default "testfamily";
}
```

Create `src/test/java/com/familyhub/demo/security/WithMockFamilySecurityContextFactory.java`:

```java
public class WithMockFamilySecurityContextFactory implements WithSecurityContextFactory<WithMockFamily> {

    @Override
    public SecurityContext createSecurityContext(WithMockFamily annotation) {
        Family family = new Family();
        family.setId(UUID.fromString(annotation.id()));
        family.setName(annotation.name());
        family.setUsername(annotation.username());
        family.setPasswordHash("encoded-password");
        family.setFamilyMembers(new ArrayList<>());

        UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(family, null, family.getAuthorities());

        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(auth);
        return context;
    }
}
```

**Why this exists:** `@WebMvcTest` controller tests mock the service layer but still need the security filter chain to be active (to test 401s). `@WithMockFamily` on a test method populates the security context as if a valid JWT was present, without actually going through the JWT filter.

The default ID (`00000000-0000-0000-0000-000000000001`) should match the `FAMILY_ID` constant in `TestDataFactory` (below).

---

## 5. Test Data Factory

Create `src/test/java/com/familyhub/demo/TestDataFactory.java` — a central place for creating test entities and DTOs.

See PR #12's `TestDataFactory.java` for the full implementation. It should include:
- Static UUID constants: `FAMILY_ID`, `MEMBER_ID`, `EVENT_ID`, `OTHER_FAMILY_ID`
- Factory methods for entities: `createFamily()`, `createOtherFamily()`, `createFamilyMember(Family)`, `createCalendarEvent(Family, FamilyMember)`
- Factory methods for DTOs: `createRegisterRequest()`, `createLoginRequest()`, `createFamilyMemberRequest()`, `createCalendarEventRequest(UUID memberId)`, `createFamilyRequest()`, `createFamilyResponse(Family)`, `createFamilyMemberResponse()`, `createCalendarEventResponse()`

**Important:**
- Entity factory methods are for unit tests (Mockito-based) where you need in-memory objects with known IDs
- Integration tests should NOT use these entity factories — they should create data through the API (HTTP requests) to test the real flow
- `FAMILY_ID` should be `00000000-0000-0000-0000-000000000001` to match `@WithMockFamily` defaults

---

## 6. CI Workflow

Create `.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      - name: Run tests
        run: ./mvnw verify
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: target/surefire-reports/
```

**Notes:**
- `ubuntu-latest` has Docker pre-installed — Testcontainers works out of the box
- Testcontainers pulls `postgres:16-alpine`, starts it, Flyway runs migrations, tests execute, container is torn down at JVM exit
- The Docker image build+push step will come in a separate PR — this workflow is just the test gate

---

## 7. Remove Placeholder Test

Delete `src/test/java/com/familyhub/demo/FamilyHubApplicationTests.java` if it exists (the default Spring Boot context load test). It'll fail without a database configured, and the integration tests will cover context loading.

---

## Do NOT

- Add `@OneToMany` fields to `Family` or `FamilyMember` pointing to `CalendarEvent`. Production code doesn't use them and the cascade is handled by `ON DELETE CASCADE` in the Flyway migration.
- Use `ddl-auto: create-drop`. The schema source of truth is Flyway.
- Modify any production source code. This prompt is test infrastructure only.

---

## Verification

After completing this prompt:
1. `./mvnw compile -pl . -q` should succeed (no compilation errors)
2. The test profile should be loadable — a simple `@SpringBootTest @Import(TestcontainersConfig.class) @ActiveProfiles("test")` context load test can verify Flyway runs migrations and Hibernate validates the schema
3. Commit with: `test(infra): add test configuration and shared utilities`
