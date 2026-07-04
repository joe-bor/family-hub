# Meals Recipe Ingredients To Grocery List Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an **Add ingredients** action to the visible, editable Meals week that gathers verbatim ingredients from recipe-backed planned meals, lets a parent review/edit/select rows, and appends the selected rows to a chosen grocery list in one confirmed, transactional write.

**Architecture:** The backend adds one **generic** bulk list-item append endpoint (`POST /api/lists/{listId}/items/bulk`) that reuses the existing single-item create/validation path (`ListService.createItem`) inside one transaction and returns the created items in request order — it has no meal/recipe/grocery concepts. The frontend layers a Meals action and review sheet over the existing `MealsView`: it reads the visible `MealBoard` from cache, fetches `RecipeDetail` per distinct `recipeId` for verbatim ingredient rows, groups rows by meal, routes quick/no-ingredient meals into a "no recipe ingredients" section, picks or creates a grocery list, and calls the released bulk endpoint once.

**Tech Stack:** Java 21, Spring Boot 4, JPA/Hibernate, PostgreSQL/Flyway, Maven, Lombok; React 19, TypeScript, TanStack Query v5, Zod, shadcn/Radix/Vaul, MSW, Vitest/Testing Library, Playwright.

**Spec:** `docs/superpowers/specs/2026-07-04-meals-recipe-ingredients-to-grocery-list.md`

**Story:** `docs/product/backlog/module-foundations/meals-recipe-ingredients-to-grocery-list.md`

---

## Delivery and repository boundaries

- Execute Tasks 1–2 in `backend/family-hub-api/`. Run Maven there and commit there.
- Merge the backend PR, merge the generated backend release PR, and confirm the published backend semver **before** starting released-contract frontend E2E.
- Execute Tasks 3–7 in `frontend/`. MSW-backed unit/component work (Tasks 3–6) can start before the backend release, but live E2E (Task 7) must consume the published backend release.
- Do not write production code from the root workspace. This root plan is the contract for the delivery-repo Issues.
- Do not open execution Issues until this root plan has passed spec-to-plan review.
- Delivery Issues must link Story, Spec, and Plan at the top and copy the non-negotiable execution contract below.

## Implementation decision: a generic bulk endpoint is required

The spec recommends a backend bulk append instead of N single-item writes. This plan requires it.

Reasoning:

- A week of recipe-backed dinners produces many ingredient rows; N independent `POST /items` calls can partially succeed, which the spec forbids (D5 "all-or-nothing").
- A single transaction validates every item first, then inserts all or nothing, and returns the created items in request order for a clean cache update.
- The endpoint stays **generic**: it accepts `{ text, categoryId? }` for any list kind and knows nothing about meals, recipes, or groceries (D6). "Groceries" is a frontend targeting choice.
- It reuses the existing `ListService.createItem` validation and locking exactly, so category/family/kind rules and the `(family, kind)` scope-lock ordering are identical to today.

The bulk endpoint must not replace the existing single-item `POST /api/lists/{listId}/items`; that endpoint and all other Lists behavior stay unchanged.

## Execution contract

- The **Add ingredients** action appears only on editable current/future Meals weeks (`!isPastWeek(visibleWeekStartDate)`) **and** only when the visible board has ≥1 recipe-backed entry (`sourceType === "recipe"` with non-null `recipeId`) in any slot `primary` or `extras[]`. It never appears on past weeks.
- Candidate rows derive only from the visible week. Each recipe row is one verbatim string from `RecipeDetail.ingredients[]` — no parsing, splitting, quantity extraction, normalization, or de-duplication.
- Quick meals, recipe meals with empty `ingredients[]`, and recipe meals whose recipe was deleted (detail 404) appear in a "no recipe ingredients" section with manual add rows; nothing is auto-generated for them.
- Rows group by meal (day/slot + snapshot title). Recipe rows default to selected; each row is editable, removable, and selectable before any write.
- The target is a grocery list only: default to the single grocery list when exactly one exists, choose among several, or create one via `POST /api/lists` `{ name, kind: "grocery" }`. Non-grocery lists are never offered.
- Appending is one confirmed action → `POST /api/lists/{listId}/items/bulk` with the selected rows (uncategorized in v1). Success updates cache and offers **View list**.
- Offline disables **Add to list** and **Create grocery list** with honest copy; reads still work from cache. A failed or partial append is never shown as success; a created-then-failed-append reports failure and offers retry.
- Backend endpoint is generic, family-scoped, bounded (`MAX_BULK_ITEMS = 100`, a named constant enforced at the request DTO and re-checked by a service-level guard), validates optional categories against family + list kind by reusing existing item-create validation, returns created items **in the bulk response** in request order, and rolls back entirely on any failure. Persisted read-back order is deterministic (`@OrderBy("createdAt ASC, id ASC")`) but not guaranteed to equal request order for same-instant items. No Flyway migration.
- Backend JSON stays in the existing `ApiResponse<T>` envelope; wire values already lowercase.

## Execution issue mapping

### Backend issue body

```markdown
Story: https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/module-foundations/meals-recipe-ingredients-to-grocery-list.md
Spec: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/specs/2026-07-04-meals-recipe-ingredients-to-grocery-list.md
Plan: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-07-04-meals-recipe-ingredients-to-grocery-list.md

Execution contract:
- Add POST /api/lists/{listId}/items/bulk: a generic bulk checklist-item append. No meal/recipe/grocery concepts in the Lists API.
- Body { items: [{ text, categoryId? }] }; items non-empty and bounded to MAX_BULK_ITEMS = 100; each item reuses CreateListItemRequest validation.
- Authenticated family scope: the target list must belong to the family (404 otherwise, do not reveal cross-family).
- Optional categoryId must belong to the same family and match the list kind, reusing ListService.createItem's resolveCategory + (family, kind) scope-lock ordering.
- One transaction: validate all items first, then insert all in request order appended after existing items; any failure rolls back the whole batch.
- Response 201 with ApiResponse<List<ListItemResponse>> containing the created items in request order.
- Existing single-item POST /items, move/update/delete/clear, and list behavior must not regress.
- Tests cover success + ordering, empty payload, over-max and boundary, wrong-family list, wrong-kind category, missing category, transactional rollback, and a generic to-do/general list append proving no grocery coupling.
```

### Frontend issue body

```markdown
Story: https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/module-foundations/meals-recipe-ingredients-to-grocery-list.md
Spec: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/specs/2026-07-04-meals-recipe-ingredients-to-grocery-list.md
Plan: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-07-04-meals-recipe-ingredients-to-grocery-list.md

Execution contract:
- Add an Add ingredients action to editable current/future Meals weeks only, shown only when the visible board has >=1 recipe-backed entry.
- Derive rows from the visible week; fetch RecipeDetail per distinct recipeId; each ingredients[] string is one verbatim row grouped by meal.
- Quick meals + empty-ingredient/deleted recipes go to a "no recipe ingredients" section with manual add rows; never auto-generate ingredients.
- Review sheet: per-row edit/remove/select, recipe rows selected by default, duplicates preserved (no de-dupe).
- Grocery-list picker: default the single grocery list, choose among several, or create one (POST /api/lists { name, kind: "grocery" }); non-grocery lists never offered.
- Append selected rows once via the released POST /api/lists/{listId}/items/bulk (uncategorized); success updates ListDetail/ListSummary cache and offers View list.
- Offline disables Add to list and Create grocery list with honest copy; reads still work. Failure is never shown as success; created-then-failed-append reports failure and offers retry.
- Released-contract E2E names the tested backend semver.
```

## Verified codebase facts

### Backend (`backend/family-hub-api/`)

