# Focused Meal Planning Sessions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a focused planning session inside the existing Meals board so a parent can fill multiple empty slots for the visible editable week as temporary drafts, review them, and commit them to the real board with one save.

**Architecture:** Backend adds an atomic batch-save endpoint for empty meal-plan targets, preserving the existing single-slot editor contract while giving `Save to week` an all-or-nothing server operation. Frontend layers a temporary planning-session state machine over the existing `MealsView`, reuses existing recipe/quick-meal candidate behavior, renders draft blocks on the current board, and only writes to the API when the user chooses `Save to week`.

**Tech Stack:** Java 21, Spring Boot 4, JPA/Hibernate, PostgreSQL/Flyway, Maven; React 19, TypeScript, TanStack Query v5, Zod, existing shadcn/Radix/Vaul primitives, MSW, Vitest/Testing Library, Playwright.

**Spec:** `docs/superpowers/specs/2026-06-28-focused-meal-planning-sessions.md`

**Story:** `docs/product/backlog/module-foundations/focused-meal-planning-sessions.md`

---

## Delivery and repository boundaries

- Execute Tasks 1-2 in `backend/family-hub-api/`. Run Maven commands there and commit there.
- Merge the backend PR, merge the generated backend release PR, and confirm the published backend semver before starting released-contract frontend E2E.
- Execute Tasks 3-8 in `frontend/`. Mock-backed unit and component work can start before the backend release, but live E2E must consume the published backend release.
- Do not write production code from the root workspace. This root plan is the contract for delivery-repo Issues.
- Do not open execution Issues until this root plan has passed spec-to-plan review.
- Delivery Issues must link Story, Spec, and Plan at the top and copy the non-negotiable execution contract below.

## Implementation decision: batch save is required

The spec asks the plan to decide whether `Save to week` needs a dedicated batch-save contract or can be layered over existing single-slot writes. This plan requires a backend batch endpoint.

Reasoning:

- The v1 contract says drafts stay temporary until `Save to week`.
- Product preference is to avoid partial success where practical.
- Existing `PUT /api/meals/slots` is intentionally one slot at a time and can partially save if the frontend loops through several requests.
- A single backend transaction can validate all draft targets first, reject save-time conflicts, and roll back all writes if a concurrent insert or validation error appears.

The batch endpoint must not replace normal slot editing. Existing `PUT /api/meals/slots`, move, duplicate, remove, collision, and editor behavior remain unchanged.

## Execution contract

- `Fill empty slots` appears only for editable current and future Meals weeks; past weeks do not allow planning sessions.
- The action applies to the visible week only.
- V1 scope options are `Empty dinners` (default), `All empty slots`, and `Selected days`.
- An empty target is a slot with no persisted primary and no persisted extras. Extras-only data is treated as non-empty and excluded.
- Filled slots remain visible as context and are never targeted by the default flow.
- Draft choices are local UI state until `Save to week`.
- Draft blocks can be added, skipped, removed, changed, and canceled without writing to the real board.
- V1 focused planning drafts are primary-meal only. Extras remain available through the normal one-slot editor, not the planning session.
- Cancel leaves persisted Meals data unchanged.
- Quick meals and recipe-backed meals both work in the session.
- Save-time conflicts are detected before overwrite. V1 must not offer replace or add-as-extra during planning-save conflict resolution.
- On save conflict, the user can skip conflicted drafts and save the remaining non-conflicted drafts, return to editing, or cancel the save.
- `Save to week` writes drafted meals through one backend batch transaction and returns the refreshed Meals board.
- Existing one-slot editing, recipe-to-meals handoff, move, duplicate, replace, add-extra, remove, and past-week review behavior must keep passing.
- Backend JSON enum wire values stay lowercase (`breakfast`, `lunch`, `dinner`, `recipe`, `quick`).

## Execution issue mapping

### Backend issue body

```markdown
Story: https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/module-foundations/focused-meal-planning-sessions.md
Spec: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/specs/2026-06-28-focused-meal-planning-sessions.md
Plan: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-06-28-focused-meal-planning-sessions.md

Execution contract:
- Add an atomic Meals batch-save endpoint for focused planning sessions.
- The endpoint only writes empty target slots; occupied primary or extras-only slots are conflicts.
- A request with any conflict writes nothing and returns 409.
- Duplicate targets in one request are rejected with 400.
- Quick meals and recipe-backed meals use the existing meal entry validation and recipe snapshot behavior.
- Existing single-slot upsert, move, duplicate, remove, lowercase enum wire values, and board shape must not regress.
- Add service, controller, and integration coverage proving all-or-nothing behavior and family isolation.
```

### Frontend issue body

```markdown
Story: https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/module-foundations/focused-meal-planning-sessions.md
Spec: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/specs/2026-06-28-focused-meal-planning-sessions.md
Plan: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-06-28-focused-meal-planning-sessions.md

Execution contract:
- Add `Fill empty slots` to editable current/future Meals weeks only.
- Implement scopes: `Empty dinners` default, `All empty slots`, `Selected days`.
- Keep draft choices local until `Save to week`; cancel must leave persisted data unchanged.
- Reuse existing candidate behavior for favorite recipes, recent recipes, recipe search, and quick meals.
- Show current slot, queue progress, draft blocks on the board, review summary, and save/cancel controls.
- Save through the released backend batch endpoint and handle 409 conflicts with skip-conflicted, keep-editing, and cancel-save paths.
- Existing Meals board/editor/recipe handoff behavior must keep passing.
- Released-contract mobile E2E must name the tested backend semver.
```

## Verified codebase facts

- Backend Meals foundation exists in `backend/family-hub-api/src/main/java/com/familyhub/demo/service/MealService.java`.
- Existing backend writes are slot-scoped: `PUT /api/meals/slots`, `POST /api/meals/slots/move`, `POST /api/meals/slots/duplicate`, and `DELETE /api/meals/slots`.
- `MealService.upsertSlot` already rejects occupied primaries when `collisionMode` is absent.
- `ConflictException` and `GlobalExceptionHandler` already support `409`.
- Backend `meal_slot` has a unique key on `(family_id, week_start_date, day_index, meal_type)` from `V16__create_meal_planning_tables.sql`.
- Frontend `MealsView` owns visible week, read-only past-week behavior, board rendering, recipe-placement draft consumption, and composer/editor opening.
- Frontend meal candidates currently live in `MealComposerSheet`: favorite recipes, recent recipes, matching recipes, all recipes, and quick meal entry.
- MSW mock storage already mutates `MealBoard` data in memory for existing meal writes.
- No new runtime dependency is needed.

## File structure

### Backend - create

- `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/SaveMealPlanRequest.java` - top-level batch-save request.
- `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/SaveMealPlanSlotRequest.java` - one draft target inside the batch request.

### Backend - modify

- `backend/family-hub-api/src/main/java/com/familyhub/demo/service/MealService.java` - add transactional `savePlan`.
- `backend/family-hub-api/src/main/java/com/familyhub/demo/controller/MealController.java` - expose `POST /api/meals/plans`.
- `backend/family-hub-api/src/test/java/com/familyhub/demo/service/MealServiceTest.java` - batch-save unit coverage.
- `backend/family-hub-api/src/test/java/com/familyhub/demo/controller/MealControllerTest.java` - wire contract coverage.
- `backend/family-hub-api/src/test/java/com/familyhub/demo/integration/MealIntegrationTest.java` - persistence, rollback, and isolation coverage.

### Frontend - create

- `frontend/src/components/meals/meal-planning-session.ts` - pure session helpers: queue building, draft board projection, request conversion, conflict diffing.
- `frontend/src/components/meals/meal-planning-session.test.ts` - pure behavior tests.
- `frontend/src/components/meals/meal-planning-scope-dialog.tsx` - scope picker for `Empty dinners`, `All empty slots`, and `Selected days`.
- `frontend/src/components/meals/meal-planning-panel.tsx` - persistent planning tray/sheet, queue progress, candidate selection, draft edit controls, review/save/cancel flow.
- `frontend/src/components/meals/meal-planning-panel.test.tsx` - component-level session behavior tests.

### Frontend - modify

- `frontend/src/lib/types/meals.ts` - batch-save request types and planning scope types.
- `frontend/src/lib/validations/meals.ts` and `frontend/src/lib/validations/meals.test.ts` - batch-save schema.
- `frontend/src/api/services/meals.service.ts` and `frontend/src/api/services/meals.service.test.ts` - `savePlan`.
- `frontend/src/api/hooks/use-meals.ts` and `frontend/src/api/hooks/use-meals.test.tsx` - `useSaveMealPlan` cache behavior.
- `frontend/src/api/hooks/index.ts` and `frontend/src/api/index.ts` - export `useSaveMealPlan`.
- `frontend/src/test/mocks/handlers.ts` and `frontend/src/test/mocks/server.ts` - MSW `POST /api/meals/plans`.
- `frontend/src/test/fixtures/meals.ts` - partially filled board and extras-only fixture helpers.
- `frontend/src/components/meals/meal-slot-card.tsx` - draft rendering and current planning target highlight.
- `frontend/src/components/meals/meal-day-card.tsx` and `frontend/src/components/meals/meal-grid.tsx` - pass draft/highlight props through.
- `frontend/src/components/meals-view.tsx` and `frontend/src/components/meals-view.test.tsx` - action entry, session orchestration, save conflict flow, regression coverage.
- `frontend/e2e/mobile-meals.spec.ts` - released-contract mobile session coverage.

