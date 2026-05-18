# Recurring Chores Present-Focused Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the shipped one-off chores module with a recurring, present-focused chores board that computes `Today`, `This Week`, and `This Month` server-side, renders one scope at a time on mobile and three columns on larger screens, and keeps completed routines visible but secondary within the current period.

**Architecture:** Extend the backend family model with a stored timezone so chores can compute current periods consistently, then replace the current `chore` table/API with `chore_template + chore_period_completion + GET /api/chores/board`. On the frontend, reset the chores types, hooks, mocks, and UI around the new board contract, keep the current mobile sheet creation pattern, and render a mobile scope switcher plus a larger-screen three-column layout from the same payload.

**Tech Stack:** Spring Boot 4, Java 21, JPA/Hibernate, Flyway, Maven, React 19, TypeScript, TanStack Query, React Hook Form, Zod, Vitest, MSW, Playwright.

**Spec:** `docs/superpowers/specs/2026-05-17-recurring-chores-present-focused-design.md`

---

## Execution Order

1. Root docs are aligned first so the replacement chores story exists before FE and BE execution issues are created.
2. Root orchestration creates or updates BE and FE execution issues that link the replacement story, approved spec, and this plan.
3. Backend work happens first in `backend/family-hub-api/`: family timezone support, recurring chores schema reset, board contract, stale-period-safe completion writes, template archive/update behavior, and member-delete guard.
4. Backend tests pass and the backend contract is released before FE live verification that depends on the new `/api/chores/*` behavior.
5. Frontend work happens in `frontend/`: contract reset, mocks, hooks, mobile/desktop chores UI, create + archive + complete/uncomplete flows, then Playwright verification.
6. Remaining root docs are finalized last so the PRD and story metadata describe the shipped recurring-routines contract, not the replaced one-off chores contract.

Per root shipping rules, mock-backed FE work can start before the BE release exists, but FE E2E that depends on the real backend should verify against a published backend release, not unreleased BE `main`.

## Execution Contract

- This is a contract reset, not a compatibility migration.
- `Chores` becomes recurring routines only. One-off assigned chores are removed from the chores contract.
- `Lists` remains the ad-hoc shared checklist surface and must not absorb assignment, cadence, or period-completion semantics in this story.
- `Calendar` remains the scheduled-commitments surface. Chores can share the same effective time policy, but chores must not become calendar events or expand RRULE-style instances.
- Chores mirrors the current calendar week behavior exactly for MVP: Sunday-first week boundaries, no chores-specific week-start preference.
- The backend owns current-period computation. The client must send `scope` and `periodStartDate` when writing completion state so the server can reject stale requests after midnight, week rollover, or month-end.
- Completed routines remain visible within the current period, but only after incomplete routines.
- Active recurring chore templates block family-member deletion until they are archived or reassigned.

## File Structure

### Root docs / orchestration

- Create: `docs/superpowers/plans/2026-05-17-recurring-chores-present-focused.md`
- Create: `docs/product/backlog/module-foundations/chores-recurring-routines.md`
- Modify: `docs/product/backlog/module-foundations/chores-core-loop.md`
- Modify: `docs/product/roadmap.md`
- Modify: `docs/product/prd.md`
- Modify: `docs/product/backlog/module-foundations/module-surface-foundations.md`

### Backend (`backend/family-hub-api/`)

Expected create / modify / delete set:

- Create: `src/main/resources/db/migration/V13__add_family_timezone.sql`
- Create: `src/main/resources/db/migration/V14__replace_chores_with_recurring_tables.sql`
- Create: `src/main/java/com/familyhub/demo/config/TimeConfig.java`
- Modify: `src/main/java/com/familyhub/demo/model/Family.java`
- Modify: `src/main/java/com/familyhub/demo/dto/RegisterRequest.java`
- Modify: `src/main/java/com/familyhub/demo/service/AuthService.java`
- Modify: `src/test/java/com/familyhub/demo/service/AuthServiceTest.java`
- Create: `src/main/java/com/familyhub/demo/model/ChoreCadence.java`
- Create: `src/main/java/com/familyhub/demo/model/ChoreTemplate.java`
- Create: `src/main/java/com/familyhub/demo/model/ChorePeriodCompletion.java`
- Create: `src/main/java/com/familyhub/demo/model/ChoreScope.java`
- Create: `src/main/java/com/familyhub/demo/dto/CreateChoreTemplateRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/UpdateChoreTemplateRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/UpdateCurrentPeriodCompletionRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/ChoreTemplateResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/ChoreBoardItemResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/ChoreAssigneeGroupResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/ChoreScopeBoardResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/ChoreBoardResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/ChoreCurrentPeriodStateResponse.java`
- Create: `src/main/java/com/familyhub/demo/repository/ChoreTemplateRepository.java`
- Create: `src/main/java/com/familyhub/demo/repository/ChorePeriodCompletionRepository.java`
- Modify: `src/main/java/com/familyhub/demo/service/ChoreService.java`
- Modify: `src/main/java/com/familyhub/demo/controller/ChoreController.java`
- Modify: `src/main/java/com/familyhub/demo/service/FamilyMemberService.java`
- Modify: `src/test/java/com/familyhub/demo/TestDataFactory.java`
- Modify: `src/test/java/com/familyhub/demo/service/ChoreServiceTest.java`
- Modify: `src/test/java/com/familyhub/demo/controller/ChoreControllerTest.java`
- Modify: `src/test/java/com/familyhub/demo/integration/ChoreIntegrationTest.java`
- Modify: `src/test/java/com/familyhub/demo/service/FamilyMemberServiceTest.java`
- Modify: `src/test/java/com/familyhub/demo/controller/FamilyMemberControllerTest.java`
- Delete: `src/main/java/com/familyhub/demo/model/Chore.java`
- Delete: `src/main/java/com/familyhub/demo/dto/CreateChoreRequest.java`
- Delete: `src/main/java/com/familyhub/demo/dto/UpdateChoreRequest.java`
- Delete: `src/main/java/com/familyhub/demo/dto/ChoreResponse.java`
- Delete: `src/main/java/com/familyhub/demo/mapper/ChoreMapper.java`
- Delete: `src/main/java/com/familyhub/demo/repository/ChoreRepository.java`

### Frontend (`frontend/`)

Expected create / modify set:

- Modify: `src/lib/types/auth.ts`
- Modify: `src/lib/types/chores.ts`
- Modify: `src/lib/types/index.ts`
- Modify: `src/components/onboarding/onboarding-flow.tsx`
- Modify: `src/components/onboarding/onboarding-flow.test.tsx`
- Modify: `src/api/services/chores.service.ts`
- Modify: `src/api/services/index.ts`
- Modify: `src/api/hooks/use-chores.ts`
- Modify: `src/api/hooks/index.ts`
- Modify: `src/api/index.ts`
- Modify: `src/lib/validations/chores.ts`
- Modify: `src/lib/validations/chores.test.ts`
- Modify: `src/lib/validations/index.ts`
- Modify: `src/test/mocks/handlers.ts`
- Modify: `src/test/mocks/server.ts`
- Create: `src/components/chores/chores-scope-switcher.tsx`
- Create: `src/components/chores/chores-scope-column.tsx`
- Create: `src/components/chores/chore-assignee-group.tsx`
- Modify: `src/components/chores/chore-form.tsx`
- Modify: `src/components/chores/chore-form-sheet.tsx`
- Modify: `src/components/chores/chore-row.tsx`
- Modify: `src/components/chores/chore-lane.tsx`
- Modify: `src/components/chores-view.tsx`
- Modify: `src/api/hooks/use-chores.test.tsx`
- Modify: `src/components/chores-view.test.tsx`
- Modify: `e2e/mobile-chores.spec.ts`

This structure keeps the backend replacement focused on one service/controller pair with small DTOs, while the frontend keeps contract logic in `@/api`, rendering in `src/components/chores/`, and mobile vs desktop presentation in a thin top-level `ChoresView`.

## Task 0: Prepare Root Story Links Before FE And BE Issue Creation

**Files:**
- Create: `docs/product/backlog/module-foundations/chores-recurring-routines.md`
- Modify: `docs/product/backlog/module-foundations/chores-core-loop.md`
- Modify: `docs/product/roadmap.md`
- Modify: `docs/product/backlog/module-foundations/module-surface-foundations.md`

- [ ] **Step 1: Create the replacement chores story that FE and BE issues will link to**

```md
---
id: chores-recurring-routines
title: Chores recurring routines
epic: module-foundations
status: in-progress
priority: P1
created: 2026-05-17
updated: 2026-05-17
issues: []
prs: []
spec: ../../../superpowers/specs/2026-05-17-recurring-chores-present-focused-design.md
plan: ../../../superpowers/plans/2026-05-17-recurring-chores-present-focused.md
---

## Context

This story replaces the shipped one-off chores board with a recurring, present-focused routines module.

## Scope

- recurring routines only
- day / week / month scope rendering
- present-focused completion state
- stale-safe current-period writes
- no carry-forward

## Out of Scope

- one-off assigned chores
- history UI
- reminders, rewards, and streaks
```

- [ ] **Step 2: Mark the old shipped chores story as historical and superseded**

```md
## Supersession

This shipped story remains historical context for the first chores release only.
The live chores contract now continues in [Chores recurring routines](./chores-recurring-routines.md).
```

- [ ] **Step 3: Update roadmap and module-foundations indexes so new FE/BE issues can use the replacement story link**

```md
- [Chores recurring routines](backlog/module-foundations/chores-recurring-routines.md) — current chores contract replacing the initial one-off chores release
- [Chores core loop](backlog/module-foundations/chores-core-loop.md) — historical shipped one-off chores release
```

```md
Current chores story:
- [Chores recurring routines](./chores-recurring-routines.md)

Historical shipped chores story:
- [Chores core loop (real data, create, complete)](./chores-core-loop.md) — superseded by recurring routines on 2026-05-17
```

- [ ] **Step 4: Verify the replacement story is in place before execution issues are created**

Run:

```bash
cd /Users/joe.bor/code/family-hub
rg -n "chores-recurring-routines|superseded" docs/product/roadmap.md docs/product/backlog/module-foundations/chores-core-loop.md docs/product/backlog/module-foundations/module-surface-foundations.md docs/product/backlog/module-foundations/chores-recurring-routines.md
```

Expected: the new recurring-routines story exists, the old chores story is explicitly marked historical/superseded, and root indexes point at the replacement story for new work.

- [ ] **Step 5: Commit**

```bash
cd /Users/joe.bor/code/family-hub
git add docs/product/backlog/module-foundations/chores-recurring-routines.md \
  docs/product/backlog/module-foundations/chores-core-loop.md \
  docs/product/roadmap.md \
  docs/product/backlog/module-foundations/module-surface-foundations.md
git commit -m "docs(chores): prepare recurring chores story for issue derivation"
```

## Task 1: Add Family Timezone Groundwork On The Backend

