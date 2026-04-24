# BE Task: Testing Foundation — Part 3: Slice Tests (Controllers & Repositories)

## Context

This is prompt 3 of 4. This prompt adds Spring slice tests — `@WebMvcTest` for controllers and `@DataJpaTest` for repositories. These test specific layers with real Spring context but without loading the full application.

**Branch:** Continue on `test/testing-foundation`.

**Reference:** PR #12 has a previous implementation. The patterns are mostly correct. Key corrections are noted below.

---

## Controller Tests (`@WebMvcTest`)

### How `@WebMvcTest` Works

`@WebMvcTest(FooController.class)` loads ONLY the web layer — the specified controller, MockMvc, and any `@ControllerAdvice`. It excludes `@Service`, `@Repository`, `@Configuration`.

**Because it excludes `@Configuration`, your `SecurityConfig` is not loaded automatically.** You must import it manually, along with the JWT filter and entry point, so that security is enforced in the test (otherwise 401 tests would break).

Additionally, `JwtAuthenticationFilter` depends on `JwtService` and `FamilyService` (it calls `FamilyService.findFamilyById()` to load the family from the JWT subject). Since the filter is in the context, Spring needs mock beans for both — **even for controllers that don't need authentication** (like `HealthController`).

### Required Preamble for Every Controller Test

```java
@WebMvcTest(SomeController.class)
@Import({SecurityConfig.class, JwtAuthenticationFilter.class, JwtAuthenticationEntryPoint.class})
@ActiveProfiles("test")
class SomeControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private JwtService jwtService;       // Required by JwtAuthenticationFilter
    @MockitoBean
    private FamilyService familyService; // Required by JwtAuthenticationFilter

    // Add @MockitoBean for the controller's actual service dependency
}
```

**Why `FamilyService` is in `HealthControllerTest`:** Not because the health endpoint needs it, but because the security filter chain is in the context and Spring can't wire the filter without it. The mock is never called during the test. This is a consequence of `Family implements UserDetails` causing tight coupling between the security layer and the domain service. It's a known tradeoff — not worth fixing now, but worth understanding.

### Controller Tests to Write

**`AuthControllerTest`** — tests public (unauthenticated) endpoints:
- `register` — returns 200 with token
- `register` — duplicate username returns 409
- `register` — validation errors return 400 **with field-specific assertions** (see below)
- `login` — returns 200 with token
- `login` — invalid credentials returns 401
- `checkUsername` — returns 200 with availability

**`FamilyControllerTest`** — authenticated endpoints, use `@WithMockFamily`:
- `getFamily` — returns 200
- `getFamily` — unauthenticated returns 401
- `updateFamily` — returns 200
- `deleteFamily` — returns 204

**`FamilyMemberControllerTest`** — authenticated endpoints:
- `getMembers` — returns 200
- `getMemberById` — returns 200
- `addMember` — returns 201 with Location header
- `updateMember` — returns 200
- `deleteMember` — returns 204
- `getMembers` — unauthenticated returns 401

**`CalendarEventControllerTest`** — authenticated endpoints:
- `getEvents` — returns 200, verify response shape
- `getEvents` — with filter params
- `getEventById` — returns 200
- `addEvent` — returns 201 with Location header
- `addEvent` — validation error returns 400 **with field-specific assertions**
- `updateEvent` — returns 200
- `deleteEvent` — returns 204
- `getEvents` — unauthenticated returns 401

**`HealthControllerTest`** — public endpoint, no `@WithMockFamily`:
- `health` — returns 200 with status "UP", no auth required

### Validation Test Specificity

Don't just assert `status().isBadRequest()` + `jsonPath("$.errors").isArray()`. Assert on **which field** failed validation. This catches regressions if someone changes a `@Size(min=3)` annotation.

```java
@Test
void register_shortUsername_returns400WithFieldError() throws Exception {
    mockMvc.perform(post("/api/auth/register")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            {
                                "username": "ab",
                                "password": "password123",
                                "familyName": "Test Family",
                                "members": [{"name": "John", "color": "coral"}]
                            }
                            """))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors[?(@.field == 'username')]").exists());
}
```

Add field-specific validation tests for: short username, short password, empty familyName, empty members list. Also for CalendarEvent: empty title, null date, null memberId.

---

## Repository Tests (`@DataJpaTest`)

### How `@DataJpaTest` Works

Loads only the JPA layer — entities, repositories, Hibernate, and the datasource. No controllers, no services, no security.

### Required Preamble

```java
@DataJpaTest
@Import(TestcontainersConfig.class)
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@ActiveProfiles("test")
class SomeRepositoryTest {
    // ...
}
```

- `@Import(TestcontainersConfig.class)` — connects to the Testcontainers Postgres
- `AutoConfigureTestDatabase(replace = NONE)` — prevents Spring from replacing the datasource with an embedded DB
- `@ActiveProfiles("test")` — uses `application-test.yml` which has `flyway.enabled: true` and `ddl-auto: validate`

**With Flyway enabled, the test database schema matches production exactly.** This means cascade behavior (`ON DELETE CASCADE` in `V1__initial_schema.sql`) works at the DB level.

### Repository Test Data Setup

Repository tests create entities directly through repositories (not through the API). Use local helper methods, not `TestDataFactory`, because repository tests need persisted entities with DB-generated IDs:

```java
private Family createAndSaveFamily(String username) {
    Family family = new Family();
    family.setName("Test Family");
    family.setUsername(username);
    family.setPasswordHash("encoded");
    family.setFamilyMembers(new ArrayList<>());
    return familyRepository.save(family);
}
```

**Use unique usernames per test** — `@DataJpaTest` classes share a cached context and database. Data accumulates across test methods.

### Repository Tests to Write

**`FamilyRepositoryTest`:**
- `existsByUsername` — true and false cases
- `findByUsername` — present and empty
- `uniqueUsernameConstraint` — saving duplicate throws `DataIntegrityViolationException`

**`FamilyMemberRepositoryTest`:**
- `findByFamily` — returns members for the right family
- `findByFamily` — doesn't return other family's members

**`CalendarEventRepositoryTest`:**
- `findByFamily` — returns events
- `findByFamilyAndId` — match, no match (wrong family), no match (wrong ID)
- `cascadeDelete_whenFamilyDeleted` — delete a family that has members and events, assert all are gone. **This tests the DB-level `ON DELETE CASCADE` from the Flyway migration.** It should be simple:

```java
@Test
void cascadeDelete_whenFamilyDeleted() {
    Family family = createAndSaveFamily("cascade_family");
    FamilyMember member = createAndSaveMember(family);
    createAndSaveEvent(family, member);

    familyRepository.delete(family);
    familyRepository.flush();

    assertThat(calendarEventRepository.findAll()).isEmpty();
    assertThat(familyMemberRepository.findAll()).isEmpty();
}
```

**Do NOT manually delete events before deleting the family.** The whole point of this test is to verify that the DB cascade handles cleanup. If this test fails, it means the Flyway migration is missing `ON DELETE CASCADE` — which is a real bug to catch.

---

## Do NOT

- Add `@OneToMany` fields to `Family` or `FamilyMember` pointing to `CalendarEvent`. The cascade is DB-level, not JPA-level. Production code never calls `family.getCalendarEvents()`.
- Use `ddl-auto: create-drop` — the test profile uses Flyway + validate.
- Modify production code.

---

## Commits

Two atomic commits:
1. `test(slice): add controller tests with @WebMvcTest`
2. `test(slice): add repository tests with @DataJpaTest`
