# Meals Weekly Planning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the placeholder `Meals` module with a real week-by-week household meal planner that consumes the `Recipes` foundation, supports quick meals and recipe-backed slots, renders mobile day-cards plus a larger-screen weekly grid, and persists stable board-level recipe snapshots for reviewable past weeks.

**Architecture:** Build a backend meal-slot aggregate in `backend/family-hub-api/` where each family owns week/day/meal-type slots and each slot stores a primary planned item plus optional extras as snapshot data. On the frontend in `frontend/`, replace the sample `MealsView` and placeholder `meals-store` with TanStack Query-backed week navigation, a text-first composer with recipe suggestions, a planning-focused slot editor, and a shared draft handoff from `Recipes`.

**Tech Stack:** Spring Boot 4, Java 21, JPA/Hibernate, Flyway, Maven, React 19, TypeScript, TanStack Query, React Hook Form, Zod, date-fns, Vitest, MSW, Playwright.

**Spec:** `docs/superpowers/specs/2026-06-01-meals-and-recipes-foundation-design.md`

---

## Execution Order

1. Re-verify the `Recipes` handoff contract before touching `Meals`
2. Backend meals schema and board contract
3. Backend slot write / move / duplicate flows
4. Frontend meals types, hooks, mocks, and utility reset
5. Frontend mobile board and composer
6. Frontend larger-screen grid, collision handling, and recipes handoff verification

Per root shipping rules, FE live verification that depends on real `/api/meals` behavior should wait for a published backend release. Mock-backed FE work can start earlier, but E2E should use the released backend image instead of unreleased backend `main`.

## Execution Contract

- `Meals` is a household-level weekly planning surface, not per-person meal assignment.
- `Recipes` already exists before this work starts and remains the source of reusable cooking content.
- Breakfast, lunch, and dinner are equal first-class slots.
- Current and future weeks are editable; past weeks are reviewable only in the FE.
- `Meals` has no status model. There is no cooked/skipped/done state.
- Empty slots must invite adding a meal.
- The composer is text-first with recipe suggestions, recent recipes, favorite recipes, and access to the full recipe library.
- A recipe-backed planned meal snapshots board-display state at placement time: source type, recipe id, title, image URL, and note.
- Recipe-backed meal detail opens the live recipe when it still exists; v1 does not snapshot full ingredients/instructions for historical recipe-detail fidelity.
- Move means move; duplicate is explicit.
- Slot collisions from move, duplicate, or recipe placement must prompt `Replace primary`, `Add as extra`, or `Cancel`.
- Past-week immutability is a frontend product guardrail in v1; backend write endpoints do not enforce it unless a later story adds server-side authorization.
- Backend JSON enum wire values must be lowercase (`breakfast`, `lunch`, `dinner`, `recipe`, `quick`, `replace_primary`, `add_as_extra`) even if Java enum constants are uppercase.
- This plan must consume the `Recipes` handoff contract; it must not redesign it in-place.

## Execution Issue Mapping

- Backend execution issue in `backend/family-hub-api`: Tasks 1-2. Contract: consumes the released Recipes backend contract `>= V15`, ships meal schema/API behavior, lowercase enum wire values, board-level snapshots, collision semantics, and backend tests. It should produce published backend release `>= V16` before real FE verification depends on it.
- Frontend execution issue in `frontend`: Tasks 3-5. Contract: consumes the released backend contract `>= V16` for live verification, while mock-backed unit work may start earlier.
- Root docs issue, if needed: Task 0/spec reconciliation only. Do not implement production code from the root workspace.

## File Structure

### Root docs / orchestration

- Create: `docs/superpowers/plans/2026-06-01-meals-weekly-planning.md`
- Read before implementation: `docs/superpowers/specs/2026-06-01-meals-and-recipes-foundation-design.md`
- Read before implementation: `docs/superpowers/plans/2026-06-01-recipes-module-foundation.md`

### Backend (`backend/family-hub-api/`)

Expected create / modify set:

- Create: `src/main/resources/db/migration/V16__create_meal_planning_tables.sql`
- Create: `src/main/java/com/familyhub/demo/model/MealType.java`
- Create: `src/main/java/com/familyhub/demo/model/MealEntrySourceType.java`
- Create: `src/main/java/com/familyhub/demo/model/MealCollisionMode.java`
- Create: `src/main/java/com/familyhub/demo/model/MealSlotRole.java`
- Create: `src/main/java/com/familyhub/demo/model/MealSlot.java`
- Create: `src/main/java/com/familyhub/demo/model/MealSlotEntry.java`
- Create: `src/main/java/com/familyhub/demo/repository/MealSlotRepository.java`
- Create: `src/main/java/com/familyhub/demo/dto/MealBoardResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/MealDayResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/MealSlotResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/MealSlotEntryResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/MealEntryRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/UpsertMealSlotRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/MoveMealSlotRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/DuplicateMealSlotRequest.java`
- Create: `src/main/java/com/familyhub/demo/mapper/MealMapper.java`
- Create: `src/main/java/com/familyhub/demo/service/MealService.java`
- Create: `src/main/java/com/familyhub/demo/controller/MealController.java`
- Modify: `src/test/java/com/familyhub/demo/TestDataFactory.java`
- Create: `src/test/java/com/familyhub/demo/service/MealServiceTest.java`
- Create: `src/test/java/com/familyhub/demo/controller/MealControllerTest.java`
- Create: `src/test/java/com/familyhub/demo/integration/MealIntegrationTest.java`

### Frontend (`frontend/`)

Expected create / modify set:

- Modify: `src/lib/types/meals.ts`
- Modify: `src/lib/types/index.ts`
- Create: `src/api/services/meals.service.ts`
- Modify: `src/api/services/index.ts`
- Create: `src/api/hooks/use-meals.ts`
- Modify: `src/api/hooks/index.ts`
- Modify: `src/api/index.ts`
- Create: `src/lib/validations/meals.ts`
- Modify: `src/lib/validations/index.ts`
- Create: `src/lib/validations/meals.test.ts`
- Modify: `src/test/mocks/handlers.ts`
- Modify: `src/test/mocks/server.ts`
- Modify: `src/test/fixtures/recipes.ts`
- Delete: `src/stores/meals-store.ts`
- Modify: `src/stores/index.ts`
- Modify: `src/lib/calendar-data.ts`
- Modify: `src/lib/time-utils.ts`
- Create: `src/components/meals/week-header.tsx`
- Create: `src/components/meals/meal-day-card.tsx`
- Create: `src/components/meals/meal-grid.tsx`
- Create: `src/components/meals/meal-slot-card.tsx`
- Create: `src/components/meals/meal-composer-sheet.tsx`
- Create: `src/components/meals/meal-editor-sheet.tsx`
- Create: `src/components/meals/recipe-match-list.tsx`
- Modify: `src/components/meals-view.tsx`
- Create: `src/api/hooks/use-meals.test.tsx`
- Create: `src/components/meals-view.test.tsx`
- Create: `src/components/meals/meal-composer-sheet.test.tsx`
- Create: `e2e/mobile-meals.spec.ts`

This structure keeps `Meals` data in `@/api`, shared week/slot modeling in `@/lib/types` and `@/lib/validations`, and view-specific rendering in small `src/components/meals/*` files so the rewritten `MealsView` remains a coordinator rather than a giant all-in-one board file.

## Task 0: Re-Verify The Recipes-To-Meals Handoff Contract Before Building Meals

**Files:**
- Read: `docs/superpowers/specs/2026-06-01-meals-and-recipes-foundation-design.md`
- Read: `docs/superpowers/plans/2026-06-01-recipes-module-foundation.md`
- Read: `frontend/src/stores/app-store.ts`

- [ ] **Step 1: Confirm the app store exposes the shared placement draft created by the Recipes plan**

```ts
interface MealPlacementDraft {
  recipeId: string;
  requestedAtWeekStartDate: string;
  source:
    | { kind: "recipes-library" }
    | { kind: "meals-slot"; dayIndex: number; mealType: "breakfast" | "lunch" | "dinner" };
}

interface RecipeCreationDraft {
  requestedAtWeekStartDate: string;
  dayIndex: number;
  mealType: "breakfast" | "lunch" | "dinner";
  typedTitle: string;
}

startMealPlacementFromRecipe(draft: MealPlacementDraft): void;
consumeMealPlacementDraft(): MealPlacementDraft | null;
startRecipeCreationFromMealSlot(draft: RecipeCreationDraft): void;
consumeRecipeCreationDraft(): RecipeCreationDraft | null;
```

- [ ] **Step 2: Verify the Recipes plan ended with Add-to-Meals intent state instead of ad hoc navigation-only behavior**

Run:

```bash
cd /Users/joe.bor/code/family-hub
rg -n "mealPlacementDraft|startMealPlacementFromRecipe|Add to Meals" \
  docs/superpowers/plans/2026-06-01-recipes-module-foundation.md \
  frontend/src/stores/app-store.ts \
  frontend/src/components/recipes/recipe-detail-view.tsx
```

Expected: the shared draft contract exists and `Meals` can consume it without redesigning the handoff.

- [ ] **Step 3: If the contract drifted during Recipes implementation, update the spec before continuing**

```md
## Cross-Module Contract

- `Recipes` owns the placement draft trigger.
- `Meals` owns slot selection and persisted placement.
- `Recipes -> Meals` placement drafts carry `recipeId`, requested week, and source context.
- `Meals -> Recipes` creation drafts carry week, day, meal type, and typed meal text so recipe creation can return to planning.
- Shared meal type wire values remain lowercase across FE and BE.
- Meal snapshots preserve board-display state; live recipe detail remains best-effort.
```

- [ ] **Step 4: Commit any spec-only contract clarification before coding Meals**

```bash
cd /Users/joe.bor/code/family-hub
git add docs/superpowers/specs/2026-06-01-meals-and-recipes-foundation-design.md
git commit -m "docs(meals): clarify recipes handoff contract"
```

## Task 1: Add Backend Meal Planning Schema, Board Read Model, And Initial Upsert

**Files:**
- Create: `backend/family-hub-api/src/main/resources/db/migration/V16__create_meal_planning_tables.sql`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/MealType.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/MealEntrySourceType.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/MealCollisionMode.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/MealSlotRole.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/MealSlot.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/MealSlotEntry.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/MealSlotRepository.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/MealEntryRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/UpsertMealSlotRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/MealBoardResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/MealDayResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/MealSlotResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/MealSlotEntryResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/mapper/MealMapper.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/MealService.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/TestDataFactory.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/service/MealServiceTest.java`

- [ ] **Step 1: Write the failing backend tests for the week board and snapshot reads**

```java
@Test
void getBoard_returnsSevenDaysWithBreakfastLunchDinnerSlots() {
    LocalDate weekStart = LocalDate.of(2026, 6, 7);

    MealBoardResponse board = mealService.getBoard(weekStart, family);

    assertThat(board.weekStartDate()).isEqualTo(weekStart);
    assertThat(board.days()).hasSize(7);
    assertThat(board.days().getFirst().slots()).extracting(MealSlotResponse::mealType)
            .containsExactly(MealType.BREAKFAST, MealType.LUNCH, MealType.DINNER);
}

@Test
void getBoard_returnsRecipeSnapshotsRatherThanLiveRecipeFields() {
    Recipe recipe = recipeRepository.saveAndFlush(TestDataFactory.recipe(family, "Original Tacos"));
    mealService.upsertSlot(new UpsertMealSlotRequest(
            LocalDate.of(2026, 6, 7),
            1,
            MealType.DINNER,
            new MealEntryRequest(MealEntrySourceType.RECIPE, recipe.getId(), null, null, null),
            List.of(),
            null,
            null
    ), family);

    recipe.setTitle("Updated Tacos");
    recipeRepository.saveAndFlush(recipe);

    MealBoardResponse board = mealService.getBoard(LocalDate.of(2026, 6, 7), family);

    assertThat(board.days().get(1).slots().get(2).primary().title()).isEqualTo("Original Tacos");
}

