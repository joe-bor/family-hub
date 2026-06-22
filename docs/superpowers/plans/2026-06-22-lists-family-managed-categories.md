# Lists Family-Managed Categories Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let each family manage one ordered category catalog per Lists kind, use those categories across matching lists and item forms, and safely support General grouping without weakening data isolation, concurrency, offline-read, or release guarantees.

**Architecture:** Backend delivery comes first. V17 adds the General-capable constraints, normalized-name uniqueness, and one persisted catalog-scope row per `(family, kind)`; every catalog-dependent write pessimistically locks that scope row before authoritative validation. A dedicated category service/controller owns catalog DTOs and atomic CRUD/reorder behavior, while the existing list service reuses the same lock for list defaults, grouped-mode changes, and item assignment. After a category-capable BE release is published, FE adds the released DTOs/query family, one responsive category manager, inline create-and-select, General grouping, precise cache convergence, and released-contract E2E.

**Tech Stack:** Java 21, Spring Boot 4, Spring Data JPA, Flyway, PostgreSQL 16/Testcontainers, Maven; React 19, TypeScript, TanStack Query v5, React Hook Form/Zod, Vaul/Radix primitives already installed, MSW, Vitest/Testing Library, Playwright.

**Spec:** `docs/superpowers/specs/2026-06-22-lists-family-managed-categories-design.md`

**Story:** `docs/product/backlog/module-foundations/lists-family-managed-categories.md`

---

## Delivery and repository boundaries

- Execute Tasks 1–6 in `backend/family-hub-api/`. Run Maven commands there and commit there.
- Merge the BE PR, merge the generated BE release PR, and confirm the semver image exists before starting Tasks 7–13.
- Execute Tasks 7–13 in `frontend/`. Run npm/Playwright commands there and commit there.
- Do not edit released `V12__create_lists_tables.sql`.
- Do not create execution Issues until this root plan has passed spec-to-plan review.
- The root story/spec/plan remain the contract. Delivery Issues must link all three and copy the non-negotiable execution contract.

## Verified codebase facts

- V16 is currently the latest BE migration, so V17 is available. Recheck immediately before implementation; if another migration lands first, rename only the new migration and update root docs before executing it.
- Production/test use Flyway plus PostgreSQL with `ddl-auto: validate`; local `dev` uses H2 with Flyway disabled and `ddl-auto: update`.
- `shared_list_item` carries `(category_id, family_id, list_kind)` as a composite FK. Delete must null `category_id` before deleting its category.
- The existing FE list-detail category type excludes General and requires `seeded`; runtime API responses are not Zod-parsed, but persisted list detail is structurally validated.
- Normal FE CI resolves `/releases/latest` and pulls the release semver. Its only unsafe behavior is the empty-resolution fallback to `latest`; BE `main` publishes mutable `latest` on every push.
- `MobileSheet` already owns Vaul focus handling and one `useBackHandler`; Vaul exposes `onAnimationEnd(open)` for a close-then-open handoff.
- No new runtime dependency is needed.

## File structure

### Backend — create

- `src/main/resources/db/migration/V17__enable_family_managed_list_categories.sql` — schema expansion, normalized uniqueness, catalog-scope rows, existing-family backfill.
- `src/main/java/com/familyhub/demo/model/ListCategoryCatalogScope.java` — lockable family + kind scope entity.
- `src/main/java/com/familyhub/demo/repository/ListCategoryCatalogScopeRepository.java` — pessimistic lock queries, including category-ID route-and-lock.
- `src/main/java/com/familyhub/demo/repository/SharedListItemRepository.java` — usage counts and delete-to-null bulk update.
- `src/main/java/com/familyhub/demo/service/ListCategoryService.java` — catalog reads and serialized create/rename/delete/reorder.
- `src/main/java/com/familyhub/demo/controller/ListCategoryController.java` — static `/api/lists/categories...` routes.
- `src/main/java/com/familyhub/demo/exception/ConflictException.java` — explicit application `409`.
- `src/main/java/com/familyhub/demo/dto/CreateListCategoryRequest.java`
- `src/main/java/com/familyhub/demo/dto/RenameListCategoryRequest.java`
- `src/main/java/com/familyhub/demo/dto/ReorderListCategoriesRequest.java`
- `src/main/java/com/familyhub/demo/dto/ListCategoryOption.java`
- `src/main/java/com/familyhub/demo/dto/ListCategoryManagementEntry.java`
- `src/main/java/com/familyhub/demo/dto/ListCategoryCatalogResponse.java`
- `src/main/java/com/familyhub/demo/dto/CategoryDeleteResult.java`
- `src/test/java/com/familyhub/demo/migration/ListCategoriesMigrationTest.java`
- `src/test/java/com/familyhub/demo/service/ListCategoryServiceTest.java`
- `src/test/java/com/familyhub/demo/controller/ListCategoryControllerTest.java`
- `src/test/java/com/familyhub/demo/integration/ListCategoryIntegrationTest.java`
- `scripts/resolve-release-version.sh` — resolve only a published BE semver and expose a sourceable parser for tests.
- `scripts/resolve-release-version.test.sh` — prove empty/malformed release responses fail closed.

### Backend — modify

- `src/main/java/com/familyhub/demo/model/ListCategory.java` — remove application mapping for `seeded`.
- `src/main/java/com/familyhub/demo/repository/ListCategoryRepository.java` — normalized lookup, ordered catalog, count/order operations.
- `src/main/java/com/familyhub/demo/repository/SharedListRepository.java` — grouped-list count and atomic flatten update.
- `src/main/java/com/familyhub/demo/service/ListSeedService.java` — create three catalog scopes and only Grocery/To-do starters during registration.
- `src/main/java/com/familyhub/demo/service/ListService.java` — General categories, catalog lock reuse, empty-catalog defaults/grouping rule.
- `src/main/java/com/familyhub/demo/mapper/ListMapper.java`
- `src/main/java/com/familyhub/demo/dto/ListDetailResponse.java`
- `src/main/java/com/familyhub/demo/exception/GlobalExceptionHandler.java`
- `src/test/java/com/familyhub/demo/TestDataFactory.java`
- `src/test/java/com/familyhub/demo/service/ListServiceTest.java`
- `src/test/java/com/familyhub/demo/controller/ListControllerTest.java`
- `src/test/java/com/familyhub/demo/integration/ListIntegrationTest.java`
- `scripts/deploy.sh` — fail closed when no published release resolves.