---

## Task 1: Add backend atomic meal-plan batch save

**Files:**
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/SaveMealPlanRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/SaveMealPlanSlotRequest.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/MealService.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/service/MealServiceTest.java`

- [ ] **Step 1: Add failing service tests for atomic batch save**

Append these tests to `MealServiceTest` before the helper methods:

```java
@Test
void savePlan_writesMultipleEmptyTargetsAndReturnsUpdatedBoard() {
    when(mealSlotRepository.findByFamilyAndWeekStartDateOrderByDayIndexAscMealTypeAsc(family, WEEK_START))
            .thenReturn(List.of())
            .thenReturn(List.of(
                    mealSlot(1, MealType.DINNER, "Tacos", List.of(), null),
                    mealSlot(3, MealType.DINNER, "Leftovers", List.of(), null)
            ));
    when(mealSlotRepository.saveAll(any())).thenAnswer(invocation -> invocation.getArgument(0));

    MealBoardResponse board = mealService.savePlan(new SaveMealPlanRequest(
            WEEK_START,
            List.of(
                    new SaveMealPlanSlotRequest(1, MealType.DINNER, quickMeal("Tacos"), List.of(), null),
                    new SaveMealPlanSlotRequest(3, MealType.DINNER, quickMeal("Leftovers"), List.of(), null)
            )
    ), family);

    assertThat(board.days().get(1).slots().get(2).primary().title()).isEqualTo("Tacos");
    assertThat(board.days().get(3).slots().get(2).primary().title()).isEqualTo("Leftovers");
}

@Test
void savePlan_snapshotsRecipeTargetsThroughTheAuthenticatedFamily() {
    Recipe recipe = createRecipe(family, "Sheet Pan Chicken");
    recipe.setImageUrl("https://cdn.example.com/chicken.jpg");
    recipe.setNote("Use the sheet pan with the rack");
    AtomicReference<List<MealSlot>> savedSlots = new AtomicReference<>(List.of());
    when(mealSlotRepository.findByFamilyAndWeekStartDateOrderByDayIndexAscMealTypeAsc(family, WEEK_START))
            .thenReturn(List.of())
            .thenAnswer(invocation -> savedSlots.get());
    when(recipeRepository.findByIdAndFamily(recipe.getId(), family)).thenReturn(Optional.of(recipe));
    when(mealSlotRepository.saveAll(any())).thenAnswer(invocation -> {
        List<MealSlot> slots = invocation.getArgument(0);
        slots.forEach(this::savedSlot);
        savedSlots.set(slots);
        return slots;
    });

    MealBoardResponse board = mealService.savePlan(new SaveMealPlanRequest(
            WEEK_START,
            List.of(new SaveMealPlanSlotRequest(2, MealType.DINNER, recipeMeal(recipe.getId()), List.of(), null))
    ), family);

    MealSlotEntryResponse plannedMeal = board.days().get(2).slots().get(2).primary();
    assertThat(plannedMeal.sourceType()).isEqualTo(MealEntrySourceType.RECIPE);
    assertThat(plannedMeal.recipeId()).isEqualTo(recipe.getId());
    assertThat(plannedMeal.title()).isEqualTo("Sheet Pan Chicken");
    assertThat(plannedMeal.imageUrl()).isEqualTo("https://cdn.example.com/chicken.jpg");
    assertThat(plannedMeal.note()).isEqualTo("Use the sheet pan with the rack");
    verify(recipeRepository).findByIdAndFamily(recipe.getId(), family);
}

@Test
void savePlan_rejectsOccupiedTargetAndWritesNothing() {
    MealSlot occupied = mealSlot(1, MealType.DINNER, "Pizza", List.of(), null);
    when(mealSlotRepository.findByFamilyAndWeekStartDateOrderByDayIndexAscMealTypeAsc(family, WEEK_START))
            .thenReturn(List.of(occupied));

    assertThatThrownBy(() -> mealService.savePlan(new SaveMealPlanRequest(
            WEEK_START,
            List.of(new SaveMealPlanSlotRequest(1, MealType.DINNER, quickMeal("Tacos"), List.of(), null))
    ), family)).isInstanceOf(ConflictException.class)
            .hasMessageContaining("Some meal slots are no longer empty.");

    verify(mealSlotRepository, never()).saveAll(any());
}

@Test
void savePlan_rejectsDuplicateTargetsInRequest() {
    assertThatThrownBy(() -> mealService.savePlan(new SaveMealPlanRequest(
            WEEK_START,
            List.of(
                    new SaveMealPlanSlotRequest(1, MealType.DINNER, quickMeal("Tacos"), List.of(), null),
                    new SaveMealPlanSlotRequest(1, MealType.DINNER, quickMeal("Pasta"), List.of(), null)
            )
    ), family)).isInstanceOf(BadRequestException.class)
            .hasMessageContaining("Meal plan contains duplicate target slots.");
}

@Test
void savePlan_treatsExtrasOnlySlotAsConflict() {
    MealSlot extrasOnly = new MealSlot();
    extrasOnly.setId(UUID.randomUUID());
    extrasOnly.setFamily(family);
    extrasOnly.setWeekStartDate(WEEK_START);
    extrasOnly.setDayIndex(4);
    extrasOnly.setMealType(MealType.DINNER);
    extrasOnly.getEntries().add(entry(extrasOnly, MealSlotRole.EXTRA, 0, "Garlic bread"));
    when(mealSlotRepository.findByFamilyAndWeekStartDateOrderByDayIndexAscMealTypeAsc(family, WEEK_START))
            .thenReturn(List.of(extrasOnly));

    assertThatThrownBy(() -> mealService.savePlan(new SaveMealPlanRequest(
            WEEK_START,
            List.of(new SaveMealPlanSlotRequest(4, MealType.DINNER, quickMeal("Soup"), List.of(), null))
    ), family)).isInstanceOf(ConflictException.class);
}
```

Add imports:

```java
import com.familyhub.demo.dto.SaveMealPlanRequest;
import com.familyhub.demo.dto.SaveMealPlanSlotRequest;
import com.familyhub.demo.exception.ConflictException;
```

- [ ] **Step 2: Run the focused backend service test and verify it fails**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=MealServiceTest test
```

Expected: FAIL because `SaveMealPlanRequest`, `SaveMealPlanSlotRequest`, and `MealService.savePlan` do not exist.

- [ ] **Step 3: Add the batch-save DTOs**

Create `SaveMealPlanSlotRequest.java`:

```java
package com.familyhub.demo.dto;

import com.familyhub.demo.model.MealType;
import jakarta.validation.Valid;
import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotNull;

import java.util.List;

public record SaveMealPlanSlotRequest(
        @NotNull @Min(0) @Max(6) Integer dayIndex,
        @NotNull MealType mealType,
        @Valid @NotNull MealEntryRequest primary,
        List<@Valid MealEntryRequest> extras,
        String note
) {
}
```

Create `SaveMealPlanRequest.java`:

```java
package com.familyhub.demo.dto;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

import java.time.LocalDate;
import java.util.List;

public record SaveMealPlanRequest(
        @NotNull LocalDate weekStartDate,
        @Valid @NotEmpty @Size(max = 21) List<SaveMealPlanSlotRequest> slots
) {
}
```

- [ ] **Step 4: Add service support**

Use the existing `findByFamilyAndWeekStartDateOrderByDayIndexAscMealTypeAsc` repository method for save-time conflict validation.

Add this public method to `MealService`:

```java
@Transactional
public MealBoardResponse savePlan(SaveMealPlanRequest request, Family family) {
    validateWeekStartDate(request.weekStartDate());
    validateUniqueTargets(request.slots());

    List<MealSlot> existingSlots = mealSlotRepository.findByFamilyAndWeekStartDateOrderByDayIndexAscMealTypeAsc(
            family,
            request.weekStartDate()
    );
    Map<SlotKey, MealSlot> existingByKey = existingSlots.stream()
            .collect(Collectors.toMap(
                    slot -> new SlotKey(slot.getDayIndex(), slot.getMealType()),
                    Function.identity(),
                    (existing, duplicate) -> existing
            ));

    List<MealSlot> toSave = new ArrayList<>();
    for (SaveMealPlanSlotRequest slotRequest : request.slots()) {
        SlotKey key = new SlotKey(slotRequest.dayIndex(), slotRequest.mealType());
        MealSlot existing = existingByKey.get(key);
        if (existing != null && !existing.getEntries().isEmpty()) {
            throw new ConflictException("Some meal slots are no longer empty.");
        }

        MealSlot slot = existing == null
                ? newSlot(family, request.weekStartDate(), slotRequest.dayIndex(), slotRequest.mealType())
                : existing;
        List<MealSlotEntry> requestedEntries = snapshotEntries(
                slot,
                slotRequest.primary(),
                normalizedExtras(slotRequest.extras())
        );
        replaceEntries(slot, requestedEntries);
        slot.setNote(RecipeFieldValidator.optionalText(slotRequest.note()));
        toSave.add(slot);
    }

    mealSlotRepository.saveAll(toSave);
    mealSlotRepository.flush();
    return getBoard(request.weekStartDate(), family);
}
```