@Test
void upsertSlot_createsQuickMealPrimaryAndExtras() {
    UpsertMealSlotRequest request = new UpsertMealSlotRequest(
            LocalDate.of(2026, 6, 7),
            2,
            MealType.DINNER,
            new MealEntryRequest(MealEntrySourceType.QUICK, null, "Leftovers", null, null),
            List.of(new MealEntryRequest(MealEntrySourceType.QUICK, null, "Salad", null, null)),
            null,
            null
    );

    MealSlotResponse response = mealService.upsertSlot(request, family);

    assertThat(response.primary().title()).isEqualTo("Leftovers");
    assertThat(response.extras()).extracting(MealSlotEntryResponse::title).containsExactly("Salad");
}
```

- [ ] **Step 2: Run the backend meal-service test to verify the planning aggregate does not exist yet**

Run:

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
./mvnw -Dtest=MealServiceTest test
```

Expected: FAIL with missing `Meal*` models, DTOs, and service.

- [ ] **Step 3: Add the migration, entities, repository, and read DTOs**

```sql
CREATE TABLE meal_slot (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL,
    week_start_date DATE NOT NULL,
    day_index INTEGER NOT NULL,
    meal_type VARCHAR(20) NOT NULL,
    note TEXT,
    created_at TIMESTAMP(6) DEFAULT now(),
    updated_at TIMESTAMP(6) DEFAULT now(),
    CONSTRAINT fk_meal_slot_family
        FOREIGN KEY (family_id) REFERENCES family(id) ON DELETE CASCADE,
    CONSTRAINT uk_meal_slot_family_week_day_type UNIQUE (family_id, week_start_date, day_index, meal_type)
);

CREATE TABLE meal_slot_entry (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slot_id UUID NOT NULL,
    role VARCHAR(20) NOT NULL,
    sort_order INTEGER NOT NULL,
    source_type VARCHAR(20) NOT NULL,
    recipe_id UUID,
    title_snapshot VARCHAR(160) NOT NULL,
    image_url_snapshot TEXT,
    note_snapshot TEXT,
    created_at TIMESTAMP(6) DEFAULT now(),
    updated_at TIMESTAMP(6) DEFAULT now(),
    CONSTRAINT fk_meal_slot_entry_slot
        FOREIGN KEY (slot_id) REFERENCES meal_slot(id) ON DELETE CASCADE,
    CONSTRAINT fk_meal_slot_entry_recipe
        FOREIGN KEY (recipe_id) REFERENCES recipe(id) ON DELETE SET NULL,
    CONSTRAINT uk_meal_slot_entry_slot_sort UNIQUE (slot_id, sort_order)
);
```

```java
public enum MealType {
    BREAKFAST("breakfast"),
    LUNCH("lunch"),
    DINNER("dinner");

    private final String wireValue;

    MealType(String wireValue) {
        this.wireValue = wireValue;
    }

    @JsonValue
    public String wireValue() {
        return wireValue;
    }

    @JsonCreator
    public static MealType fromValue(String value) {
        return Arrays.stream(values())
                .filter(type -> type.wireValue.equalsIgnoreCase(value) || type.name().equalsIgnoreCase(value))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("Unknown meal type: " + value));
    }
}

public enum MealEntrySourceType {
    RECIPE("recipe"),
    QUICK("quick");

    private final String wireValue;

    MealEntrySourceType(String wireValue) {
        this.wireValue = wireValue;
    }

    @JsonValue
    public String wireValue() {
        return wireValue;
    }

    @JsonCreator
    public static MealEntrySourceType fromValue(String value) {
        return Arrays.stream(values())
                .filter(type -> type.wireValue.equalsIgnoreCase(value) || type.name().equalsIgnoreCase(value))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("Unknown meal entry source type: " + value));
    }
}

public enum MealCollisionMode {
    REPLACE_PRIMARY("replace_primary"),
    ADD_AS_EXTRA("add_as_extra");

    private final String wireValue;

    MealCollisionMode(String wireValue) {
        this.wireValue = wireValue;
    }

    @JsonValue
    public String wireValue() {
        return wireValue;
    }

    @JsonCreator
    public static MealCollisionMode fromValue(String value) {
        return Arrays.stream(values())
                .filter(mode -> mode.wireValue.equalsIgnoreCase(value) || mode.name().equalsIgnoreCase(value))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("Unknown meal collision mode: " + value));
    }
}
```

```java
public record MealEntryRequest(
        @NotNull MealEntrySourceType sourceType,
        UUID recipeId,
        String title,
        String imageUrl,
        String note
) {}

public record UpsertMealSlotRequest(
        @NotNull LocalDate weekStartDate,
        @Min(0) @Max(6) int dayIndex,
        @NotNull MealType mealType,
        @Valid @NotNull MealEntryRequest primary,
        List<@Valid MealEntryRequest> extras,
        String note,
        MealCollisionMode collisionMode
) {}
```

```java
public record MealSlotEntryResponse(
        UUID id,
        MealEntrySourceType sourceType,
        UUID recipeId,
        String title,
        String imageUrl,
        String note
) {}
```

```java
public interface MealSlotRepository extends JpaRepository<MealSlot, UUID> {
    List<MealSlot> findByFamilyAndWeekStartDateOrderByDayIndexAscMealTypeAsc(Family family, LocalDate weekStartDate);
    Optional<MealSlot> findByFamilyAndWeekStartDateAndDayIndexAndMealType(
            Family family,
            LocalDate weekStartDate,
            int dayIndex,
            MealType mealType
    );
}
```