- `ListController` (`.../controller/ListController.java`) maps `/api/lists`, injects `ListService`, uses `@AuthenticationPrincipal Family family`, and builds `new ApiResponse<>(data, message)`. Item create is `@PostMapping("/{id}/items")` → `listService.createItem(id, request, family)` returning `ListItemResponse`.
- `ListService.createItem(UUID listId, CreateListItemRequest request, Family family)` is `@Transactional`. When `categoryId != null` it reads `sharedListRepository.findKindByFamilyAndId(family, listId)` (→ `ResourceNotFoundException("List", listId)`) then `lockScope(family, kind)`; loads `getListOrThrow(listId, family)`; builds a `SharedListItem` (`setList`, `setText(request.text().trim())`, `setCompleted(false)`, `setCompletedAt(null)`, `setCategory(resolveCategory(list, request.categoryId()))`), adds it to `list.getItems()`, then `sharedListRepository.saveAndFlush(list)` and returns `ListMapper.toItemDto(...)`.
- `resolveCategory(SharedList list, UUID categoryId)`: null → null; else `listCategoryRepository.findByFamilyAndId(list.getFamily(), categoryId)` (→ `ResourceNotFoundException("List Category", id)`, mapped to 404); wrong kind → `BadRequestException("Category kind does not match list kind.")` (mapped to 400).
- `CreateListItemRequest(@NotBlank @Size(max = 100) String text, UUID categoryId)`.
- `ListItemResponse(UUID id, String text, boolean completed, LocalDateTime completedAt, UUID categoryId, LocalDateTime createdAt, LocalDateTime updatedAt)`.
- `ListMapper.toItemDto(SharedListItem)` produces `ListItemResponse`. `ResourceNotFoundException`→404, `BadRequestException`→400, `ConflictException`→409 via `GlobalExceptionHandler`.
- Test infrastructure exists: `ListServiceTest`, `ListControllerTest`, `ListIntegrationTest`; controller tests use `@WithMockFamily` + MockMvc + jsonPath (see `MealControllerTest`).

### Frontend (`frontend/`)

- Types: `MealBoard { weekStartDate, days: MealDay[] }`, `MealDay { date, dayIndex, slots: MealSlot[] }`, `MealSlot { id, weekStartDate, dayIndex, mealType, primary: MealSlotEntry | null, extras: MealSlotEntry[], note }`, `MealSlotEntry { id, role, sourceType: "recipe" | "quick", recipeId: string | null, title, imageUrl, note }` (`src/lib/types/meals.ts`).
- `RecipeDetail { ...RecipeSummary, ingredients: string[], instructions, note, sourceUrl }`; ingredients live only on detail (`src/lib/types/recipes.ts`).
- Lists: `ListSummary { id, name, kind: "grocery" | "to-do" | "general", totalItems, completedItems }`, `ListItem { id, text, completed, completedAt, categoryId, createdAt, updatedAt }`, `CreateListRequest { name, kind }`, `CreateListItemRequest { text, categoryId? }` (`src/lib/types/lists.ts`).
- Services: `recipesService.getRecipe(id)` → `GET /recipes/${id}`; `listsService.getLists()` → `GET /lists`; `listsService.createList(req)` → `POST /lists`; item create → `POST /lists/${listId}/items` (`src/api/services/*.service.ts`).
- Hooks: `useRecipe(id)` (`use-recipes.ts`); `useLists()`, `useList(id)`, `useCreateList()`, `useCreateListItem(listId)` (`use-lists.ts`).
- Query keys (exact names): `listsKeys.hub()` (summaries) and `listsKeys.detail(id)` (`use-lists.ts`); `recipesKeys.detail(id)` (`use-recipes.ts`). `useCreateListItem` calls `assertOnlineForWrite()` (from `@/lib/offline/read-only-guard`) at the start of `onMutate`, mutates `listsKeys.detail(listId)` via the `updateDetailItems` helper, and invalidates `listsKeys.hub()`. `useBulkCreateListItems` reuses all of these.
- Offline: Lists write hooks call `assertOnlineForWrite()` (throws when offline). `ApiException` (`@/api/client`) exposes `.status` for HTTP-code checks (used to map recipe-detail 404 → "missing"). For a variable-length set of recipe fetches, use `useQueries` (never `useRecipe` in a loop — Rules of Hooks).
- `MealsView` (`src/components/meals-view.tsx`) owns the visible week; `readOnly = isPastWeek(visibleWeekStartDate)` (line ~163) is the editable gate; the `Fill empty slots` action button lives near line ~482. Reuse both.
- Week starts Sunday (cross-stack contract): `dayIndex` 0 = Sunday … 6 = Saturday.

## File structure

### Backend — create

- `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/BulkCreateListItemsRequest.java` — the bulk request wrapping a bounded list of `CreateListItemRequest`.

### Backend — modify

- `.../service/ListService.java` — add transactional `createItemsBulk`.
- `.../controller/ListController.java` — expose `POST /{id}/items/bulk`.
- `.../service/ListServiceTest.java` — bulk service coverage.
- `.../controller/ListControllerTest.java` — wire contract coverage.
- `.../integration/ListIntegrationTest.java` — persistence, ordering, rollback, isolation coverage.

### Frontend — create

- `frontend/src/components/meals/meal-ingredient-extraction.ts` — pure helpers: collect planned entries, distinct recipe ids, `RecipeResolution` model, build review groups (recipe / no-ingredient / error), convert selected rows to a bulk request.
- `frontend/src/components/meals/meal-ingredient-extraction.test.ts` — pure behavior tests.
- `frontend/src/components/meals/use-recipe-details.ts` — `useRecipeDetails(recipeIds)` via `useQueries`, mapping each recipe id to a `RecipeResolution` (loaded / missing (404) / error).
- `frontend/src/components/meals/use-recipe-details.test.ts` — hook resolution-mapping tests.
- `frontend/src/components/meals/add-ingredients-sheet.tsx` — review sheet: groups, edit/remove/select, no-ingredient section, retryable error rows, list picker/create, append + success/View list, offline handling.
- `frontend/src/components/meals/add-ingredients-sheet.test.tsx` — component-level tests.

### Frontend — modify

- `frontend/src/lib/types/lists.ts` — `BulkCreateListItemsRequest`, `ListItemsApiResponse`.
- `frontend/src/lib/validations/lists.ts` and `.test.ts` — bulk schema.
- `frontend/src/api/services/lists.service.ts` and `.test.ts` — `createItemsBulk`.
- `frontend/src/api/hooks/use-lists.ts` and `use-lists.test.tsx` — `useBulkCreateListItems`.
- `frontend/src/api/hooks/index.ts` and `frontend/src/api/index.ts` — export `useBulkCreateListItems`.
- `frontend/src/test/mocks/handlers.ts` — MSW `POST /lists/:id/items/bulk`.
- `frontend/src/components/meals-view.tsx` and `meals-view.test.tsx` — action entry + sheet orchestration + regression coverage.
- `frontend/e2e/mobile-meals.spec.ts` (or a new `e2e/meals-ingredients.spec.ts`) — released-contract E2E.

---

## Task 1: Backend generic bulk list-item append service