### Frontend — create

- `src/components/lists/category-manager.tsx` — responsive manager orchestration, load/error/offline/empty states.
- `src/components/lists/category-manager-list.tsx` — catalog rows, inline rename, accessible local reorder draft.
- `src/components/lists/category-confirm-dialog.tsx` — delete and dirty-close confirmation using existing Dialog primitives.
- `src/components/lists/category-manager.test.tsx`
- `src/components/lists/category-manager-list.test.tsx`
- `src/components/lists/list-item-sheet.test.tsx`

### Frontend — modify

- `src/lib/types/lists.ts` — released category contracts; remove `CategoryAwareListKind` and `seeded`.
- `src/lib/validations/lists.ts` and `src/lib/validations/lists.test.ts` — category-name validation.
- `src/api/services/lists.service.ts`
- `src/api/hooks/use-lists.ts` and `src/api/hooks/use-lists.test.tsx`
- `src/api/hooks/index.ts` and `src/api/index.ts`
- `src/test/mocks/handlers.ts` and `src/test/mocks/server.ts` — shared family + kind mock catalogs; static routes before `:id`.
- `src/lib/offline/validators.ts`, `src/lib/offline/validators.test.ts`, and `src/lib/offline/dehydrate.test.ts`.
- `src/components/ui/mobile-sheet.tsx` and `src/components/ui/mobile-sheet.test.tsx` — explicit focus-return target, optional restore suppression, animation completion.
- `src/components/ui/responsive-form-dialog.tsx` and its tests — forward mobile focus-return behavior.
- `src/components/lists/list-create-sheet.tsx`
- `src/components/lists/list-options-controls.tsx`
- `src/components/lists/list-item-sheet.tsx`
- `src/components/lists/list-detail-view.tsx` and `src/components/lists/list-detail-view.test.tsx`
- `src/components/lists/build-list-sections.ts` and `src/components/lists/build-list-sections.test.ts`
- `src/components/lists-view.test.tsx`
- `.github/workflows/ci.yml` and `docker-compose.e2e.yml` — released version resolution fails closed.
- `.github/scripts/resolve-backend-version.sh` and `.github/scripts/resolve-backend-version.test.sh` — sourceable release parser plus failure tests used by CI.
- `e2e/mobile-lists.spec.ts` — released-contract manager/inline/group/reorder/delete/hardware-back flow.
- `e2e/offline-persistence.spec.ts` — new and legacy category cache shapes remain readable.

---

## Task 1: Add V17 and a lockable catalog scope

**Files:**
- Create: `backend/family-hub-api/src/main/resources/db/migration/V17__enable_family_managed_list_categories.sql`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/ListCategoryCatalogScope.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/ListCategoryCatalogScopeRepository.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/migration/ListCategoriesMigrationTest.java`

- [ ] **Step 1: Write migration tests for both paths**

Use one PostgreSQL 16 Testcontainer. In one test migrate an empty schema through latest and assert all V17 constraints/index/table/columns. In the second, migrate only through V16, insert a family plus representative Grocery/To-do/General list data, migrate through latest, and assert the rows are unchanged and exactly three scope rows exist. Also insert `Documents` then assert `documents` and whitespace/case variant `  DOCUMENTS  ` violate the normalized index. Assert `list_category.seeded` still exists.

Core Flyway setup:

```java
private Flyway flyway(String schema, String target) {
    return Flyway.configure()
            .dataSource(postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword())
            .schemas(schema)
            .defaultSchema(schema)
            .createSchemas(true)
            .target(target)
            .load();
}
```

Run: `./mvnw -Dtest=ListCategoriesMigrationTest test`

Expected: FAIL because V17 and the scope table do not exist.

- [ ] **Step 2: Add the V17 SQL**

Use the shipped constraint names exactly and abort rather than silently merging unexpected duplicates:

```sql
DO $$
BEGIN
    IF EXISTS (
        SELECT 1
        FROM list_category
        GROUP BY family_id, kind, lower(btrim(name))
        HAVING count(*) > 1
    ) THEN
        RAISE EXCEPTION 'Cannot enable normalized list-category names: duplicates exist';
    END IF;
END $$;

ALTER TABLE list_category DROP CONSTRAINT ck_list_category_supported_kind;
ALTER TABLE list_category ADD CONSTRAINT ck_list_category_supported_kind
    CHECK (kind IN ('GROCERY', 'TODO', 'GENERAL'));

ALTER TABLE shared_list DROP CONSTRAINT ck_shared_list_general_display_mode;
ALTER TABLE shared_list_item DROP CONSTRAINT ck_shared_list_item_general_category;

ALTER TABLE list_category DROP CONSTRAINT uk_list_category_family_kind_name;
CREATE UNIQUE INDEX uk_list_category_family_kind_normalized_name
    ON list_category (family_id, kind, lower(btrim(name)));

CREATE TABLE list_category_catalog_scope (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL,
    kind VARCHAR(20) NOT NULL,
    CONSTRAINT fk_list_category_catalog_scope_family
        FOREIGN KEY (family_id) REFERENCES family(id) ON DELETE CASCADE,
    CONSTRAINT ck_list_category_catalog_scope_kind
        CHECK (kind IN ('GROCERY', 'TODO', 'GENERAL')),
    CONSTRAINT uk_list_category_catalog_scope_family_kind
        UNIQUE (family_id, kind)
);