```java
private MealSlotEntry snapshotEntry(MealEntryRequest request, MealSlot slot, MealSlotRole role, int sortOrder) {
    MealSlotEntry entry = new MealSlotEntry();
    entry.setSlot(slot);
    entry.setRole(role);
    entry.setSortOrder(sortOrder);
    entry.setSourceType(request.sourceType());

    if (request.sourceType() == MealEntrySourceType.RECIPE) {
        Recipe recipe = recipeRepository.findByIdAndFamily(request.recipeId(), slot.getFamily())
                .orElseThrow(() -> new ResourceNotFoundException("Recipe", request.recipeId()));
        entry.setRecipe(recipe);
        entry.setTitleSnapshot(recipe.getTitle());
        entry.setImageUrlSnapshot(recipe.getImageUrl());
        entry.setNoteSnapshot(recipe.getNote());
    } else {
        entry.setTitleSnapshot(request.title().trim());
        entry.setImageUrlSnapshot(request.imageUrl());
        entry.setNoteSnapshot(request.note());
    }

    return entry;
}
```

- [ ] **Step 4: Run the backend meal-service test to verify the board and initial upsert behavior now pass**

Run:

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
./mvnw -Dtest=MealServiceTest test
```

Expected: PASS for the week board shape, initial slot upsert, and board-level recipe snapshot behavior.

- [ ] **Step 5: Commit**

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
git add src/main/resources/db/migration/V16__create_meal_planning_tables.sql \
  src/main/java/com/familyhub/demo/model/MealType.java \
  src/main/java/com/familyhub/demo/model/MealEntrySourceType.java \
  src/main/java/com/familyhub/demo/model/MealCollisionMode.java \
  src/main/java/com/familyhub/demo/model/MealSlotRole.java \
  src/main/java/com/familyhub/demo/model/MealSlot.java \
  src/main/java/com/familyhub/demo/model/MealSlotEntry.java \
  src/main/java/com/familyhub/demo/repository/MealSlotRepository.java \
  src/main/java/com/familyhub/demo/dto/MealEntryRequest.java \
  src/main/java/com/familyhub/demo/dto/UpsertMealSlotRequest.java \
  src/main/java/com/familyhub/demo/dto/MealBoardResponse.java \
  src/main/java/com/familyhub/demo/dto/MealDayResponse.java \
  src/main/java/com/familyhub/demo/dto/MealSlotResponse.java \
  src/main/java/com/familyhub/demo/dto/MealSlotEntryResponse.java \
  src/main/java/com/familyhub/demo/mapper/MealMapper.java \
  src/main/java/com/familyhub/demo/service/MealService.java \
  src/test/java/com/familyhub/demo/TestDataFactory.java \
  src/test/java/com/familyhub/demo/service/MealServiceTest.java
git commit -m "feat(meals): add meal board foundation"
```

## Task 2: Add Backend Slot Write, Move, Duplicate, And Collision Handling

**Files:**
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/MoveMealSlotRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/DuplicateMealSlotRequest.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/MealService.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/controller/MealController.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/controller/MealControllerTest.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/integration/MealIntegrationTest.java`

- [ ] **Step 1: Write the failing tests for controller upsert, move, duplicate, and collision resolution**

```java
void moveSlot_addAsExtra_flattensMovedUnitIntoDestinationExtras() {
    // given a source slot with primary + extra and a destination slot with an existing primary
    MoveMealSlotRequest request = new MoveMealSlotRequest(
            LocalDate.of(2026, 6, 7), 1, MealType.DINNER,
            LocalDate.of(2026, 6, 7), 2, MealType.DINNER,
            MealCollisionMode.ADD_AS_EXTRA
    );

    mealService.moveSlot(request, family);

    MealBoardResponse board = mealService.getBoard(LocalDate.of(2026, 6, 7), family);
    assertThat(board.days().get(2).slots().get(2).extras())
            .extracting(MealSlotEntryResponse::title)
            .contains("Source Primary", "Source Side");
}
```

```java
@Test
@WithMockFamily
void upsertSlot_returns200WithLowercaseWireValues() throws Exception {
    mockMvc.perform(put("/api/meals/slots")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            {
                              "weekStartDate": "2026-06-07",
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
                              "note": null,
                              "collisionMode": null
                            }
                            """))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.mealType").value("dinner"))
            .andExpect(jsonPath("$.data.primary.sourceType").value("quick"));
}

@Test
@WithMockFamily
void duplicateSlot_returns200() throws Exception {
    mockMvc.perform(post("/api/meals/slots/duplicate")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            {
                              "sourceWeekStartDate": "2026-06-07",
                              "sourceDayIndex": 1,
                              "sourceMealType": "dinner",
                              "destinationWeekStartDate": "2026-06-07",
                              "destinationDayIndex": 4,
                              "destinationMealType": "dinner",
                              "collisionMode": "replace_primary"
                            }
                            """))
            .andExpect(status().isOk());
}
```

- [ ] **Step 2: Run the controller and integration tests to verify writes are still missing**

Run:

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
./mvnw -Dtest=MealControllerTest,MealIntegrationTest test
```

Expected: FAIL with missing move/duplicate request DTOs, controller endpoints, and collision handling.

- [ ] **Step 3: Implement move/duplicate DTOs, controller endpoints, and collision semantics**

Extend the existing Task 1 `upsertSlot` service method so an occupied primary slot cannot be overwritten unless the request includes an explicit `collisionMode`. `replace_primary` replaces the primary; `add_as_extra` preserves the existing primary and appends the requested meal as an extra.

