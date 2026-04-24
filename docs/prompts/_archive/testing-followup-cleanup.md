> Archived 2026-04-24 — implemented by joe-bor/family-hub-api#15

# Testing Follow-up: Cleanup & Hardening

> Follow-up to PR #14 (testing foundation). Addresses review findings that weren't blockers but improve consistency, security posture, and test coverage.

## Context

PR #14 established the testing foundation across all 4 layers. During the final review, several non-critical items were identified. This prompt groups them into a single follow-up PR.

Branch from: `main` (after PR #14 is merged)
Branch name: `test/testing-followup-cleanup`

---

## 1. Remove hardcoded JWT test secret from `application-test.yml`

**Why:** GitGuardian flags the base64 string `dGVzdC1zZWNyZXQta2V5LXRoYXQtaXMtbG9uZy1lbm91Z2gtZm9yLWhtYWMtc2hhLTI1Ng==` as a leaked secret on every PR. It's not a real secret (decodes to `test-secret-key-that-is-long-enough-for-hmac-sha-256`), but the false positives create noise and train the team to ignore GitGuardian alerts — which is the real danger.

**What to do:**
- Remove the `jwt.secret`, `jwt.expiration`, and `jwt.issuer` values from `src/test/resources/application-test.yml`
- Instead, set them via environment variables in the CI workflow (`.github/workflows/ci.yml`), e.g.:
  ```yaml
  - name: Build and test
    run: ./mvnw verify
    env:
      JWT_SECRET: dGVzdC1zZWNyZXQta2V5LXRoYXQtaXMtbG9uZy1lbm91Z2gtZm9yLWhtYWMtc2hhLTI1Ng==
      JWT_EXPIRATION: 86400000
      JWT_ISSUER: family-hub-api-test
  ```
- Update `application-test.yml` to reference the env vars, same pattern as `application.yml`:
  ```yaml
  jwt:
    secret: ${JWT_SECRET}
    expiration: ${JWT_EXPIRATION:86400000}
    issuer: ${JWT_ISSUER:family-hub-api-test}
  ```
- Note: `expiration` and `issuer` aren't sensitive, so they have defaults. Only `secret` is truly required from the env.
- For local test runs, developers set `JWT_SECRET` in their shell or IDE run config. Document this briefly in the project README or a `CONTRIBUTING.md` note if one exists.
- Verify `JwtServiceTest` still works — it constructs `JwtService` manually with its own secret, so it doesn't depend on `application-test.yml` at all. No changes needed there.

**Verify:** Run `./mvnw verify` with the env var set. All tests pass. The secret string no longer appears in any committed file.

---

## 2. `FamilyMemberService` — return DTOs instead of entities

**Why:** `addFamilyMember` and `updateFamilyMember` in `FamilyMemberService` return `FamilyMember` (entity), forcing the controller to call `FamilyMemberMapper.toDto()` itself. Every other service method returns DTOs. This inconsistency:
- Leaks JPA entities into the controller layer
- Forces `FamilyMemberControllerTest` to construct entity objects (see `sampleMemberEntity()`) instead of mocking with DTOs like every other controller test

**Current code** (`FamilyMemberController.java:47`):
```java
FamilyMember familyMember = familyMemberService.addFamilyMember(family, request);
URI location = URI.create("/api/family/members/" + familyMember.getId());
return ResponseEntity.created(location).body(
        new ApiResponse<>(FamilyMemberMapper.toDto(familyMember), "Family Member Added")
);
```

**What to do:**
- Change `FamilyMemberService.addFamilyMember` to return `FamilyMemberResponse` instead of `FamilyMember`
- Same for `updateFamilyMember`
- The service should call `FamilyMemberMapper.toDto()` internally before returning
- **Problem:** The controller needs the `id` for the `Location` header URI. Options:
  - Return the DTO (which already contains the `id` field) and use `response.id()` for the URI
  - This is the cleanest approach since `FamilyMemberResponse` already has the `id`
- Update `FamilyMemberController` to use the DTO directly
- Update `FamilyMemberControllerTest` — replace `sampleMemberEntity()` with a `sampleMemberResponse()` pattern matching the other controller tests
- Update `FamilyMemberServiceTest` assertions accordingly (assert on DTO fields, not entity fields)

**Verify:** All tests pass. No JPA entity types appear in controller method signatures.

---

## 3. Add `FamilyService.findFamilyResponse` unit test

**Why:** `FamilyServiceTest` covers `updateFamily` and `deleteFamily` but not `findFamilyResponse`. The method has a findById + map pattern with a `ResourceNotFoundException` branch. Integration tests cover it indirectly, but the unit layer should test both branches for consistency with how other services are tested.

**What to add** in `FamilyServiceTest.java`:
```java
@Test
void findFamilyResponse_found_returnsResponse() {
    when(familyRepository.findById(FAMILY_ID)).thenReturn(Optional.of(family));

    FamilyResponse result = familyService.findFamilyResponse(FAMILY_ID);

    assertThat(result.id()).isEqualTo(FAMILY_ID);
    assertThat(result.name()).isEqualTo("Test Family");
}

@Test
void findFamilyResponse_notFound_throws() {
    UUID unknownId = UUID.randomUUID();
    when(familyRepository.findById(unknownId)).thenReturn(Optional.empty());

    assertThatThrownBy(() -> familyService.findFamilyResponse(unknownId))
            .isInstanceOf(ResourceNotFoundException.class);
}
```

**Verify:** Tests pass. `FamilyServiceTest` now covers all public methods.

---

## 4. Controller tests — use `TestDataFactory` constants for UUIDs

**Why:** Controller tests define their own `private static final UUID FAMILY_ID = ...` instead of importing from `TestDataFactory`. The values happen to match, but this creates a maintenance risk — if `TestDataFactory` IDs change, controller tests silently diverge.

**What to do:**
- In `AuthControllerTest`, `CalendarEventControllerTest`, `FamilyControllerTest`, `FamilyMemberControllerTest`:
  - Remove local UUID constants (`FAMILY_ID`, `MEMBER_ID`, `EVENT_ID`)
  - Import from `TestDataFactory` via `import static com.familyhub.demo.TestDataFactory.*`
  - Update references accordingly
- `HealthControllerTest` has no UUIDs, so no changes needed
- The `WithMockFamily` annotation default `familyId` already matches `TestDataFactory.FAMILY_ID` — verify this stays in sync

**Verify:** Tests pass. `grep -r "UUID.fromString" src/test/java/com/familyhub/demo/controller/` returns no results (all UUIDs come from TestDataFactory).

---

## Acceptance Criteria

- [ ] No JWT secret string appears in any committed file (grep for the base64 value)
- [ ] `./mvnw verify` passes with `JWT_SECRET` set as env var
- [ ] `FamilyMemberService.addFamilyMember` and `updateFamilyMember` return `FamilyMemberResponse`
- [ ] No JPA entity types in `FamilyMemberController` method bodies (only DTOs)
- [ ] `FamilyServiceTest` covers `findFamilyResponse` (found + not found)
- [ ] Controller tests import UUIDs from `TestDataFactory`, no local UUID constants
- [ ] All existing tests still pass
- [ ] GitGuardian no longer flags the PR

## Commit strategy

Atomic commits per item:
1. `chore(test): externalize JWT test secret to env var`
2. `refactor: return DTOs from FamilyMemberService`
3. `test(unit): add FamilyService.findFamilyResponse tests`
4. `refactor(test): use TestDataFactory constants in controller tests`