INSERT INTO list_category_catalog_scope (family_id, kind)
SELECT family.id, kinds.kind
FROM family
CROSS JOIN (VALUES ('GROCERY'), ('TODO'), ('GENERAL')) AS kinds(kind);
```

Do not alter or drop `seeded`.

- [ ] **Step 3: Map and lock the scope row**

`ListCategoryCatalogScope` contains generated `id`, non-null `Family family`, and enumerated `ListKind kind`. The repository exposes:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("""
        select scope from ListCategoryCatalogScope scope
        where scope.family = :family and scope.kind = :kind
        """)
Optional<ListCategoryCatalogScope> lockByFamilyAndKind(Family family, ListKind kind);

@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("""
        select scope from ListCategoryCatalogScope scope, ListCategory category
        where category.family = :family
          and category.id = :categoryId
          and scope.family = :family
          and scope.kind = category.kind
        """)
Optional<ListCategoryCatalogScope> lockByFamilyAndCategoryId(Family family, UUID categoryId);
```

The second query performs a family-scoped route-and-lock in one statement. Services must refetch the category after the scope lock before using it as authoritative state.

- [ ] **Step 4: Run migration tests**

Run: `./mvnw -Dtest=ListCategoriesMigrationTest test`

Expected: PASS for clean and V16-upgrade paths.

- [ ] **Step 5: Commit**

```bash
git add src/main/resources/db/migration/V17__enable_family_managed_list_categories.sql \
  src/main/java/com/familyhub/demo/model/ListCategoryCatalogScope.java \
  src/main/java/com/familyhub/demo/repository/ListCategoryCatalogScopeRepository.java \
  src/test/java/com/familyhub/demo/migration/ListCategoriesMigrationTest.java
git commit -m "feat(lists): add category catalog scope migration"
```

## Task 2: Define category contracts, repositories, and conflict mapping

**Files:**
- Create the eight backend DTO/exception files listed above.
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/SharedListItemRepository.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/ListCategory.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/ListCategoryRepository.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/SharedListRepository.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/exception/GlobalExceptionHandler.java`

- [ ] **Step 1: Add validated request records and response records**

```java
public record CreateListCategoryRequest(
        @NotNull ListKind kind,
        @NotBlank @Size(max = 100) String name
) {}

public record RenameListCategoryRequest(
        @NotBlank @Size(max = 100) String name
) {}

public record ReorderListCategoriesRequest(
        @NotNull ListKind kind,
        @NotNull List<@NotNull UUID> expectedCategoryIds,
        @NotNull List<@NotNull UUID> categoryIds
) {}

public record ListCategoryOption(UUID id, ListKind kind, String name, int sortOrder) {}
public record ListCategoryManagementEntry(
        UUID id, ListKind kind, String name, int sortOrder, long itemCount
) {}
public record ListCategoryCatalogResponse(
        ListKind kind, long groupedListCount, List<ListCategoryManagementEntry> categories
) {}
public record CategoryDeleteResult(long uncategorizedItemCount, long flattenedListCount) {}
```

Remove `seeded` from `ListCategory`; leave the physical column untouched.

- [ ] **Step 2: Add repository operations without overcounting joins**

`ListCategoryRepository` must provide ordered family+kind reads, family+ID reads, case-insensitive trimmed existence excluding an optional current ID, and `countByFamilyAndKind`.

`SharedListItemRepository` must provide one grouped projection query and one scoped bulk update:

```java
interface CategoryUsageCount {
    UUID getCategoryId();
    long getItemCount();
}

@Query("""
        select item.category.id as categoryId, count(item.id) as itemCount
        from SharedListItem item
        where item.familyId = :familyId
          and item.listKind = :kind
          and item.category is not null
        group by item.category.id
        """)
List<CategoryUsageCount> countUsage(UUID familyId, ListKind kind);

@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("""
        update SharedListItem item
        set item.category = null, item.updatedAt = CURRENT_TIMESTAMP
        where item.familyId = :familyId
          and item.listKind = :kind
          and item.category.id = :categoryId
        """)
int clearCategory(UUID familyId, ListKind kind, UUID categoryId);
```

`SharedListRepository` adds an exact grouped count and a scoped bulk update from `GROUPED` to `FLAT` returning the affected row count.

- [ ] **Step 3: Map explicit application conflicts to `409`**

Add `ConflictException` and a `GlobalExceptionHandler` method returning the existing `ErrorResponse` shape. Keep `DataIntegrityViolationException` mapped to `409`; do not parse vendor messages or expose constraint details.

- [ ] **Step 4: Compile the exact contracts and commit**

Run: `./mvnw -DskipTests compile`

Expected: BUILD SUCCESS. Repository behavior is exercised through the service unit tests in Task 3 and PostgreSQL integration tests in Task 5; this task contains only contracts and data-access seams.

```bash
git add src/main/java/com/familyhub/demo/dto src/main/java/com/familyhub/demo/exception \
  src/main/java/com/familyhub/demo/model/ListCategory.java \
  src/main/java/com/familyhub/demo/repository
git commit -m "feat(lists): define managed category contracts"
```

## Task 3: Implement serialized catalog CRUD, delete, and reorder

**Files:**
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/ListCategoryService.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/service/ListCategoryServiceTest.java`

- [ ] **Step 1: Complete failing unit tests before service code**

Cover these observable cases with Mockito/InOrder where lock ordering matters:

- read returns ordered entries, exact item counts, and exact grouped-list count;
- create locks scope, trims, rejects normalized duplicate with `ConflictException`, and appends at current count;
- rename route-and-locks by family/category, refetches, trims, preserves order/kind, and returns current item count;
- delete route-and-locks, clears matching items before delete, compacts dense order, flattens only when final, and returns exact counts;
- reorder checks `expectedCategoryIds` before target membership; stale baseline is `409`; duplicate/missing/extra/foreign/wrong-kind target membership is `400`; empty/single/unchanged are `200`-equivalent responses; changed order is dense;
- all missing or cross-family category IDs produce `404` without existence leakage.

Run: `./mvnw -Dtest=ListCategoryServiceTest test`

Expected: FAIL because `ListCategoryService` is absent.

- [ ] **Step 2: Implement one lock-first service boundary**

Expose exactly:

```java
public ListCategoryCatalogResponse getCatalog(ListKind kind, Family family);
@Transactional public ListCategoryManagementEntry create(CreateListCategoryRequest request, Family family);
@Transactional public ListCategoryManagementEntry rename(UUID id, RenameListCategoryRequest request, Family family);
@Transactional public CategoryDeleteResult delete(UUID id, Family family);
@Transactional public ListCategoryCatalogResponse reorder(ReorderListCategoriesRequest request, Family family);
```

Use helpers with these semantics:

```java
private ListCategoryCatalogScope lock(Family family, ListKind kind) {
    return scopeRepository.lockByFamilyAndKind(family, kind)
            .orElseThrow(() -> new IllegalStateException("List category catalog scope is missing"));
}

private ListCategory lockAndRefetch(Family family, UUID categoryId) {
    scopeRepository.lockByFamilyAndCategoryId(family, categoryId)
            .orElseThrow(() -> new ResourceNotFoundException("List Category", categoryId));
    return categoryRepository.findByFamilyAndId(family, categoryId)
            .orElseThrow(() -> new ResourceNotFoundException("List Category", categoryId));
}
```

Every normalized-name precheck runs after the scope lock. Catching a precheck race is not sufficient: allow the database unique index to remain authoritative and let `DataIntegrityViolationException` map to `409`.

- [ ] **Step 3: Implement atomic delete in the required order**

Within one transaction: route-and-lock; refetch; clear item FKs; delete and flush category; load remaining ordered categories and rewrite `sortOrder = index`; if none remain, bulk-flatten matching grouped lists; return the two bulk-operation counts. Do not use cascade delete for item assignment.

- [ ] **Step 4: Implement reorder comparison and response**

After the scope lock, capture current ordered IDs. Compare that list to `expectedCategoryIds` using list equality and throw `ConflictException` on mismatch. Only then require `categoryIds.size == current.size`, unique target IDs, and set equality; throw `BadRequestException` for any membership failure. Rewrite all positions and return a freshly assembled catalog.

- [ ] **Step 5: Run and commit**

Run: `./mvnw -Dtest=ListCategoryServiceTest test`

Expected: PASS.

```bash
git add src/main/java/com/familyhub/demo/service/ListCategoryService.java \
  src/test/java/com/familyhub/demo/service/ListCategoryServiceTest.java
git commit -m "feat(lists): manage serialized category catalogs"
```

## Task 4: Integrate catalog rules into registration, lists, and items

**Files:**
- Modify: backend model/DTO/mapper/seed/list service files listed in File structure.
- Modify tests: `TestDataFactory`, `ListServiceTest`, `ListIntegrationTest`.

- [ ] **Step 1: Replace the obsolete service tests with the new invariants**

Add failing tests for:

- registration creates three scope rows, Grocery/To-do starters, and no General starter;
- starter rename/delete are ordinary data and calling login does not reseed;
- Grocery/To-do list creation is grouped only when its catalog is non-empty; General is always flat;
- grouped PATCH works for General with at least one category and returns `ConflictException` for every empty kind;
- first category creation and later recreation do not change existing list modes;
- General item create/update accepts a matching category; family/kind mismatch retains `404`/`400` semantics;
- list detail embeds ordered `ListCategoryOption` values for all kinds and never returns `seeded`.

Run: `./mvnw -Dtest=ListServiceTest,ListIntegrationTest test`

Expected: FAIL on the shipped General restrictions and category DTO.

- [ ] **Step 2: Make registration one-time and create catalog scopes**

`ListSeedService.seedDefaultsForFamily` creates and flushes one scope for each `ListKind`, then inserts the fixed Grocery/To-do names with dense order. It creates no General categories and never runs from login, reads, or background reconciliation. Remove all `setSeeded` calls.

- [ ] **Step 3: Reuse the catalog lock in `ListService`**

- Create list: lock `(family, request.kind)`; Grocery/To-do choose grouped iff category count is positive; General chooses flat.
- Update list: family-scope lookup determines immutable kind, lock that scope, then reject grouped if category count is zero; otherwise apply the request.
- Item create/update with non-null `categoryId`: determine immutable list kind, lock that scope, then refetch the category through family and validate exact kind.
- Detail mapping: query categories for every kind and map `ListCategoryOption`; remove the `GENERAL ? List.of()` branch.

Simple list reads need no scope lock. Flat display continues retaining category assignments.

- [ ] **Step 4: Update mapper/fixtures and run focused tests**

Run: `./mvnw -Dtest=ListServiceTest,ListIntegrationTest,AuthIntegrationTest test`

Expected: PASS, including registration starters and General behavior.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/familyhub/demo/model src/main/java/com/familyhub/demo/dto \
  src/main/java/com/familyhub/demo/mapper/ListMapper.java \
  src/main/java/com/familyhub/demo/service/ListSeedService.java \
  src/main/java/com/familyhub/demo/service/ListService.java \
  src/test/java/com/familyhub/demo/TestDataFactory.java \
  src/test/java/com/familyhub/demo/service/ListServiceTest.java \
  src/test/java/com/familyhub/demo/integration/ListIntegrationTest.java