**Files:**
- Create: `backend/family-hub-api/src/main/resources/db/migration/V13__add_family_timezone.sql`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/config/TimeConfig.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/Family.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/RegisterRequest.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/AuthService.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/service/AuthServiceTest.java`

- [ ] **Step 1: Write the failing auth-service tests for persisted family timezone**

```java
@Test
void register_withExplicitTimezone_persistsTimezone() {
    RegisterRequest request = new RegisterRequest(
            "smithfamily",
            "password123",
            "Smith Family",
            List.of(createFamilyMemberRequest()),
            "America/Chicago"
    );
    when(familyRepository.existsByUsername(request.username())).thenReturn(false);
    when(passwordEncoder.encode(anyString())).thenReturn("encoded-password");
    when(familyRepository.saveAndFlush(any(Family.class))).thenAnswer(invocation -> invocation.getArgument(0));
    when(jwtService.generateToken(any(Family.class))).thenReturn("jwt-token");

    authService.register(request);

    verify(familyRepository).saveAndFlush(argThat(saved ->
            "America/Chicago".equals(saved.getTimezone())
    ));
}

@Test
void register_withoutTimezone_fallsBackToPacificDefault() {
    RegisterRequest request = new RegisterRequest(
            "smithfamily",
            "password123",
            "Smith Family",
            List.of(createFamilyMemberRequest()),
            null
    );
    when(familyRepository.existsByUsername(request.username())).thenReturn(false);
    when(passwordEncoder.encode(anyString())).thenReturn("encoded-password");
    when(familyRepository.saveAndFlush(any(Family.class))).thenAnswer(invocation -> invocation.getArgument(0));
    when(jwtService.generateToken(any(Family.class))).thenReturn("jwt-token");

    authService.register(request);

    verify(familyRepository).saveAndFlush(argThat(saved ->
            "America/Los_Angeles".equals(saved.getTimezone())
    ));
}
```

- [ ] **Step 2: Run the auth-service test to verify it fails because the timezone field does not exist yet**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=AuthServiceTest test
```

Expected: FAIL with compilation errors for `RegisterRequest(..., timezone)` and `Family.getTimezone()`.

- [ ] **Step 3: Add the family timezone migration, model field, config bean, and register fallback**

```sql
ALTER TABLE family
    ADD COLUMN timezone VARCHAR(100) NOT NULL DEFAULT 'America/Los_Angeles';

UPDATE family
SET timezone = 'America/Los_Angeles'
WHERE timezone IS NULL;
```

```java
@Configuration
public class TimeConfig {
    @Bean
    Clock clock() {
        return Clock.systemUTC();
    }
}
```

```java
@Column(nullable = false, length = 100)
private String timezone = "America/Los_Angeles";
```

```java
public record RegisterRequest(
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 20, message = "Username must be between 3 and 20 characters")
    @Pattern(regexp = "^[a-z0-9_]+$", message = "Username can only contain lowercase letters, numbers, and underscores")
    String username,

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100, message = "Password must be between 8 - 100 characters")
    String password,

    @NotBlank(message = "Family name is required")
    @Size(max = 50, message = "Family name must be 50 characters or less")
    String familyName,

    @NotEmpty(message = "A family must have at least one member")
    List<@Valid FamilyMemberRequest> members,

    @Size(max = 100, message = "Timezone must be 100 characters or less")
    String timezone
) {}
```

```java
private static final String DEFAULT_FAMILY_TIMEZONE = "America/Los_Angeles";

@Transactional
public AuthResponse register(RegisterRequest registerRequest) {
    if (familyRepository.existsByUsername(registerRequest.username())) {
        throw new UsernameAlreadyExists(registerRequest.username());
    }

    Family family = new Family();
    family.setName(registerRequest.familyName());
    family.setUsername(registerRequest.username());
    family.setPasswordHash(passwordEncoder.encode(registerRequest.password()));
    family.setTimezone(
            registerRequest.timezone() == null || registerRequest.timezone().isBlank()
                    ? DEFAULT_FAMILY_TIMEZONE
                    : registerRequest.timezone().trim()
    );
    family.setFamilyMembers(
            registerRequest.members().stream()
                    .map(request -> FamilyMemberMapper.toEntity(request, family))
                    .toList()
    );

    Family saved = familyRepository.saveAndFlush(family);
    listSeedService.seedDefaultsForFamily(saved);
    String token = jwtService.generateToken(saved);
    return new AuthResponse(token, FamilyMapper.toDto(saved));
}
```

- [ ] **Step 4: Run the auth-service test to verify timezone persistence now passes**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=AuthServiceTest test
```

Expected: PASS for the two timezone tests and the existing registration/login coverage.

- [ ] **Step 5: Commit**

```bash
cd backend/family-hub-api
git add src/main/resources/db/migration/V13__add_family_timezone.sql \
  src/main/java/com/familyhub/demo/config/TimeConfig.java \
  src/main/java/com/familyhub/demo/model/Family.java \
  src/main/java/com/familyhub/demo/dto/RegisterRequest.java \
  src/main/java/com/familyhub/demo/service/AuthService.java \
  src/test/java/com/familyhub/demo/service/AuthServiceTest.java
git commit -m "feat(chores): add family timezone groundwork"
```

## Task 2: Replace The Chores Persistence Model With Templates And Period Completions

**Files:**
- Create: `backend/family-hub-api/src/main/resources/db/migration/V14__replace_chores_with_recurring_tables.sql`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/ChoreCadence.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/ChoreTemplate.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/ChorePeriodCompletion.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/ChoreScope.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/ChoreTemplateRepository.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/ChorePeriodCompletionRepository.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/TestDataFactory.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/service/ChoreServiceTest.java`
- Delete: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/Chore.java`
- Delete: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/ChoreRepository.java`

- [ ] **Step 1: Write the failing chore-service tests for cadence scope mapping, activeFrom, and completed-last ordering**

```java
@Test
void getBoard_groupsTemplatesIntoTodayWeekAndMonth() {
    family.setTimezone("America/Los_Angeles");
    FamilyMember leo = createFamilyMember(family);

    ChoreTemplate daily = createChoreTemplate(family, leo, "🪥 Brush teeth", ChoreCadence.DAILY, LocalDate.of(2026, 5, 17));
    ChoreTemplate weekly = createChoreTemplate(family, leo, "🗑️ Take out trash", ChoreCadence.WEEKLY, LocalDate.of(2026, 5, 17));
    ChoreTemplate monthly = createChoreTemplate(family, leo, "🧼 Deep clean fridge", ChoreCadence.MONTHLY, LocalDate.of(2026, 5, 17));

    when(choreTemplateRepository.findActiveByFamily(family)).thenReturn(List.of(daily, weekly, monthly));
    when(chorePeriodCompletionRepository.findByTemplateIdsAndPeriod(any(), any(), any())).thenReturn(List.of());

    ChoreBoardResponse board = choreService.getBoard(family);

    assertThat(board.today().summary().total()).isEqualTo(1);
    assertThat(board.thisWeek().summary().total()).isEqualTo(1);
    assertThat(board.thisMonth().summary().total()).isEqualTo(1);
}

@Test
void getBoard_excludesTemplateBeforeActiveFromDate() {
    family.setTimezone("America/Los_Angeles");
    FamilyMember leo = createFamilyMember(family);

    ChoreTemplate futureMonthly = createChoreTemplate(
            family,
            leo,
            "🧼 Deep clean fridge",
            ChoreCadence.MONTHLY,
            LocalDate.of(2026, 6, 1)
    );

    when(choreTemplateRepository.findActiveByFamily(family)).thenReturn(List.of(futureMonthly));
    when(chorePeriodCompletionRepository.findByTemplateIdsAndPeriod(any(), any(), any())).thenReturn(List.of());

    ChoreBoardResponse board = choreService.getBoard(family);

    assertThat(board.thisMonth().summary().total()).isZero();
}

@Test
void getBoard_ordersIncompleteBeforeCompleted_thenByCreatedAtAndTitle() {
    family.setTimezone("America/Los_Angeles");
    FamilyMember leo = createFamilyMember(family);

    ChoreTemplate bravo = createChoreTemplate(
            family,
            leo,
            "Bravo",
            ChoreCadence.DAILY,
            LocalDate.of(2026, 5, 17)
    );
    bravo.setCreatedAt(LocalDateTime.of(2026, 5, 17, 9, 0));

    ChoreTemplate alpha = createChoreTemplate(
            family,
            leo,
            "Alpha",
            ChoreCadence.DAILY,
            LocalDate.of(2026, 5, 17)
    );
    alpha.setCreatedAt(LocalDateTime.of(2026, 5, 17, 8, 0));

    ChoreTemplate completed = createChoreTemplate(
            family,
            leo,
            "Completed",
            ChoreCadence.DAILY,
            LocalDate.of(2026, 5, 17)
    );
    completed.setCreatedAt(LocalDateTime.of(2026, 5, 17, 7, 0));

    ChorePeriodCompletion completion = new ChorePeriodCompletion();
    completion.setChoreTemplate(completed);
    completion.setPeriodStartDate(LocalDate.of(2026, 5, 17));
    completion.setPeriodEndDate(LocalDate.of(2026, 5, 17));
    completion.setCompletedAt(LocalDateTime.of(2026, 5, 17, 10, 0));

    when(choreTemplateRepository.findActiveByFamily(family)).thenReturn(List.of(bravo, completed, alpha));
    when(chorePeriodCompletionRepository.findByTemplateIdsAndPeriod(any(), any(), any()))
            .thenReturn(List.of(completion));

    ChoreBoardResponse board = choreService.getBoard(family);

    assertThat(board.today().assignees().get(0).chores())
            .extracting(ChoreBoardItemResponse::title)
            .containsExactly("Alpha", "Bravo", "Completed");
    assertThat(board.today().assignees().get(0).chores().get(2).completed()).isTrue();
}
```

- [ ] **Step 2: Run the chore-service test to verify it fails because the recurring model does not exist yet**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=ChoreServiceTest test
```

Expected: FAIL with compilation errors for `ChoreTemplate`, `ChoreCadence`, and `ChoreBoardResponse`.

- [ ] **Step 3: Add the recurring chores migration, entities, repositories, and test fixtures**

```sql
DROP TABLE IF EXISTS chore;

CREATE TABLE chore_template (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL,
    assigned_to_member_id UUID NOT NULL,
    title VARCHAR(100) NOT NULL,
    cadence VARCHAR(20) NOT NULL,
    active_from DATE NOT NULL,
    archived_at TIMESTAMP(6),
    created_at TIMESTAMP(6) DEFAULT now(),
    updated_at TIMESTAMP(6) DEFAULT now(),
    CONSTRAINT fk_chore_template_family
        FOREIGN KEY (family_id) REFERENCES family(id) ON DELETE CASCADE,
    CONSTRAINT fk_chore_template_assignee_in_family
        FOREIGN KEY (family_id, assigned_to_member_id)
        REFERENCES family_member(family_id, id)
        ON DELETE CASCADE
);