```java
public record MoveMealSlotRequest(
        @NotNull LocalDate sourceWeekStartDate,
        @Min(0) @Max(6) int sourceDayIndex,
        @NotNull MealType sourceMealType,
        @NotNull LocalDate destinationWeekStartDate,
        @Min(0) @Max(6) int destinationDayIndex,
        @NotNull MealType destinationMealType,
        @NotNull MealCollisionMode collisionMode
) {}

public record DuplicateMealSlotRequest(
        @NotNull LocalDate sourceWeekStartDate,
        @Min(0) @Max(6) int sourceDayIndex,
        @NotNull MealType sourceMealType,
        @NotNull LocalDate destinationWeekStartDate,
        @Min(0) @Max(6) int destinationDayIndex,
        @NotNull MealType destinationMealType,
        @NotNull MealCollisionMode collisionMode
) {}
```

```java
@PutMapping("/slots")
public ResponseEntity<ApiResponse<MealSlotResponse>> upsertSlot(
        @Valid @RequestBody UpsertMealSlotRequest request,
        @AuthenticationPrincipal Family family
) {
    return ResponseEntity.ok(new ApiResponse<>(mealService.upsertSlot(request, family), "Meal slot updated successfully"));
}

@PostMapping("/slots/move")
public ResponseEntity<ApiResponse<MealBoardResponse>> moveSlot(
        @Valid @RequestBody MoveMealSlotRequest request,
        @AuthenticationPrincipal Family family
) {
    return ResponseEntity.ok(new ApiResponse<>(mealService.moveSlot(request, family), "Meal slot moved successfully"));
}

@PostMapping("/slots/duplicate")
public ResponseEntity<ApiResponse<MealBoardResponse>> duplicateSlot(
        @Valid @RequestBody DuplicateMealSlotRequest request,
        @AuthenticationPrincipal Family family
) {
    return ResponseEntity.ok(new ApiResponse<>(mealService.duplicateSlot(request, family), "Meal slot duplicated successfully"));
}
```

- [ ] **Step 4: Run the backend write-path tests to verify slot upsert, move, duplicate, and collision behavior**

Run:

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
./mvnw -Dtest=MealServiceTest,MealControllerTest,MealIntegrationTest test
```

Expected: PASS for quick meals, board-level recipe snapshotting, move/duplicate behavior, and collision resolution semantics.

- [ ] **Step 5: Commit**

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
git add src/main/java/com/familyhub/demo/dto/MoveMealSlotRequest.java \
  src/main/java/com/familyhub/demo/dto/DuplicateMealSlotRequest.java \
  src/main/java/com/familyhub/demo/service/MealService.java \
  src/main/java/com/familyhub/demo/controller/MealController.java \
  src/test/java/com/familyhub/demo/controller/MealControllerTest.java \
  src/test/java/com/familyhub/demo/integration/MealIntegrationTest.java
git commit -m "feat(meals): add meal slot write flows"
```

## Task 3: Replace Placeholder Frontend Meals Contracts, Utilities, And Mocks

**Files:**
- Modify: `frontend/src/lib/types/meals.ts`
- Modify: `frontend/src/lib/types/index.ts`
- Create: `frontend/src/api/services/meals.service.ts`
- Modify: `frontend/src/api/services/index.ts`
- Create: `frontend/src/api/hooks/use-meals.ts`
- Modify: `frontend/src/api/hooks/index.ts`
- Modify: `frontend/src/api/index.ts`
- Create: `frontend/src/lib/validations/meals.ts`
- Modify: `frontend/src/lib/validations/index.ts`
- Create: `frontend/src/lib/validations/meals.test.ts`
- Modify: `frontend/src/test/mocks/handlers.ts`
- Modify: `frontend/src/test/mocks/server.ts`
- Modify: `frontend/src/test/fixtures/recipes.ts`
- Delete: `frontend/src/stores/meals-store.ts`
- Modify: `frontend/src/stores/index.ts`
- Modify: `frontend/src/lib/calendar-data.ts`
- Modify: `frontend/src/lib/time-utils.ts`
- Create: `frontend/src/api/hooks/use-meals.test.tsx`

- [ ] **Step 1: Write the failing frontend tests for week navigation, board loading, and composer validation**

```tsx
it("loads a meals board for the selected week", async () => {
  const { result } = renderHook(() => useMealsBoard("2026-06-07"), { wrapper: createWrapper() });

  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data?.data.days).toHaveLength(7);
});
```

```ts
it("requires a quick meal name when source type is quick", () => {
  const parsed = mealEntrySchema.safeParse({
    sourceType: "quick",
    recipeId: null,
    title: "",
    imageUrl: null,
    note: null,
  });

  expect(parsed.success).toBe(false);
});
```

- [ ] **Step 2: Run the focused tests to verify the meals contract still uses placeholder types**

Run:

```bash
cd /Users/joe.bor/code/family-hub/frontend
npm test -- --run src/api/hooks/use-meals.test.tsx src/lib/validations/meals.test.ts
```

Expected: FAIL with missing API-backed `Meals` types/hooks and validation.

- [ ] **Step 3: Add the API contract, delete the placeholder store, and extend time utilities for Sunday-first week navigation**

`getWeekStartSunday` must already exist from the completed Recipes plan. This task adds `addWeeksLocal` and `isPastWeek`; if the week-start helper is missing, stop and fix the Recipes handoff contract instead of redefining it here.

```ts
export interface MealSlotEntry {
  id: string;
  sourceType: "recipe" | "quick";
  recipeId: string | null;
  title: string;
  imageUrl: string | null;
  note: string | null;
}

export interface MealSlot {
  mealType: "breakfast" | "lunch" | "dinner";
  primary: MealSlotEntry | null;
  extras: MealSlotEntry[];
  note: string | null;
}

export interface MealsBoard {
  weekStartDate: string;
  days: Array<{
    date: string;
    dayIndex: number;
    slots: MealSlot[];
  }>;
}

export type MealCollisionMode = "replace_primary" | "add_as_extra";

export interface MealEntryRequest {
  sourceType: "recipe" | "quick";
  recipeId: string | null;
  title: string | null;
  imageUrl: string | null;
  note: string | null;
}

export interface UpsertMealSlotRequest {
  weekStartDate: string;
  dayIndex: number;
  mealType: "breakfast" | "lunch" | "dinner";
  primary: MealEntryRequest;
  extras: MealEntryRequest[];
  note: string | null;
  collisionMode: MealCollisionMode | null;
}

export interface MoveMealSlotRequest {
  sourceWeekStartDate: string;
  sourceDayIndex: number;
  sourceMealType: "breakfast" | "lunch" | "dinner";
  destinationWeekStartDate: string;
  destinationDayIndex: number;
  destinationMealType: "breakfast" | "lunch" | "dinner";
  collisionMode: MealCollisionMode;
}

export interface DuplicateMealSlotRequest extends MoveMealSlotRequest {}
```