git commit -m "feat(lists): enable family categories across list kinds"
```

## Task 5: Publish the category API and prove isolation/concurrency

**Files:**
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/controller/ListCategoryController.java`
- Create: controller/integration tests listed in File structure.
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/controller/ListControllerTest.java`

- [ ] **Step 1: Write controller contract tests**

Test the exact envelope/status/header for:

```text
GET    /api/lists/categories?kind=general        -> 200 ListCategoryCatalogResponse
POST   /api/lists/categories                     -> 201 + Location + management entry
PATCH  /api/lists/categories/{categoryId}        -> 200 management entry
DELETE /api/lists/categories/{categoryId}        -> 200 CategoryDeleteResult
PUT    /api/lists/categories/order               -> 200 ListCategoryCatalogResponse
```

Also prove `/api/lists/categories` is not captured by `/api/lists/{id}`, validation failures are `400`, unauthenticated requests are `401`, family-scoped missing IDs are `404`, and explicit conflicts are `409` in the existing `ErrorResponse` shape.

Run: `./mvnw -Dtest=ListCategoryControllerTest,ListControllerTest test`

Expected: FAIL because the controller is absent.

- [ ] **Step 2: Add the dedicated controller**

Use `@RequestMapping("/api/lists/categories")`; put `/order` before `/{categoryId}` for readability even though Spring chooses the static route. POST builds `Location` from the returned ID. Do not accept family IDs from requests.

- [ ] **Step 3: Write PostgreSQL integration and concurrency tests**

Use authenticated MockMvc flows and direct JDBC only for setup/failure injection. Cover:

- two families and two kinds cannot read, mutate, assign, count, or reorder each other's categories;
- counts span two same-kind lists without join multiplication;
- delete moves all matching items to null, returns exact count, compacts order, and final delete flattens every grouped matching list;
- a PostgreSQL trigger that rejects the category DELETE causes the HTTP request to fail and proves item assignments, list modes, category row, and order all rolled back;
- two executor threads POST `Travel` and `travel` behind a latch: exactly one `201`, one `409`, zero `500`, one row;
- concurrent empty-catalog creates produce dense distinct positions;
- reorder racing create/delete/reorder produces a stale-baseline `409` and no lost/duplicate order;
- empty/single/unchanged reorder succeeds; malformed/foreign/wrong-kind membership is `400` without revealing foreign existence.

Run: `./mvnw -Dtest=ListCategoryIntegrationTest test`

Expected: PASS.

- [ ] **Step 4: Run the backend verification suite**

Run: `./mvnw verify`

Expected: BUILD SUCCESS with all unit, migration, controller, and PostgreSQL integration tests passing.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/familyhub/demo/controller/ListCategoryController.java \
  src/test/java/com/familyhub/demo/controller \
  src/test/java/com/familyhub/demo/integration
git commit -m "feat(lists): expose family category catalog API"
```

## Task 6: Make BE deployment resolution fail closed and publish the release

**Files:**
- Create: `backend/family-hub-api/scripts/resolve-release-version.sh`
- Create: `backend/family-hub-api/scripts/resolve-release-version.test.sh`
- Modify: `backend/family-hub-api/scripts/deploy.sh`

- [ ] **Step 1: Write the failing release-parser test**

The test sources `resolve-release-version.sh`, asserts `resolve_release_version '{"tag_name":"v1.2.3"}'` prints `1.2.3`, and asserts missing, null, blank, or malformed JSON return non-zero. It never invokes SSH.

Run: `bash scripts/resolve-release-version.test.sh`

Expected: FAIL because the resolver does not exist.

- [ ] **Step 2: Extract a sourceable fail-closed resolver**

The resolver defines `resolve_release_version()` with `jq -er '.tag_name | select(type == "string" and length > 0)'`, strips one leading `v`, and has an executable main path that fetches `/releases/latest`. `deploy.sh` captures its output and refuses to continue when empty:

```bash
if [ -z "$BE_VERSION" ]; then
  echo "No published backend release could be resolved; refusing to deploy." >&2
  exit 1
fi
```

There is no assignment to `latest`.

- [ ] **Step 3: Run parser tests, verify shell syntax, and commit**

Run: `bash scripts/resolve-release-version.test.sh && bash -n scripts/deploy.sh scripts/resolve-release-version.sh`

Expected: all parser cases pass and shell syntax exits 0.

```bash
git add scripts/deploy.sh scripts/resolve-release-version.sh scripts/resolve-release-version.test.sh
git commit -m "fix(release): fail closed when backend release is unavailable"
```

- [ ] **Step 4: Merge and publish before FE work**

Open the BE PR with the root Story/Spec/Plan links and the non-negotiable lock/migration/API/test contract. After review, merge it, merge the generated BE release PR, and verify both GitHub `/releases/latest` and `ghcr.io/joe-bor/family-hub-api:<semver>` resolve to that release. Record the semver for the FE PR/E2E evidence. Do not begin Task 7 against unreleased BE `main`.

---

## Task 7: Add the released FE contract, API hooks, mocks, and offline schema

**Files:**
- Modify: FE type, service, hook, barrel, validation, MSW, and offline files listed above.

- [ ] **Step 1: Write failing type/service/hook tests**

Add tests for all service paths and exact DTOs; catalog query key `listsKeys.categories(kind)`; online-only catalog fetch; create/rename/delete/reorder requests; static MSW category routes before `/lists/:id`; and category mutation cache effects across two cached same-kind details while a different kind remains untouched.

Run: `npm test -- --run src/api/hooks/use-lists.test.tsx src/lib/validations/lists.test.ts`

Expected: FAIL because category contracts/hooks do not exist.

- [ ] **Step 2: Replace category types and add requests/responses**

```ts
export interface ListCategoryOption {
  id: string;
  kind: ListKind;
  name: string;
  sortOrder: number;
}
export interface ListCategoryManagementEntry extends ListCategoryOption {
  itemCount: number;
}
export interface ListCategoryCatalog {
  kind: ListKind;
  groupedListCount: number;
  categories: ListCategoryManagementEntry[];
}
export interface CategoryDeleteResult {
  uncategorizedItemCount: number;
  flattenedListCount: number;
}
export interface CreateListCategoryRequest { kind: ListKind; name: string }
export interface RenameListCategoryRequest { name: string }
export interface ReorderListCategoriesRequest {
  kind: ListKind;
  expectedCategoryIds: string[];
  categoryIds: string[];
}
```

`ListDetail.categories` becomes `ListCategoryOption[]`. Delete `CategoryAwareListKind`, `ListCategory`, and every `seeded` fixture/reference. Add `categoryNameSchema` using trim, required, max 100.

- [ ] **Step 3: Add service methods and query hooks**

Service paths exactly match the spec. Query key:

```ts
categories: (kind: ListKind) => [...listsKeys.all, "categories", kind] as const,
```

`useListCategories(kind, enabled)` sets `enabled` from manager-open plus online state and is never added to the offline persistence allowlist.

Mutation cache rules:

- create: append returned option to every cached same-kind list detail and append management entry to catalog;
- rename: replace matching entries in catalog and same-kind details;
- delete: remove category, null matching item assignments, and set list mode flat if no options remain; decrement grouped count by authoritative `flattenedListCount`;
- reorder: replace catalog from response and reorder same-kind detail options from its canonical IDs;
- after each immediate update, invalidate active same-kind list-detail queries as correctness backstop; never invalidate unrelated kinds or Lists hub for category-only changes.

All write hooks call `assertOnlineForWrite()` before cache mutation.

- [ ] **Step 4: Make MSW model one shared catalog per kind**

Store `mockCategoryCatalogs: Record<ListKind, ListCategoryOption[]>`. Every list detail derives categories from its kind catalog rather than owning a private catalog. Add static GET/POST/PUT handlers before `GET /lists/:id`, then PATCH/DELETE category-ID handlers. Enforce trim/case duplicate conflicts, complete reorder membership, delete-to-null across matching mock lists, final flattening, and authoritative counts.

- [ ] **Step 5: Tighten offline validators without persisting management catalogs**

Replace `z.array(z.unknown())` with a structural category-option schema that requires `id/kind/name/sortOrder` and is lenient to legacy extra `seeded`. Tests cover new options without `seeded`, legacy cached options with it, grouped General detail, and `listsKeys.categories(kind)` returning false from `isOfflineReadQueryKey`.

- [ ] **Step 6: Run and commit**

Run: `npm test -- --run src/api/hooks/use-lists.test.tsx src/lib/validations/lists.test.ts src/lib/offline/validators.test.ts src/lib/offline/dehydrate.test.ts`

Expected: PASS.

```bash
git add src/lib/types/lists.ts src/lib/validations/lists.ts src/lib/validations/lists.test.ts \
  src/api/services/lists.service.ts src/api/hooks/use-lists.ts src/api/hooks/use-lists.test.tsx \
  src/api/hooks/index.ts src/api/index.ts src/test/mocks/handlers.ts src/test/mocks/server.ts \
  src/lib/offline/validators.ts src/lib/offline/validators.test.ts src/lib/offline/dehydrate.test.ts
git commit -m "feat(lists): consume managed category catalog contract"
```

## Task 8: Enable General rendering and category controls for every kind

**Files:**
- Modify: `list-create-sheet.tsx`, `list-options-controls.tsx`, `build-list-sections.ts`, and their tests.

- [ ] **Step 1: Write failing rendering/options/copy tests**

Assert General grouped detail follows catalog order, hides empty groups, adds synthetic Uncategorized only when non-empty, and flat mode retains assignments without headings. Assert every kind shows the category-mode control, empty catalog disables grouped with `Create a category first.`, and creating a category alone never mutates display mode. Assert revised kind copy exactly matches the spec.

Run: `npm test -- --run src/components/lists/build-list-sections.test.ts src/components/lists/list-detail-view.test.tsx src/components/lists-view.test.tsx`

Expected: FAIL because General is hard-coded flat/categoryless.

- [ ] **Step 2: Remove General special-casing**

`buildListSections` uses only `categoryDisplayMode === "flat"` for its flat branch. `ListOptionsControls` renders the category-mode control for every kind; its grouped `<option>` is disabled when `list.categories.length === 0`, followed by the explanatory text.

- [ ] **Step 3: Update creation copy and run tests**

Use:

```text
Grocery — Customizable shopping categories with starter examples
To-do — Customizable planning buckets with starter examples
General — Flexible checklist; add categories whenever useful
```

Run the Step 1 command; expect PASS.

- [ ] **Step 4: Commit**

```bash
git add src/components/lists/list-create-sheet.tsx src/components/lists/list-options-controls.tsx \
  src/components/lists/build-list-sections.ts src/components/lists/build-list-sections.test.ts \
  src/components/lists/list-detail-view.test.tsx src/components/lists-view.test.tsx
git commit -m "feat(lists): enable categories for every list kind"
```

## Task 9: Build category manager CRUD, states, confirmations, and cache feedback

**Files:**
- Create: `category-manager.tsx`, `category-manager-list.tsx`, `category-confirm-dialog.tsx`, and tests.
- Modify: `list-options-controls.tsx` and `list-detail-view.tsx` for manager entry and desktop opening; mobile handoff lands in Task 11.

- [ ] **Step 1: Write manager behavior tests**

Cover loading skeleton/text, GET error + Retry, connectivity dropping while open with no query/write, empty state, shared-scope copy, add/rename validation, input-adjacent `400/409/network` errors, preflight item/grouped counts in delete confirmation, delete failure retaining confirmation context, authoritative response counts in success toast, and final delete visibly flattening an open list. Confirm create/rename/delete pending disables only that operation outside reorder. List Options tests prove Manage Categories appears for every kind and is disabled with explanatory copy while offline.

Run: `npm test -- --run src/components/lists/category-manager.test.tsx`

Expected: FAIL because manager components do not exist.

- [ ] **Step 2: Implement one responsive manager**

`CategoryManager` takes `open`, `onOpenChange`, `kind`, and optional `returnFocusRef`. It calls `useOnlineStatus`; offline renders explanatory content and does not enable `useListCategories`. Desktop and mobile share identical content through `ResponsiveFormDialog`, not duplicate manager trees.

`CategoryManagerList` renders ordered rows with name and `itemCount`, inline rename, delete, and Reorder entry. Add uses `categoryNameSchema`; successful create clears only the category-name input. Error text is adjacent to the active form.

`ListOptionsControls` adds `onManageCategories` and `categoriesOnline` props, renders the shared-scope copy based on kind, and disables the manager entry while offline. `ListDetailView` passes `useOnlineStatus()` and opens the desktop manager directly.

- [ ] **Step 3: Implement honest delete confirmation**

The initial dialog text uses catalog preflight counts. The mutation retains the selected entry until success. On success, close confirmation and toast using `CategoryDeleteResult`; when `flattenedListCount > 0`, mention the exact number switched to flat. Register `useBackHandler(confirmOpen, closeConfirm)` so hardware back closes confirmation before manager.

- [ ] **Step 4: Run and commit**

Run: `npm test -- --run src/components/lists/category-manager.test.tsx src/components/lists/list-detail-view.test.tsx`