Add this helper:

```java
private void validateUniqueTargets(List<SaveMealPlanSlotRequest> slots) {
    if (slots == null || slots.isEmpty()) {
        throw new BadRequestException("At least one meal plan slot is required.");
    }

    long uniqueTargets = slots.stream()
            .map(slot -> new SlotKey(slot.dayIndex(), slot.mealType()))
            .distinct()
            .count();
    if (uniqueTargets != slots.size()) {
        throw new BadRequestException("Meal plan contains duplicate target slots.");
    }
}
```

Add imports:

```java
import com.familyhub.demo.dto.SaveMealPlanRequest;
import com.familyhub.demo.dto.SaveMealPlanSlotRequest;
import com.familyhub.demo.exception.ConflictException;
```

- [ ] **Step 5: Run the focused backend service test and verify it passes**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=MealServiceTest test
```

Expected: PASS for existing meal service tests plus the new `savePlan_*` tests.

- [ ] **Step 6: Commit**

```bash
cd backend/family-hub-api
git add src/main/java/com/familyhub/demo/dto/SaveMealPlanRequest.java \
  src/main/java/com/familyhub/demo/dto/SaveMealPlanSlotRequest.java \
  src/main/java/com/familyhub/demo/service/MealService.java \
  src/test/java/com/familyhub/demo/service/MealServiceTest.java
git commit -m "feat(meals): add atomic meal plan save"
```

## Task 2: Expose and verify the backend batch-save API

**Files:**
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/controller/MealController.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/controller/MealControllerTest.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/integration/MealIntegrationTest.java`

- [ ] **Step 1: Add failing controller and integration tests**

Add a controller test:

```java
@Test
@WithMockFamily
void savePlan_returnsUpdatedBoard() throws Exception {
    given(mealService.savePlan(any(), any(Family.class))).willReturn(sampleBoard());

    mockMvc.perform(post("/api/meals/plans")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            {
                              "weekStartDate": "2026-06-07",
                              "slots": [
                                {
                                  "dayIndex": 1,
                                  "mealType": "dinner",
                                  "primary": {
                                    "sourceType": "quick",
                                    "recipeId": null,
                                    "title": "Tacos",
                                    "imageUrl": null,
                                    "note": null
                                  },
                                  "extras": [],
                                  "note": null
                                }
                              ]
                            }
                            """))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.message").value("Meal plan saved successfully"))
            .andExpect(jsonPath("$.data.weekStartDate").value("2026-06-07"));
}

@Test
@WithMockFamily
void savePlan_conflictReturns409() throws Exception {
    given(mealService.savePlan(any(), any(Family.class))).willThrow(new ConflictException(
            "Some meal slots are no longer empty."
    ));

    mockMvc.perform(post("/api/meals/plans")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            {
                              "weekStartDate": "2026-06-07",
                              "slots": [
                                {
                                  "dayIndex": 1,
                                  "mealType": "dinner",
                                  "primary": {
                                    "sourceType": "quick",
                                    "recipeId": null,
                                    "title": "Tacos",
                                    "imageUrl": null,
                                    "note": null
                                  },
                                  "extras": [],
                                  "note": null
                                }
                              ]
                            }
                            """))
            .andExpect(status().isConflict())
            .andExpect(jsonPath("$.message").value("Some meal slots are no longer empty."));
}
```

Add an integration sequence after the existing meal flow has registered a token:

```java
mockMvc.perform(post("/api/meals/plans")
                .header("Authorization", "Bearer " + token)
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        {
                          "weekStartDate": "2026-06-14",
                          "slots": [
                            {
                              "dayIndex": 1,
                              "mealType": "dinner",
                              "primary": {
                                "sourceType": "quick",
                                "recipeId": null,
                                "title": "Tacos",
                                "imageUrl": null,
                                "note": null
                              },
                              "extras": [],
                              "note": null
                            },
                            {
                              "dayIndex": 2,
                              "mealType": "dinner",
                              "primary": {
                                "sourceType": "quick",
                                "recipeId": null,
                                "title": "Leftovers",
                                "imageUrl": null,
                                "note": null
                              },
                              "extras": [],
                              "note": null
                            }
                          ]
                        }
                        """))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.data.days[1].slots[2].primary.title").value("Tacos"))
        .andExpect(jsonPath("$.data.days[2].slots[2].primary.title").value("Leftovers"));

String batchRecipeBody = mockMvc.perform(post("/api/recipes")
                .header("Authorization", "Bearer " + token)
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        {
                          "title": "Sheet Pan Chicken",
                          "imageUrl": "https://cdn.example.com/chicken.jpg",
                          "note": "Use thighs"
                        }
                        """))
        .andExpect(status().isCreated())
        .andReturn()
        .getResponse()
        .getContentAsString();
String batchRecipeId = JsonPath.read(batchRecipeBody, "$.data.id");

mockMvc.perform(post("/api/meals/plans")
                .header("Authorization", "Bearer " + token)
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        {
                          "weekStartDate": "2026-06-14",
                          "slots": [
                            {
                              "dayIndex": 4,
                              "mealType": "dinner",
                              "primary": {
                                "sourceType": "recipe",
                                "recipeId": "%s",
                                "title": null,
                                "imageUrl": null,
                                "note": null
                              },
                              "extras": [],
                              "note": null
                            }
                          ]
                        }
                        """.formatted(batchRecipeId)))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.data.days[4].slots[2].primary.sourceType").value("recipe"))
        .andExpect(jsonPath("$.data.days[4].slots[2].primary.title").value("Sheet Pan Chicken"))
        .andExpect(jsonPath("$.data.days[4].slots[2].primary.imageUrl").value("https://cdn.example.com/chicken.jpg"));

mockMvc.perform(post("/api/meals/plans")
                .header("Authorization", "Bearer " + token)
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        {
                          "weekStartDate": "2026-06-14",
                          "slots": [
                            {
                              "dayIndex": 1,
                              "mealType": "dinner",
                              "primary": {
                                "sourceType": "quick",
                                "recipeId": null,
                                "title": "Should Not Save",
                                "imageUrl": null,
                                "note": null
                              },
                              "extras": [],
                              "note": null
                            },
                            {
                              "dayIndex": 3,
                              "mealType": "dinner",
                              "primary": {
                                "sourceType": "quick",
                                "recipeId": null,
                                "title": "Also Should Not Save",
                                "imageUrl": null,
                                "note": null
                              },
                              "extras": [],
                              "note": null
                            }
                          ]
                        }
                        """))
        .andExpect(status().isConflict());

mockMvc.perform(get("/api/meals/board")
                .header("Authorization", "Bearer " + token)
                .param("weekStartDate", "2026-06-14"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.data.days[1].slots[2].primary.title").value("Tacos"))
        .andExpect(jsonPath("$.data.days[3].slots[2].primary").doesNotExist());
```

Add imports in `MealControllerTest`:

```java
import com.familyhub.demo.exception.ConflictException;
```

- [ ] **Step 2: Run focused tests and verify they fail**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=MealControllerTest,MealIntegrationTest test
```

Expected: FAIL because `POST /api/meals/plans` is not mapped.

- [ ] **Step 3: Add the controller endpoint**

Modify `MealController`:

```java
@PostMapping("/plans")
public ResponseEntity<ApiResponse<MealBoardResponse>> savePlan(
        @Valid @RequestBody SaveMealPlanRequest request,
        @AuthenticationPrincipal Family family
) {
    return ResponseEntity.ok(new ApiResponse<>(
            mealService.savePlan(request, family),
            "Meal plan saved successfully"
    ));
}
```

Add import:

```java
import com.familyhub.demo.dto.SaveMealPlanRequest;
```

- [ ] **Step 4: Run backend verification for Meals**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=MealServiceTest,MealControllerTest,MealIntegrationTest test
```

Expected: PASS for service, controller, and integration meal coverage.

- [ ] **Step 5: Commit**

```bash
cd backend/family-hub-api
git add src/main/java/com/familyhub/demo/controller/MealController.java \
  src/test/java/com/familyhub/demo/controller/MealControllerTest.java \
  src/test/java/com/familyhub/demo/integration/MealIntegrationTest.java
git commit -m "feat(meals): expose meal plan batch save"
```

## Task 3: Add frontend batch-save contracts, validation, hooks, and mocks