**Files:**
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/BulkCreateListItemsRequest.java`
- Modify: `.../service/ListService.java`
- Modify: `.../service/ListServiceTest.java`

- [ ] **Step 1: Write failing service tests**

Append to `ListServiceTest`, reusing its existing fixtures and stubs (`LIST_ID`, the `groceryList` fixture with a mutable `getItems()`, `family`, `produceCategory` (GROCERY), `groceryScope`, and the `sharedListRepository.findDetailByFamilyAndId` / `findKindByFamilyAndId` / `scopeRepository.lockByFamilyAndKind` / `listCategoryRepository.findByFamilyAndId` mocks used by the existing `createItem_*` tests):

```java
@Test
void createItemsBulk_appendsAllItemsInRequestOrder() {
    when(sharedListRepository.findDetailByFamilyAndId(family, LIST_ID)).thenReturn(Optional.of(groceryList));
    when(sharedListRepository.saveAndFlush(any(SharedList.class))).thenReturn(groceryList);

    List<ListItemResponse> created = listService.createItemsBulk(
            LIST_ID,
            new BulkCreateListItemsRequest(List.of(
                    new CreateListItemRequest("2 chicken breasts", null),
                    new CreateListItemRequest("1 tbsp olive oil", null),
                    new CreateListItemRequest("2 cups broccoli", null)
            )),
            family
    );

    assertThat(created).extracting(ListItemResponse::text)
            .containsExactly("2 chicken breasts", "1 tbsp olive oil", "2 cups broccoli");
}

@Test
void createItemsBulk_acceptsMaxItems() {
    when(sharedListRepository.findDetailByFamilyAndId(family, LIST_ID)).thenReturn(Optional.of(groceryList));
    when(sharedListRepository.saveAndFlush(any(SharedList.class))).thenReturn(groceryList);
    List<CreateListItemRequest> items = IntStream.range(0, 100)
            .mapToObj(i -> new CreateListItemRequest("item " + i, null))
            .toList();

    List<ListItemResponse> created = listService.createItemsBulk(
            LIST_ID, new BulkCreateListItemsRequest(items), family);

    assertThat(created).hasSize(100);
    assertThat(created.get(0).text()).isEqualTo("item 0");
    assertThat(created.get(99).text()).isEqualTo("item 99");
}

@Test
void createItemsBulk_listNotFound_throws404() {
    when(sharedListRepository.findDetailByFamilyAndId(family, LIST_ID)).thenReturn(Optional.empty());

    assertThatThrownBy(() -> listService.createItemsBulk(
            LIST_ID,
            new BulkCreateListItemsRequest(List.of(new CreateListItemRequest("milk", null))),
            family
    )).isInstanceOf(ResourceNotFoundException.class);
}

@Test
void createItemsBulk_assignsValidSameKindCategory() {
    when(sharedListRepository.findKindByFamilyAndId(family, LIST_ID)).thenReturn(Optional.of(ListKind.GROCERY));
    when(scopeRepository.lockByFamilyAndKind(family, ListKind.GROCERY)).thenReturn(Optional.of(groceryScope));
    when(sharedListRepository.findDetailByFamilyAndId(family, LIST_ID)).thenReturn(Optional.of(groceryList));
    when(listCategoryRepository.findByFamilyAndId(family, produceCategory.getId()))
            .thenReturn(Optional.of(produceCategory));
    when(sharedListRepository.saveAndFlush(any(SharedList.class))).thenReturn(groceryList);

    List<ListItemResponse> created = listService.createItemsBulk(
            LIST_ID,
            new BulkCreateListItemsRequest(List.of(
                    new CreateListItemRequest("2 chicken breasts", produceCategory.getId())
            )),
            family
    );

    assertThat(created).hasSize(1);
    assertThat(created.get(0).categoryId()).isEqualTo(produceCategory.getId());
}

@Test
void createItemsBulk_missingCategory_throws404AndAppendsNothing() {
    UUID missingCategoryId = UUID.randomUUID();
    int before = groceryList.getItems().size();
    when(sharedListRepository.findKindByFamilyAndId(family, LIST_ID)).thenReturn(Optional.of(ListKind.GROCERY));
    when(scopeRepository.lockByFamilyAndKind(family, ListKind.GROCERY)).thenReturn(Optional.of(groceryScope));
    when(sharedListRepository.findDetailByFamilyAndId(family, LIST_ID)).thenReturn(Optional.of(groceryList));
    when(listCategoryRepository.findByFamilyAndId(family, missingCategoryId)).thenReturn(Optional.empty());

    assertThatThrownBy(() -> listService.createItemsBulk(
            LIST_ID,
            new BulkCreateListItemsRequest(List.of(new CreateListItemRequest("mystery", missingCategoryId))),
            family
    )).isInstanceOf(ResourceNotFoundException.class);

    assertThat(groceryList.getItems()).hasSize(before);
}

// Proves prevalidation: the wrong-kind second item is rejected DURING category
// resolution, before any row is appended and before saveAndFlush is called.
@Test
void createItemsBulk_wrongKindCategory_throwsAndAppendsNothing() {
    ListCategory todoCategory = createListCategory(family, ListKind.TODO, "Urgent", 0);
    int before = groceryList.getItems().size();
    when(sharedListRepository.findKindByFamilyAndId(family, LIST_ID)).thenReturn(Optional.of(ListKind.GROCERY));
    when(scopeRepository.lockByFamilyAndKind(family, ListKind.GROCERY)).thenReturn(Optional.of(groceryScope));
    when(sharedListRepository.findDetailByFamilyAndId(family, LIST_ID)).thenReturn(Optional.of(groceryList));
    when(listCategoryRepository.findByFamilyAndId(family, produceCategory.getId()))
            .thenReturn(Optional.of(produceCategory));
    when(listCategoryRepository.findByFamilyAndId(family, todoCategory.getId()))
            .thenReturn(Optional.of(todoCategory));

    assertThatThrownBy(() -> listService.createItemsBulk(
            LIST_ID,
            new BulkCreateListItemsRequest(List.of(
                    new CreateListItemRequest("ok row", produceCategory.getId()),
                    new CreateListItemRequest("bad row", todoCategory.getId()) // wrong-kind category
            )),
            family
    )).isInstanceOf(BadRequestException.class);

    assertThat(groceryList.getItems()).hasSize(before);
    verify(sharedListRepository, org.mockito.Mockito.never()).saveAndFlush(any(SharedList.class));
}
```

Add imports (some may already be present from existing tests):

```java
import com.familyhub.demo.dto.BulkCreateListItemsRequest;
import com.familyhub.demo.exception.BadRequestException;
import com.familyhub.demo.exception.ResourceNotFoundException;
import java.util.stream.IntStream;
```

- [ ] **Step 2: Run the focused service test and verify it fails**

```bash
cd backend/family-hub-api
./mvnw -Dtest=ListServiceTest test
```

Expected: FAIL — `BulkCreateListItemsRequest` and `ListService.createItemsBulk` do not exist.

- [ ] **Step 3: Add the bulk request DTO**

Create `BulkCreateListItemsRequest.java`:

```java
package com.familyhub.demo.dto;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.Size;

import java.util.List;