```ts
export const mealsService = {
  getBoard(weekStartDate: string) {
    return httpClient.get<ApiResponse<MealsBoard>>(`/meals/board?weekStartDate=${weekStartDate}`);
  },
  upsertSlot(request: UpsertMealSlotRequest) {
    return httpClient.put<ApiResponse<MealSlot>>("/meals/slots", request);
  },
  moveSlot(request: MoveMealSlotRequest) {
    return httpClient.post<ApiResponse<MealsBoard>>("/meals/slots/move", request);
  },
  duplicateSlot(request: DuplicateMealSlotRequest) {
    return httpClient.post<ApiResponse<MealsBoard>>("/meals/slots/duplicate", request);
  },
};
```

```ts
export function addWeeksLocal(date: Date, amount: number): Date {
  const copy = new Date(date);
  copy.setDate(copy.getDate() + amount * 7);
  return copy;
}

export function isPastWeek(weekStartDate: string, now: Date = new Date()): boolean {
  return parseLocalDate(weekStartDate).getTime() < getWeekStartSunday(now).getTime();
}
```

- [ ] **Step 4: Run the contract-layer tests to verify the new board query and validation pass**

Run:

```bash
cd /Users/joe.bor/code/family-hub/frontend
npm test -- --run src/api/hooks/use-meals.test.tsx src/lib/validations/meals.test.ts
```

Expected: PASS for board loading, slot mutation contract, quick-meal validation, and Sunday-first week utilities.

- [ ] **Step 5: Commit**

```bash
cd /Users/joe.bor/code/family-hub/frontend
git add src/lib/types/meals.ts \
  src/lib/types/index.ts \
  src/api/services/meals.service.ts \
  src/api/services/index.ts \
  src/api/hooks/use-meals.ts \
  src/api/hooks/index.ts \
  src/api/index.ts \
  src/lib/validations/meals.ts \
  src/lib/validations/index.ts \
  src/lib/validations/meals.test.ts \
  src/test/mocks/handlers.ts \
  src/test/mocks/server.ts \
  src/test/fixtures/recipes.ts \
  src/stores/index.ts \
  src/lib/calendar-data.ts \
  src/lib/time-utils.ts \
  src/api/hooks/use-meals.test.tsx
git rm src/stores/meals-store.ts
git commit -m "feat(meals): replace placeholder meals contract"
```

## Task 4: Build The Mobile-First Meals Board, Composer, And Quick Meal Flow

**Files:**
- Create: `frontend/src/components/meals/week-header.tsx`
- Create: `frontend/src/components/meals/meal-day-card.tsx`
- Create: `frontend/src/components/meals/meal-slot-card.tsx`
- Create: `frontend/src/components/meals/meal-composer-sheet.tsx`
- Create: `frontend/src/components/meals/recipe-match-list.tsx`
- Modify: `frontend/src/components/meals-view.tsx`
- Create: `frontend/src/components/meals/meal-composer-sheet.test.tsx`
- Create: `frontend/src/components/meals-view.test.tsx`

- [ ] **Step 1: Write the failing UI tests for empty slots, composer suggestions, quick meal creation, and recipe-placement collisions**

```tsx
it("shows add affordances for empty meal slots", async () => {
  render(<MealsView />);
  expect(await screen.findAllByRole("button", { name: /add meal/i })).not.toHaveLength(0);
});

it("suggests matching recipes while typing and still allows a quick meal", async () => {
  render(<MealComposerSheet open slot={mondayDinnerSlot} onOpenChange={vi.fn()} />);

  await user.type(screen.getByLabelText(/meal name/i), "Left");
  expect(await screen.findByText("Leftovers")).toBeInTheDocument();
  expect(screen.getByRole("button", { name: /create quick meal/i })).toBeInTheDocument();
  expect(screen.getByRole("button", { name: /create recipe from this/i })).toBeInTheDocument();
});

it("returns to Recipes with a creation draft when Create recipe from this is chosen", async () => {
  render(<MealComposerSheet open slot={mondayDinnerSlot} onOpenChange={vi.fn()} />);

  await user.type(screen.getByLabelText(/meal name/i), "Leftovers");
  await user.click(screen.getByRole("button", { name: /create recipe from this/i }));

  expect(useAppStore.getState().activeModule).toBe("recipes");
  expect(useAppStore.getState().recipeCreationDraft).toEqual({
    requestedAtWeekStartDate: "2026-06-07",
    dayIndex: 1,
    mealType: "dinner",
    typedTitle: "Leftovers",
  });
});

it("prompts before placing a recipe draft into an occupied slot", async () => {
  render(
    <MealComposerSheet
      open
      slot={{ ...occupiedDinnerSlot, seededRecipeId: "recipe-1" }}
      readOnly={false}
      onOpenChange={vi.fn()}
    />,
  );

  await user.click(await screen.findByRole("button", { name: /add recipe to slot/i }));

  expect(await screen.findByText("That slot already has a meal")).toBeInTheDocument();
  expect(screen.getByRole("button", { name: /replace primary/i })).toBeInTheDocument();
  expect(screen.getByRole("button", { name: /add as extra/i })).toBeInTheDocument();
});
```