CREATE TABLE chore_period_completion (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chore_template_id UUID NOT NULL,
    period_start_date DATE NOT NULL,
    period_end_date DATE NOT NULL,
    completed_at TIMESTAMP(6) NOT NULL,
    CONSTRAINT fk_chore_period_completion_template
        FOREIGN KEY (chore_template_id) REFERENCES chore_template(id) ON DELETE CASCADE,
    CONSTRAINT uk_chore_period_completion_template_period
        UNIQUE (chore_template_id, period_start_date, period_end_date)
);
```

```java
public enum ChoreCadence {
    DAILY,
    WEEKLY,
    MONTHLY
}
```

```java
@Entity
@Getter
@Setter
public class ChoreTemplate {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @ManyToOne
    @JoinColumn(name = "family_id", nullable = false)
    private Family family;

    @ManyToOne
    @JoinColumn(name = "assigned_to_member_id", nullable = false)
    private FamilyMember assignedToMember;

    @Column(nullable = false, length = 100)
    private String title;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private ChoreCadence cadence;

    @Column(nullable = false)
    private LocalDate activeFrom;

    private LocalDateTime archivedAt;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

```java
@Entity
@Getter
@Setter
public class ChorePeriodCompletion {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @ManyToOne
    @JoinColumn(name = "chore_template_id", nullable = false)
    private ChoreTemplate choreTemplate;

    @Column(nullable = false)
    private LocalDate periodStartDate;

    @Column(nullable = false)
    private LocalDate periodEndDate;

    @Column(nullable = false)
    private LocalDateTime completedAt;
}
```

```java
public interface ChoreTemplateRepository extends JpaRepository<ChoreTemplate, UUID> {
    @Query("""
        select ct
        from ChoreTemplate ct
        join fetch ct.assignedToMember m
        where ct.family = :family
          and ct.archivedAt is null
        order by lower(m.name), ct.createdAt, lower(ct.title)
        """)
    List<ChoreTemplate> findActiveByFamily(@Param("family") Family family);

    Optional<ChoreTemplate> findByFamilyAndId(Family family, UUID id);

    boolean existsByAssignedToMemberAndArchivedAtIsNull(FamilyMember member);
}
```

```java
public interface ChorePeriodCompletionRepository extends JpaRepository<ChorePeriodCompletion, UUID> {
    @Query("""
        select c
        from ChorePeriodCompletion c
        where c.choreTemplate.id in :templateIds
          and c.periodStartDate = :periodStart
          and c.periodEndDate = :periodEnd
        """)
    List<ChorePeriodCompletion> findByTemplateIdsAndPeriod(
            @Param("templateIds") List<UUID> templateIds,
            @Param("periodStart") LocalDate periodStart,
            @Param("periodEnd") LocalDate periodEnd
    );

    Optional<ChorePeriodCompletion> findByChoreTemplateAndPeriodStartDateAndPeriodEndDate(
            ChoreTemplate choreTemplate,
            LocalDate periodStartDate,
            LocalDate periodEndDate
    );
}
```

```java
public static final UUID CHORE_TEMPLATE_ID = UUID.fromString("00000000-0000-0000-0000-000000000004");

public static ChoreTemplate createChoreTemplate(
        Family family,
        FamilyMember member,
        String title,
        ChoreCadence cadence,
        LocalDate activeFrom
) {
    ChoreTemplate template = new ChoreTemplate();
    template.setId(UUID.randomUUID());
    template.setFamily(family);
    template.setAssignedToMember(member);
    template.setTitle(title);
    template.setCadence(cadence);
    template.setActiveFrom(activeFrom);
    template.setCreatedAt(activeFrom.atStartOfDay());
    template.setUpdatedAt(activeFrom.atStartOfDay());
    return template;
}
```

- [ ] **Step 4: Run the chore-service test to verify the recurring schema and fixtures compile**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=ChoreServiceTest test
```

Expected: FAIL with missing DTO/service implementation errors instead of missing entity/repository errors.

- [ ] **Step 5: Commit**

```bash
cd backend/family-hub-api
git add src/main/resources/db/migration/V14__replace_chores_with_recurring_tables.sql \
  src/main/java/com/familyhub/demo/model/ChoreCadence.java \
  src/main/java/com/familyhub/demo/model/ChoreTemplate.java \
  src/main/java/com/familyhub/demo/model/ChorePeriodCompletion.java \
  src/main/java/com/familyhub/demo/model/ChoreScope.java \
  src/main/java/com/familyhub/demo/repository/ChoreTemplateRepository.java \
  src/main/java/com/familyhub/demo/repository/ChorePeriodCompletionRepository.java \
  src/test/java/com/familyhub/demo/TestDataFactory.java \
  src/test/java/com/familyhub/demo/service/ChoreServiceTest.java
git commit -m "feat(chores): add recurring chores persistence model"
```

## Task 3: Implement The Backend Board Contract, Stale-Safe Completion Writes, And Member Delete Guard

**Files:**
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/CreateChoreTemplateRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/UpdateChoreTemplateRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/UpdateCurrentPeriodCompletionRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ChoreTemplateResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ChoreBoardItemResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ChoreAssigneeGroupResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ChoreScopeBoardResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ChoreBoardResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ChoreCurrentPeriodStateResponse.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/ChoreService.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/controller/ChoreController.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/FamilyMemberService.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/service/ChoreServiceTest.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/controller/ChoreControllerTest.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/integration/ChoreIntegrationTest.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/service/FamilyMemberServiceTest.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/controller/FamilyMemberControllerTest.java`

- [ ] **Step 1: Write the failing controller and service tests for the new board contract, stale completion writes, and member-delete block**

```java
@Test
@WithMockFamily
void getBoard_returnsTodayWeekAndMonth() throws Exception {
    ChoreBoardResponse board = new ChoreBoardResponse(
            "America/Los_Angeles",
            new ChoreScopeBoardResponse(
                    ChoreScope.TODAY,
                    LocalDate.of(2026, 5, 17),
                    LocalDate.of(2026, 5, 17),
                    new ChoreScopeBoardResponse.Summary(1, 0, 1),
                    List.of()
            ),
            new ChoreScopeBoardResponse(
                    ChoreScope.THIS_WEEK,
                    LocalDate.of(2026, 5, 17).with(TemporalAdjusters.previousOrSame(DayOfWeek.SUNDAY)),
                    LocalDate.of(2026, 5, 17).with(TemporalAdjusters.previousOrSame(DayOfWeek.SUNDAY)).plusDays(6),
                    new ChoreScopeBoardResponse.Summary(1, 0, 1),
                    List.of()
            ),
            new ChoreScopeBoardResponse(
                    ChoreScope.THIS_MONTH,
                    LocalDate.of(2026, 5, 1),
                    LocalDate.of(2026, 5, 31),
                    new ChoreScopeBoardResponse.Summary(1, 0, 1),
                    List.of()
            )
    );
    given(choreService.getBoard(any(Family.class))).willReturn(board);

    mockMvc.perform(get("/api/chores/board"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.today.scope").value("TODAY"))
            .andExpect(jsonPath("$.data.thisWeek.scope").value("THIS_WEEK"))
            .andExpect(jsonPath("$.data.thisMonth.scope").value("THIS_MONTH"));
}

@Test
@WithMockFamily
void completeCurrentPeriod_staleRequest_returns400() throws Exception {
    given(choreService.completeCurrentPeriod(eq(CHORE_TEMPLATE_ID), any(), any(Family.class)))
            .willThrow(new BadRequestException("Chore period is stale. Refresh and try again."));

    mockMvc.perform(put("/api/chores/templates/{id}/current-period-completion", CHORE_TEMPLATE_ID)
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                        {"scope":"THIS_WEEK","periodStartDate":"2026-05-04"}
                        """))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.message").value("Chore period is stale. Refresh and try again."));
}

@Test
@WithMockFamily
void updateTemplate_archive_returns200() throws Exception {
    given(choreService.updateTemplate(eq(CHORE_TEMPLATE_ID), any(), any(Family.class)))
            .willReturn(new ChoreTemplateResponse(
                    CHORE_TEMPLATE_ID,
                    "🪥 Brush teeth",
                    MEMBER_ID,
                    ChoreCadence.DAILY,
                    LocalDate.of(2026, 5, 17),
                    true,
                    LocalDateTime.of(2026, 5, 17, 8, 0),
                    LocalDateTime.of(2026, 5, 17, 9, 0)
            ));

    mockMvc.perform(patch("/api/chores/templates/{id}", CHORE_TEMPLATE_ID)
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                        {"archived":true}
                        """))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.archived").value(true));
}

@Test
@WithMockFamily
void uncompleteCurrentPeriod_returns200() throws Exception {
    given(choreService.uncompleteCurrentPeriod(eq(CHORE_TEMPLATE_ID), any(), any(Family.class)))
            .willReturn(new ChoreCurrentPeriodStateResponse(
                    ChoreScope.TODAY,
                    LocalDate.of(2026, 5, 17),
                    LocalDate.of(2026, 5, 17),
                    new ChoreBoardItemResponse(
                            CHORE_TEMPLATE_ID,
                            "🪥 Brush teeth",
                            ChoreCadence.DAILY,
                            MEMBER_ID,
                            false,
                            null
                    )
            ));

    mockMvc.perform(delete("/api/chores/templates/{id}/current-period-completion", CHORE_TEMPLATE_ID)
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                        {"scope":"TODAY","periodStartDate":"2026-05-17"}
                        """))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.item.completed").value(false));
}
```

```java
@Test
void uncompleteCurrentPeriod_staleRequest_throwsBadRequest() {
    family.setTimezone("America/Los_Angeles");
    FamilyMember leo = createFamilyMember(family);
    ChoreTemplate daily = createChoreTemplate(
            family,
            leo,
            "🪥 Brush teeth",
            ChoreCadence.DAILY,
            LocalDate.of(2026, 5, 17)
    );

    when(choreTemplateRepository.findByFamilyAndId(family, daily.getId()))
            .thenReturn(Optional.of(daily));

    assertThatThrownBy(() -> choreService.uncompleteCurrentPeriod(
            daily.getId(),
            new UpdateCurrentPeriodCompletionRequest(ChoreScope.TODAY, LocalDate.of(2026, 5, 16)),
            family
    ))
            .isInstanceOf(BadRequestException.class)
            .hasMessage("Chore period is stale. Refresh and try again.");
}

@Test
void deleteFamilyMember_withActiveRecurringChores_throwsBadRequest() {
    when(familyMemberRepository.findById(MEMBER_ID)).thenReturn(Optional.of(member));
    when(choreTemplateRepository.existsByAssignedToMemberAndArchivedAtIsNull(member)).thenReturn(true);

    assertThatThrownBy(() -> familyMemberService.deleteFamilyMember(family, MEMBER_ID))
            .isInstanceOf(BadRequestException.class)
            .hasMessage("Reassign or archive this member's recurring chores before deleting them.");
}
```

- [ ] **Step 2: Run the backend tests to verify they fail against the old controller/service contract**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=ChoreServiceTest,ChoreControllerTest,ChoreIntegrationTest,FamilyMemberServiceTest,FamilyMemberControllerTest test
```

Expected: FAIL with missing board DTOs, missing `/api/chores/board`, `PATCH /api/chores/templates/{id}`, `DELETE /api/chores/templates/{id}/current-period-completion` endpoints, and missing member-delete guard.

- [ ] **Step 3: Implement the DTOs, board assembly, stale-safe writes, and delete guard**

```java
public record UpdateCurrentPeriodCompletionRequest(
        @NotNull ChoreScope scope,
        @NotNull @JsonFormat(pattern = "yyyy-MM-dd") LocalDate periodStartDate
) {}
```

```java
public record CreateChoreTemplateRequest(
        @NotBlank @Size(max = 100) String title,
        @NotNull UUID assignedToMemberId,
        @NotNull ChoreCadence cadence,
        @NotNull @JsonFormat(pattern = "yyyy-MM-dd") LocalDate activeFrom
) {}
```

```java
public record UpdateChoreTemplateRequest(
        @Size(max = 100) String title,
        UUID assignedToMemberId,
        ChoreCadence cadence,
        @JsonFormat(pattern = "yyyy-MM-dd") LocalDate activeFrom,
        Boolean archived
) {}
```

```java
public record ChoreTemplateResponse(
        UUID id,
        String title,
        UUID assignedToMemberId,
        ChoreCadence cadence,
        LocalDate activeFrom,
        boolean archived,
        LocalDateTime createdAt,
        LocalDateTime updatedAt
) {}
```

```java
public record ChoreBoardItemResponse(
        UUID templateId,
        String title,
        ChoreCadence cadence,
        UUID assignedToMemberId,
        boolean completed,
        LocalDateTime completedAt
) {}
```

```java
public record ChoreAssigneeGroupResponse(
        FamilyMemberResponse member,
        Summary summary,
        List<ChoreBoardItemResponse> chores
) {
    public record Summary(int total, int completed, int remaining) {}
}
```

```java
public record ChoreScopeBoardResponse(
        ChoreScope scope,
        LocalDate periodStartDate,
        LocalDate periodEndDate,
        Summary summary,
        List<ChoreAssigneeGroupResponse> assignees
) {
    public record Summary(int total, int completed, int remaining) {}
}
```

```java
public record ChoreBoardResponse(
        String timezone,
        ChoreScopeBoardResponse today,
        ChoreScopeBoardResponse thisWeek,
        ChoreScopeBoardResponse thisMonth
) {}
```

```java
public record ChoreCurrentPeriodStateResponse(
        ChoreScope scope,
        LocalDate periodStartDate,
        LocalDate periodEndDate,
        ChoreBoardItemResponse item
) {}
```

```java
public ChoreBoardResponse getBoard(Family family) {
    ZoneId zoneId = ZoneId.of(family.getTimezone());
    LocalDate today = LocalDate.now(clock.withZone(zoneId));

    ChoreScopeBoardResponse todayBoard = buildScopeBoard(family, ChoreScope.TODAY, today, today);
    LocalDate weekStart = today.with(TemporalAdjusters.previousOrSame(DayOfWeek.SUNDAY));
    LocalDate weekEnd = weekStart.plusDays(6);
    ChoreScopeBoardResponse weekBoard = buildScopeBoard(family, ChoreScope.THIS_WEEK, weekStart, weekEnd);
    LocalDate monthStart = today.withDayOfMonth(1);
    LocalDate monthEnd = today.withDayOfMonth(today.lengthOfMonth());
    ChoreScopeBoardResponse monthBoard = buildScopeBoard(family, ChoreScope.THIS_MONTH, monthStart, monthEnd);

    return new ChoreBoardResponse(family.getTimezone(), todayBoard, weekBoard, monthBoard);
}

private record ResolvedPeriod(
        ChoreScope scope,
        LocalDate periodStartDate,
        LocalDate periodEndDate
) {}

private ResolvedPeriod resolveCurrentPeriod(ChoreTemplate template, Family family) {
    ZoneId zoneId = ZoneId.of(family.getTimezone());
    LocalDate today = LocalDate.now(clock.withZone(zoneId));

    return switch (template.getCadence()) {
        case DAILY -> new ResolvedPeriod(ChoreScope.TODAY, today, today);
        case WEEKLY -> {
            LocalDate weekStart = today.with(TemporalAdjusters.previousOrSame(DayOfWeek.SUNDAY));
            yield new ResolvedPeriod(ChoreScope.THIS_WEEK, weekStart, weekStart.plusDays(6));
        }
        case MONTHLY -> {
            LocalDate monthStart = today.withDayOfMonth(1);
            yield new ResolvedPeriod(ChoreScope.THIS_MONTH, monthStart, monthStart.withDayOfMonth(today.lengthOfMonth()));
        }
    };
}

private ChoreBoardItemResponse toBoardItem(
        ChoreTemplate template,
        ChorePeriodCompletion completion
) {
    return new ChoreBoardItemResponse(
            template.getId(),
            template.getTitle(),
            template.getCadence(),
            template.getAssignedToMember().getId(),
            completion != null,
            completion != null ? completion.getCompletedAt() : null
    );
}

private ChoreTemplateResponse toTemplateResponse(ChoreTemplate template) {
    return new ChoreTemplateResponse(
            template.getId(),
            template.getTitle(),
            template.getAssignedToMember().getId(),
            template.getCadence(),
            template.getActiveFrom(),
            template.getArchivedAt() != null,
            template.getCreatedAt(),
            template.getUpdatedAt()
    );
}

private FamilyMember requireAssignedMember(Family family, UUID memberId) {
    FamilyMember assignee = familyMemberRepository.findById(memberId)
            .orElseThrow(() -> new ResourceNotFoundException("Family Member", memberId));
    if (!assignee.getFamily().getId().equals(family.getId())) {
        throw new AccessDeniedException("Unauthorized");
    }
    return assignee;
}

public ChoreCurrentPeriodStateResponse completeCurrentPeriod(
        UUID templateId,
        UpdateCurrentPeriodCompletionRequest request,
        Family family
) {
    ChoreTemplate template = choreTemplateRepository.findByFamilyAndId(family, templateId)
            .orElseThrow(() -> new ResourceNotFoundException("Chore Template", templateId));

    ResolvedPeriod period = resolveCurrentPeriod(template, family);
    if (request.scope() != period.scope() || !request.periodStartDate().equals(period.periodStartDate())) {
        throw new BadRequestException("Chore period is stale. Refresh and try again.");
    }

    ChorePeriodCompletion completion = chorePeriodCompletionRepository
            .findByChoreTemplateAndPeriodStartDateAndPeriodEndDate(
                    template,
                    period.periodStartDate(),
                    period.periodEndDate()
            )
            .orElseGet(() -> {
                ChorePeriodCompletion created = new ChorePeriodCompletion();
                created.setChoreTemplate(template);
                created.setPeriodStartDate(period.periodStartDate());
                created.setPeriodEndDate(period.periodEndDate());
                created.setCompletedAt(LocalDateTime.now(clock));
                return created;
            });

    ChorePeriodCompletion saved = chorePeriodCompletionRepository.save(completion);
    return new ChoreCurrentPeriodStateResponse(
            period.scope(),
            period.periodStartDate(),
            period.periodEndDate(),
            toBoardItem(template, saved)
    );
}

public ChoreCurrentPeriodStateResponse uncompleteCurrentPeriod(
        UUID templateId,
        UpdateCurrentPeriodCompletionRequest request,
        Family family
) {
    ChoreTemplate template = choreTemplateRepository.findByFamilyAndId(family, templateId)
            .orElseThrow(() -> new ResourceNotFoundException("Chore Template", templateId));

    ResolvedPeriod period = resolveCurrentPeriod(template, family);
    if (request.scope() != period.scope() || !request.periodStartDate().equals(period.periodStartDate())) {
        throw new BadRequestException("Chore period is stale. Refresh and try again.");
    }

    chorePeriodCompletionRepository
            .findByChoreTemplateAndPeriodStartDateAndPeriodEndDate(
                    template,
                    period.periodStartDate(),
                    period.periodEndDate()
            )
            .ifPresent(chorePeriodCompletionRepository::delete);

    return new ChoreCurrentPeriodStateResponse(
            period.scope(),
            period.periodStartDate(),
            period.periodEndDate(),
            toBoardItem(template, null)
    );
}

public ChoreTemplateResponse updateTemplate(
        UUID templateId,
        UpdateChoreTemplateRequest request,
        Family family
) {
    ChoreTemplate template = choreTemplateRepository.findByFamilyAndId(family, templateId)
            .orElseThrow(() -> new ResourceNotFoundException("Chore Template", templateId));

    if (request.title() != null) {
        template.setTitle(request.title().trim());
    }
    if (request.cadence() != null) {
        template.setCadence(request.cadence());
    }
    if (request.activeFrom() != null) {
        template.setActiveFrom(request.activeFrom());
    }
    if (request.assignedToMemberId() != null) {
        FamilyMember assignee = requireAssignedMember(family, request.assignedToMemberId());
        template.setAssignedToMember(assignee);
    }
    if (Boolean.TRUE.equals(request.archived())) {
        template.setArchivedAt(LocalDateTime.now(clock));
    }

    ChoreTemplate saved = choreTemplateRepository.save(template);
    return toTemplateResponse(saved);
}
```

```java
@Transactional
public void deleteFamilyMember(Family family, UUID familyMemberId) {
    FamilyMember toBeDeleted = findById(familyMemberId);
    if (!isMemberOfFamily(family, toBeDeleted)) {
        throw new AccessDeniedException("Unauthorized");
    }
    if (choreTemplateRepository.existsByAssignedToMemberAndArchivedAtIsNull(toBeDeleted)) {
        throw new BadRequestException("Reassign or archive this member's recurring chores before deleting them.");
    }
    familyMemberRepository.delete(toBeDeleted);
    log.info("Family member deleted, memberId={}, familyId={}", familyMemberId, family.getId());
}
```

```java
@GetMapping("/board")
public ResponseEntity<ApiResponse<ChoreBoardResponse>> getBoard(@AuthenticationPrincipal Family family) {
    return ResponseEntity.ok(new ApiResponse<>(choreService.getBoard(family), ""));
}

@PostMapping("/templates")
public ResponseEntity<ApiResponse<ChoreTemplateResponse>> createTemplate(
        @Valid @RequestBody CreateChoreTemplateRequest request,
        @AuthenticationPrincipal Family family
) {
    ChoreTemplateResponse response = choreService.createTemplate(request, family);
    return ResponseEntity.created(URI.create("/api/chores/templates/" + response.id()))
            .body(new ApiResponse<>(response, "Chore template created successfully"));
}

@PutMapping("/templates/{id}/current-period-completion")
public ResponseEntity<ApiResponse<ChoreCurrentPeriodStateResponse>> completeCurrentPeriod(
        @PathVariable UUID id,
        @Valid @RequestBody UpdateCurrentPeriodCompletionRequest request,
        @AuthenticationPrincipal Family family
) {
    return ResponseEntity.ok(new ApiResponse<>(
            choreService.completeCurrentPeriod(id, request, family),
            "Chore completion updated successfully"
    ));
}

@PatchMapping("/templates/{id}")
public ResponseEntity<ApiResponse<ChoreTemplateResponse>> updateTemplate(
        @PathVariable UUID id,
        @Valid @RequestBody UpdateChoreTemplateRequest request,
        @AuthenticationPrincipal Family family
) {
    return ResponseEntity.ok(new ApiResponse<>(
            choreService.updateTemplate(id, request, family),
            "Chore template updated successfully"
    ));
}

@DeleteMapping("/templates/{id}/current-period-completion")
public ResponseEntity<ApiResponse<ChoreCurrentPeriodStateResponse>> uncompleteCurrentPeriod(
        @PathVariable UUID id,
        @Valid @RequestBody UpdateCurrentPeriodCompletionRequest request,
        @AuthenticationPrincipal Family family
) {
    return ResponseEntity.ok(new ApiResponse<>(
            choreService.uncompleteCurrentPeriod(id, request, family),
            "Chore completion updated successfully"
    ));
}
```

- [ ] **Step 4: Run the backend contract tests to verify the new API passes**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=ChoreServiceTest,ChoreControllerTest,ChoreIntegrationTest,FamilyMemberServiceTest,FamilyMemberControllerTest test
```

Expected: PASS for board responses, template archive/update, stale-safe complete/uncomplete behavior, and blocked member deletion when active recurring chores exist.

- [ ] **Step 5: Commit**

```bash
cd backend/family-hub-api
git add src/main/java/com/familyhub/demo/dto \
  src/main/java/com/familyhub/demo/service/ChoreService.java \
  src/main/java/com/familyhub/demo/controller/ChoreController.java \
  src/main/java/com/familyhub/demo/service/FamilyMemberService.java \
  src/test/java/com/familyhub/demo/service/ChoreServiceTest.java \
  src/test/java/com/familyhub/demo/controller/ChoreControllerTest.java \
  src/test/java/com/familyhub/demo/integration/ChoreIntegrationTest.java \
  src/test/java/com/familyhub/demo/service/FamilyMemberServiceTest.java \
  src/test/java/com/familyhub/demo/controller/FamilyMemberControllerTest.java
git commit -m "feat(chores): ship recurring chores board contract"
```

## Task 4: Reset The Frontend Contract Around Board Data And Registration Timezone

**Files:**
- Modify: `frontend/src/lib/types/auth.ts`
- Modify: `frontend/src/lib/types/chores.ts`
- Modify: `frontend/src/lib/types/index.ts`
- Modify: `frontend/src/components/onboarding/onboarding-flow.tsx`
- Modify: `frontend/src/components/onboarding/onboarding-flow.test.tsx`
- Modify: `frontend/src/api/client/http-client.ts`
- Modify: `frontend/src/api/services/chores.service.ts`
- Modify: `frontend/src/api/services/index.ts`
- Modify: `frontend/src/api/hooks/use-chores.ts`
- Modify: `frontend/src/api/hooks/index.ts`
- Modify: `frontend/src/api/index.ts`
- Modify: `frontend/src/lib/validations/chores.ts`
- Modify: `frontend/src/lib/validations/chores.test.ts`
- Modify: `frontend/src/lib/validations/index.ts`
- Modify: `frontend/src/test/mocks/handlers.ts`
- Modify: `frontend/src/test/mocks/server.ts`
- Modify: `frontend/src/api/hooks/use-chores.test.tsx`

- [ ] **Step 1: Write the failing frontend tests for the new board hook contract and registration timezone payload**

```tsx
it("sends browser timezone during onboarding registration", async () => {
  let capturedRegisterBody: RegisterRequest | null = null;
  server.use(
    http.post(`${API_BASE}/auth/register`, async ({ request }) => {
      capturedRegisterBody = (await request.json()) as RegisterRequest;
      return HttpResponse.json({
        data: {
          token: "jwt-token",
          family: {
            id: "family-1",
            name: "Smith Family",
            members: [{ id: "leo", name: "Leo", color: "coral" }],
            createdAt: "2026-05-17T09:00:00Z",
          },
        },
      });
    }),
  );
  const timezoneSpy = vi
    .spyOn(Intl.DateTimeFormat.prototype, "resolvedOptions")
    .mockReturnValue({ timeZone: "America/Chicago" } as Intl.ResolvedDateTimeFormatOptions);

  const { user } = renderWithUser(<OnboardingFlow />);
  await user.click(screen.getByRole("button", { name: /get started/i }));
  await user.type(screen.getByRole("textbox"), "Smith Family");
  await user.click(screen.getByRole("button", { name: /continue/i }));
  await user.click(screen.getByRole("button", { name: /add family member/i }));
  await user.type(screen.getByLabelText(/name/i), "Leo");
  await user.click(screen.getByRole("button", { name: /select coral color/i }));
  await user.click(screen.getByRole("button", { name: /^add$/i }));
  await user.click(screen.getByRole("button", { name: /continue/i }));
  await user.type(screen.getByLabelText(/username/i), "smithfamily");
  await user.type(screen.getByLabelText(/^password/i), "password123");
  await user.click(screen.getByRole("button", { name: /complete setup/i }));

  await waitFor(() => {
    expect(capturedRegisterBody).not.toBeNull();
    expect(capturedRegisterBody).toMatchObject({
      familyName: "Smith Family",
      timezone: "America/Chicago",
    });
  });

  timezoneSpy.mockRestore();
});
```

```tsx
it("fetches the chores board and sends stale-safe completion and uncompletion payloads", async () => {
  let capturedCompletionBody: UpdateCurrentPeriodCompletionRequest | null = null;
  let capturedUncompletionBody: UpdateCurrentPeriodCompletionRequest | null = null;
  const queryClient = createTestQueryClient();

  server.use(
    http.get(`${API_BASE}/chores/board`, () =>
      HttpResponse.json({
        data: {
          timezone: "America/Los_Angeles",
          today: {
            scope: "TODAY",
            periodStartDate: "2026-05-17",
            periodEndDate: "2026-05-17",
            summary: { total: 1, completed: 0, remaining: 1 },
            assignees: [
              {
                member: { id: "leo", name: "Leo", color: "coral" },
                summary: { total: 1, completed: 0, remaining: 1 },
                chores: [
                  {
                    templateId: "brush-teeth-id",
                    title: "🪥 Brush teeth",
                    cadence: "DAILY",
                    assignedToMemberId: "leo",
                    completed: false,
                    completedAt: null,
                  },
                ],
              },
            ],
          },
          thisWeek: {
            scope: "THIS_WEEK",
            periodStartDate: "2026-05-17",
            periodEndDate: "2026-05-23",
            summary: { total: 0, completed: 0, remaining: 0 },
            assignees: [],
          },
          thisMonth: {
            scope: "THIS_MONTH",
            periodStartDate: "2026-05-01",
            periodEndDate: "2026-05-31",
            summary: { total: 0, completed: 0, remaining: 0 },
            assignees: [],
          },
        },
      }),
    ),
    http.put(
      `${API_BASE}/chores/templates/brush-teeth-id/current-period-completion`,
      async ({ request }) => {
        capturedCompletionBody =
          (await request.json()) as UpdateCurrentPeriodCompletionRequest;
        return HttpResponse.json({
          data: {
            scope: "TODAY",
            periodStartDate: "2026-05-17",
            periodEndDate: "2026-05-17",
            item: {
              templateId: "brush-teeth-id",
              title: "🪥 Brush teeth",
              cadence: "DAILY",
              assignedToMemberId: "leo",
              completed: true,
              completedAt: "2026-05-17T09:15:00Z",
            },
          },
        });
      },
    ),
    http.delete(
      `${API_BASE}/chores/templates/brush-teeth-id/current-period-completion`,
      async ({ request }) => {
        capturedUncompletionBody =
          (await request.json()) as UpdateCurrentPeriodCompletionRequest;
        return HttpResponse.json({
          data: {
            scope: "TODAY",
            periodStartDate: "2026-05-17",
            periodEndDate: "2026-05-17",
            item: {
              templateId: "brush-teeth-id",
              title: "🪥 Brush teeth",
              cadence: "DAILY",
              assignedToMemberId: "leo",
              completed: false,
              completedAt: null,
            },
          },
        });
      },
    ),
  );

  const { result } = renderHook(() => ({
    board: useChoresBoard(),
    complete: useCompleteChoreForCurrentPeriod(),
    uncomplete: useUncompleteChoreForCurrentPeriod(),
  }), {
    wrapper: ({ children }) => (
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    ),
  });

  await waitFor(() => expect(result.current.board.data?.data.today.scope).toBe("TODAY"));

  act(() => {
    result.current.complete.mutate({
      templateId: "brush-teeth-id",
      request: {
        scope: "TODAY",
        periodStartDate: "2026-05-17",
      },
    });
  });

  await waitFor(() => {
    expect(capturedCompletionBody).toEqual({
      scope: "TODAY",
      periodStartDate: "2026-05-17",
    });
  });

  act(() => {
    result.current.uncomplete.mutate({
      templateId: "brush-teeth-id",
      request: {
        scope: "TODAY",
        periodStartDate: "2026-05-17",
      },
    });
  });

  await waitFor(() => {
    expect(capturedUncompletionBody).toEqual({
      scope: "TODAY",
      periodStartDate: "2026-05-17",
    });
  });
});
```

```tsx
it("sends archive updates for recurring templates", async () => {
  let capturedUpdateTemplateBody: UpdateChoreTemplateRequest | null = null;
  const queryClient = createTestQueryClient();

  server.use(
    http.patch(`${API_BASE}/chores/templates/brush-teeth-id`, async ({ request }) => {
      capturedUpdateTemplateBody =
        (await request.json()) as UpdateChoreTemplateRequest;
      return HttpResponse.json({
        data: {
          id: "brush-teeth-id",
          title: "🪥 Brush teeth",
          assignedToMemberId: "leo",
          cadence: "DAILY",
          activeFrom: "2026-05-17",
          archived: true,
          createdAt: "2026-05-17T08:00:00Z",
          updatedAt: "2026-05-17T09:00:00Z",
        },
      });
    }),
  );

  const { result } = renderHook(() => useUpdateChoreTemplate(), {
    wrapper: ({ children }) => (
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    ),
  });

  act(() => {
    result.current.mutate({
      id: "brush-teeth-id",
      request: { archived: true },
    });
  });

  await waitFor(() => {
    expect(capturedUpdateTemplateBody).toEqual({ archived: true });
  });
});
```

- [ ] **Step 2: Run the targeted frontend tests to verify the old hook/service contract fails**

Run:

```bash
cd frontend
npm run test -- --run src/api/hooks/use-chores.test.tsx src/components/onboarding/onboarding-flow.test.tsx
```

Expected: FAIL because `useChoresBoard`, `useUncompleteChoreForCurrentPeriod`, `useUpdateChoreTemplate`, the board payload types, and the timezone field do not exist yet.

- [ ] **Step 3: Replace the types, services, hooks, validation, and mocks**

```ts
export interface RegisterRequest {
  username: string;
  password: string;
  familyName: string;
  members: Array<Omit<FamilyMember, "id">>;
  timezone: string;
}
```

```ts
export type ChoreCadence = "DAILY" | "WEEKLY" | "MONTHLY";
export type ChoreScope = "TODAY" | "THIS_WEEK" | "THIS_MONTH";

export interface ChoreBoardItem {
  templateId: string;
  title: string;
  cadence: ChoreCadence;
  assignedToMemberId: string;
  completed: boolean;
  completedAt: string | null;
}

export interface ChoreAssigneeGroup {
  member: {
    id: string;
    name: string;
    color: FamilyColor;
  };
  summary: {
    total: number;
    completed: number;
    remaining: number;
  };
  chores: ChoreBoardItem[];
}

export interface ChoreScopeBoard {
  scope: ChoreScope;
  periodStartDate: string;
  periodEndDate: string;
  summary: {
    total: number;
    completed: number;
    remaining: number;
  };
  assignees: ChoreAssigneeGroup[];
}

export interface ChoresBoard {
  timezone: string;
  today: ChoreScopeBoard;
  thisWeek: ChoreScopeBoard;
  thisMonth: ChoreScopeBoard;
}

export interface CreateChoreTemplateRequest {
  title: string;
  assignedToMemberId: string;
  cadence: ChoreCadence;
  activeFrom: string;
}

export interface UpdateChoreTemplateRequest {
  title?: string;
  assignedToMemberId?: string;
  cadence?: ChoreCadence;
  activeFrom?: string;
  archived?: boolean;
}

export interface ChoreTemplate {
  id: string;
  title: string;
  assignedToMemberId: string;
  cadence: ChoreCadence;
  activeFrom: string;
  archived: boolean;
  createdAt: string;
  updatedAt: string;
}

export interface UpdateCurrentPeriodCompletionRequest {
  scope: ChoreScope;
  periodStartDate: string;
}

export interface ChoreCurrentPeriodState {
  scope: ChoreScope;
  periodStartDate: string;
  periodEndDate: string;
  item: ChoreBoardItem;
}
```

```ts
export const choreService = {
  getBoard(): Promise<ApiResponse<ChoresBoard>> {
    return httpClient.get<ApiResponse<ChoresBoard>>("/chores/board");
  },

  createTemplate(request: CreateChoreTemplateRequest) {
    return httpClient.post<ApiResponse<ChoreTemplate>>("/chores/templates", request);
  },

  updateTemplate(id: string, request: UpdateChoreTemplateRequest) {
    return httpClient.patch<ApiResponse<ChoreTemplate>>(`/chores/templates/${id}`, request);
  },

  completeCurrentPeriod(id: string, request: UpdateCurrentPeriodCompletionRequest) {
    return httpClient.put<ApiResponse<ChoreCurrentPeriodState>>(
      `/chores/templates/${id}/current-period-completion`,
      request,
    );
  },

  uncompleteCurrentPeriod(id: string, request: UpdateCurrentPeriodCompletionRequest) {
    return httpClient.delete<ApiResponse<ChoreCurrentPeriodState>>(
      `/chores/templates/${id}/current-period-completion`,
      request,
    );
  },
};
```

```ts
delete: <T>(endpoint: string, data?: unknown, config?: RequestConfig) =>
  request<T>(endpoint, {
    ...config,
    method: "DELETE",
    body: data,
  }),
```

```ts
let mockChoresBoard: ChoresBoard = {
  timezone: "America/Los_Angeles",
  today: {
    scope: "TODAY",
    periodStartDate: "2026-05-17",
    periodEndDate: "2026-05-17",
    summary: { total: 0, completed: 0, remaining: 0 },
    assignees: [],
  },
  thisWeek: {
    scope: "THIS_WEEK",
    periodStartDate: "2026-05-17",
    periodEndDate: "2026-05-23",
    summary: { total: 0, completed: 0, remaining: 0 },
    assignees: [],
  },
  thisMonth: {
    scope: "THIS_MONTH",
    periodStartDate: "2026-05-01",
    periodEndDate: "2026-05-31",
    summary: { total: 0, completed: 0, remaining: 0 },
    assignees: [],
  },
};

export function resetMockChoresBoard(): void {
  mockChoresBoard = {
    timezone: "America/Los_Angeles",
    today: {
      scope: "TODAY",
      periodStartDate: "2026-05-17",
      periodEndDate: "2026-05-17",
      summary: { total: 0, completed: 0, remaining: 0 },
      assignees: [],
    },
    thisWeek: {
      scope: "THIS_WEEK",
      periodStartDate: "2026-05-17",
      periodEndDate: "2026-05-23",
      summary: { total: 0, completed: 0, remaining: 0 },
      assignees: [],
    },
    thisMonth: {
      scope: "THIS_MONTH",
      periodStartDate: "2026-05-01",
      periodEndDate: "2026-05-31",
      summary: { total: 0, completed: 0, remaining: 0 },
      assignees: [],
    },
  };
}

export function seedMockChoresBoard(board: ChoresBoard): void {
  mockChoresBoard = structuredClone(board);
}

http.get(`${API_BASE}/chores/board`, () => {
  return HttpResponse.json(createApiResponse(mockChoresBoard));
});
```

```ts
export function setupMswServer() {
  beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
  afterEach(() => {
    server.resetHandlers();
    resetMockChores();
    resetMockChoresBoard();
    resetMockEvents();
    resetMockFamily();
    resetMockLists();
    resetMockUsers();
  });
  afterAll(() => server.close());
}

export {
  seedMockChoresBoard,
  resetMockChoresBoard,
};
```

```ts
export const choreKeys = {
  all: ["chores"] as const,
  board: () => [...choreKeys.all, "board"] as const,
};

export function useChoresBoard() {
  return useQuery({
    queryKey: choreKeys.board(),
    queryFn: () => choreService.getBoard(),
  });
}

export function useCompleteChoreForCurrentPeriod() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ templateId, request }: { templateId: string; request: UpdateCurrentPeriodCompletionRequest }) =>
      choreService.completeCurrentPeriod(templateId, request),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: choreKeys.board() });
    },
  });
}

export function useUncompleteChoreForCurrentPeriod() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ templateId, request }: { templateId: string; request: UpdateCurrentPeriodCompletionRequest }) =>
      choreService.uncompleteCurrentPeriod(templateId, request),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: choreKeys.board() });
    },
  });
}

export function useCreateChoreTemplate() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (request: CreateChoreTemplateRequest) => choreService.createTemplate(request),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: choreKeys.board() });
    },
  });
}

export function useUpdateChoreTemplate() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, request }: { id: string; request: UpdateChoreTemplateRequest }) =>
      choreService.updateTemplate(id, request),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: choreKeys.board() });
    },
  });
}
```

```ts
const handleCredentialsNext = (username: string, password: string) => {
  setRegistrationError(null);

  registerFamily.mutate(
    {
      username,
      password,
      familyName: draftName,
      members: draftMembers.map(({ name, color }) => ({ name, color })),
      timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
    },
    {
      onSuccess: () => setAuthenticated(true),
      onError: (error) => setRegistrationError(getRegistrationErrorMessage(error)),
    },
  );
};
```

- [ ] **Step 4: Run the hook and onboarding tests to verify the new contract passes**

Run:

```bash
cd frontend
npm run test -- --run src/api/hooks/use-chores.test.tsx src/components/onboarding/onboarding-flow.test.tsx src/lib/validations/chores.test.ts
```

Expected: PASS for board fetching, stale-safe complete/uncomplete payloads, archive updates, timezone registration, and the recurring chore form validation.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/lib/types/auth.ts \
  src/lib/types/chores.ts \
  src/lib/types/index.ts \
  src/components/onboarding/onboarding-flow.tsx \
  src/components/onboarding/onboarding-flow.test.tsx \
  src/api/client/http-client.ts \
  src/api/services/chores.service.ts \
  src/api/services/index.ts \
  src/api/hooks/use-chores.ts \
  src/api/hooks/index.ts \
  src/api/index.ts \
  src/lib/validations/chores.ts \
  src/lib/validations/chores.test.ts \
  src/lib/validations/index.ts \
  src/test/mocks/handlers.ts \
  src/test/mocks/server.ts \
  src/api/hooks/use-chores.test.tsx
git commit -m "feat(chores): reset frontend chores contract"
```

## Task 5: Build The Present-Focused Chores UI For Mobile And Larger Screens

**Files:**
- Create: `frontend/src/components/chores/chores-scope-switcher.tsx`
- Create: `frontend/src/components/chores/chores-scope-column.tsx`
- Create: `frontend/src/components/chores/chore-assignee-group.tsx`
- Modify: `frontend/src/components/chores/chore-row.tsx`
- Modify: `frontend/src/components/chores/chore-lane.tsx`
- Modify: `frontend/src/components/chores-view.tsx`
- Modify: `frontend/src/components/chores-view.test.tsx`

- [ ] **Step 1: Write the failing component tests for mobile single-scope rendering and larger-screen three-column rendering**

```tsx
let mockIsMobile = false;
vi.mock("@/hooks", async (importOriginal) => {
  const actual = await importOriginal<typeof import("@/hooks")>();
  return {
    ...actual,
    useIsMobile: () => mockIsMobile,
  };
});

function sampleChoresBoard(): ChoresBoard {
  return {
    timezone: "America/Los_Angeles",
    today: {
      scope: "TODAY",
      periodStartDate: "2026-05-17",
      periodEndDate: "2026-05-17",
      summary: { total: 1, completed: 0, remaining: 1 },
      assignees: [
        {
          member: { id: "leo", name: "Leo", color: "coral" },
          summary: { total: 1, completed: 0, remaining: 1 },
          chores: [
            {
              templateId: "brush-teeth-id",
              title: "🪥 Brush teeth",
              cadence: "DAILY",
              assignedToMemberId: "leo",
              completed: false,
              completedAt: null,
            },
          ],
        },
      ],
    },
    thisWeek: {
      scope: "THIS_WEEK",
      periodStartDate: "2026-05-17",
      periodEndDate: "2026-05-23",
      summary: { total: 1, completed: 0, remaining: 1 },
      assignees: [
        {
          member: { id: "leo", name: "Leo", color: "coral" },
          summary: { total: 1, completed: 0, remaining: 1 },
          chores: [
            {
              templateId: "trash-id",
              title: "🗑️ Take out trash",
              cadence: "WEEKLY",
              assignedToMemberId: "leo",
              completed: false,
              completedAt: null,
            },
          ],
        },
      ],
    },
    thisMonth: {
      scope: "THIS_MONTH",
      periodStartDate: "2026-05-01",
      periodEndDate: "2026-05-31",
      summary: { total: 1, completed: 1, remaining: 0 },
      assignees: [
        {
          member: { id: "maya", name: "Maya", color: "teal" },
          summary: { total: 1, completed: 1, remaining: 0 },
          chores: [
            {
              templateId: "fridge-id",
              title: "🧼 Deep clean fridge",
              cadence: "MONTHLY",
              assignedToMemberId: "maya",
              completed: true,
              completedAt: "2026-05-10T09:00:00Z",
            },
          ],
        },
      ],
    },
  };
}

function emptyChoresBoard(): ChoresBoard {
  return {
    timezone: "America/Los_Angeles",
    today: {
      scope: "TODAY",
      periodStartDate: "2026-05-17",
      periodEndDate: "2026-05-17",
      summary: { total: 0, completed: 0, remaining: 0 },
      assignees: [],
    },
    thisWeek: {
      scope: "THIS_WEEK",
      periodStartDate: "2026-05-17",
      periodEndDate: "2026-05-23",
      summary: { total: 0, completed: 0, remaining: 0 },
      assignees: [],
    },
    thisMonth: {
      scope: "THIS_MONTH",
      periodStartDate: "2026-05-01",
      periodEndDate: "2026-05-31",
      summary: { total: 0, completed: 0, remaining: 0 },
      assignees: [],
    },
  };
}

it("shows one selected scope at a time on mobile", async () => {
  mockIsMobile = true;
  seedMockChoresBoard(sampleChoresBoard());

  const { user } = renderWithUser(<ChoresView />);

  expect(await screen.findByRole("button", { name: "Day" })).toHaveAttribute("aria-pressed", "true");
  expect(screen.getByRole("heading", { name: "Today" })).toBeVisible();
  expect(screen.queryByRole("heading", { name: "This Week" })).not.toBeInTheDocument();

  await user.click(screen.getByRole("button", { name: "Week" }));

  expect(screen.getByRole("heading", { name: "This Week" })).toBeVisible();
  expect(screen.queryByRole("heading", { name: "Today" })).not.toBeInTheDocument();
});

it("shows today, this week, and this month columns on larger screens", async () => {
  mockIsMobile = false;
  seedMockChoresBoard(sampleChoresBoard());

  render(<ChoresView />);

  expect(await screen.findByRole("heading", { name: "Today" })).toBeVisible();
  expect(screen.getByRole("heading", { name: "This Week" })).toBeVisible();
  expect(screen.getByRole("heading", { name: "This Month" })).toBeVisible();
});

it("shows a family-level empty state when no recurring routines exist anywhere", async () => {
  mockIsMobile = false;
  seedMockChoresBoard(emptyChoresBoard());

  render(<ChoresView />);

  expect(await screen.findByText("No recurring chores yet")).toBeVisible();
});

it("shows a scope empty state when one timeframe has no routines", async () => {
  mockIsMobile = false;
  const board = sampleChoresBoard();
  board.thisWeek = {
    scope: "THIS_WEEK",
    periodStartDate: "2026-05-17",
    periodEndDate: "2026-05-23",
    summary: { total: 0, completed: 0, remaining: 0 },
    assignees: [],
  };
  seedMockChoresBoard(board);

  render(<ChoresView />);

  expect(await screen.findByText("No routines in this timeframe")).toBeVisible();
  expect(screen.getByText("🪥 Brush teeth")).toBeVisible();
});

it("shows all caught up while keeping completed routines visible in a fully complete scope", async () => {
  mockIsMobile = false;
  seedMockChoresBoard(sampleChoresBoard());

  render(<ChoresView />);

  expect(await screen.findByText("All caught up")).toBeVisible();
  expect(screen.getByText("🧼 Deep clean fridge")).toBeVisible();
});
```

- [ ] **Step 2: Run the component test to verify the old view fails the new scope contract**

Run:

```bash
cd frontend
npm run test -- --run src/components/chores-view.test.tsx
```

Expected: FAIL because the current view is still a one-off chores board with no scope switcher, no multi-scope empty states, and no three-column layout.

- [ ] **Step 3: Implement the scope switcher, scope columns, and present-focused board rendering**

```tsx
export function ChoreScopeSwitcher({
  value,
  onChange,
}: {
  value: "today" | "thisWeek" | "thisMonth";
  onChange: (value: "today" | "thisWeek" | "thisMonth") => void;
}) {
  const options = [
    { key: "today", label: "Day" },
    { key: "thisWeek", label: "Week" },
    { key: "thisMonth", label: "Month" },
  ] as const;

  return (
    <div className="inline-flex rounded-xl bg-muted p-1" role="tablist" aria-label="Chore scope">
      {options.map((option) => (
        <button
          key={option.key}
          type="button"
          role="tab"
          aria-pressed={value === option.key}
          onClick={() => onChange(option.key)}
          className={cn(
            "min-h-10 rounded-lg px-3 text-sm font-semibold",
            value === option.key ? "bg-background text-foreground shadow-sm" : "text-muted-foreground",
          )}
        >
          {option.label}
        </button>
      ))}
    </div>
  );
}
```

```tsx
export function ChoresView() {
  const isMobile = useIsMobile();
  const [selectedScope, setSelectedScope] = useState<"today" | "thisWeek" | "thisMonth">("today");
  const [isCreateOpen, setCreateOpen] = useState(false);
  const { data, isLoading, isError } = useChoresBoard();

  const board = data?.data;
  const hasAnyRoutines = board
    ? [board.today, board.thisWeek, board.thisMonth].some((scope) => scope.summary.total > 0)
    : false;
  const visibleScopes = isMobile
    ? selectedScope === "today"
      ? [board?.today]
      : selectedScope === "thisWeek"
        ? [board?.thisWeek]
        : [board?.thisMonth]
    : [board?.today, board?.thisWeek, board?.thisMonth];

  return (
    <>
      <div className="flex-1 overflow-y-auto p-4 sm:p-6">
        <div className="mx-auto max-w-6xl">
          <div className="mb-6 flex items-center justify-between gap-3">
            <div className="space-y-3">
              <h1 className="text-[24px] leading-8 font-semibold text-foreground">Chores</h1>
              {isMobile && (
                <ChoreScopeSwitcher value={selectedScope} onChange={setSelectedScope} />
              )}
            </div>
            <Button type="button" aria-label="Add recurring chore" size="icon" onClick={() => setCreateOpen(true)}>
              <Plus className="h-5 w-5" />
            </Button>
          </div>

          {isLoading && <div className="py-16 text-center text-sm text-muted-foreground">Loading chores...</div>}
          {isError && <div className="rounded-lg border border-destructive/30 bg-destructive/10 p-4 text-sm text-destructive">Could not load chores. Try again in a moment.</div>}
          {!isLoading && !isError && board && !hasAnyRoutines && (
            <div className="rounded-3xl border border-dashed border-border/70 bg-card/70 px-6 py-14 text-center">
              <h2 className="text-lg font-semibold text-foreground">No recurring chores yet</h2>
              <p className="mt-2 text-sm text-muted-foreground">Add a daily, weekly, or monthly routine to get started.</p>
            </div>
          )}

          {!isLoading && !isError && board && hasAnyRoutines && (
            <div className={cn("grid gap-4", !isMobile && "lg:grid-cols-3")}>
              {visibleScopes.filter(Boolean).map((scope) => (
                <ChoreScopeColumn key={scope!.scope} scope={scope!} />
              ))}
            </div>
          )}
        </div>
      </div>

      <ChoreFormSheet isOpen={isCreateOpen} onClose={() => setCreateOpen(false)} />
    </>
  );
}
```

```tsx
export function ChoreScopeColumn({ scope }: { scope: ChoreScopeBoard }) {
  const heading =
    scope.scope === "TODAY"
      ? "Today"
      : scope.scope === "THIS_WEEK"
        ? "This Week"
        : "This Month";
  const isFullyComplete = scope.summary.total > 0 && scope.summary.remaining === 0;

  return (
    <section aria-labelledby={`${scope.scope}-heading`} className="rounded-3xl border bg-card p-4 shadow-sm">
      <div className="mb-4">
        <h2 id={`${scope.scope}-heading`} className="text-lg font-semibold text-foreground">{heading}</h2>
      </div>

      {scope.summary.total === 0 ? (
        <p className="text-sm text-muted-foreground">No routines in this timeframe</p>
      ) : (
        <div className="space-y-4">
          {isFullyComplete && (
            <p className="text-sm font-medium text-muted-foreground">All caught up</p>
          )}
          {scope.assignees.map((assignee) => (
            <ChoreAssigneeGroup key={assignee.member.id} group={assignee} scope={scope} />
          ))}
        </div>
      )}
    </section>
  );
}
```

- [ ] **Step 4: Run the component tests to verify the new mobile and desktop presentations pass**

Run:

```bash
cd frontend
npm run test -- --run src/components/chores-view.test.tsx
```

Expected: PASS for single-scope mobile rendering, three-column larger-screen rendering, family-level empty state, scope-level empty state, and fully-complete scope rendering.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/chores/chores-scope-switcher.tsx \
  src/components/chores/chores-scope-column.tsx \
  src/components/chores/chore-assignee-group.tsx \
  src/components/chores/chore-row.tsx \
  src/components/chores/chore-lane.tsx \
  src/components/chores-view.tsx \
  src/components/chores-view.test.tsx
git commit -m "feat(chores): add present-focused chores board UI"
```

## Task 6: Wire Create, Archive, And Current-Period Toggle UX End-To-End

**Files:**
- Modify: `frontend/src/components/chores/chore-form.tsx`
- Modify: `frontend/src/components/chores/chore-form-sheet.tsx`
- Modify: `frontend/src/components/chores/chore-row.tsx`
- Modify: `frontend/src/components/chores-view.tsx`
- Modify: `frontend/src/components/chores/chore-form.test.tsx`
- Modify: `frontend/src/components/chores-view.test.tsx`
- Modify: `frontend/e2e/mobile-chores.spec.ts`

- [ ] **Step 1: Write the failing UI tests for creating a recurring routine, keeping completed routines visible, and archiving a routine**

```tsx
it("submits title, assignee, and cadence from the recurring chore form", async () => {
  const onSubmit = vi.fn();
  const { user } = renderWithUser(
    <ChoreForm
      defaultValues={{
        assignedToMemberId: "leo",
        cadence: "WEEKLY",
      }}
      onSubmit={onSubmit}
      onCancel={vi.fn()}
    />,
  );

  await waitForMemberSelected("Leo");
  await typeAndWait(user, screen.getByLabelText(/chore name/i), "🗑️ Take out trash");
  await user.click(screen.getByRole("button", { name: "Weekly" }));
  await user.click(screen.getByRole("button", { name: /save chore/i }));

  await waitFor(() => {
    expect(onSubmit).toHaveBeenCalledWith({
      title: "🗑️ Take out trash",
      assignedToMemberId: "leo",
      cadence: "WEEKLY",
    });
  });
});
```

```tsx
it("creates a weekly recurring routine with activeFrom defaulted to today", async () => {
  let capturedCreateTemplateBody: CreateChoreTemplateRequest | null = null;
  server.use(
    http.post(`${API_BASE}/chores/templates`, async ({ request }) => {
      capturedCreateTemplateBody =
        (await request.json()) as CreateChoreTemplateRequest;
      return HttpResponse.json(
        {
          data: {
            id: "template-1",
            title: "🗑️ Take out trash",
            assignedToMemberId: "leo",
            cadence: "WEEKLY",
            activeFrom: "2026-05-17",
            archived: false,
            createdAt: "2026-05-17T09:00:00Z",
            updatedAt: "2026-05-17T09:00:00Z",
          },
        },
        { status: 201 },
      );
    }),
  );
  seedMockChoresBoard(emptyChoresBoard());
  const { user } = renderWithUser(<ChoresView />);

  await user.click(screen.getByRole("button", { name: /add recurring chore/i }));
  await typeAndWait(user, screen.getByLabelText(/chore name/i), "🗑️ Take out trash");
  await user.click(screen.getByRole("button", { name: "Weekly" }));
  await user.click(screen.getByRole("button", { name: /save chore/i }));

  await waitFor(() => {
    expect(capturedCreateTemplateBody).toEqual({
      title: "🗑️ Take out trash",
      assignedToMemberId: "leo",
      cadence: "WEEKLY",
      activeFrom: "2026-05-17",
    });
  });
});

it("completes and uncompletes a routine for the current period while keeping completed rows visible", async () => {
  let capturedCompletionBody: UpdateCurrentPeriodCompletionRequest | null = null;
  let capturedUncompletionBody: UpdateCurrentPeriodCompletionRequest | null = null;

  server.use(
    http.put(`${API_BASE}/chores/templates/brush-teeth-id/current-period-completion`, async ({ request }) => {
      capturedCompletionBody =
        (await request.json()) as UpdateCurrentPeriodCompletionRequest;
      return HttpResponse.json({
        data: {
          scope: "TODAY",
          periodStartDate: "2026-05-17",
          periodEndDate: "2026-05-17",
          item: {
            templateId: "brush-teeth-id",
            title: "🪥 Brush teeth",
            cadence: "DAILY",
            assignedToMemberId: "leo",
            completed: true,
            completedAt: "2026-05-17T09:30:00Z",
          },
        },
      });
    }),
    http.delete(`${API_BASE}/chores/templates/brush-teeth-id/current-period-completion`, async ({ request }) => {
      capturedUncompletionBody =
        (await request.json()) as UpdateCurrentPeriodCompletionRequest;
      return HttpResponse.json({
        data: {
          scope: "TODAY",
          periodStartDate: "2026-05-17",
          periodEndDate: "2026-05-17",
          item: {
            templateId: "brush-teeth-id",
            title: "🪥 Brush teeth",
            cadence: "DAILY",
            assignedToMemberId: "leo",
            completed: false,
            completedAt: null,
          },
        },
      });
    }),
  );

  seedMockChoresBoard(sampleChoresBoard());
  const { user } = renderWithUser(<ChoresView />);

  await user.click(await screen.findByRole("button", { name: /mark 🪥 brush teeth complete/i }));

  await waitFor(() => {
    expect(capturedCompletionBody).toEqual({
      scope: "TODAY",
      periodStartDate: "2026-05-17",
    });
  });

  expect(screen.getByTestId("chore-row-brush-teeth-id")).toHaveClass("bg-muted/40");

  await user.click(screen.getByRole("button", { name: /mark 🪥 brush teeth incomplete/i }));

  await waitFor(() => {
    expect(capturedUncompletionBody).toEqual({
      scope: "TODAY",
      periodStartDate: "2026-05-17",
    });
  });
});

it("archives a recurring routine from the row action", async () => {
  let capturedUpdateTemplateBody: UpdateChoreTemplateRequest | null = null;
  server.use(
    http.patch(`${API_BASE}/chores/templates/brush-teeth-id`, async ({ request }) => {
      capturedUpdateTemplateBody =
        (await request.json()) as UpdateChoreTemplateRequest;
      return HttpResponse.json({
        data: {
          id: "brush-teeth-id",
          title: "🪥 Brush teeth",
          assignedToMemberId: "leo",
          cadence: "DAILY",
          activeFrom: "2026-05-17",
          archived: true,
          createdAt: "2026-05-17T08:00:00Z",
          updatedAt: "2026-05-17T09:05:00Z",
        },
      });
    }),
  );
  seedMockChoresBoard(sampleChoresBoard());
  const { user } = renderWithUser(<ChoresView />);

  await user.click(await screen.findByRole("button", { name: /archive 🪥 brush teeth/i }));

  await waitFor(() => {
    expect(capturedUpdateTemplateBody).toEqual({ archived: true });
  });
});
```

- [ ] **Step 2: Run the form and view tests to verify the old due-date form and delete action no longer fit**

Run:

```bash
cd frontend
npm run test -- --run src/components/chores/chore-form.test.tsx src/components/chores-view.test.tsx
```

Expected: FAIL because the form still asks for `dueDate`, the create flow still uses the device clock, and the row still uses destructive delete instead of recurring template create/archive/current-period toggle behavior.

- [ ] **Step 3: Replace the form fields and row actions with recurring-template semantics**

```tsx
type ChoreFormValues = z.infer<typeof choreFormSchema>;

interface ChoreFormProps {
  defaultValues?: Partial<ChoreFormValues>;
  onSubmit: (values: ChoreFormValues) => void;
  onCancel: () => void;
  isPending?: boolean;
}

export function ChoreForm({
  defaultValues,
  onSubmit,
  onCancel,
  isPending = false,
}: ChoreFormProps) {
  const familyMembers = useFamilyMembers();
  const form = useForm<ChoreFormValues>({
    resolver: zodResolver(choreFormSchema),
    defaultValues: {
      title: defaultValues?.title ?? "",
      assignedToMemberId: defaultValues?.assignedToMemberId ?? familyMembers[0]?.id ?? "",
      cadence: defaultValues?.cadence ?? "DAILY",
    },
  });

  return (
    <form id="chore-form" onSubmit={form.handleSubmit(onSubmit)} className="space-y-5">
      <div className="space-y-2">
        <Label htmlFor="chore-title">Chore Name</Label>
        <Input id="chore-title" {...form.register("title")} placeholder="What routine should repeat?" />
      </div>

      <div className="space-y-2">
        <Label>Assign To</Label>
        <MemberSelector
          members={familyMembers}
          value={form.watch("assignedToMemberId")}
          onChange={(memberId) => form.setValue("assignedToMemberId", memberId, { shouldDirty: true, shouldValidate: true })}
        />
      </div>

      <div className="space-y-2">
        <Label>Repeats</Label>
        <div className="grid grid-cols-3 gap-2">
          {(["DAILY", "WEEKLY", "MONTHLY"] as const).map((cadence) => (
            <Button
              key={cadence}
              type="button"
              variant={form.watch("cadence") === cadence ? "default" : "outline"}
              onClick={() => form.setValue("cadence", cadence, { shouldDirty: true, shouldValidate: true })}
            >
              {cadence === "DAILY" ? "Daily" : cadence === "WEEKLY" ? "Weekly" : "Monthly"}
            </Button>
          ))}
        </div>
      </div>
    </form>
  );
}
```

```tsx
<Button
  type="button"
  variant={chore.completed ? "secondary" : "default"}
  aria-label={chore.completed ? `Mark ${chore.title} incomplete` : `Mark ${chore.title} complete`}
  onClick={chore.completed ? onUncomplete : onComplete}
>
  {chore.completed ? "Completed" : "Mark done"}
</Button>

<Button
  type="button"
  variant="ghost"
  size="icon-sm"
  aria-label={`Archive ${chore.title}`}
  onClick={onArchive}
  className="text-muted-foreground hover:text-foreground"
>
  <Archive className="h-4 w-4" />
</Button>
```

```tsx
export function ChoreFormSheet({
  isOpen,
  onClose,
  defaultActiveFrom,
}: {
  isOpen: boolean;
  onClose: () => void;
  defaultActiveFrom: string | null;
}) {
  const createTemplate = useCreateChoreTemplate();

  if (!defaultActiveFrom) {
    return null;
  }

  const handleSubmit = (values: ChoreFormValues) => {
    createTemplate.mutate({
      title: values.title,
      assignedToMemberId: values.assignedToMemberId,
      cadence: values.cadence,
      activeFrom: defaultActiveFrom,
    });
  };
}
```

```tsx
<ChoreFormSheet
  isOpen={isCreateOpen}
  defaultActiveFrom={board?.today.periodStartDate ?? null}
  onClose={() => setCreateOpen(false)}
/>
```

- [ ] **Step 4: Run the mobile Playwright test plus the updated component tests**

Run:

```bash
cd frontend
npm run test -- --run src/components/chores/chore-form.test.tsx src/components/chores-view.test.tsx
npx playwright test e2e/mobile-chores.spec.ts --project="Mobile Chrome"
```

Expected: PASS for create recurring routine using the family-local current date from the board payload, complete/uncomplete in the current period, completed-visible-but-secondary rendering, and archive behavior.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/chores/chore-form.tsx \
  src/components/chores/chore-form-sheet.tsx \
  src/components/chores/chore-row.tsx \
  src/components/chores-view.tsx \
  src/components/chores/chore-form.test.tsx \
  src/components/chores-view.test.tsx \
  e2e/mobile-chores.spec.ts
git commit -m "feat(chores): wire recurring chores interactions"
```

## Task 7: Finalize Root Product Docs After FE And BE Delivery

**Files:**
- Modify: `docs/product/prd.md`
- Modify: `docs/product/backlog/module-foundations/chores-recurring-routines.md`

- [ ] **Step 1: Update the replacement chores story with execution metadata once FE and BE issues/PRs exist**

```md
issues:
  - BE #<backend-issue>
  - FE #<frontend-issue>
prs:
  - BE PR #<backend-pr>
  - FE PR #<frontend-pr>
```

- [ ] **Step 2: Update PRD language to match the shipped recurring-routines contract**

```md
**Requirements:**
- `Chores` is the recurring routines surface
- daily routines appear in `Today`
- weekly routines appear in `This Week`
- monthly routines appear in `This Month`
- completed routines remain visible in the current scope, but secondary
```

- [ ] **Step 3: Verify the live product docs no longer describe `dueDate`-driven chores or a `Show Completed` toggle as the active contract**

Run:

```bash
cd /Users/joe.bor/code/family-hub
rg -n "Show Completed|optional due date|due-date semantics" docs/product/prd.md docs/product/backlog/module-foundations/chores-core-loop.md
```

Expected: the PRD no longer describes due-date-driven chores as the live contract, and the superseded story is the only place where historical one-off chores semantics remain.

- [ ] **Step 4: Commit**

```bash
cd /Users/joe.bor/code/family-hub
git add docs/product/prd.md \
  docs/product/backlog/module-foundations/chores-recurring-routines.md
git commit -m "docs(chores): align product docs to recurring routines contract"
```

## Self-Review Checklist

- The plan covers root story preflight before FE/BE issue creation, timezone groundwork, recurring schema reset, board read contract, stale-safe complete/uncomplete writes, template archive/update behavior, member deletion guard, mobile/desktop FE rendering, empty states, create/archive/toggle interactions, and root doc alignment.
- The plan explicitly removes one-off due-date chores from the chores contract and keeps `Lists` and `Calendar` separate.
- The plan locks Sunday-first weekly behavior to mirror the current calendar module and uses the family-local board date rather than the device clock for `activeFrom`.
- No task depends on hidden context outside the referenced files and commands.