public record BulkCreateListItemsRequest(
        @Valid
        @NotEmpty(message = "At least one item is required")
        @Size(max = 100, message = "A bulk append may contain at most 100 items")
        List<CreateListItemRequest> items
) {
}
```

- [ ] **Step 4: Add the service method (reuse the single-item path)**

Add to `ListService`, mirroring `createItem`'s kind-projection → scope-lock → load ordering, but for the whole batch in one transaction:

```java
@Transactional
public List<ListItemResponse> createItemsBulk(UUID listId, BulkCreateListItemsRequest request, Family family) {
    // Same scope-lock ordering as createItem: if any item assigns a category, read the immutable
    // kind projection first, then lock the (family, kind) scope, then load the aggregate.
    boolean anyCategory = request.items().stream().anyMatch(item -> item.categoryId() != null);
    if (anyCategory) {
        ListKind kind = sharedListRepository.findKindByFamilyAndId(family, listId)
                .orElseThrow(() -> new ResourceNotFoundException("List", listId));
        lockScope(family, kind);
    }

    SharedList list = getListOrThrow(listId, family);

    // Prevalidation contract: resolve and validate EVERY category BEFORE mutating the aggregate,
    // so a single bad item never appends a partial row. resolveCategory runs queries, so appending
    // items first could let Hibernate autoflush persist a partial batch before a later item throws.
    // `.toList()` forces resolution eagerly in request order; the first invalid item throws here,
    // before any item is added to the collection and before saveAndFlush runs.
    List<ListCategory> resolvedCategories = request.items().stream()
            .map(item -> resolveCategory(list, item.categoryId()))
            .toList();

    List<SharedListItem> created = new ArrayList<>(request.items().size());
    for (int i = 0; i < request.items().size(); i++) {
        SharedListItem item = new SharedListItem();
        item.setList(list);
        item.setText(request.items().get(i).text().trim());
        item.setCompleted(false);
        item.setCompletedAt(null);
        item.setCategory(resolvedCategories.get(i));
        list.getItems().add(item);
        created.add(item);
    }

    sharedListRepository.saveAndFlush(list); // one transaction; rolls back entirely on any failure
    return created.stream().map(ListMapper::toItemDto).toList(); // request order preserved
}
```

Add imports if missing:

```java
import com.familyhub.demo.dto.BulkCreateListItemsRequest;
import com.familyhub.demo.model.ListCategory;
import java.util.ArrayList;
```

- [ ] **Step 5: Run the focused service test and verify it passes**

```bash
cd backend/family-hub-api
./mvnw -Dtest=ListServiceTest test
```

Expected: PASS for existing list-service tests plus the six `createItemsBulk_*` tests.

- [ ] **Step 6: Commit**

```bash
cd backend/family-hub-api
git add src/main/java/com/familyhub/demo/dto/BulkCreateListItemsRequest.java \
  src/main/java/com/familyhub/demo/service/ListService.java \
  src/test/java/com/familyhub/demo/service/ListServiceTest.java
git commit -m "feat(lists): add transactional bulk item append service"
```

## Task 2: Expose and verify the bulk-append API

**Files:**
- Modify: `.../controller/ListController.java`
- Modify: `.../controller/ListControllerTest.java`
- Modify: `.../integration/ListIntegrationTest.java`

- [ ] **Step 1: Write failing controller + integration tests**

Add to `ListControllerTest` (mirror existing `@WithMockFamily` MockMvc tests):

```java
@Test
@WithMockFamily
void createItemsBulk_returnsCreatedItemsInOrder() throws Exception {
    given(listService.createItemsBulk(eq(LIST_ID), any(), any(Family.class)))
            .willReturn(List.of(
                    sampleItem("2 chicken breasts"),
                    sampleItem("1 tbsp olive oil")
            ));

    mockMvc.perform(post("/api/lists/{id}/items/bulk", LIST_ID)
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            { "items": [ { "text": "2 chicken breasts" }, { "text": "1 tbsp olive oil" } ] }
                            """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.data[0].text").value("2 chicken breasts"))
            .andExpect(jsonPath("$.data[1].text").value("1 tbsp olive oil"));
}

@Test
@WithMockFamily
void createItemsBulk_rejectsEmptyItems() throws Exception {
    mockMvc.perform(post("/api/lists/{id}/items/bulk", LIST_ID)
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            { "items": [] }
                            """))
            .andExpect(status().isBadRequest());
}

@Test
@WithMockFamily
void createItemsBulk_rejectsOverMaxItems() throws Exception {
    String items = IntStream.range(0, 101)
            .mapToObj(i -> "{ \"text\": \"item " + i + "\" }")
            .collect(Collectors.joining(","));

    mockMvc.perform(post("/api/lists/{id}/items/bulk", LIST_ID)
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{ \"items\": [" + items + "] }"))
            .andExpect(status().isBadRequest());
}

@Test
@WithMockFamily
void createItemsBulk_rejectsBlankOrOverlongItemText() throws Exception {
    mockMvc.perform(post("/api/lists/{id}/items/bulk", LIST_ID)
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{ \"items\": [ { \"text\": \"   \" } ] }"))
            .andExpect(status().isBadRequest());

    String overlong = "x".repeat(101);
    mockMvc.perform(post("/api/lists/{id}/items/bulk", LIST_ID)
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{ \"items\": [ { \"text\": \"" + overlong + "\" } ] }"))
            .andExpect(status().isBadRequest());
}
```

(Add `import java.util.stream.IntStream;` and `import java.util.stream.Collectors;` to `ListControllerTest` if not already present, and reuse its existing `LIST_ID` constant and `sampleItem(...)` helper.)

Add to `ListIntegrationTest`, after the existing token/list setup, a full-stack sequence proving ordering, generic (non-grocery) support, and rollback:

```java
// Append two items to a grocery list; assert order and persistence.
mockMvc.perform(post("/api/lists/{id}/items/bulk", groceryListId)
                .header("Authorization", "Bearer " + token)
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        { "items": [ { "text": "2 chicken breasts" }, { "text": "1 tbsp olive oil" } ] }
                        """))
        .andExpect(status().isCreated())
        .andExpect(jsonPath("$.data[0].text").value("2 chicken breasts"))
        .andExpect(jsonPath("$.data[1].text").value("1 tbsp olive oil"));

mockMvc.perform(get("/api/lists/{id}", groceryListId)
                .header("Authorization", "Bearer " + token))
        .andExpect(status().isOk())
        // Read-back order is deterministic (@OrderBy "createdAt ASC, id ASC") but not
        // guaranteed to equal request order — same-batch items can tie on createdAt.
        // Assert membership, not position. (Request order is asserted on the POST response.)
        .andExpect(jsonPath("$.data.items.length()").value(2))
        .andExpect(jsonPath("$.data.items[*].text",
                containsInAnyOrder("2 chicken breasts", "1 tbsp olive oil")));

// Generic proof: the same endpoint appends to a to-do list (no grocery coupling).
mockMvc.perform(post("/api/lists/{id}/items/bulk", todoListId)
                .header("Authorization", "Bearer " + token)
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        { "items": [ { "text": "call plumber" } ] }
                        """))
        .andExpect(status().isCreated());