**Files:**
- Modify: `frontend/src/lib/types/meals.ts`
- Modify: `frontend/src/lib/validations/meals.ts`
- Modify: `frontend/src/lib/validations/meals.test.ts`
- Modify: `frontend/src/api/services/meals.service.ts`
- Modify: `frontend/src/api/services/meals.service.test.ts`
- Modify: `frontend/src/api/hooks/use-meals.ts`
- Modify: `frontend/src/api/hooks/use-meals.test.tsx`
- Modify: `frontend/src/api/hooks/index.ts`
- Modify: `frontend/src/api/index.ts`
- Modify: `frontend/src/test/mocks/handlers.ts`
- Modify: `frontend/src/test/mocks/server.ts`

- [ ] **Step 1: Add failing contract tests**

In `meals.service.test.ts`, add:

```ts
describe("mealsService.savePlan", () => {
  it("posts a batch save request to /meals/plans", async () => {
    const captured = { body: "", url: "" };
    server.use(
      http.post(`${API_BASE}/meals/plans`, async ({ request }) => {
        captured.url = request.url;
        captured.body = await request.text();
        return HttpResponse.json({
          data: { weekStartDate: "2026-06-07", days: [] },
          message: "Meal plan saved successfully",
        });
      }),
    );

    await mealsService.savePlan({
      weekStartDate: "2026-06-07",
      slots: [
        {
          dayIndex: 1,
          mealType: "dinner",
          primary: {
            sourceType: "quick",
            recipeId: null,
            title: "Tacos",
            imageUrl: null,
            note: null,
          },
          extras: [],
          note: null,
        },
      ],
    });

    expect(new URL(captured.url).pathname).toBe("/api/meals/plans");
    expect(JSON.parse(captured.body)).toMatchObject({
      weekStartDate: "2026-06-07",
      slots: [{ dayIndex: 1, mealType: "dinner" }],
    });
  });
});
```

In `use-meals.test.tsx`, add a mutation test that seeds `mealsKeys.board("2026-06-07")`, calls `useSaveMealPlan`, and expects the returned `MealBoardApiResponse` to replace that board cache.

- [ ] **Step 2: Run focused frontend contract tests and verify they fail**

Run:

```bash
cd frontend
npm test -- --run src/api/services/meals.service.test.ts src/api/hooks/use-meals.test.tsx src/lib/validations/meals.test.ts
```

Expected: FAIL because `savePlan` contracts do not exist.

- [ ] **Step 3: Add frontend types**

Append to `src/lib/types/meals.ts`:

```ts
export type MealPlanningScope =
  | { kind: "empty_dinners" }
  | { kind: "all_empty_slots" }
  | { kind: "selected_days"; dayIndexes: number[] };

export interface SaveMealPlanSlotRequest {
  dayIndex: number;
  mealType: MealType;
  primary: MealEntryRequest;
  extras: MealEntryRequest[];
  note: string | null;
}

export interface SaveMealPlanRequest {
  weekStartDate: string;
  slots: SaveMealPlanSlotRequest[];
}
```

- [ ] **Step 4: Add validation schema and request conversion reuse**

Append to `src/lib/validations/meals.ts`:

```ts
export const saveMealPlanSlotSchema = z.object({
  dayIndex: z.number().int().min(0).max(6),
  mealType: mealTypeSchema,
  primary: mealEntrySchema,
  extras: z.array(mealEntrySchema).optional().default([]),
  note: optionalTextSchema,
});

export const saveMealPlanSchema = z.object({
  weekStartDate: localDateSchema,
  slots: z.array(saveMealPlanSlotSchema).min(1).max(21),
});
```

Add tests covering:

- empty `slots` fails
- more than 21 `slots` fails
- quick meal title trims
- recipe slot requires a UUID recipe id

- [ ] **Step 5: Add service and hook**

Modify `meals.service.ts`:

```ts
import type { SaveMealPlanRequest } from "@/lib/types";

savePlan(request: SaveMealPlanRequest): Promise<MealBoardApiResponse> {
  return httpClient.post<MealBoardApiResponse>("/meals/plans", request);
},
```

Modify `use-meals.ts`:

```ts
import type { SaveMealPlanRequest } from "@/lib/types";

export function useSaveMealPlan(callbacks?: {
  onSuccess?: (data: MealBoardApiResponse) => void;
  onError?: (error: Error) => void;
}) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (request: SaveMealPlanRequest) =>
      mealsService.savePlan(request),
    onSuccess: (response, request) => {
      cacheBoard(queryClient, response);
      invalidateBoard(queryClient, request.weekStartDate);
      callbacks?.onSuccess?.(response);
    },
    onError: (error) => {
      callbacks?.onError?.(
        error instanceof Error ? error : new Error(String(error)),
      );
    },
  });
}
```

Export `useSaveMealPlan` from `src/api/hooks/index.ts` and `src/api/index.ts` alongside the other Meals hooks.

- [ ] **Step 6: Add MSW batch-save behavior**

In `handlers.ts`, add a `POST ${API_BASE}/meals/plans` handler near the existing Meals handlers.
Add `SaveMealPlanRequest` to the existing `@/lib/types` import list in the same file.

Expected mock behavior:

- reject duplicate targets with `400`
- reject occupied targets with `409`
- write all requested slots to the mock board if all targets are empty
- return the updated board

Core handler logic:

```ts
http.post(`${API_BASE}/meals/plans`, async ({ request }) => {
  const requestBody = (await request.json()) as SaveMealPlanRequest;
  const board = getMealsBoard(requestBody.weekStartDate);
  const seenTargets = new Set<string>();

  for (const slotRequest of requestBody.slots) {
    const key = `${slotRequest.dayIndex}:${slotRequest.mealType}`;
    if (seenTargets.has(key)) {
      return HttpResponse.json(
        {
          status: 400,
          path: "/api/meals/plans",
          message: "Meal plan contains duplicate target slots.",
        },
        { status: 400 },
      );
    }
    seenTargets.add(key);

    const slot = findMealSlot(board, slotRequest.dayIndex, slotRequest.mealType);
    if (slot.primary || slot.extras.length > 0) {
      return HttpResponse.json(
        {
          status: 409,
          path: "/api/meals/plans",
          message: "Some meal slots are no longer empty.",
        },
        { status: 409 },
      );
    }
  }

  for (const slotRequest of requestBody.slots) {
    const slot = findMealSlot(board, slotRequest.dayIndex, slotRequest.mealType);
    const entries = snapshotRequestedEntries(slotRequest.primary, slotRequest.extras);
    if (entries instanceof HttpResponse) return entries;

    const updatedSlot = resolveMealSlotPlacement(
      slot,
      entries.primary,
      entries.extras,
      slotRequest.note ?? null,
      null,
    );
    if (updatedSlot instanceof HttpResponse) return updatedSlot;
    setMealSlot(board, updatedSlot);
  }

  return HttpResponse.json(
    createApiResponse(board, "Meal plan saved successfully"),
  );
}),
```

- [ ] **Step 7: Run focused frontend contract tests and verify they pass**

Run:

```bash
cd frontend
npm test -- --run src/api/services/meals.service.test.ts src/api/hooks/use-meals.test.tsx src/lib/validations/meals.test.ts
```

Expected: PASS for batch-save service, hook, validation, and existing meal contract tests.

- [ ] **Step 8: Commit**

```bash
cd frontend
git add src/lib/types/meals.ts src/lib/validations/meals.ts \
  src/lib/validations/meals.test.ts src/api/services/meals.service.ts \
  src/api/services/meals.service.test.ts src/api/hooks/use-meals.ts \
  src/api/hooks/use-meals.test.tsx src/api/hooks/index.ts src/api/index.ts \
  src/test/mocks/handlers.ts src/test/mocks/server.ts
git commit -m "feat(meals): add focused planning save contract"
```

## Task 4: Add pure frontend planning-session helpers

**Files:**
- Create: `frontend/src/components/meals/meal-planning-session.ts`
- Create: `frontend/src/components/meals/meal-planning-session.test.ts`
- Modify: `frontend/src/test/fixtures/meals.ts`

- [ ] **Step 1: Write failing pure helper tests**

