# BE Task: Testing Foundation Review Fixes

## Context

The `test/testing-foundation` branch adds the full test suite for the API — unit, slice, and integration tests plus CI workflow. A mentor review identified several issues ranging from a real production bug to test quality improvements. This prompt captures the action items from that review.

**Review files:** `reviews/testing-foundation/` — read the SUMMARY.md and thread files for full context on *why* each change is needed.

**Branch:** Create a new branch off `test/testing-foundation` (e.g., `fix/testing-review-fixes`). This will be merged into the testing branch before it goes to main.

---

## Fixes — Sorted by Priority

### 1. Cascade Gap: Family → CalendarEvent (Bug — High)

**Current state:** `Family` has `cascade = CascadeType.ALL` to `FamilyMember`, but neither `Family` nor `FamilyMember` has a cascade relationship to `CalendarEvent`. The `CalendarEvent` entity has `@ManyToOne` references to both `Family` and `FamilyMember`, but there's no inverse `@OneToMany` on either parent.

**Problem:** Calling `familyRepository.delete(family)` when the family has calendar events will throw a foreign key constraint violation. This is a real production bug — `FamilyService.deleteFamily()` would fail.

**Fix:** Add a `@OneToMany` with cascade on `Family` (or `FamilyMember`, or both — your call) so that deleting a family cleans up its events. The most straightforward approach:

```java
// In Family.java — add alongside the existing familyMembers field
@OneToMany(mappedBy = "family", orphanRemoval = true, cascade = CascadeType.ALL)
private List<CalendarEvent> calendarEvents;
```

**Verify your choice:** Consider whether deleting a `FamilyMember` should also cascade-delete their events, or whether those events should be reassignable. This is a product decision — make it explicit.

**Then fix the test:** Rewrite `CalendarEventRepositoryTest.cascadeDelete_whenFamilyDeleted()` to actually test the cascade. It should just delete the family and assert that events (and members) are gone — no manual cleanup:

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

If this fails, the cascade isn't configured correctly. That's the point of the test.

---

### 2. Test Data Isolation in AuthIntegrationTest (Flaky Test Risk — Medium)

**Current state:** `AuthIntegrationTest` uses hardcoded usernames (`"integrationtest"`, `"duplicate_user"`). `CalendarEventIntegrationTest` uses `nanoTime`-based unique names. The inconsistency means `AuthIntegrationTest` can break if test execution order changes or new tests reuse those usernames.

**Problem:** `create-drop` only drops the schema at context shutdown, not between test methods. All methods within a `@SpringBootTest` class share the same database. Hardcoded usernames create order-dependent tests.

**Fix:** Adopt the `nanoTime` pattern in `AuthIntegrationTest`. Each test method that registers a user should generate a unique username:

```java
String username = "auth" + (System.nanoTime() % 100000000);
```

Apply this to:
- `fullRegisterLoginFlow()` — replace `"integrationtest"`
- `register_duplicateUsername_returns409()` — the first registration should use a unique name, the second should reuse it (that's the point of the test)
- Any other test that creates data

The `register_duplicateUsername_returns409` test is slightly tricky — it *needs* to use the same username twice, but that username should still be unique per run. Generate it once at the top of the method and use it for both registrations.

---

### 3. Validation Test Specificity (Test Quality — Low)

**Current state:** Controller tests like `AuthControllerTest.register_validationError_returns400()` send invalid payloads and assert `status().isBadRequest()` + `jsonPath("$.errors").isArray()`. They don't assert *which* validation errors fire.

**Problem:** If someone changes `@Size(min=3)` to `@Size(min=5)` on the username field, no test would catch it. The tests prove "bad input returns 400" but not "this specific rule is enforced."

**Fix:** Add targeted validation tests that assert on error messages or field names. For example:

```java
@Test
void register_shortUsername_returns400WithSpecificError() throws Exception {
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

Don't go overboard — you don't need a test for every validation annotation. Focus on the rules that matter most: username length, password length, non-empty members list, non-empty familyName.

---

## Not Code Fixes — Awareness Items

These came up in the review but don't need code changes right now:

- **Controller test boilerplate:** The repeated `@Import({SecurityConfig.class, ...})` and mock beans across all 5 controller tests is a known tradeoff from `Family implements UserDetails`. At 5 controllers, copy-paste is fine. If the controller count grows past ~15, consider extracting a `BaseControllerTest` class.

- **FamilyServiceTest thin tests:** Tests like `findFamilyResponse_success` and `deleteFamily_success` test pure delegation (no branching). They're low-value but also low-cost. Don't delete them, but when adding new modules, only write service unit tests for methods with real branching logic.

- **CI pipeline staging:** Current single-job `./mvnw verify` is correct for this app size. When CI time crosses ~3-5 minutes (likely after adding tasks/chores/meals modules), consider splitting into fast unit tests and slower integration tests using JUnit 5 `@Tag` annotations.

---

## Summary Checklist

- [ ] Add cascade from `Family` (and/or `FamilyMember`) to `CalendarEvent`
- [ ] Rewrite `cascadeDelete_whenFamilyDeleted` to test actual cascade behavior
- [ ] Decide: should deleting a `FamilyMember` cascade-delete their events?
- [ ] Replace hardcoded usernames in `AuthIntegrationTest` with `nanoTime`-based unique names
- [ ] Add 3-4 targeted validation tests that assert on specific field errors
- [ ] Run full `./mvnw verify` to confirm everything passes after changes