- [ ] **Step 2: Run the focused Meals UI tests to verify the board is still the old placeholder**

Run:

```bash
cd /Users/joe.bor/code/family-hub/frontend
npm test -- --run src/components/meals-view.test.tsx src/components/meals/meal-composer-sheet.test.tsx
```

Expected: FAIL because `MealsView` still renders sample breakfast/lunch/dinner cards instead of API-backed slots and composer behavior.

- [ ] **Step 3: Rewrite MealsView around the board query, week header, and mobile composer flow**

```tsx
export function MealsView() {
  const [visibleWeekStartDate, setVisibleWeekStartDate] = useState(
    formatLocalDate(getWeekStartSunday(new Date())),
  );
  const [selectedSlot, setSelectedSlot] = useState<MealSlotSelection | null>(null);
  const [placementDraft, setPlacementDraft] = useState<MealPlacementDraft | null>(
    null,
  );
  const board = useMealsBoard(visibleWeekStartDate);
  const pendingPlacementDraft = useAppStore((state) => state.mealPlacementDraft);
  const consumeMealPlacementDraft = useAppStore(
    (state) => state.consumeMealPlacementDraft,
  );
  const isReadOnlyWeek = isPastWeek(visibleWeekStartDate);

  useEffect(() => {
    if (!pendingPlacementDraft) return;
    setVisibleWeekStartDate(pendingPlacementDraft.requestedAtWeekStartDate);
    setPlacementDraft(pendingPlacementDraft);
    if (pendingPlacementDraft.source.kind === "meals-slot") {
      setSelectedSlot({
        weekStartDate: pendingPlacementDraft.requestedAtWeekStartDate,
        dayIndex: pendingPlacementDraft.source.dayIndex,
        mealType: pendingPlacementDraft.source.mealType,
        seededRecipeId: pendingPlacementDraft.recipeId,
      });
    }
    consumeMealPlacementDraft();
  }, [pendingPlacementDraft, consumeMealPlacementDraft]);

  return (
    <>
      <div className="flex-1 overflow-y-auto p-4 sm:p-6">
        <div className="mx-auto max-w-5xl space-y-6">
          <WeekHeader
            weekStartDate={visibleWeekStartDate}
            onPrevious={() => setVisibleWeekStartDate(formatLocalDate(addWeeksLocal(parseLocalDate(visibleWeekStartDate), -1)))}
            onNext={() => setVisibleWeekStartDate(formatLocalDate(addWeeksLocal(parseLocalDate(visibleWeekStartDate), 1)))}
          />

          <div className="space-y-4 lg:hidden">
            {board.data?.data.days.map((day) => (
              <MealDayCard
                key={day.date}
                day={day}
                readOnly={isReadOnlyWeek}
                pendingPlacementRecipeId={placementDraft?.source.kind === "recipes-library" ? placementDraft.recipeId : null}
                onSelectSlot={(slot) => {
                  setSelectedSlot(
                    placementDraft?.source.kind === "recipes-library"
                      ? {
                          ...slot,
                          seededRecipeId: placementDraft.recipeId,
                        }
                      : slot,
                  );
                  setPlacementDraft(null);
                }}
              />
            ))}
          </div>
        </div>
      </div>

      <MealComposerSheet
        open={selectedSlot !== null}
        slot={selectedSlot}
        readOnly={isReadOnlyWeek}
        onOpenChange={(open) => !open && setSelectedSlot(null)}
      />
    </>
  );
}
```

```tsx
<Button type="button" variant="outline" onClick={handleCreateQuickMeal}>
  Create quick meal
</Button>
<Button type="button" variant="secondary" onClick={handleCreateRecipeFromTypedText}>
  Create recipe from this
</Button>
<RecipeMatchList
  query={searchValue}
  recentRecipes={recentRecipes}
  favoriteRecipes={favoriteRecipes}
  onSelectRecipe={handleSelectRecipe}
/>

const startRecipeCreationFromMealSlot = useAppStore(
  (state) => state.startRecipeCreationFromMealSlot,
);

function handleCreateRecipeFromTypedText() {
  startRecipeCreationFromMealSlot({
    requestedAtWeekStartDate: slot.weekStartDate,
    dayIndex: slot.dayIndex,
    mealType: slot.mealType,
    typedTitle: searchValue.trim(),
  });
}
```

```tsx
function submitSeededRecipe(collisionMode: MealCollisionMode | null = null) {
  if (!slot?.seededRecipeId) return;
  if (slot.primary && collisionMode === null) {
    setCollisionOpen(true);
    return;
  }

  upsertSlot.mutate({
    weekStartDate: slot.weekStartDate,
    dayIndex: slot.dayIndex,
    mealType: slot.mealType,
    primary: {
      sourceType: "recipe",
      recipeId: slot.seededRecipeId,
      title: null,
      imageUrl: null,
      note: null,
    },
    extras: [],
    note: null,
    collisionMode,
  });
}

<AlertDialog open={collisionOpen} onOpenChange={setCollisionOpen}>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>That slot already has a meal</AlertDialogTitle>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel>Cancel</AlertDialogCancel>
      <AlertDialogAction onClick={() => submitSeededRecipe("add_as_extra")}>
        Add as extra
      </AlertDialogAction>
      <AlertDialogAction onClick={() => submitSeededRecipe("replace_primary")}>
        Replace primary
      </AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

- [ ] **Step 4: Run the unit tests to verify mobile board rendering and composer behavior pass**

Run:

```bash
cd /Users/joe.bor/code/family-hub/frontend
npm test -- --run src/components/meals-view.test.tsx src/components/meals/meal-composer-sheet.test.tsx
```

Expected: PASS for empty-slot affordances, recipe suggestions, quick meals, Add-to-Meals occupied-slot prompting, and week navigation.

- [ ] **Step 5: Commit**

```bash
cd /Users/joe.bor/code/family-hub/frontend
git add src/components/meals/week-header.tsx \
  src/components/meals/meal-day-card.tsx \
  src/components/meals/meal-slot-card.tsx \
  src/components/meals/meal-composer-sheet.tsx \
  src/components/meals/recipe-match-list.tsx \
  src/components/meals-view.tsx \
  src/components/meals/meal-composer-sheet.test.tsx \
  src/components/meals-view.test.tsx