Expected: PASS for desktop and manager behavior.

```bash
git add src/components/lists/category-manager*.tsx src/components/lists/category-confirm-dialog.tsx \
  src/components/lists/category-manager*.test.tsx src/components/lists/list-detail-view.tsx \
  src/components/lists/list-detail-view.test.tsx src/components/lists/list-options-controls.tsx
git commit -m "feat(lists): add family category manager"
```

## Task 10: Add explicit accessible reorder mode

**Files:**
- Modify: `category-manager.tsx`, `category-manager-list.tsx`, and their tests.

- [ ] **Step 1: Write failing reorder state tests**

Assert entering captures immutable baseline entries/IDs; up/down clicks make no request; boundary buttons are disabled; controls are named `Move {name} up/down`; focus remains on the moved row's applicable control; and an aria-live region announces `{name} moved to position X of Y`. Save is disabled unchanged and sends one full expected/target request when dirty. Create/rename/delete are disabled throughout reorder. Pending disables move/Save/Cancel/close. Failure keeps draft with Retry/Cancel. Success replaces baseline. `409` refetches, exits reorder, and explains remote change. The 44x44 computed touch geometry is asserted in Playwright in Task 13 because jsdom cannot measure CSS layout.

Run: `npm test -- --run src/components/lists/category-manager-list.test.tsx`

Expected: FAIL on missing reorder behavior.

- [ ] **Step 2: Implement local draft mechanics**

On entry snapshot `baselineEntries` and `baselineIds`; derive `draftIds`. Moves update only `draftIds`. Render rows from the snapshot so background catalog refresh cannot mutate the local baseline. Save calls reorder once with both arrays. On success replace catalog/baseline from response and exit. On non-409 failure keep mode/draft; on 409 await catalog refetch, exit, and toast.

- [ ] **Step 3: Implement dirty-close confirmation**

Clean close exits immediately. Dirty close opens `CategoryConfirmDialog`; `Keep editing` returns to the draft and `Discard order` exits/ closes. Register its back handler after the manager's. While PUT is pending, ignore overlay/Escape/back close requests and disable all buttons.

- [ ] **Step 4: Run and commit**

Run the Step 1 command; expect PASS.

```bash
git add src/components/lists/category-manager.tsx src/components/lists/category-manager-list.tsx \
  src/components/lists/category-manager.test.tsx src/components/lists/category-manager-list.test.tsx
git commit -m "feat(lists): add accessible batched category reorder"
```

## Task 11: Implement mobile Options-to-manager handoff and focus ownership

**Files:**
- Modify: `mobile-sheet.tsx`, `responsive-form-dialog.tsx`, `list-detail-view.tsx`, and tests.

- [ ] **Step 1: Write failing primitive and integration tests**

Primitive tests prove optional close-focus suppression, explicit `returnFocusRef`, and `onAnimationEnd(false)`. List-detail tests prove Manage Categories first closes Options; manager opens only after close animation; at no point are two dialogs/back handlers active; manager receives focus; manager close returns focus to the visible List Options trigger; desktop opens centered manager without the mobile handoff.

Run: `npm test -- --run src/components/ui/mobile-sheet.test.tsx src/components/ui/responsive-form-dialog.test.tsx src/components/lists/list-detail-view.test.tsx`

Expected: FAIL on missing handoff props/state.

- [ ] **Step 2: Extend primitives without changing existing defaults**

Add optional MobileSheet props:

```ts
restoreFocusOnClose?: boolean; // default true
returnFocusRef?: RefObject<HTMLElement | null>;
onAnimationEnd?: (open: boolean) => void;
```

Forward `onAnimationEnd` to `Drawer.Root`. In `onCloseAutoFocus`, always prevent Radix default, then focus `returnFocusRef.current ?? openerRef.current` only when restoration is enabled. Forward `returnFocusRef` through `ResponsiveFormDialog` on mobile; existing callers retain current behavior.

- [ ] **Step 3: Sequence the handoff in `ListDetailView`**

Keep an `optionsButtonRef`, `managerOpen`, and `managerHandoffPending`. Manage click sets pending then closes Options. Options passes `restoreFocusOnClose={!managerHandoffPending}` and in `onAnimationEnd(false)` opens manager and clears pending. Manager receives `returnFocusRef={optionsButtonRef}`. The Options and manager open booleans are never true together.

- [ ] **Step 4: Run and commit**

Run the Step 1 command; expect PASS.

```bash
git add src/components/ui/mobile-sheet.tsx src/components/ui/mobile-sheet.test.tsx \
  src/components/ui/responsive-form-dialog.tsx src/components/ui/responsive-form-dialog.test.tsx \
  src/components/lists/list-detail-view.tsx src/components/lists/list-detail-view.test.tsx
git commit -m "fix(lists): hand off mobile category manager without stacking"
```

## Task 12: Add inline create-and-select and stale item recovery

**Files:**
- Modify: `frontend/src/components/lists/list-item-sheet.tsx`
- Create: `frontend/src/components/lists/list-item-sheet.test.tsx`

- [ ] **Step 1: Write failing item-sheet tests**

Cover every kind showing category selection; inline New Category expands inside the existing sheet; successful create selects the returned ID without changing text/other draft fields; create failure preserves draft and category name; later item-save failure keeps the created category and selection. Offline hides/disables New Category with explanation.

Add four item-save `404` tests: refetch returns list 404 -> list missing message; refetch succeeds but edited item missing -> item missing message; selected category missing -> set only `categoryId` to null and request save again; list/item/category still present -> preserve draft and show original error.

Run: `npm test -- --run src/components/lists/list-item-sheet.test.tsx`

Expected: FAIL because General/inline/recovery paths are absent.

- [ ] **Step 2: Implement inline creation inside React Hook Form**

Remove `supportsCategories`. Keep inline name state outside `form.reset` so only opening a new item cycle resets it. Call `useCreateListCategory`; on success `form.setValue("categoryId", response.data.id, { shouldDirty: true })`, clear inline name, and leave all item fields intact. Do not auto-change list display mode.