Create `meal-planning-session.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createEmptyMealsBoard, createExtrasOnlyMealsBoard, createOccupiedMealsBoard } from "@/test/fixtures/meals";
import {
  applyPlanningDraftsToBoard,
  buildPlanningQueue,
  getConflictedDraftTargets,
  toSaveMealPlanRequest,
  type MealPlanningDraft,
} from "./meal-planning-session";

describe("buildPlanningQueue", () => {
  it("defaults to empty dinners in day order", () => {
    const board = createOccupiedMealsBoard();

    const queue = buildPlanningQueue(board, { kind: "empty_dinners" });

    expect(queue.map((slot) => `${slot.dayIndex}:${slot.mealType}`)).toEqual([
      "0:dinner",
      "3:dinner",
      "4:dinner",
      "5:dinner",
      "6:dinner",
    ]);
  });

  it("can include every empty breakfast, lunch, and dinner slot", () => {
    const queue = buildPlanningQueue(createEmptyMealsBoard(), {
      kind: "all_empty_slots",
    });

    expect(queue).toHaveLength(21);
    expect(queue[0]).toMatchObject({ dayIndex: 0, mealType: "breakfast" });
    expect(queue[20]).toMatchObject({ dayIndex: 6, mealType: "dinner" });
  });

  it("can include all empty slots for selected days only", () => {
    const queue = buildPlanningQueue(createEmptyMealsBoard(), {
      kind: "selected_days",
      dayIndexes: [2, 4],
    });

    expect(queue.map((slot) => `${slot.dayIndex}:${slot.mealType}`)).toEqual([
      "2:breakfast",
      "2:lunch",
      "2:dinner",
      "4:breakfast",
      "4:lunch",
      "4:dinner",
    ]);
  });

  it("treats extras-only slots as non-empty", () => {
    const queue = buildPlanningQueue(createExtrasOnlyMealsBoard(), {
      kind: "empty_dinners",
    });

    expect(queue.map((slot) => `${slot.dayIndex}:${slot.mealType}`)).not.toContain("4:dinner");
  });
});

describe("planning drafts", () => {
  it("projects drafts onto a board without mutating the source board", () => {
    const board = createEmptyMealsBoard();
    const drafts: MealPlanningDraft[] = [
      {
        target: { dayIndex: 1, mealType: "dinner" },
        displayTitle: "Tacos",
        displayImageUrl: null,
        displayNote: null,
        primary: {
          sourceType: "quick",
          recipeId: null,
          title: "Tacos",
          imageUrl: null,
          note: null,
        },
        note: null,
      },
    ];

    const projected = applyPlanningDraftsToBoard(board, drafts);

    expect(projected.days[1].slots[2].primary?.title).toBe("Tacos");
    expect(board.days[1].slots[2].primary).toBeNull();
  });

  it("converts drafts to the backend batch-save request", () => {
    const request = toSaveMealPlanRequest("2026-06-07", [
      {
        target: { dayIndex: 1, mealType: "dinner" },
        displayTitle: "Tacos",
        displayImageUrl: null,
        displayNote: null,
        primary: {
          sourceType: "quick",
          recipeId: null,
          title: "Tacos",
          imageUrl: null,
          note: null,
        },
        note: null,
      },
    ]);

    expect(request).toEqual({
      weekStartDate: "2026-06-07",
      slots: [
        {
          dayIndex: 1,
          mealType: "dinner",
          primary: {
            sourceType: "quick",
            recipeId: null,
            title: "Tacos",
            imageUrl: null,
            note: null,
          },
          extras: [],
          note: null,
        },
      ],
    });
  });

  it("finds conflicted draft targets after a board refetch", () => {
    const conflicts = getConflictedDraftTargets(createOccupiedMealsBoard(), [
      {
        target: { dayIndex: 1, mealType: "dinner" },
        displayTitle: "Tacos",
        displayImageUrl: null,
        displayNote: null,
        primary: {
          sourceType: "quick",
          recipeId: null,
          title: "Tacos",
          imageUrl: null,
          note: null,
        },
        note: null,
      },
    ]);

    expect(conflicts).toEqual([{ dayIndex: 1, mealType: "dinner" }]);
  });
});
```

- [ ] **Step 2: Run the focused pure helper test and verify it fails**

Run:

```bash
cd frontend
npm test -- --run src/components/meals/meal-planning-session.test.ts
```

Expected: FAIL because `meal-planning-session.ts` and `createExtrasOnlyMealsBoard` do not exist.

- [ ] **Step 3: Add meal fixture for extras-only data**

Append to `src/test/fixtures/meals.ts`:

```ts
export function createExtrasOnlyMealsBoard(): MealBoard {
  const board = createEmptyMealsBoard();
  board.days[4].slots[2] = {
    id: "slot-extras-only-dinner",
    weekStartDate: testWeekStartDate,
    dayIndex: 4,
    mealType: "dinner",
    primary: null,
    extras: [
      {
        id: "entry-extras-only",
        role: "extra",
        sourceType: "quick",
        recipeId: null,
        title: "Garlic bread",
        imageUrl: null,
        note: null,
      },
    ],
    note: null,
  };
  return board;
}
```

- [ ] **Step 4: Implement pure planning helpers**

Create `meal-planning-session.ts`:

```ts
import type {
  MealBoard,
  MealEntryRequest,
  MealPlanningScope,
  MealSlot,
  MealType,
  SaveMealPlanRequest,
} from "@/lib/types";

export interface MealPlanningTarget {
  dayIndex: number;
  mealType: MealType;
}

export interface MealPlanningDraft {
  target: MealPlanningTarget;
  displayTitle: string;
  displayImageUrl: string | null;
  displayNote: string | null;
  primary: MealEntryRequest;
  note: string | null;
}

const MEAL_ORDER: MealType[] = ["breakfast", "lunch", "dinner"];

function isSlotEmpty(slot: MealSlot) {
  return slot.primary === null && slot.extras.length === 0;
}

function targetKey(target: MealPlanningTarget) {
  return `${target.dayIndex}:${target.mealType}`;
}

export function buildPlanningQueue(
  board: MealBoard,
  scope: MealPlanningScope,
): MealSlot[] {
  const selectedDays =
    scope.kind === "selected_days" ? new Set(scope.dayIndexes) : null;
  const allowedMealTypes =
    scope.kind === "empty_dinners" ? new Set<MealType>(["dinner"]) : null;

  return board.days.flatMap((day) => {
    if (selectedDays && !selectedDays.has(day.dayIndex)) return [];
    return MEAL_ORDER.flatMap((mealType) => {
      if (allowedMealTypes && !allowedMealTypes.has(mealType)) return [];
      const slot = day.slots.find((candidate) => candidate.mealType === mealType);
      return slot && isSlotEmpty(slot) ? [slot] : [];
    });
  });
}

export function applyPlanningDraftsToBoard(
  board: MealBoard,
  drafts: MealPlanningDraft[],
): MealBoard {
  const draftsByTarget = new Map(drafts.map((draft) => [targetKey(draft.target), draft]));
  return {
    ...board,
    days: board.days.map((day) => ({
      ...day,
      slots: day.slots.map((slot) => {
        const draft = draftsByTarget.get(targetKey(slot));
        if (!draft) return slot;
        return {
          ...slot,
          primary: {
            id: `draft-${targetKey(slot)}`,
            role: "primary",
            ...draft.primary,
            title: draft.displayTitle,
            imageUrl: draft.displayImageUrl,
            note: draft.displayNote,
          },
          extras: [],
          note: draft.note,
        };
      }),
    })),
  };
}

export function toSaveMealPlanRequest(
  weekStartDate: string,
  drafts: MealPlanningDraft[],
): SaveMealPlanRequest {
  return {
    weekStartDate,
    slots: drafts.map((draft) => ({
      dayIndex: draft.target.dayIndex,
      mealType: draft.target.mealType,
      primary: draft.primary,
      extras: [],
      note: draft.note,
    })),
  };
}

export function getConflictedDraftTargets(
  board: MealBoard,
  drafts: MealPlanningDraft[],
): MealPlanningTarget[] {
  return drafts
    .filter((draft) => {
      const slot = board.days[draft.target.dayIndex]?.slots.find(
        (candidate) => candidate.mealType === draft.target.mealType,
      );
      return !slot || !isSlotEmpty(slot);
    })
    .map((draft) => draft.target);
}
```

Recipe-backed drafts must set `displayTitle`, `displayImageUrl`, and `displayNote` from the selected `RecipeSummary` so draft cards never render generic labels.

- [ ] **Step 5: Run focused helper tests and verify they pass**

Run:

```bash
cd frontend
npm test -- --run src/components/meals/meal-planning-session.test.ts
```

Expected: PASS for queue building, draft projection, request conversion, and conflict diffing.

- [ ] **Step 6: Commit**

```bash
cd frontend
git add src/components/meals/meal-planning-session.ts \
  src/components/meals/meal-planning-session.test.ts \
  src/test/fixtures/meals.ts
git commit -m "feat(meals): add planning session helpers"
```

## Task 5: Add focused planning session UI without saving drafts early

**Files:**
- Create: `frontend/src/components/meals/meal-planning-scope-dialog.tsx`
- Create: `frontend/src/components/meals/meal-planning-panel.tsx`
- Create: `frontend/src/components/meals/meal-planning-panel.test.tsx`
- Modify: `frontend/src/components/meals/meal-slot-card.tsx`
- Modify: `frontend/src/components/meals/meal-day-card.tsx`
- Modify: `frontend/src/components/meals/meal-grid.tsx`
- Modify: `frontend/src/components/meals-view.tsx`
- Modify: `frontend/src/components/meals-view.test.tsx`

- [ ] **Step 1: Add failing component tests for entry, scopes, drafts, and cancel**

In `MealsView.test.tsx`, add tests:

```ts
it("shows Fill empty slots on editable weeks and hides it on past weeks", async () => {
  seedMockMealsBoard(createEmptyMealsBoard());
  seedMockMealsBoard(createEmptyMealsBoard("2026-05-31"));
  const { user } = renderWithUser(<MealsView />);

  expect(
    await screen.findByRole("button", { name: "Fill empty slots" }),
  ).toBeInTheDocument();

  await user.click(screen.getByRole("button", { name: "Previous week" }));

  expect(await screen.findByText("Review only")).toBeInTheDocument();
  expect(
    screen.queryByRole("button", { name: "Fill empty slots" }),
  ).not.toBeInTheDocument();
});

it("starts a default empty-dinners session and drafts quick meals without saving", async () => {
  seedMockMealsBoard(createEmptyMealsBoard());
  const { user } = renderWithUser(<MealsView />);

  await user.click(await screen.findByRole("button", { name: "Fill empty slots" }));
  await user.click(screen.getByRole("button", { name: "Start planning" }));

  expect(screen.getByText("Sunday dinner - 1 of 7")).toBeInTheDocument();
  await user.type(screen.getByLabelText("Meal name"), "Tacos");
  await user.click(screen.getByRole("button", { name: "Add quick meal draft" }));

  expect(screen.getByText("Monday dinner - 2 of 7")).toBeInTheDocument();
  expect(screen.getByRole("button", { name: /draft dinner: tacos/i })).toBeInTheDocument();
  expect(getMockMealsBoard(testWeekStartDate).days[0].slots[2].primary).toBeNull();
});

it("cancels a planning session without changing the persisted board", async () => {
  seedMockMealsBoard(createEmptyMealsBoard());
  const { user } = renderWithUser(<MealsView />);

  await user.click(await screen.findByRole("button", { name: "Fill empty slots" }));
  await user.click(screen.getByRole("button", { name: "Start planning" }));
  await user.type(screen.getByLabelText("Meal name"), "Tacos");
  await user.click(screen.getByRole("button", { name: "Add quick meal draft" }));
  await user.click(screen.getByRole("button", { name: "Cancel planning" }));

  expect(screen.queryByRole("dialog", { name: /meal planning/i })).not.toBeInTheDocument();
  expect(getMockMealsBoard(testWeekStartDate).days[0].slots[2].primary).toBeNull();
});
```

Add a selected-days scope test that chooses Monday and Wednesday, starts planning, and expects only those days' empty slots in the queue.

Add a future-week test:

```ts
it("plans a blank future week without leaving Meals", async () => {
  seedMockMealsBoard(createEmptyMealsBoard());
  seedMockMealsBoard(createEmptyMealsBoard("2026-06-14"));
  const { user } = renderWithUser(<MealsView />);

  await user.click(await screen.findByRole("button", { name: "Next week" }));
  expect(await screen.findByText("Jun 14 - Jun 20")).toBeInTheDocument();

  await user.click(screen.getByRole("button", { name: "Fill empty slots" }));
  await user.click(screen.getByRole("button", { name: "Start planning" }));

  expect(screen.getByText("Sunday dinner - 1 of 7")).toBeInTheDocument();
  await user.type(screen.getByLabelText("Meal name"), "Tacos");
  await user.click(screen.getByRole("button", { name: "Add quick meal draft" }));
  await user.click(screen.getByRole("button", { name: "Review plan" }));
  await user.click(screen.getByRole("button", { name: "Save to week" }));

  await waitFor(() => {
    expect(getMockMealsBoard("2026-06-14").days[0].slots[2].primary?.title).toBe("Tacos");
  });
  expect(screen.getByText("Jun 14 - Jun 20")).toBeInTheDocument();
});
```

Add a recipe-backed draft test:

```ts
it("adds a recipe-backed draft from planning candidates without saving early", async () => {
  seedMockMealsBoard(createEmptyMealsBoard());
  seedMockRecipes([testRecipeDetail]);
  const { user } = renderWithUser(<MealsView />);

  await user.click(await screen.findByRole("button", { name: "Fill empty slots" }));
  await user.click(screen.getByRole("button", { name: "Start planning" }));
  await user.click(screen.getByRole("button", { name: `Select recipe: ${testRecipeDetail.title}` }));

  expect(
    screen.getByRole("button", { name: new RegExp(`draft dinner: ${testRecipeDetail.title}`, "i") }),
  ).toBeInTheDocument();
  expect(getMockMealsBoard(testWeekStartDate).days[0].slots[2].primary).toBeNull();
});
```

Add a draft-controls test that proves `Skip this slot`, `Remove draft`, and `Change draft` update the queue without saving:

```ts
it("supports skip, remove draft, and change draft during a planning session", async () => {
  seedMockMealsBoard(createEmptyMealsBoard());
  const { user } = renderWithUser(<MealsView />);

  await user.click(await screen.findByRole("button", { name: "Fill empty slots" }));
  await user.click(screen.getByRole("button", { name: "Start planning" }));
  await user.click(screen.getByRole("button", { name: "Skip this slot" }));
  expect(screen.getByText("Monday dinner - 2 of 7")).toBeInTheDocument();

  await user.type(screen.getByLabelText("Meal name"), "Tacos");
  await user.click(screen.getByRole("button", { name: "Add quick meal draft" }));
  await user.click(screen.getByRole("button", { name: /change draft: monday dinner/i }));
  await user.clear(screen.getByLabelText("Meal name"));
  await user.type(screen.getByLabelText("Meal name"), "Pasta");
  await user.click(screen.getByRole("button", { name: "Update draft" }));
  expect(screen.getByRole("button", { name: /draft dinner: pasta/i })).toBeInTheDocument();

  await user.click(screen.getByRole("button", { name: /remove draft: monday dinner/i }));
  expect(screen.queryByRole("button", { name: /draft dinner: pasta/i })).not.toBeInTheDocument();
  expect(getMockMealsBoard(testWeekStartDate).days[1].slots[2].primary).toBeNull();
});
```

- [ ] **Step 2: Run focused component tests and verify they fail**

Run:

```bash
cd frontend
npm test -- --run src/components/meals-view.test.tsx
```

Expected: FAIL because the planning entry point and UI do not exist.

- [ ] **Step 3: Implement the scope picker**

Create `meal-planning-scope-dialog.tsx` with these props:

```ts
interface MealPlanningScopeDialogProps {
  isOpen: boolean;
  weekLabel: string;
  days: Array<{ dayIndex: number; label: string }>;
  onStart: (scope: MealPlanningScope) => void;
  onOpenChange: (open: boolean) => void;
}
```

Required UI behavior:

- default selection is `Empty dinners`
- options are radio controls for `Empty dinners`, `All empty slots`, and `Selected days`
- selected-days mode shows day checkboxes with at least one day required
- primary action label is `Start planning`
- cancel closes the dialog without session state

- [ ] **Step 4: Implement the planning panel**

Create `meal-planning-panel.tsx` with these props:

```ts
interface MealPlanningPanelProps {
  isOpen: boolean;
  board: MealBoard;
  queue: MealSlot[];
  drafts: MealPlanningDraft[];
  currentIndex: number;
  recipes: RecipeSummary[];
  isSaving: boolean;
  saveError: Error | null;
  conflictedTargets: MealPlanningTarget[];
  onAddDraft: (draft: MealPlanningDraft) => void;
  onRemoveDraft: (target: MealPlanningTarget) => void;
  onSkip: () => void;
  onBack: () => void;
  onReview: () => void;
  onSave: () => void;
  onSaveNonConflicted: () => void;
  onKeepEditing: () => void;
  onCancelSave: () => void;
  onCancel: () => void;
}
```

Required behavior:

- render as `MobileSheet` with title `Meal planning`
- show progress as `{day label} {meal type} - {position} of {queue length}`
- keep the meal tray open across draft choices
- show favorite recipes and recent recipes when search is empty
- show matching recipes when meal name search has text
- support `Show all recipes`
- support quick meal draft creation with button label `Add quick meal draft`
- support recipe draft creation by selecting a recipe
- support `Skip this slot`, `Remove draft`, `Change draft`, `Keep editing`, `Save to week`, and `Cancel planning`
- when all queue items are skipped or drafted, show review summary with `{count} meals ready to add`
- show conflict summary when `conflictedTargets.length > 0` with actions `Skip conflicted and save remaining`, `Keep editing`, and `Cancel save`
- `Cancel save` clears the conflict summary and returns to the review state without discarding the planning session; `Cancel planning` discards the whole draft session

Reuse the candidate ordering from `MealComposerSheet`: favorites first, then recent by `updatedAt`, with search matching title or tags.

- [ ] **Step 5: Wire drafts into board rendering**

Modify `MealSlotCard` props:

```ts
interface MealSlotCardProps {
  slot: MealSlot;
  readOnly: boolean;
  pendingRecipeId?: string | null;
  draft?: MealPlanningDraft | null;
  isPlanningTarget?: boolean;
  onSelectSlot: (slot: MealSlot) => void;
}
```

Required rendering:

- if `draft` exists, render a button with aria-label `Draft {mealType}: {draft title}`
- add a visible `Draft` badge inside the card
- if `isPlanningTarget`, add a visible highlight class and `aria-current="true"`
- normal occupied, empty, read-only, and pending recipe placement behavior remain unchanged

Pass these props through `MealDayCard` and `MealGrid`.

- [ ] **Step 6: Orchestrate session state in `MealsView`**

Add state:

```ts
const [scopeOpen, setScopeOpen] = useState(false);
const [planningScope, setPlanningScope] = useState<MealPlanningScope | null>(null);
const [planningQueue, setPlanningQueue] = useState<MealSlot[]>([]);
const [planningDrafts, setPlanningDrafts] = useState<MealPlanningDraft[]>([]);
const [currentPlanningIndex, setCurrentPlanningIndex] = useState(0);
const [conflictedTargets, setConflictedTargets] = useState<MealPlanningTarget[]>([]);
```

In `MealsView`, render `Fill empty slots` near `WeekHeader` only when `board.data?.data` exists and `readOnly` is false.

When starting:

```ts
const queue = buildPlanningQueue(board.data.data, scope);
setPlanningScope(scope);
setPlanningQueue(queue);
setPlanningDrafts([]);
setCurrentPlanningIndex(0);
setConflictedTargets([]);
```

Use `applyPlanningDraftsToBoard(board.data.data, planningDrafts)` for display while the session is active.

When week changes, close scope/session state, clear selected slot/editor state, and keep existing recipe placement draft behavior unchanged.

- [ ] **Step 7: Run focused component tests and verify they pass**

Run:

```bash
cd frontend
npm test -- --run src/components/meals-view.test.tsx src/components/meals/meal-planning-panel.test.tsx
```

Expected: PASS for entry point, scope selection, draft behavior, cancel behavior, and existing Meals regressions.

- [ ] **Step 8: Commit**

```bash
cd frontend
git add src/components/meals/meal-planning-scope-dialog.tsx \
  src/components/meals/meal-planning-panel.tsx \
  src/components/meals/meal-planning-panel.test.tsx \
  src/components/meals/meal-slot-card.tsx \
  src/components/meals/meal-day-card.tsx \
  src/components/meals/meal-grid.tsx \
  src/components/meals-view.tsx \
  src/components/meals-view.test.tsx
git commit -m "feat(meals): add focused planning session UI"
```

## Task 6: Add save, conflict resolution, and regression coverage

**Files:**
- Modify: `frontend/src/components/meals-view.tsx`
- Modify: `frontend/src/components/meals-view.test.tsx`
- Modify: `frontend/src/components/meals/meal-planning-panel.tsx`
- Modify: `frontend/src/components/meals/meal-planning-panel.test.tsx`

- [ ] **Step 1: Add failing tests for save and conflict paths**

Add tests to `MealsView.test.tsx`:

```ts
it("saves drafted meals to the week with one batch request", async () => {
  seedMockMealsBoard(createEmptyMealsBoard());
  const { user } = renderWithUser(<MealsView />);

  await user.click(await screen.findByRole("button", { name: "Fill empty slots" }));
  await user.click(screen.getByRole("button", { name: "Start planning" }));
  await user.type(screen.getByLabelText("Meal name"), "Tacos");
  await user.click(screen.getByRole("button", { name: "Add quick meal draft" }));
  await user.click(screen.getByRole("button", { name: "Review plan" }));
  await user.click(screen.getByRole("button", { name: "Save to week" }));

  await waitFor(() => {
    expect(getMockMealsBoard(testWeekStartDate).days[0].slots[2].primary?.title).toBe("Tacos");
  });
  expect(screen.queryByRole("dialog", { name: /meal planning/i })).not.toBeInTheDocument();
});

it("handles save-time conflicts without overwriting and can skip conflicted drafts", async () => {
  seedMockMealsBoard(createEmptyMealsBoard());
  const { user } = renderWithUser(<MealsView />);

  await user.click(await screen.findByRole("button", { name: "Fill empty slots" }));
  await user.click(screen.getByRole("button", { name: "Start planning" }));
  await user.type(screen.getByLabelText("Meal name"), "Tacos");
  await user.click(screen.getByRole("button", { name: "Add quick meal draft" }));
  await user.type(screen.getByLabelText("Meal name"), "Leftovers");
  await user.click(screen.getByRole("button", { name: "Add quick meal draft" }));

  seedMockMealsBoard(createOccupiedMealsBoard());

  await user.click(screen.getByRole("button", { name: "Review plan" }));
  await user.click(screen.getByRole("button", { name: "Save to week" }));

  expect(await screen.findByText(/Some meal slots are no longer empty/i)).toBeInTheDocument();
  await user.click(screen.getByRole("button", { name: "Skip conflicted and save remaining" }));

  await waitFor(() => {
    const board = getMockMealsBoard(testWeekStartDate);
    expect(board.days[1].slots[2].primary?.title).toBe("Pasta");
    expect(board.days[0].slots[2].primary?.title).toBe("Tacos");
  });
});
```

- [ ] **Step 2: Run focused tests and verify they fail**

Run:

```bash
cd frontend
npm test -- --run src/components/meals-view.test.tsx src/components/meals/meal-planning-panel.test.tsx
```

Expected: FAIL until save and conflict state are wired.

- [ ] **Step 3: Wire `useSaveMealPlan` into `MealsView`**

Add:

```ts
import { ApiException } from "@/api/client";

const saveMealPlan = useSaveMealPlan({
  onSuccess: () => {
    setPlanningScope(null);
    setPlanningQueue([]);
    setPlanningDrafts([]);
    setCurrentPlanningIndex(0);
    setConflictedTargets([]);
  },
  onError: async (error) => {
    if (ApiException.isApiException(error) && error.status === 409) {
      const refreshed = await board.refetch();
      const nextBoard = refreshed.data?.data;
      if (nextBoard) {
        setConflictedTargets(getConflictedDraftTargets(nextBoard, planningDrafts));
      }
    }
  },
});
```

When saving:

```ts
saveMealPlan.mutate(toSaveMealPlanRequest(visibleWeekStartDate, planningDrafts));
```

When saving non-conflicted drafts:

```ts
const conflictKeys = new Set(conflictedTargets.map((target) => `${target.dayIndex}:${target.mealType}`));
const nonConflicted = planningDrafts.filter(
  (draft) => !conflictKeys.has(`${draft.target.dayIndex}:${draft.target.mealType}`),
);
if (nonConflicted.length === 0) {
  setConflictedTargets([]);
  setPlanningDrafts([]);
  return;
}
saveMealPlan.mutate(toSaveMealPlanRequest(visibleWeekStartDate, nonConflicted));
```

Do not call single-slot `useUpsertMealSlot` from the planning session.

- [ ] **Step 4: Run focused tests and verify they pass**

Run:

```bash
cd frontend
npm test -- --run src/components/meals-view.test.tsx src/components/meals/meal-planning-panel.test.tsx
```

Expected: PASS for save, conflict resolution, cancel, scopes, draft rendering, and existing Meals behavior.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/meals-view.tsx src/components/meals-view.test.tsx \
  src/components/meals/meal-planning-panel.tsx \
  src/components/meals/meal-planning-panel.test.tsx