git commit -m "feat(meals): add mobile-first meals board"
```

## Task 5: Add Larger-Screen Grid, Slot Editor, Move/Duplicate, And Collision Prompts

**Files:**
- Create: `frontend/src/components/meals/meal-grid.tsx`
- Create: `frontend/src/components/meals/meal-editor-sheet.tsx`
- Modify: `frontend/src/components/meals-view.tsx`
- Modify: `frontend/src/components/meals/meal-slot-card.tsx`
- Modify: `frontend/src/components/meals-view.test.tsx`
- Create: `frontend/e2e/mobile-meals.spec.ts`

- [ ] **Step 1: Write the failing UI tests for tablet grid rendering, duplicate, and collision prompts**

```tsx
it("renders a weekly grid on large screens", async () => {
  mockIsMobile(false);
  render(<MealsView />);
  expect(await screen.findByRole("grid", { name: /weekly meals/i })).toBeInTheDocument();
});

it("prompts when moving into an occupied slot", async () => {
  render(<MealEditorSheet open plannedSlot={occupiedSlot} onOpenChange={vi.fn()} />);

  await user.click(screen.getByRole("button", { name: /move meal/i }));
  expect(await screen.findByText("Replace primary")).toBeInTheDocument();
  expect(screen.getByText("Add as extra")).toBeInTheDocument();
});

it("opens the real recipe detail from a recipe-backed planned meal", async () => {
  render(<MealEditorSheet open plannedSlot={recipeBackedSlot} onOpenChange={vi.fn()} />);

  await user.click(screen.getByRole("button", { name: /view recipe/i }));
  expect(await screen.findByText("Ingredients")).toBeInTheDocument();
  expect(screen.getByText("Instructions")).toBeInTheDocument();
});
```

- [ ] **Step 2: Run the focused tests to verify larger-screen and collision behavior are still missing**

Run:

```bash
cd /Users/joe.bor/code/family-hub/frontend
npm test -- --run src/components/meals-view.test.tsx
```

Expected: FAIL with no weekly grid, duplicate flow, or collision prompt.

- [ ] **Step 3: Add the weekly grid, editor actions, and collision resolution UX**

```tsx
{!isMobile && board.data?.data && (
  <MealGrid
    board={board.data.data}
    readOnly={isReadOnlyWeek}
    onSelectSlot={setSelectedSlot}
  />
)}
```

```tsx
<AlertDialog open={collisionOpen} onOpenChange={setCollisionOpen}>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>That slot already has a meal</AlertDialogTitle>
      <AlertDialogDescription>
        Choose whether to replace the primary meal or add this meal as an extra.
      </AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel>Cancel</AlertDialogCancel>
      <AlertDialogAction onClick={() => resolveCollision("add_as_extra")}>
        Add as extra
      </AlertDialogAction>
      <AlertDialogAction onClick={() => resolveCollision("replace_primary")}>
        Replace primary
      </AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

```tsx
<Button type="button" variant="outline" onClick={handleDuplicate}>
  Duplicate meal
</Button>
<Button type="button" variant="outline" onClick={handleMove}>
  Move meal
</Button>
<Button type="button" variant="destructive" onClick={handleRemove}>
  Remove meal
</Button>
<Button type="button" variant="ghost" onClick={() => setRecipeDetailOpen(true)}>
  View recipe
</Button>

{plannedSlot.primary?.recipeId && (
  <MobileSheet open={recipeDetailOpen} onOpenChange={setRecipeDetailOpen} title="Recipe">
    <RecipeDetailView
      recipeId={plannedSlot.primary.recipeId}
      onBack={() => setRecipeDetailOpen(false)}
    />
  </MobileSheet>
)}
```

- [ ] **Step 4: Run unit and E2E verification for move/duplicate/collision behavior**

Run:

```bash
cd /Users/joe.bor/code/family-hub/frontend
npm test -- --run src/components/meals-view.test.tsx
npm run test:e2e -- e2e/mobile-meals.spec.ts
```

Expected: PASS for large-screen grid rendering, duplicate flow, collision resolution, and the recipes-to-meals handoff on mobile.

- [ ] **Step 5: Commit**

```bash
cd /Users/joe.bor/code/family-hub/frontend
git add src/components/meals/meal-grid.tsx \
  src/components/meals/meal-editor-sheet.tsx \
  src/components/meals-view.tsx \
  src/components/meals/meal-slot-card.tsx \
  src/components/meals-view.test.tsx \
  e2e/mobile-meals.spec.ts
git commit -m "feat(meals): add grid and slot management"
```

## Spec Coverage Checklist

- Week-by-week planning board: Tasks 1, 3, 4, 5
- Equal breakfast/lunch/dinner treatment: Tasks 1 and 4
- Past weeks review-only in FE: Tasks 3 and 4
- Quick meals plus recipe suggestions: Tasks 3 and 4
- `Create recipe from this` and return-to-planning handoff: Tasks 0 and 4
- Board-level recipe snapshots at placement time: Tasks 1 and 2
- Explicit move, duplicate, and collision handling: Tasks 2 and 5
- Recipe-backed meal navigation into real recipe detail: Task 5
- Recipes handoff consumption: Tasks 0, 4, and 5

## Risks To Watch During Execution

- The “Add as extra” collision rule must be implemented explicitly; do not leave source-slot extras ambiguous.
- Remove the placeholder sample data and `meals-store` completely so the module does not accidentally drift into split state ownership.
- Keep mobile interactions reliable even if desktop/tablet supports more literal drag behavior later; action-based fallback must remain trustworthy.