- [ ] **Step 3: Implement deterministic `404` recovery**

On item mutation error, act only when `ApiException.isApiException(error)`, `status === 404`, and selected category is non-null. Fetch `listsService.getList(list.id)` and replace `listsKeys.detail(list.id)` on success. Then evaluate in order: list fetch 404; edited item absent; selected category absent; otherwise original error. Category absence sets `categoryId` null and a form-root message asking the user to save again. Never discard `text` or completion state.

- [ ] **Step 4: Run and commit**

Run the Step 1 command; expect PASS.

```bash
git add src/components/lists/list-item-sheet.tsx src/components/lists/list-item-sheet.test.tsx
git commit -m "feat(lists): create categories inline without losing item drafts"
```

## Task 13: Enforce released-contract CI and complete browser verification

**Files:**
- Create: `frontend/.github/scripts/resolve-backend-version.sh`
- Create: `frontend/.github/scripts/resolve-backend-version.test.sh`
- Modify: `frontend/.github/workflows/ci.yml`
- Modify: `frontend/docker-compose.e2e.yml`
- Modify: `frontend/e2e/mobile-lists.spec.ts`
- Modify: `frontend/e2e/offline-persistence.spec.ts`

- [ ] **Step 1: Write a failing sourceable resolver test**

Mirror the BE parser contract: valid `v1.2.3` prints `1.2.3`; absent/null/blank/malformed tags return non-zero. The script's executable path uses `GITHUB_TOKEN` to call the latest-release API and prints only the version on stdout.

Run: `bash .github/scripts/resolve-backend-version.test.sh`

Expected: FAIL because the resolver does not exist.

- [ ] **Step 2: Make FE release resolution fail closed**

Replace the warning/fallback with:

```bash
if [ -z "$BE_VERSION" ]; then
  echo "::error::No published backend release could be resolved"
  exit 1
fi
echo "BE_IMAGE_TAG=${BE_VERSION}" >> "$GITHUB_ENV"
echo "Using published BE image tag: ${BE_VERSION}"
```

Change Compose to:

```yaml
image: ghcr.io/joe-bor/family-hub-api:${BE_IMAGE_TAG:?BE_IMAGE_TAG must be a published release}
```

The workflow obtains `BE_VERSION=$(./.github/scripts/resolve-backend-version.sh)`; shell failure stops the step. After the parser test, run `! rg -n 'BE_VERSION:-latest|BE_IMAGE_TAG:-latest' .github/workflows/ci.yml docker-compose.e2e.yml` to prove no fallback remains.

Run: `bash .github/scripts/resolve-backend-version.test.sh`

Expected: PASS.

- [ ] **Step 3: Expand released-contract mobile Lists E2E**

Against the published BE image, execute one coherent family flow:

1. Create General list and confirm flat/empty catalog.
2. Open Add Item, create `Documents` inline, keep the item draft, save assigned item.
3. Open Options then manager; prove Options is gone, manager is full-height, and hardware back dismisses only the manager.
4. Reopen, create a second category, reorder with arrows, Save once, reload, and verify order.
5. Opt the list into grouped mode; verify empty groups hidden and assigned item visible.
6. Create/open a second General list and observe the shared category/order without automatic grouping.
7. Rename and verify both list details converge.
8. Delete a used category and verify authoritative success count plus `Uncategorized`.
9. Delete the final category and verify all grouped General lists become flat; recreate one and verify they remain flat.

Use request counting for reorder to assert arrows send zero PUTs and Save sends exactly one.

Measure both move controls with Playwright `boundingBox()` and assert width and height are each at least 44 CSS pixels. Focus a move control with the keyboard, activate it, and assert focus plus the live position announcement remain correct.

- [ ] **Step 4: Expand offline E2E**

Online, load a grouped General list with a new category option, wait for persistence, go offline/reload, and verify headings/items remain readable while Manage Categories and New Category are unavailable. A validator unit fixture supplies the legacy extra `seeded`; browser E2E covers the new released shape.

- [ ] **Step 5: Run complete FE verification**

Run in order:

```bash
npm run lint
npm test -- --run
npm run build
BE_IMAGE_TAG=<published-semver> docker compose -f docker-compose.e2e.yml up -d --wait
npm run test:e2e -- e2e/mobile-lists.spec.ts
npm run test:e2e -- e2e/offline-persistence.spec.ts
docker compose -f docker-compose.e2e.yml down
```

Expected: lint, all Vitest tests, build, released-contract mobile Lists E2E, and offline persistence E2E pass. Record the exact BE semver in the FE PR.

- [ ] **Step 6: Commit**

```bash
git add .github/workflows/ci.yml .github/scripts/resolve-backend-version.sh \
  .github/scripts/resolve-backend-version.test.sh docker-compose.e2e.yml \
  e2e/mobile-lists.spec.ts e2e/offline-persistence.spec.ts
git commit -m "test(lists): verify managed categories against released backend"
```

---

## Final contract audit before opening or merging PRs

- [ ] Every approved product decision in the spec maps to a named BE or FE test above.
- [ ] V12 is unchanged; clean and V16-upgrade migration tests both pass; `seeded` remains physical but absent from contracts.
- [ ] Scope locks cover category writes, list default/group changes, and non-null item assignment, including an empty catalog.
- [ ] Family/kind isolation and exact counts are proven through authenticated PostgreSQL integration tests.
- [ ] Delete rollback, duplicate race, dense concurrent order, and stale reorder conflict are observable integration tests rather than transaction-annotation assertions.
- [ ] General is flat initially; empty catalogs cannot group; first/recreated categories never regroup.
- [ ] FE manager, inline form, stale `404`, cache convergence, offline, mobile handoff, hardware back, keyboard, announcements, and touch targets have explicit coverage.
- [ ] FE CI and BE deploy resolve only published releases and fail closed; no execution path substitutes mutable `latest`.
- [ ] BE release exists before FE implementation/E2E begins, and the FE PR names the tested semver.
- [ ] Delivery Issues link Story, Spec, and Plan and copy these non-negotiables before implementation starts.