git commit -m "feat(meals): save focused plans with conflict handling"
```

## Task 7: Add released-contract mobile E2E

**Files:**
- Modify: `frontend/e2e/mobile-meals.spec.ts`
- Modify: `frontend/e2e/helpers/api-helpers.ts`

- [ ] **Step 1: Add a recipe helper for mobile E2E setup**

Add to `e2e/helpers/api-helpers.ts`:

```ts
export async function createRecipe(
  request: APIRequestContext,
  token: string,
  recipe: {
    title: string;
    imageUrl?: string | null;
    ingredients?: string[];
    instructions?: string[];
    note?: string | null;
    tags?: string[];
    favorite?: boolean;
  },
): Promise<{ id: string; title: string }> {
  const response = await request.post(`${API_BASE}/recipes`, {
    headers: {
      Authorization: `Bearer ${token}`,
    },
    data: recipe,
  });

  if (!response.ok()) {
    const body = await response.text();
    throw new Error(`Create recipe failed (${response.status()}): ${body}`);
  }

  const json = await response.json();
  return json.data;
}
```

- [ ] **Step 2: Add failing mobile E2E for the full planning session**

Modify the existing helper import at the top of `e2e/mobile-meals.spec.ts`:

```ts
import { createRecipe, registerFamily, seedBrowserAuth } from "./helpers/api-helpers";
```

Add a test:

```ts
test("fills empty dinners in one focused planning session", async ({ page, request }) => {
  test.setTimeout(60000);

  const registration = await registerFamily(request, {
    familyName: "Focused Planning Crew",
    members: [{ name: "Sam", color: "teal" }],
  });

  await seedBrowserAuth(page, registration);
  await page.reload();
  await waitForHydration(page);
  await openMealsBoard(page);

  await safeClick(page.getByRole("button", { name: "Fill empty slots" }));
  const scopeDialog = page.getByRole("dialog", { name: /fill empty slots/i });
  await expect(scopeDialog).toBeVisible();
  await safeClick(scopeDialog.getByRole("button", { name: "Start planning" }));

  const planningSheet = page.getByRole("dialog", { name: "Meal planning" });
  await waitForSheetSettled(planningSheet);
  await expect(planningSheet.getByText("Sunday dinner - 1 of 7")).toBeVisible();

  await planningSheet.getByLabel("Meal name").fill("Tacos");
  await safeClick(planningSheet.getByRole("button", { name: "Add quick meal draft" }));
  await expect(planningSheet.getByText("Monday dinner - 2 of 7")).toBeVisible();

  await planningSheet.getByLabel("Meal name").fill("Leftovers");
  await safeClick(planningSheet.getByRole("button", { name: "Add quick meal draft" }));
  await expect(planningSheet.getByText("Tuesday dinner - 3 of 7")).toBeVisible();

  await planningSheet.getByLabel("Meal name").fill("Pasta");
  await safeClick(planningSheet.getByRole("button", { name: "Add quick meal draft" }));
  await safeClick(planningSheet.getByRole("button", { name: "Review plan" }));
  await expect(planningSheet.getByText("3 meals ready to add")).toBeVisible();
  await safeClick(planningSheet.getByRole("button", { name: "Save to week" }));

  await expect(planningSheet).toBeHidden();
  await expect(page.getByRole("button", { name: /open dinner: tacos/i })).toBeVisible();
  await expect(page.getByRole("button", { name: /open dinner: leftovers/i })).toBeVisible();
  await expect(page.getByRole("button", { name: /open dinner: pasta/i })).toBeVisible();

  await page.reload();
  await waitForHydration(page);
  await openMealsBoard(page);
  await expect(page.getByRole("button", { name: /open dinner: tacos/i })).toBeVisible();
  await expect(page.getByRole("button", { name: /open dinner: leftovers/i })).toBeVisible();
  await expect(page.getByRole("button", { name: /open dinner: pasta/i })).toBeVisible();
});
```

Add a recipe-backed future-week test:

```ts
test("plans a blank future week with a recipe-backed dinner", async ({ page, request }) => {
  test.setTimeout(60000);

  const registration = await registerFamily(request, {
    familyName: "Future Planning Crew",
    members: [{ name: "Sam", color: "teal" }],
  });
  await createRecipe(request, registration.token, {
    title: "Sheet Pan Chicken",
    ingredients: ["Chicken", "Potatoes"],
    instructions: ["Roast everything"],
    tags: ["Dinner"],
    favorite: true,
  });

  await seedBrowserAuth(page, registration);
  await page.reload();
  await waitForHydration(page);
  await openMealsBoard(page);

  await safeClick(page.getByRole("button", { name: "Next week" }));
  await safeClick(page.getByRole("button", { name: "Fill empty slots" }));
  const scopeDialog = page.getByRole("dialog", { name: /fill empty slots/i });
  await safeClick(scopeDialog.getByRole("button", { name: "Start planning" }));

  const planningSheet = page.getByRole("dialog", { name: "Meal planning" });
  await waitForSheetSettled(planningSheet);
  await safeClick(planningSheet.getByRole("button", { name: "Select recipe: Sheet Pan Chicken" }));
  await safeClick(planningSheet.getByRole("button", { name: "Review plan" }));
  await safeClick(planningSheet.getByRole("button", { name: "Save to week" }));

  await expect(planningSheet).toBeHidden();
  await expect(
    page.getByRole("button", { name: /open dinner: sheet pan chicken/i }),
  ).toBeVisible();
});
```

Add a past-week assertion to an existing or new mobile test:

```ts
await safeClick(page.getByRole("button", { name: "Previous week" }));
await expect(page.getByText("Review only")).toBeVisible();
await expect(page.getByRole("button", { name: "Fill empty slots" })).toBeHidden();
```

- [ ] **Step 3: Run focused E2E after the backend batch-save release exists**

Run with the same released backend resolution flow FE CI uses:

```bash
cd frontend
npm run test:e2e -- --project="Mobile Chrome" e2e/mobile-meals.spec.ts
```

Expected: PASS against the published backend release that contains `POST /api/meals/plans`. Record that backend semver in the FE PR body.

- [ ] **Step 4: Record the manual flow timing check**

In the FE PR body, record one local or device smoke check where three empty dinners are drafted and saved from the planning session in under three minutes without opening three slot composers. This is a product success criterion, not a separate automated performance test.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add e2e/mobile-meals.spec.ts e2e/helpers/api-helpers.ts
git commit -m "test(meals): cover focused planning session on mobile"
```

## Task 8: Final frontend verification and release handoff

**Files:**
- Read: `docs/superpowers/specs/2026-06-28-focused-meal-planning-sessions.md`
- Read: `docs/superpowers/plans/2026-06-28-focused-meal-planning-sessions.md`

- [ ] **Step 1: Run frontend focused verification**

Run:

```bash
cd frontend
npm test -- --run src/api/services/meals.service.test.ts src/api/hooks/use-meals.test.tsx src/lib/validations/meals.test.ts src/components/meals/meal-planning-session.test.ts src/components/meals/meal-planning-panel.test.tsx src/components/meals-view.test.tsx
```

Expected: PASS for planning contracts, helpers, panel, board integration, and existing Meals regressions.

- [ ] **Step 2: Run frontend standard checks**

Run:

```bash
cd frontend
npm run lint
npm test -- --run
npm run build
```

Expected: lint, all Vitest tests, and build pass.

- [ ] **Step 3: Run released-contract mobile E2E**

Run:

```bash
cd frontend
npm run test:e2e -- --project="Mobile Chrome" e2e/mobile-meals.spec.ts
```

Expected: PASS against the published backend release containing `POST /api/meals/plans`.

- [ ] **Step 4: Final PR checklist**

Before opening or merging the frontend PR, include:

- backend release semver tested by E2E
- mapping from every non-negotiable execution-contract bullet to code and tests
- confirmation that normal one-slot editing, recipe handoff, move, duplicate, remove, and past-week review behavior still pass
- confirmation that no grocery, pantry, AI planning, copy-week, status, or saved draft-plan scope was added

- [ ] **Step 5: Commit any final test-only updates**

```bash
cd frontend
git status --short
```

Expected: clean. If this is not clean, stop and inspect the changed files before PR handoff; the planned implementation commits above should already contain every required code and test change.

## Final contract audit before opening or merging PRs

- [ ] Backend batch save is a single transaction and never loops single-slot writes from the frontend.
- [ ] Backend rejects occupied primary, extras-only, duplicate targets, non-Sunday week starts, bad meal types, missing quick titles, missing recipe IDs, and recipes outside the authenticated family.
- [ ] Backend returns a refreshed board after successful save and 409 on save-time conflicts.
- [ ] Frontend planning drafts are local state and do not mutate TanStack Query board data before save.
- [ ] `Fill empty slots` is visible only for editable current/future weeks and hidden on past weeks.
- [ ] Scope picker supports `Empty dinners`, `All empty slots`, and `Selected days`.
- [ ] Board shows current target and draft blocks while filled persisted slots remain visible as context.
- [ ] User can skip, remove, change, cancel, review, save, keep editing, and skip conflicted drafts.
- [ ] Quick meals and recipe-backed meals both work in the planning session.
- [ ] Existing Meals editor, recipe placement handoff, move, duplicate, collision, remove, mobile layout, and past-week review tests remain green.
- [ ] Released-contract mobile E2E runs against the backend release that contains the batch endpoint, and the FE PR records the semver.
- [ ] Delivery Issues link Story, Spec, and Plan and copy the non-negotiables before implementation starts.

## Spec coverage checklist

- Visible-week planning entry: Tasks 5-7.
- Current and future week support: Tasks 5-7.
- Past weeks review-only: Tasks 5 and 7.
- Scope options: Tasks 4-5.
- Queue of empty slots: Task 4.
- Existing meals protected and visible: Tasks 4-6.
- Drafts local until save: Tasks 4-6.
- Review summary: Task 5.
- Atomic `Save to week`: Tasks 1-3 and 6.
- Save-time conflict paths: Tasks 1-3 and 6.
- Quick meal and recipe-backed candidates: Tasks 5-7.
- Faster-than-manual session evidence: Task 7 proves adding three dinners without opening separate slot composers and records the under-three-minute manual check.

## Risks to watch during execution

- The planning panel must not accidentally reuse `useUpsertMealSlot`; that would violate the draft contract.
- Conflict handling must not offer `replace_primary` or `add_as_extra` in v1 planning save.
- Draft board projection must not write into the query cache before save.
- Recipe-backed draft cards need a real display title from the selected `RecipeSummary`; do not ship generic recipe labels.
- The mobile panel must keep current slot, progress, candidates, and save/cancel actions reachable without nested sheets fighting for focus or hardware-back ownership.
