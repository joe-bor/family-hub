> Archived 2026-04-24 — implemented by joe-bor/family-hub-api#14

# BE Task: Testing Foundation — Part 2: Unit Tests (Services & Mappers)

## Context

This is prompt 2 of 4. Prompt 1 set up test infrastructure. This prompt adds Mockito-based unit tests for the service and mapper layers.

**Branch:** Continue on `test/testing-foundation`.

**Reference:** PR #12 has a previous implementation of these tests. The test logic is mostly correct — use it as a reference for what to test. This prompt tells you *what's worth testing and what isn't*.

---

## What Unit Tests Prove

Unit tests with Mockito mock all dependencies and test one class in isolation. They're fast (milliseconds, no DB) and good for exhaustively testing **branching logic**. They're NOT useful for methods that just delegate to a repository with no decisions.

---

## Service Tests to Write

### `CalendarEventServiceTest` — Full coverage, highest value

This is the meatiest service. It has real business logic between the repository calls: ownership checks, date filtering, time range validation, all-day skip logic.

**Test these scenarios:**
- `getAllEventsByFamily` — unfiltered, filtered by date range (include + exclude), filtered by memberId, member from wrong family throws `AccessDeniedException`
- `getEventById` — success, not found throws `ResourceNotFoundException`
- `addCalendarEvent` — success, invalid time range throws `BadRequestException`, all-day event skips time validation, member from wrong family throws `AccessDeniedException`
- `updateCalendarEvent` — success (verify save is called)
- `deleteCalendarEvent` — success, not found throws

See PR #12's `CalendarEventServiceTest.java` for the full implementation — the test logic there is correct.

### `AuthServiceTest` — Full coverage

Business logic: duplicate username check, password encoding, JWT generation, credential validation.

**Test these scenarios:**
- `register` — success (verify save + token generation), duplicate username throws `UsernameAlreadyExists` (verify save is never called)
- `login` — success, invalid password throws `InvalidCredentialException`, unknown username throws `InvalidCredentialException`
- `checkUsername` — available, taken

### `FamilyMemberServiceTest` — Ownership checks matter

**Test these scenarios:**
- `findAllMembers` — returns list
- `findById` — success, not found throws, wrong family throws `AccessDeniedException`
- `addFamilyMember` — success
- `updateFamilyMember` — success (verify fields changed), wrong family throws
- `deleteFamilyMember` — success, wrong family throws

### `FamilyServiceTest` — Selective coverage

This service is mostly delegation (`findById + orElseThrow`). Only `updateFamily` has meaningful branching (null checks for partial updates).

**Test these:**
- `updateFamily` — partial update (name only), partial update (username only), full update
- `deleteFamily` — success, not found throws

**Optional / low-value** (write them if you want, but know they're just testing "mock returns value, service passes it through"):
- `findFamilyResponse` — success, not found
- `findFamilyById` — success, not found

### `JwtServiceTest` — No mocks, real crypto

This test doesn't use Mockito — it creates a real `JwtService` with a known secret and tests actual token generation/validation.

**Test these:**
- `generateToken` — contains correct subject (family ID)
- `extractSubject` — valid token, expired token throws `JwtException`, tampered token throws, wrong issuer throws

See PR #12's `JwtServiceTest.java` for the setup pattern (creating `JwtConfig` manually).

---

## Mapper Tests to Write

### `CalendarEventMapperTest` — High value, pure logic

Mappers are the ideal unit test target: pure functions, no dependencies, testable edge cases.

**Test these:**
- `toEntity` — parses "2:00 PM" → `LocalTime.of(14, 0)` correctly, handles all-day events, null `isAllDay` defaults to false, invalid time string throws `BadRequestException`
- `toDto` — formats `LocalTime.of(9, 0)` → "9:00 AM", maps all fields correctly

### `FamilyMapperTest` and `FamilyMemberMapperTest`

If these mappers exist and have logic, test them. If they're trivial (just copying fields), a single happy-path test per method is sufficient.

---

## Patterns to Follow

**Test structure:**
```java
@ExtendWith(MockitoExtension.class)
class SomeServiceTest {

    @Mock
    private SomeRepository someRepository;

    @InjectMocks
    private SomeService someService;

    // Use TestDataFactory for setup
    @BeforeEach
    void setUp() {
        family = TestDataFactory.createFamily();
    }
}
```

**Assertion style:** Use AssertJ (`assertThat`) not JUnit assertions. Use `assertThatThrownBy` for exception tests.

**Test naming:** `methodName_scenario_expectedResult` (e.g., `addCalendarEvent_invalidTimeRange_throws`)

---

## Do NOT

- Test pure delegation methods exhaustively — if a method is just `findById + orElseThrow + toDto`, one test is enough to confirm wiring. The integration tests will cover the full flow.
- Add `@SpringBootTest` or any Spring context — these are pure Mockito tests.
- Modify production code.

---

## Commits

Make two atomic commits:
1. `test(unit): add service layer unit tests`
2. `test(unit): add mapper unit tests`