// Rollback proof: a wrong-kind category in the second item writes nothing.
mockMvc.perform(post("/api/lists/{id}/items/bulk", groceryListId)
                .header("Authorization", "Bearer " + token)
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        { "items": [ { "text": "should not persist" }, { "text": "bad", "categoryId": "%s" } ] }
                        """.formatted(todoCategoryId)))
        .andExpect(status().isBadRequest());

mockMvc.perform(get("/api/lists/{id}", groceryListId)
                .header("Authorization", "Bearer " + token))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.data.items.length()").value(2)); // still only the first two
```

- [ ] **Step 2: Run focused tests and verify they fail**

```bash
cd backend/family-hub-api
./mvnw -Dtest=ListControllerTest,ListIntegrationTest test
```

Expected: FAIL — `POST /api/lists/{id}/items/bulk` is not mapped.

- [ ] **Step 3: Add the controller endpoint**

Add to `ListController` (place next to `createItem`):

```java
@PostMapping("/{id}/items/bulk")
public ResponseEntity<ApiResponse<List<ListItemResponse>>> createItemsBulk(
        @PathVariable UUID id,
        @Valid @RequestBody BulkCreateListItemsRequest request,
        @AuthenticationPrincipal Family family
) {
    List<ListItemResponse> response = listService.createItemsBulk(id, request, family);
    return ResponseEntity.status(HttpStatus.CREATED)
            .body(new ApiResponse<>(response, "List items added successfully"));
}
```

Add import:

```java
import org.springframework.http.HttpStatus;
```

(`BulkCreateListItemsRequest` and `List` are already visible via `com.familyhub.demo.dto.*` and `java.util.List`.)

- [ ] **Step 4: Run backend verification for Lists**

```bash
cd backend/family-hub-api
./mvnw -Dtest=ListServiceTest,ListControllerTest,ListIntegrationTest test
```

Expected: PASS for service, controller, and integration list coverage.

- [ ] **Step 5: Commit**

```bash
cd backend/family-hub-api
git add src/main/java/com/familyhub/demo/controller/ListController.java \
  src/test/java/com/familyhub/demo/controller/ListControllerTest.java \
  src/test/java/com/familyhub/demo/integration/ListIntegrationTest.java
git commit -m "feat(lists): expose bulk item append endpoint"
```

> After merge, merge the release-please PR and note the published `family-hub-api` semver. Task 7 E2E consumes that release.

## Task 3: Frontend bulk-append contract, hook, and mock

**Files:**
- Modify: `frontend/src/lib/types/lists.ts`
- Modify: `frontend/src/lib/validations/lists.ts`, `frontend/src/lib/validations/lists.test.ts`
- Modify: `frontend/src/api/services/lists.service.ts`, `frontend/src/api/services/lists.service.test.ts`
- Modify: `frontend/src/api/hooks/use-lists.ts`, `frontend/src/api/hooks/use-lists.test.tsx`
- Modify: `frontend/src/api/hooks/index.ts`, `frontend/src/api/index.ts`
- Modify: `frontend/src/test/mocks/handlers.ts`

- [ ] **Step 1: Write failing service + hook tests**

In `lists.service.test.ts`:

```ts
describe("listsService.createItemsBulk", () => {
  it("posts selected rows to /lists/:id/items/bulk", async () => {
    const captured = { url: "", body: "" };
    server.use(
      http.post(`${API_BASE}/lists/list-1/items/bulk`, async ({ request }) => {
        captured.url = request.url;
        captured.body = await request.text();
        return HttpResponse.json({
          data: [
            { id: "i1", text: "2 eggs", completed: false, completedAt: null, categoryId: null, createdAt: "", updatedAt: "" },
          ],
          message: "List items added successfully",
        });
      }),
    );

    await listsService.createItemsBulk("list-1", { items: [{ text: "2 eggs" }] });

    expect(new URL(captured.url).pathname).toBe("/api/lists/list-1/items/bulk");
    expect(JSON.parse(captured.body)).toEqual({ items: [{ text: "2 eggs" }] });
  });
});
```

In `use-lists.test.tsx`, add two tests (use a dedicated `QueryClient` with `gcTime: Infinity`, mirroring the existing `useCreateListItem` tests):
- Seeds `listsKeys.detail("list-1")` with a `ListDetail` (2 items), calls `useBulkCreateListItems("list-1")` with two rows, and asserts the detail cache holds the original items **followed by** the two created items in order, and that `listsKeys.hub()` is invalidated (summaries refetch).
- Offline (force the read-only guard to throw, e.g. `navigator.onLine = false`): the mutation rejects and the detail cache is unchanged — mirroring the existing `useCreateListItem` offline-guard behavior.

- [ ] **Step 2: Run focused contract tests and verify they fail**

```bash
cd frontend
npm test -- --run src/api/services/lists.service.test.ts src/api/hooks/use-lists.test.tsx src/lib/validations/lists.test.ts
```

Expected: FAIL — bulk contracts do not exist.

- [ ] **Step 3: Add types**

Append to `src/lib/types/lists.ts`:

```ts
export interface BulkCreateListItemsRequest {
  items: CreateListItemRequest[];
}

export type ListItemsApiResponse = ApiResponse<ListItem[]>;
```

- [ ] **Step 4: Add the Zod schema**

Append to `src/lib/validations/lists.ts` (reuse the existing single-item text rule — trimmed, non-empty, max 100 — mirroring BE `CreateListItemRequest`):

```ts
export const bulkCreateListItemsSchema = z.object({
  items: z
    .array(
      z.object({
        text: z.string().trim().min(1).max(100),
        categoryId: z.string().uuid().nullable().optional(),
      }),
    )
    .min(1)
    .max(100),
});
```

Add tests: empty `items` fails; 101 items fails; a blank `text` fails; a valid single row passes.

- [ ] **Step 5: Add the service method**

Add to `lists.service.ts`:

```ts
createItemsBulk(
  listId: string,
  request: BulkCreateListItemsRequest,
): Promise<ListItemsApiResponse> {
  return httpClient.post<ListItemsApiResponse>(`/lists/${listId}/items/bulk`, request);
},
```

Import `BulkCreateListItemsRequest` and `ListItemsApiResponse` from `@/lib/types`.

- [ ] **Step 6: Add the hook (mirror `useCreateListItem`'s offline guard, keys, and cache convergence)**

Add to `use-lists.ts`. Reuse what `useCreateListItem` already uses in this file: `assertOnlineForWrite` (imported from `@/lib/offline/read-only-guard`), the `updateDetailItems` helper, and the `listsKeys.detail(id)` / `listsKeys.hub()` keys:

```ts
export function useBulkCreateListItems(listId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (request: BulkCreateListItemsRequest) => {
      // Read-only offline: reject before any write, matching the other Lists write hooks.
      assertOnlineForWrite();
      return listsService.createItemsBulk(listId, request);
    },
    onSuccess: (response) => {
      // Append the created items to the cached list detail in request order, via the same
      // `updateDetailItems` helper `useCreateListItem` uses.
      queryClient.setQueryData<ListDetailApiResponse>(
        listsKeys.detail(listId),
        (current) => updateDetailItems(current, (items) => [...items, ...response.data]),
      );
      // Summaries (totalItems) changed; reconcile detail + summaries with the server.
      queryClient.invalidateQueries({ queryKey: listsKeys.hub() });
      queryClient.invalidateQueries({ queryKey: listsKeys.detail(listId) });
    },
  });
}
```

Note: unlike single-item create there is no optimistic `onMutate` here — the append is a single confirmed action, so we update the cache from the authoritative response and invalidate. Import `BulkCreateListItemsRequest` from `@/lib/types`. Export `useBulkCreateListItems` from `src/api/hooks/index.ts` and `src/api/index.ts`.

- [ ] **Step 7: Add the MSW handler**

In `src/test/mocks/handlers.ts`, add near the other lists handlers a `POST ${API_BASE}/lists/:id/items/bulk` that reads `{ items }`, appends items to the in-memory list, and returns `ApiResponse<ListItem[]>` of the created items in order (reuse the existing mock list store the single-item handler uses).

- [ ] **Step 8: Run focused contract tests and verify they pass**

```bash
cd frontend
npm test -- --run src/api/services/lists.service.test.ts src/api/hooks/use-lists.test.tsx src/lib/validations/lists.test.ts
```

Expected: PASS.

- [ ] **Step 9: Commit**

```bash
cd frontend
git add src/lib/types/lists.ts src/lib/validations/lists.ts src/lib/validations/lists.test.ts \
  src/api/services/lists.service.ts src/api/services/lists.service.test.ts \
  src/api/hooks/use-lists.ts src/api/hooks/use-lists.test.tsx \
  src/api/hooks/index.ts src/api/index.ts src/test/mocks/handlers.ts
git commit -m "feat(lists): add bulk item append contract and hook"
```

## Task 4: Pure ingredient-extraction helpers

**Files:**
- Create: `frontend/src/components/meals/meal-ingredient-extraction.ts`
- Create: `frontend/src/components/meals/meal-ingredient-extraction.test.ts`

- [ ] **Step 1: Write failing pure tests**

Create `meal-ingredient-extraction.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type { MealBoard, RecipeDetail } from "@/lib/types";
import {
  buildReviewModel,
  distinctRecipeIds,
  hasRecipeBackedEntry,
  toBulkItemsRequest,
} from "./meal-ingredient-extraction";

function board(): MealBoard {
  // Wed(3) dinner recipe r1; Thu(4) dinner recipe r1 (same recipe); Fri(5) dinner quick.
  const empty = (dayIndex: number) => ({
    date: "",
    dayIndex,
    slots: (["breakfast", "lunch", "dinner"] as const).map((mealType) => ({
      id: null,
      weekStartDate: "2026-07-05",
      dayIndex,
      mealType,
      primary: null,
      extras: [],
      note: null,
    })),
  });
  const b: MealBoard = { weekStartDate: "2026-07-05", days: [0, 1, 2, 3, 4, 5, 6].map(empty) };
  b.days[3].slots[2].primary = { id: "e1", role: "primary", sourceType: "recipe", recipeId: "r1", title: "Sheet Pan Chicken", imageUrl: null, note: null };
  b.days[4].slots[2].primary = { id: "e2", role: "primary", sourceType: "recipe", recipeId: "r1", title: "Sheet Pan Chicken", imageUrl: null, note: null };
  b.days[5].slots[2].primary = { id: "e3", role: "primary", sourceType: "quick", recipeId: null, title: "Taco night", imageUrl: null, note: null };
  return b;
}

const recipeR1: RecipeDetail = {
  id: "r1", title: "Sheet Pan Chicken", imageUrl: null, favorite: false, tags: [], updatedAt: "",
  ingredients: ["2 chicken breasts", "1 tbsp olive oil"], instructions: [], note: null, sourceUrl: null,
};

describe("hasRecipeBackedEntry", () => {
  it("is true when any slot has a recipe-backed entry", () => {
    expect(hasRecipeBackedEntry(board())).toBe(true);
  });
  it("is false for a board with only quick/empty slots", () => {
    const b = board();
    b.days[3].slots[2].primary = null;
    b.days[4].slots[2].primary = null;
    expect(hasRecipeBackedEntry(b)).toBe(false);
  });
});

describe("distinctRecipeIds", () => {
  it("dedupes recipe fetch ids across meals", () => {
    expect(distinctRecipeIds(board())).toEqual(["r1"]);
  });
});

describe("buildReviewModel", () => {
  it("makes one verbatim row per ingredient, grouped by meal, and routes quick meals to the no-ingredient section", () => {
    const model = buildReviewModel(board(), { r1: { status: "loaded", detail: recipeR1 } });

    expect(model.recipeGroups).toHaveLength(2); // Wed + Thu, both from r1
    expect(model.recipeGroups[0].label).toBe("Wednesday · Dinner — Sheet Pan Chicken");
    expect(model.recipeGroups[0].rows.map((r) => r.text)).toEqual(["2 chicken breasts", "1 tbsp olive oil"]);
    expect(model.recipeGroups[0].rows.every((r) => r.selected)).toBe(true);

    expect(model.noIngredientGroups.map((g) => g.label)).toEqual(["Friday · Dinner — Taco night"]);
    expect(model.errorGroups).toHaveLength(0);
  });

  it("routes an empty-ingredient recipe and a 404 (deleted) recipe to the no-ingredient section", () => {
    const b = board();
    b.days[4].slots[2].primary = { id: "e2", role: "primary", sourceType: "recipe", recipeId: "r2", title: "Grandma's stew", imageUrl: null, note: null };
    const model = buildReviewModel(b, {
      r1: { status: "loaded", detail: { ...recipeR1, ingredients: [] } }, // no ingredients
      r2: { status: "missing" }, // 404 → deleted/unavailable
    });
    const labels = model.noIngredientGroups.map((g) => g.label);
    expect(labels).toContain("Wednesday · Dinner — Sheet Pan Chicken");
    expect(labels).toContain("Thursday · Dinner — Grandma's stew");
    expect(model.recipeGroups).toHaveLength(0);
    expect(model.errorGroups).toHaveLength(0);
  });

  it("routes a non-404 fetch error to a retryable error group, not the no-ingredient section", () => {
    const model = buildReviewModel(board(), { r1: { status: "error" } });

    expect(model.errorGroups.map((g) => g.label)).toEqual([
      "Wednesday · Dinner — Sheet Pan Chicken",
      "Thursday · Dinner — Sheet Pan Chicken",
    ]);
    expect(model.errorGroups.every((g) => g.recipeId === "r1")).toBe(true);
    expect(model.recipeGroups).toHaveLength(0);
    // The quick meal still lands in the no-ingredient section; the errored recipe does not.
    expect(model.noIngredientGroups.map((g) => g.label)).toEqual(["Friday · Dinner — Taco night"]);
  });
});

describe("toBulkItemsRequest", () => {
  it("sends only selected rows, verbatim, uncategorized, in order", () => {
    const model = buildReviewModel(board(), { r1: { status: "loaded", detail: recipeR1 } });
    model.recipeGroups[0].rows[1].selected = false; // deselect "1 tbsp olive oil" on Wed
    const request = toBulkItemsRequest(model);
    expect(request).toEqual({
      items: [
        { text: "2 chicken breasts" },
        { text: "2 chicken breasts" },
        { text: "1 tbsp olive oil" },
      ],
    });
  });
});
```

- [ ] **Step 2: Run the focused pure test and verify it fails**

```bash
cd frontend
npm test -- --run src/components/meals/meal-ingredient-extraction.test.ts
```

Expected: FAIL — `meal-ingredient-extraction.ts` does not exist.

- [ ] **Step 3: Implement the pure helpers**

Create `meal-ingredient-extraction.ts`:

```ts
import type {
  BulkCreateListItemsRequest,
  MealBoard,
  MealSlotEntry,
  RecipeDetail,
} from "@/lib/types";

const WEEKDAY_LABELS = ["Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"];
const MEAL_LABELS: Record<string, string> = { breakfast: "Breakfast", lunch: "Lunch", dinner: "Dinner" };

export interface PlannedEntryRef {
  dayIndex: number;
  mealType: string;
  entry: MealSlotEntry;
}

export interface ReviewRow {
  id: string; // stable per (entry, ingredient index) or manual row id
  text: string;
  selected: boolean;
}

export interface RecipeGroup {
  key: string;
  label: string;
  rows: ReviewRow[];
}

export interface NoIngredientGroup {
  key: string;
  label: string;
  manualRows: ReviewRow[]; // starts empty; the UI appends manual rows
}

// A recipe-backed meal whose detail fetch failed for a non-404 reason. Rendered as a
// retryable per-meal error row; it contributes no ingredient rows and no manual rows.
export interface ErrorGroup {
  key: string;
  label: string;
  recipeId: string;
}

// Per-recipe fetch outcome the UI passes in (built by useRecipeDetails in Task 5):
//   loaded  → detail available
//   missing → 404 (deleted recipe) → honest "no recipe ingredients"
//   error   → non-404 failure → retryable per-meal error row
export type RecipeResolution =
  | { status: "loaded"; detail: RecipeDetail }
  | { status: "missing" }
  | { status: "error" };

export interface ReviewModel {
  recipeGroups: RecipeGroup[];
  noIngredientGroups: NoIngredientGroup[];
  errorGroups: ErrorGroup[];
}

function isRecipeBacked(entry: MealSlotEntry): boolean {
  return entry.sourceType === "recipe" && entry.recipeId !== null;
}

export function collectPlannedEntries(board: MealBoard): PlannedEntryRef[] {
  const refs: PlannedEntryRef[] = [];
  for (const day of board.days) {
    for (const slot of day.slots) {
      const entries = [slot.primary, ...slot.extras].filter((e): e is MealSlotEntry => e !== null);
      for (const entry of entries) {
        refs.push({ dayIndex: day.dayIndex, mealType: slot.mealType, entry });
      }
    }
  }
  return refs;
}

export function hasRecipeBackedEntry(board: MealBoard): boolean {
  return collectPlannedEntries(board).some((ref) => isRecipeBacked(ref.entry));
}

export function distinctRecipeIds(board: MealBoard): string[] {
  const ids = new Set<string>();
  for (const ref of collectPlannedEntries(board)) {
    if (isRecipeBacked(ref.entry) && ref.entry.recipeId) ids.add(ref.entry.recipeId);
  }
  return [...ids];
}

function label(ref: PlannedEntryRef): string {
  return `${WEEKDAY_LABELS[ref.dayIndex]} · ${MEAL_LABELS[ref.mealType]} — ${ref.entry.title}`;
}

function groupKey(ref: PlannedEntryRef): string {
  return `${ref.dayIndex}:${ref.mealType}:${ref.entry.id}`;
}

export function buildReviewModel(
  board: MealBoard,
  resolutionsById: Record<string, RecipeResolution | undefined>,
): ReviewModel {
  const recipeGroups: RecipeGroup[] = [];
  const noIngredientGroups: NoIngredientGroup[] = [];
  const errorGroups: ErrorGroup[] = [];

  for (const ref of collectPlannedEntries(board)) {
    const key = groupKey(ref);

    // Quick meals (and any entry without a recipe id) never have recipe ingredients.
    if (!isRecipeBacked(ref.entry) || !ref.entry.recipeId) {
      noIngredientGroups.push({ key, label: label(ref), manualRows: [] });
      continue;
    }

    const resolution = resolutionsById[ref.entry.recipeId];

    // Non-404 failure (or an unresolved id) → retryable error row; never silently dropped
    // and never treated as "no recipe ingredients".
    if (resolution === undefined || resolution.status === "error") {
      errorGroups.push({ key, label: label(ref), recipeId: ref.entry.recipeId });
      continue;
    }

    // 404 (deleted recipe) → honest "no recipe ingredients".
    if (resolution.status === "missing") {
      noIngredientGroups.push({ key, label: label(ref), manualRows: [] });
      continue;
    }

    // Loaded: one verbatim row per ingredient, or the no-ingredient section when empty.
    const ingredients = resolution.detail.ingredients;
    if (ingredients.length > 0) {
      recipeGroups.push({
        key,
        label: label(ref),
        rows: ingredients.map((text, index) => ({ id: `${key}#${index}`, text, selected: true })),
      });
    } else {
      noIngredientGroups.push({ key, label: label(ref), manualRows: [] });
    }
  }

  return { recipeGroups, noIngredientGroups, errorGroups };
}

export function toBulkItemsRequest(model: ReviewModel): BulkCreateListItemsRequest {
  const rows: ReviewRow[] = [
    ...model.recipeGroups.flatMap((g) => g.rows),
    ...model.noIngredientGroups.flatMap((g) => g.manualRows),
  ];
  return {
    items: rows
      .filter((row) => row.selected && row.text.trim().length > 0)
      .map((row) => ({ text: row.text.trim() })), // uncategorized in v1
  };
}
```

- [ ] **Step 4: Run the focused pure test and verify it passes**

```bash
cd frontend
npm test -- --run src/components/meals/meal-ingredient-extraction.test.ts
```

Expected: PASS for visibility, dedupe, grouping, no-ingredient routing, and request conversion.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/meals/meal-ingredient-extraction.ts src/components/meals/meal-ingredient-extraction.test.ts
git commit -m "feat(meals): add recipe-ingredient extraction helpers"
```

## Task 5: Add ingredients review sheet component

**Files:**
- Create: `frontend/src/components/meals/use-recipe-details.ts` — `useRecipeDetails(recipeIds)` via `useQueries`, returning `{ resolutionsById: Record<string, RecipeResolution>; isLoading: boolean }`.
- Create: `frontend/src/components/meals/use-recipe-details.test.ts`
- Create: `frontend/src/components/meals/add-ingredients-sheet.tsx`
- Create: `frontend/src/components/meals/add-ingredients-sheet.test.tsx`

The sheet takes the visible `MealBoard`, resolves `RecipeDetail` for `distinctRecipeIds(board)` through `useRecipeDetails` (which uses TanStack Query's `useQueries` — one hook call for a variable number of recipes, so the Rules of Hooks are never violated; do **not** call `useRecipe` in a loop), builds the review model from the resolutions, and renders:

- A **loading** state while recipe details resolve; a **retryable per-meal error row** for each `errorGroup` (non-404 fetch failure) whose Retry refetches that recipe; a 404 recipe treated as "no recipe ingredients".
- Recipe groups with a header label and per-row checkbox (default checked), inline-editable text, and a remove control.
- A **No recipe ingredients** section listing each quick / empty-ingredient / deleted-recipe meal with an **+ Add item** control that appends an editable, selected manual row scoped to that meal.
- The grocery-list picker/create (delegated to Task 6's wiring; the sheet exposes `onConfirm(model, targetListId)` and a `create-list` affordance).
- **Add to list** disabled while offline with copy: "You're offline. Review your ingredients now and add them to your list when you're back online." Reuse the existing offline signal used by the Lists write flows.

- [ ] **Step 1: Write failing component tests**

First add `use-recipe-details.test.ts`: with MSW, assert `useRecipeDetails(["r1", "r2", "r3"])` maps a loaded recipe to `{ status: "loaded", detail }`, a `GET /recipes/:id` 404 to `{ status: "missing" }`, and a 500 to `{ status: "error" }`.

Then create `add-ingredients-sheet.test.tsx` covering (mirror existing meals component-test setup with MSW + seeded recipes):

```ts
it("groups verbatim ingredient rows by meal and lists quick meals under 'No recipe ingredients'", async () => { /* render with a board that has a recipe dinner + a quick dinner; assert group label + verbatim rows; assert the quick meal appears under the no-ingredient heading with an Add item control */ });

it("edits and removes rows before append", async () => { /* type into a row, remove another, assert the confirmed payload reflects edits/removals */ });

it("deselects a row so it is excluded from the append", async () => { /* uncheck one row; assert it is not in the submitted items */ });

it("does not auto-generate rows for quick meals", async () => { /* assert the quick meal contributes zero rows until the user clicks Add item */ });

it("shows a retryable error row for a non-404 recipe fetch failure and excludes it from append", async () => { /* make one recipe's GET /recipes/:id return 500; assert a per-meal error row with a Retry control; assert its ingredients are not in the submitted payload; a separate 404 recipe instead appears under 'No recipe ingredients' */ });

it("disables Add to list while offline with honest copy", async () => { /* set offline; assert the confirm button is disabled and the copy is shown */ });
```

- [ ] **Step 2: Run and verify failure**

```bash
cd frontend
npm test -- --run src/components/meals/add-ingredients-sheet.test.tsx
```

Expected: FAIL — component does not exist.

- [ ] **Step 3: Implement `useRecipeDetails` and `add-ingredients-sheet.tsx`**

First create `use-recipe-details.ts` — one `useQueries` call maps each recipe id to a `RecipeResolution` (404 → `missing`, any other error → `error`):

```ts
import { useQueries } from "@tanstack/react-query";
import { ApiException } from "@/api/client";
import { recipesService } from "@/api/services";
import { recipesKeys } from "@/api/hooks/use-recipes";
import type { RecipeResolution } from "./meal-ingredient-extraction";

export function useRecipeDetails(recipeIds: string[]): {
  resolutionsById: Record<string, RecipeResolution>;
  isLoading: boolean;
} {
  const results = useQueries({
    queries: recipeIds.map((id) => ({
      queryKey: recipesKeys.detail(id),
      queryFn: () => recipesService.getRecipe(id),
    })),
  });

  const resolutionsById: Record<string, RecipeResolution> = {};
  let isLoading = false;
  results.forEach((result, index) => {
    const id = recipeIds[index];
    if (result.isPending) {
      isLoading = true;
    } else if (result.isSuccess) {
      resolutionsById[id] = { status: "loaded", detail: result.data.data };
    } else if (result.error instanceof ApiException && result.error.status === 404) {
      resolutionsById[id] = { status: "missing" };
    } else {
      resolutionsById[id] = { status: "error" };
    }
  });

  return { resolutionsById, isLoading };
}
```

(`recipesKeys` and `recipesService.getRecipe` already exist; `ApiException` from `@/api/client` exposes `.status`; `result.data` is the `RecipeDetailApiResponse`, so `result.data.data` is the `RecipeDetail`.)

Then build `add-ingredients-sheet.tsx` using the existing MobileSheet/Radix dialog primitives used by other meals sheets, `useRecipeDetails(distinctRecipeIds(board))`, and the `meal-ingredient-extraction` helpers. While `isLoading`, show the loading state; otherwise compute `buildReviewModel(board, resolutionsById)` and hold the resulting `ReviewModel` in local component state so edit/remove/select/manual-add stay local until confirm. Render `errorGroups` as retryable per-meal rows (Retry refetches those recipe queries). Keep the file focused on presentation + local review state; delegate list selection and the append mutation to props from Task 6.

- [ ] **Step 4: Run and verify pass**

```bash
cd frontend
npm test -- --run src/components/meals/add-ingredients-sheet.test.tsx
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/meals/use-recipe-details.ts src/components/meals/use-recipe-details.test.ts \
  src/components/meals/add-ingredients-sheet.tsx src/components/meals/add-ingredients-sheet.test.tsx
git commit -m "feat(meals): add ingredients review sheet"
```

## Task 6: Wire the Meals action, list picker/create, append, and success

**Files:**
- Modify: `frontend/src/components/meals-view.tsx`
- Modify: `frontend/src/components/meals-view.test.tsx`
- Modify: `frontend/src/components/meals/add-ingredients-sheet.tsx` (list picker/create + confirm wiring)

- [ ] **Step 1: Write failing MealsView tests**

Add to `meals-view.test.tsx`:

```ts
it("shows Add ingredients on an editable week with a recipe-backed meal and hides it on past weeks", async () => { /* seed a current-week board with a recipe dinner → button present; navigate to a past week → button absent (alongside the existing Review only assertion) */ });

it("hides Add ingredients when the visible week has no recipe-backed meals", async () => { /* seed a board with only quick/empty slots → button absent */ });

it("defaults to the only grocery list and appends selected rows once, then offers View list", async () => { /* one grocery list exists; open sheet; confirm; assert exactly one POST /lists/:id/items/bulk with the selected rows; assert success + View list; assert the list detail cache gained the items */ });

it("lets the user create a grocery list when none exists, then append into it", async () => { /* zero grocery lists; create-list path; then bulk append targets the created list */ });

it("does not present a failed append as success", async () => { /* make the bulk endpoint 500; assert error surfaced, no success/View list, and the reviewed selection is retained */ });
```

- [ ] **Step 2: Run and verify failure**

```bash
cd frontend
npm test -- --run src/components/meals-view.test.tsx
```

Expected: FAIL — action + wiring absent.

- [ ] **Step 3: Add the action button in `MealsView`**

Near the existing `Fill empty slots` action (line ~482), render an **Add ingredients** button gated by `!readOnly && hasRecipeBackedEntry(visibleBoard)` (import `hasRecipeBackedEntry` from `meal-ingredient-extraction`). Clicking opens `AddIngredientsSheet` with the visible board.

- [ ] **Step 4: Implement list picker/create + confirm inside the sheet**

- Read grocery lists: `useLists()` filtered to `kind === "grocery"`. Preselect when exactly one; render a picker when several; render **Create a grocery list** (name field → `useCreateList()` with `{ name, kind: "grocery" }`) when none.
- On **Add to list**: build `toBulkItemsRequest(model)` and call `useBulkCreateListItems(targetListId)`. On success, show a success state with **View list** (navigate to the target list route). On error, keep the sheet open with the reviewed selection intact and show the failure; if a list was just created, keep it selected and offer retry.
- Guard: disable confirm when zero rows are selected or when offline.

- [ ] **Step 5: Run and verify pass, then full FE checks**

```bash
cd frontend
npm test -- --run src/components/meals-view.test.tsx src/components/meals/add-ingredients-sheet.test.tsx
npm run lint
npm test -- --run
```

Expected: PASS for new coverage and no regressions across the suite.

- [ ] **Step 6: Commit**

```bash
cd frontend
git add src/components/meals-view.tsx src/components/meals-view.test.tsx src/components/meals/add-ingredients-sheet.tsx
git commit -m "feat(meals): add ingredients-to-grocery-list action and append flow"
```

## Task 7: Released-contract E2E

**Files:**
- Modify: `frontend/e2e/mobile-meals.spec.ts` (or create `frontend/e2e/meals-ingredients.spec.ts`)

- [ ] **Step 1: Write the released-contract E2E**

Against the published backend release (resolved by CI), seed a family via `registerFamily`, then through API/UI:

1. Create two recipes that store ingredients and plan them as recipe-backed dinners in the current week; add one quick-meal dinner.
2. Open **Add ingredients**; assert rows grouped by meal with verbatim ingredient text and the quick meal under **No recipe ingredients**.
3. Edit one row and remove one row.
4. Ensure a grocery list exists (create one in-flow if none); confirm **Add to list** once.
5. Assert success + **View list**, then open the grocery list and assert the selected rows are present (the bulk response preserves request order; do **not** assert strict read-back order — same-batch items can tie on `createdAt`).

- [ ] **Step 2: Run E2E locally against the released backend**

```bash
cd frontend
# Start the released BE image (CI resolves the semver automatically):
docker compose -f docker-compose.e2e.yml up -d --wait
npm run test:e2e -- meals-ingredients
```

Expected: PASS against the published backend release. Record the resolved BE semver in the FE PR description.

- [ ] **Step 3: Commit**

```bash
cd frontend
git add e2e/
git commit -m "test(meals): e2e for ingredients-to-grocery-list append"
```

---

## Self-review

- **Spec coverage:** action visibility (Task 6 Step 3), visible-week derivation + verbatim rows + grouping + dedupe fetch (Task 4), quick/empty/deleted → no-ingredient section and non-404 → retryable error group (Task 4/5), edit/remove/select (Task 5), grocery picker/default-single/create (Task 6), one transactional append + View list (Task 6 + Tasks 1–2), generic bounded family-scoped endpoint with ordered response + all-or-nothing prevalidation/rollback (Tasks 1–2), offline honest copy (Task 5), failure-not-success + create-then-fail retry (Task 6), released-contract sequencing (Delivery boundaries + Task 7). All spec sections map to a task.
- **Prevalidation (spec D5):** `createItemsBulk` resolves/validates every category before mutating the aggregate, and `createItemsBulk_wrongKindCategory_throwsAndAppendsNothing` + the integration rollback assert nothing is written on failure.
- **BE test coverage vs. acceptance:** success+ordering, exactly-100 (service) and over-100 (controller), blank/overlong text (controller), list-not-in-family 404, valid same-kind / missing / wrong-kind category, and generic to-do append are all present across Tasks 1–2.
- **Placeholder scan:** no TBD/TODO; `MAX_BULK_ITEMS = 100` is concrete; category status behavior is inherited from the named `resolveCategory` path.
- **Type consistency:** `BulkCreateListItemsRequest`/`ListItemsApiResponse`/`createItemsBulk`/`useBulkCreateListItems`/`buildReviewModel`/`toBulkItemsRequest`/`hasRecipeBackedEntry`/`distinctRecipeIds`/`RecipeResolution`/`ErrorGroup`/`useRecipeDetails` are used with identical names across tasks; the FE hook uses the real `listsKeys.hub()`/`listsKeys.detail()` keys and `assertOnlineForWrite`; BE `createItemsBulk` request/response types match the controller and tests.
