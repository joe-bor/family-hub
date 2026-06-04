# Recipes Module Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a standalone top-level `Recipes` module with family-scoped backend persistence, manual create, URL import with auto-save, cook-first recipe detail, favorites/tags/light filters, and the shared `Add to Meals` intent contract that the later `Meals` plan will consume.

**Architecture:** Implement the recipe domain first in `backend/family-hub-api/` as a family-owned recipe aggregate with ordered ingredients/instructions and lightweight tags, plus an import service that scrapes and normalizes a remote recipe into the same model. On the frontend in `frontend/`, add a new `recipes` module to the shell, load recipe data through TanStack Query, build a mobile-first recipe library/detail/add flow, and store a cross-module placement draft in the app shell so `Meals` can later pick it up without redesigning the contract.

**Tech Stack:** Spring Boot 4, Java 21, JPA/Hibernate, Flyway, Maven, Jsoup, React 19, TypeScript, TanStack Query, React Hook Form, Zod, Vitest, MSW, Playwright.

**Spec:** `docs/superpowers/specs/2026-06-01-meals-and-recipes-foundation-design.md`

---

## Execution Order

1. Backend recipe schema and aggregate model
2. Backend `/api/recipes` contract and URL import behavior
3. Frontend recipe types, hooks, mocks, and shell module registration
4. Frontend recipe library, detail, manual create, and URL import UX
5. Cross-module `Add to Meals` draft contract and final verification against the shared spec

Per root shipping rules, live FE verification that depends on real `/api/recipes` behavior should wait for a published backend release. Mock-backed frontend work can start earlier, but real FE verification should target the released backend contract, not unreleased backend `main`.

Handoff status as of 2026-06-04: backend Recipes issue [family-hub-api#52](https://github.com/joe-bor/family-hub-api/issues/52) and PR [family-hub-api#53](https://github.com/joe-bor/family-hub-api/pull/53) are merged and released in [family-hub-api v1.5.0](https://github.com/joe-bor/family-hub-api/releases/tag/v1.5.0). Frontend Recipes issue [FamilyHub#183](https://github.com/joe-bor/FamilyHub/issues/183) consumes that release for real-backend verification.

## Execution Contract

- `Recipes` is a standalone top-level module in this phase, not hidden inside `Meals`.
- Recipes are family-scoped reusable content, not per-user data.
- A recipe can be created manually with only a name required.
- A recipe can be imported from a URL; successful imports auto-save immediately.
- Recipe detail must be cook-first: image, title, ingredients, instructions.
- `Favorites` are simple stars with functional importance in search/pickers.
- Tags and filters are intentionally lightweight, not a heavy taxonomy system.
- URL import belongs inside the add flow, not as a peer action on the library home surface.
- `Recipes` must expose a shared `Add to Meals` intent, but it must not implement the weekly planner itself. The actual slot-picking and meal placement UX belongs to the later `Meals` plan.
- The shared handoff contract must support both directions:
  - `Recipes -> Meals` for adding an existing recipe to a future meal slot
  - `Meals -> Recipes -> back to Meals` for turning typed meal text into a real reusable recipe and resuming planning
- This plan may define shared handoff state for `Meals`, but it must not absorb the `Meals` board, composer, or snapshot persistence work.
- Shared handoff wire values must stay lowercase (`breakfast`, `lunch`, `dinner`) so frontend types and backend JSON contracts do not drift.
- `Recipes` owns the first implementation of the Sunday week-start helper used by `Add to Meals`; the later `Meals` plan consumes and extends that helper.
- URL import must fetch user-provided URLs with server-side guardrails: public `http`/`https` only, private-network rejection, bounded redirects, short timeouts, and a response-size cap.
- Recipe detail must include editable favorite and edit actions for saved recipes, not only create-time favorite state.

## Execution Issue Mapping

- Backend execution issue in `backend/family-hub-api`: [family-hub-api#52](https://github.com/joe-bor/family-hub-api/issues/52), closed by [PR #53](https://github.com/joe-bor/family-hub-api/pull/53). Contract: shipped schema, `/api/recipes`, import guardrails, update/favorite support, and backend tests in release [v1.5.0](https://github.com/joe-bor/family-hub-api/releases/tag/v1.5.0).
- Frontend execution issue in `frontend`: [FamilyHub#183](https://github.com/joe-bor/FamilyHub/issues/183). Contract: consumes backend release [v1.5.0](https://github.com/joe-bor/family-hub-api/releases/tag/v1.5.0) for live verification, while mock-backed unit work may start earlier.
- Root docs issue, if needed: spec/plan reconciliation only. Do not implement production code from the root workspace.

## File Structure

### Root docs / orchestration

- Create: `docs/superpowers/plans/2026-06-01-recipes-module-foundation.md`
- Read only during implementation handoff: `docs/superpowers/specs/2026-06-01-meals-and-recipes-foundation-design.md`

### Backend (`backend/family-hub-api/`)

Expected create / modify set:

- Modify: `pom.xml`
- Create: `src/main/resources/db/migration/V15__create_recipe_tables.sql`
- Create: `src/main/java/com/familyhub/demo/model/Recipe.java`
- Create: `src/main/java/com/familyhub/demo/model/RecipeIngredient.java`
- Create: `src/main/java/com/familyhub/demo/model/RecipeInstruction.java`
- Create: `src/main/java/com/familyhub/demo/model/RecipeTag.java`
- Create: `src/main/java/com/familyhub/demo/repository/RecipeRepository.java`
- Create: `src/main/java/com/familyhub/demo/dto/CreateRecipeRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/UpdateRecipeRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/ImportRecipeRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/RecipeSummaryResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/RecipeDetailResponse.java`
- Create: `src/main/java/com/familyhub/demo/mapper/RecipeMapper.java`
- Create: `src/main/java/com/familyhub/demo/service/ImportedRecipe.java`
- Create: `src/main/java/com/familyhub/demo/service/RecipeImportService.java`
- Create: `src/main/java/com/familyhub/demo/service/RecipeService.java`
- Create: `src/main/java/com/familyhub/demo/controller/RecipeController.java`
- Modify: `src/test/java/com/familyhub/demo/TestDataFactory.java`
- Create: `src/test/java/com/familyhub/demo/repository/RecipeRepositoryTest.java`
- Create: `src/test/java/com/familyhub/demo/service/RecipeServiceTest.java`
- Create: `src/test/java/com/familyhub/demo/service/RecipeImportServiceTest.java`
- Create: `src/test/java/com/familyhub/demo/controller/RecipeControllerTest.java`
- Create: `src/test/java/com/familyhub/demo/integration/RecipeIntegrationTest.java`

### Frontend (`frontend/`)

Expected create / modify set:

- Modify: `src/App.tsx`
- Modify: `src/stores/app-store.ts`
- Modify: `src/stores/index.ts`
- Modify: `src/components/shared/mobile-bottom-nav.tsx`
- Modify: `src/components/shared/navigation-tabs.tsx`
- Modify: `src/components/shared/mobile-bottom-nav.test.tsx`
- Modify: `src/components/shared/navigation-tabs.test.tsx`
- Create: `src/lib/types/recipes.ts`
- Modify: `src/lib/types/index.ts`
- Modify: `src/lib/time-utils.ts`
- Modify: `src/lib/time-utils.test.ts`
- Create: `src/api/services/recipes.service.ts`
- Modify: `src/api/services/index.ts`
- Create: `src/api/hooks/use-recipes.ts`
- Modify: `src/api/hooks/index.ts`
- Modify: `src/api/index.ts`
- Create: `src/lib/validations/recipes.ts`
- Modify: `src/lib/validations/index.ts`
- Create: `src/lib/validations/recipes.test.ts`
- Modify: `src/test/mocks/handlers.ts`
- Modify: `src/test/mocks/server.ts`
- Create: `src/test/fixtures/recipes.ts`
- Create: `src/components/recipes-view.tsx`
- Create: `src/components/recipes/recipe-library-card.tsx`
- Create: `src/components/recipes/recipe-filter-bar.tsx`
- Create: `src/components/recipes/recipe-form.tsx`
- Create: `src/components/recipes/recipe-create-sheet.tsx`
- Create: `src/components/recipes/recipe-edit-sheet.tsx`
- Create: `src/components/recipes/recipe-import-sheet.tsx`
- Create: `src/components/recipes/recipe-detail-view.tsx`
- Create: `src/api/hooks/use-recipes.test.tsx`
- Create: `src/components/recipes-view.test.tsx`
- Create: `e2e/mobile-recipes.spec.ts`

This structure keeps backend recipe persistence and import logic isolated in one aggregate, while the frontend keeps network state in `@/api`, module UI in `src/components/recipes/`, and the cross-module meal-placement handoff in the app shell store instead of burying it in a placeholder meals store.

## Task 1: Add Backend Recipe Schema And Aggregate Model

**Files:**
- Modify: `backend/family-hub-api/pom.xml`
- Create: `backend/family-hub-api/src/main/resources/db/migration/V15__create_recipe_tables.sql`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/Recipe.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/RecipeIngredient.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/RecipeInstruction.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/RecipeTag.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/RecipeRepository.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/repository/RecipeRepositoryTest.java`

- [ ] **Step 1: Write the failing backend repository tests for ordered aggregate persistence and family scoping**

```java
@Test
void saveRecipe_persistsOrderedIngredientsInstructionsAndTags() {
    Recipe recipe = recipeFor(family, "Sheet Pan Gnocchi");
    recipe.getIngredients().add(ingredient(recipe, 0, "1 lb shelf-stable gnocchi"));
    recipe.getIngredients().add(ingredient(recipe, 1, "2 cups cherry tomatoes"));
    recipe.getInstructions().add(instruction(recipe, 0, "Heat oven to 425F"));
    recipe.getInstructions().add(instruction(recipe, 1, "Roast for 20 minutes"));
    recipe.getTags().add(tag(recipe, 0, "dinner"));
    recipe.getTags().add(tag(recipe, 1, "weeknight"));

    UUID id = recipeRepository.saveAndFlush(recipe).getId();
    entityManager.clear();

    Recipe saved = recipeRepository.findByIdAndFamily(id, family).orElseThrow();

    assertThat(saved.getIngredients()).extracting(RecipeIngredient::getText)
            .containsExactly("1 lb shelf-stable gnocchi", "2 cups cherry tomatoes");
    assertThat(saved.getInstructions()).extracting(RecipeInstruction::getText)
            .containsExactly("Heat oven to 425F", "Roast for 20 minutes");
    assertThat(saved.getTags()).extracting(RecipeTag::getName)
            .containsExactly("dinner", "weeknight");
}

@Test
void findByIdAndFamily_doesNotReturnAnotherFamiliesRecipe() {
    Recipe otherFamilyRecipe = recipeRepository.saveAndFlush(recipeFor(otherFamily, "Other Soup"));

    assertThat(recipeRepository.findByIdAndFamily(otherFamilyRecipe.getId(), family)).isEmpty();
}
```

- [ ] **Step 2: Run the recipe repository test to verify it fails because the recipe domain does not exist yet**

Run:

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
./mvnw -Dtest=RecipeRepositoryTest test
```

Expected: FAIL with compilation errors for missing `Recipe*` classes and `RecipeRepository`.

- [ ] **Step 3: Add the dependency, migration, entities, and repository**

```xml
<dependency>
    <groupId>org.jsoup</groupId>
    <artifactId>jsoup</artifactId>
    <version>1.18.3</version>
</dependency>
```

```sql
CREATE TABLE recipe (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL,
    title VARCHAR(160) NOT NULL,
    image_url TEXT,
    note TEXT,
    source_url TEXT,
    favorite BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP(6) DEFAULT now(),
    updated_at TIMESTAMP(6) DEFAULT now(),
    CONSTRAINT fk_recipe_family
        FOREIGN KEY (family_id) REFERENCES family(id) ON DELETE CASCADE
);

CREATE TABLE recipe_ingredient (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipe_id UUID NOT NULL,
    sort_order INTEGER NOT NULL,
    text VARCHAR(500) NOT NULL,
    CONSTRAINT fk_recipe_ingredient_recipe
        FOREIGN KEY (recipe_id) REFERENCES recipe(id) ON DELETE CASCADE,
    CONSTRAINT uk_recipe_ingredient_recipe_sort UNIQUE (recipe_id, sort_order)
);

CREATE TABLE recipe_instruction (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipe_id UUID NOT NULL,
    sort_order INTEGER NOT NULL,
    text TEXT NOT NULL,
    CONSTRAINT fk_recipe_instruction_recipe
        FOREIGN KEY (recipe_id) REFERENCES recipe(id) ON DELETE CASCADE,
    CONSTRAINT uk_recipe_instruction_recipe_sort UNIQUE (recipe_id, sort_order)
);

CREATE TABLE recipe_tag (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipe_id UUID NOT NULL,
    name VARCHAR(60) NOT NULL,
    sort_order INTEGER NOT NULL,
    CONSTRAINT fk_recipe_tag_recipe
        FOREIGN KEY (recipe_id) REFERENCES recipe(id) ON DELETE CASCADE,
    CONSTRAINT uk_recipe_tag_recipe_name UNIQUE (recipe_id, name)
);

CREATE INDEX idx_recipe_family_created_at ON recipe(family_id, created_at DESC);
CREATE INDEX idx_recipe_family_favorite ON recipe(family_id, favorite);
```

```java
@Entity
@Table(name = "recipe")
@Getter
@Setter
public class Recipe {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @ManyToOne
    @JoinColumn(name = "family_id", nullable = false)
    private Family family;

    @Column(nullable = false, length = 160)
    private String title;

    private String imageUrl;

    @Column(columnDefinition = "TEXT")
    private String note;

    private String sourceUrl;

    @Column(nullable = false)
    private boolean favorite;

    @OneToMany(mappedBy = "recipe", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("sortOrder ASC")
    private List<RecipeIngredient> ingredients = new ArrayList<>();

    @OneToMany(mappedBy = "recipe", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("sortOrder ASC")
    private List<RecipeInstruction> instructions = new ArrayList<>();

    @OneToMany(mappedBy = "recipe", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("sortOrder ASC")
    private List<RecipeTag> tags = new ArrayList<>();

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

```java
public interface RecipeRepository extends JpaRepository<Recipe, UUID> {
    List<Recipe> findByFamilyOrderByUpdatedAtDesc(Family family);
    Optional<Recipe> findByIdAndFamily(UUID id, Family family);
}
```

- [ ] **Step 4: Run the backend recipe repository test to verify the aggregate foundation now passes**

Run:

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
./mvnw -Dtest=RecipeRepositoryTest test
```

Expected: PASS for ordered child persistence and family-scoped repository lookup.

- [ ] **Step 5: Commit**

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
git add pom.xml \
  src/main/resources/db/migration/V15__create_recipe_tables.sql \
  src/main/java/com/familyhub/demo/model/Recipe.java \
  src/main/java/com/familyhub/demo/model/RecipeIngredient.java \
  src/main/java/com/familyhub/demo/model/RecipeInstruction.java \
  src/main/java/com/familyhub/demo/model/RecipeTag.java \
  src/main/java/com/familyhub/demo/repository/RecipeRepository.java \
  src/test/java/com/familyhub/demo/repository/RecipeRepositoryTest.java
git commit -m "feat(recipes): add recipe aggregate foundation"
```

## Task 2: Add Backend Recipe DTOs, Controller, And URL Import

**Files:**
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/CreateRecipeRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/UpdateRecipeRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ImportRecipeRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/RecipeSummaryResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/RecipeDetailResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/mapper/RecipeMapper.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/ImportedRecipe.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/RecipeImportService.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/RecipeService.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/controller/RecipeController.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/TestDataFactory.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/service/RecipeServiceTest.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/service/RecipeImportServiceTest.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/controller/RecipeControllerTest.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/integration/RecipeIntegrationTest.java`

- [ ] **Step 1: Write the failing service, controller, and import-service tests for list/detail/create/update/import**

```java
@Test
void createRecipe_persistsOrderedIngredientsInstructionsTagsAndFavoriteState() {
    CreateRecipeRequest request = new CreateRecipeRequest(
            "Sheet Pan Gnocchi",
            "https://cdn.example.com/gnocchi.jpg",
            List.of("1 lb shelf-stable gnocchi", "2 cups cherry tomatoes"),
            List.of("Heat oven to 425F", "Roast for 20 minutes"),
            "Weeknight favorite",
            "https://example.com/gnocchi",
            List.of("dinner", "weeknight"),
            true
    );

    RecipeDetailResponse response = recipeService.createRecipe(request, family);

    assertThat(response.title()).isEqualTo("Sheet Pan Gnocchi");
    assertThat(response.favorite()).isTrue();
    assertThat(response.ingredients()).containsExactly(
            "1 lb shelf-stable gnocchi",
            "2 cups cherry tomatoes"
    );
    assertThat(response.instructions()).containsExactly(
            "Heat oven to 425F",
            "Roast for 20 minutes"
    );
    assertThat(response.tags()).containsExactly("dinner", "weeknight");
}

@Test
void updateRecipe_canToggleFavoriteAndEditRecipeFields() {
    Recipe recipe = recipeRepository.saveAndFlush(TestDataFactory.recipe(family, "Old Soup"));

    RecipeDetailResponse response = recipeService.updateRecipe(
            recipe.getId(),
            new UpdateRecipeRequest(
                    "New Soup",
                    null,
                    List.of("1 cup broth"),
                    List.of("Simmer"),
                    "Updated note",
                    null,
                    List.of("lunch"),
                    true
            ),
            family
    );

    assertThat(response.title()).isEqualTo("New Soup");
    assertThat(response.favorite()).isTrue();
    assertThat(response.ingredients()).containsExactly("1 cup broth");
}

@Test
void getRecipes_returnsRecentRecipesForFamilyOnly() {
    Recipe oldest = recipeRepository.saveAndFlush(TestDataFactory.recipe(family, "Oldest Soup"));
    Recipe newest = recipeRepository.saveAndFlush(TestDataFactory.recipe(family, "Newest Soup"));
    recipeRepository.saveAndFlush(TestDataFactory.recipe(otherFamily, "Other Soup"));

    List<RecipeSummaryResponse> recipes = recipeService.getRecipes(family);

    assertThat(recipes).extracting(RecipeSummaryResponse::title)
            .containsSubsequence("Newest Soup", "Oldest Soup");
    assertThat(recipes).extracting(RecipeSummaryResponse::title)
            .doesNotContain("Other Soup");
}

@Test
@WithMockFamily
void getRecipes_returns200() throws Exception {
    given(recipeService.getRecipes(any(Family.class)))
            .willReturn(List.of(TestDataFactory.recipeSummaryResponse("Sheet Pan Gnocchi")));

    mockMvc.perform(get("/api/recipes"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data[0].title").value("Sheet Pan Gnocchi"));
}

@Test
@WithMockFamily
void importRecipe_returns201() throws Exception {
    given(recipeService.importRecipe(any(), any(Family.class)))
            .willReturn(TestDataFactory.recipeDetailResponse("Imported Pasta"));

    mockMvc.perform(post("/api/recipes/import")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            {
                              "url": "https://example.com/pasta"
                            }
                            """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.data.title").value("Imported Pasta"));
}

@Test
void importDocument_extractsJsonLdRecipeFields() {
    String html = """
            <html>
              <head>
                <script type="application/ld+json">
                {
                  "@context":"https://schema.org",
                  "@type":"Recipe",
                  "name":"Weeknight Tacos",
                  "image":"https://cdn.example.com/tacos.jpg",
                  "recipeIngredient":["1 lb beef","8 tortillas"],
                  "recipeInstructions":["Brown beef","Serve in tortillas"]
                }
                </script>
              </head>
            </html>
            """;

    ImportedRecipe imported = recipeImportService.parseHtml("https://example.com/tacos", html);

    assertThat(imported.title()).isEqualTo("Weeknight Tacos");
    assertThat(imported.ingredients()).containsExactly("1 lb beef", "8 tortillas");
}

@Test
void importRecipe_rejectsPrivateNetworkTargetsBeforeFetching() {
    ImportRecipeRequest request = new ImportRecipeRequest("http://127.0.0.1:8080/private");

    assertThatThrownBy(() -> recipeService.importRecipe(request, family))
            .isInstanceOf(BadRequestException.class)
            .hasMessageContaining("Could not import recipe");
}
```

- [ ] **Step 2: Run the focused backend tests to verify the HTTP contract and import flow are still missing**

Run:

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
./mvnw -Dtest=RecipeServiceTest,RecipeControllerTest,RecipeImportServiceTest test
```

Expected: FAIL with missing DTO/controller/service/import types.

- [ ] **Step 3: Implement the DTOs, mapper, service, import parser, and controller**

The import service must validate and fetch user-provided URLs before parsing:

- accept only `http` and `https`
- reject loopback, private, link-local, and otherwise non-public resolved addresses
- apply short connection/read timeouts
- follow only a small bounded redirect chain and re-check every redirect target
- cap the downloaded response body before passing HTML to Jsoup
- return the same user-facing import failure style for blocked URLs, network failures, oversized responses, and parse failures

```java
public record CreateRecipeRequest(
        @NotBlank @Size(max = 160) String title,
        String imageUrl,
        List<@NotBlank @Size(max = 500) String> ingredients,
        List<@NotBlank String> instructions,
        String note,
        String sourceUrl,
        List<@NotBlank @Size(max = 60) String> tags,
        Boolean favorite
) {}
```

```java
public record RecipeSummaryResponse(
        UUID id,
        String title,
        String imageUrl,
        boolean favorite,
        List<String> tags,
        LocalDateTime updatedAt
) {}
```

```java
@RestController
@RequestMapping("/api/recipes")
@RequiredArgsConstructor
public class RecipeController {
    private final RecipeService recipeService;

    @GetMapping
    public ResponseEntity<ApiResponse<List<RecipeSummaryResponse>>> getRecipes(@AuthenticationPrincipal Family family) {
        return ResponseEntity.ok(new ApiResponse<>(recipeService.getRecipes(family), ""));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<RecipeDetailResponse>> getRecipe(
            @PathVariable UUID id,
            @AuthenticationPrincipal Family family
    ) {
        return ResponseEntity.ok(new ApiResponse<>(recipeService.getRecipe(id, family), ""));
    }

    @PostMapping
    public ResponseEntity<ApiResponse<RecipeDetailResponse>> createRecipe(
            @Valid @RequestBody CreateRecipeRequest request,
            @AuthenticationPrincipal Family family
    ) {
        RecipeDetailResponse response = recipeService.createRecipe(request, family);
        return ResponseEntity.created(URI.create("/api/recipes/" + response.id()))
                .body(new ApiResponse<>(response, "Recipe created successfully"));
    }

    @PatchMapping("/{id}")
    public ResponseEntity<ApiResponse<RecipeDetailResponse>> updateRecipe(
            @PathVariable UUID id,
            @Valid @RequestBody UpdateRecipeRequest request,
            @AuthenticationPrincipal Family family
    ) {
        return ResponseEntity.ok(new ApiResponse<>(recipeService.updateRecipe(id, request, family), "Recipe updated successfully"));
    }

    @PostMapping("/import")
    public ResponseEntity<ApiResponse<RecipeDetailResponse>> importRecipe(
            @Valid @RequestBody ImportRecipeRequest request,
            @AuthenticationPrincipal Family family
    ) {
        RecipeDetailResponse response = recipeService.importRecipe(request, family);
        return ResponseEntity.created(URI.create("/api/recipes/" + response.id()))
                .body(new ApiResponse<>(response, "Recipe imported successfully"));
    }
}
```

```java
public ImportedRecipe parseHtml(String sourceUrl, String html) {
    Document document = Jsoup.parse(html, sourceUrl);
    Element script = document.selectFirst("script[type=application/ld+json]");
    if (script == null) {
        throw new BadRequestException("Could not extract recipe data from URL.");
    }

    JsonNode root = objectMapper.readTree(script.data());
    JsonNode recipeNode = root.isArray()
            ? StreamSupport.stream(root.spliterator(), false)
                    .filter(node -> "Recipe".equals(node.path("@type").asText()))
                    .findFirst()
                    .orElse(root.get(0))
            : root;

    String title = recipeNode.path("name").asText(document.title()).trim();
    if (title.isBlank()) {
        throw new BadRequestException("Could not extract recipe title from URL.");
    }

    return new ImportedRecipe(
            title,
            firstImage(recipeNode),
            toStringList(recipeNode.path("recipeIngredient")),
            toInstructionList(recipeNode.path("recipeInstructions")),
            null,
            sourceUrl,
            List.of(),
            false
    );
}
```

- [ ] **Step 4: Run the backend contract tests and the full recipe integration test**

Run:

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
./mvnw -Dtest=RecipeServiceTest,RecipeControllerTest,RecipeImportServiceTest,RecipeIntegrationTest test
```

Expected: PASS for list/detail/create/update/favorite/import flows and guarded JSON-LD import parsing.

- [ ] **Step 5: Commit**

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
git add src/main/java/com/familyhub/demo/dto/CreateRecipeRequest.java \
  src/main/java/com/familyhub/demo/dto/UpdateRecipeRequest.java \
  src/main/java/com/familyhub/demo/dto/ImportRecipeRequest.java \
  src/main/java/com/familyhub/demo/dto/RecipeSummaryResponse.java \
  src/main/java/com/familyhub/demo/dto/RecipeDetailResponse.java \
  src/main/java/com/familyhub/demo/mapper/RecipeMapper.java \
  src/main/java/com/familyhub/demo/service/ImportedRecipe.java \
  src/main/java/com/familyhub/demo/service/RecipeImportService.java \
  src/main/java/com/familyhub/demo/service/RecipeService.java \
  src/main/java/com/familyhub/demo/controller/RecipeController.java \
  src/test/java/com/familyhub/demo/TestDataFactory.java \
  src/test/java/com/familyhub/demo/service/RecipeServiceTest.java \
  src/test/java/com/familyhub/demo/service/RecipeImportServiceTest.java \
  src/test/java/com/familyhub/demo/controller/RecipeControllerTest.java \
  src/test/java/com/familyhub/demo/integration/RecipeIntegrationTest.java
git commit -m "feat(recipes): add recipe api and import flow"
```

## Task 3: Add Frontend Recipe Contracts, Hooks, Validation, And Mocks

**Files:**
- Create: `frontend/src/lib/types/recipes.ts`
- Modify: `frontend/src/lib/types/index.ts`
- Create: `frontend/src/api/services/recipes.service.ts`
- Modify: `frontend/src/api/services/index.ts`
- Create: `frontend/src/api/hooks/use-recipes.ts`
- Modify: `frontend/src/api/hooks/index.ts`
- Modify: `frontend/src/api/index.ts`
- Create: `frontend/src/lib/validations/recipes.ts`
- Modify: `frontend/src/lib/validations/index.ts`
- Create: `frontend/src/lib/validations/recipes.test.ts`
- Modify: `frontend/src/test/mocks/handlers.ts`
- Modify: `frontend/src/test/mocks/server.ts`
- Create: `frontend/src/test/fixtures/recipes.ts`
- Create: `frontend/src/api/hooks/use-recipes.test.tsx`

- [ ] **Step 1: Write the failing frontend tests for recipe queries, mutations, and validation**

```tsx
it("loads recipe summaries for the library", async () => {
  const { result } = renderHook(() => useRecipes(), { wrapper: createWrapper() });

  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data?.data[0].title).toBe("Sheet Pan Gnocchi");
});

it("creates a recipe and seeds detail cache", async () => {
  const { result } = renderHook(() => useCreateRecipe(), { wrapper: createWrapper() });

  result.current.mutate({
    title: "Quick Oats",
    imageUrl: null,
    ingredients: [],
    instructions: [],
    note: null,
    sourceUrl: null,
    tags: [],
    favorite: false,
  });

  await waitFor(() => expect(result.current.isSuccess).toBe(true));
});
```

```ts
it("requires a recipe title", () => {
  const parsed = recipeFormSchema.safeParse({
    title: "",
    ingredients: [],
    instructions: [],
    tags: [],
  });

  expect(parsed.success).toBe(false);
});
```

- [ ] **Step 2: Run the frontend tests to verify the recipe contract layer is still missing**

Run:

```bash
cd /Users/joe.bor/code/family-hub/frontend
npm test -- --run src/api/hooks/use-recipes.test.tsx src/lib/validations/recipes.test.ts
```

Expected: FAIL with missing recipe types, hooks, and schema.

- [ ] **Step 3: Add recipe types, services, hooks, validation, and MSW fixtures**

```ts
export interface RecipeSummary {
  id: string;
  title: string;
  imageUrl: string | null;
  favorite: boolean;
  tags: string[];
  updatedAt: string;
}

export interface RecipeDetail extends RecipeSummary {
  ingredients: string[];
  instructions: string[];
  note: string | null;
  sourceUrl: string | null;
}

export interface CreateRecipeRequest {
  title: string;
  imageUrl: string | null;
  ingredients: string[];
  instructions: string[];
  note: string | null;
  sourceUrl: string | null;
  tags: string[];
  favorite: boolean;
}

export interface UpdateRecipeRequest extends CreateRecipeRequest {}
```

```ts
export const recipesService = {
  getRecipes() {
    return httpClient.get<ApiResponse<RecipeSummary[]>>("/recipes");
  },

  getRecipe(id: string) {
    return httpClient.get<ApiResponse<RecipeDetail>>(`/recipes/${id}`);
  },

  createRecipe(request: CreateRecipeRequest) {
    return httpClient.post<ApiResponse<RecipeDetail>>("/recipes", request);
  },

  updateRecipe(id: string, request: UpdateRecipeRequest) {
    return httpClient.patch<ApiResponse<RecipeDetail>>(`/recipes/${id}`, request);
  },

  importRecipe(url: string) {
    return httpClient.post<ApiResponse<RecipeDetail>>("/recipes/import", { url });
  },
};
```

```ts
export const recipeFormSchema = z.object({
  title: z.string().trim().min(1, "Recipe name is required").max(160),
  imageUrl: z.string().trim().url().nullable().or(z.literal("")).transform((value) => value || null),
  ingredients: z.array(z.string().trim().min(1)).default([]),
  instructions: z.array(z.string().trim().min(1)).default([]),
  note: z.string().trim().nullable().or(z.literal("")).transform((value) => value || null),
  sourceUrl: z.string().trim().url().nullable().or(z.literal("")).transform((value) => value || null),
  tags: z.array(z.string().trim().min(1).max(60)).default([]),
  favorite: z.boolean().default(false),
});
```

- [ ] **Step 4: Run the focused frontend tests to verify the contract layer passes**

Run:

```bash
cd /Users/joe.bor/code/family-hub/frontend
npm test -- --run src/api/hooks/use-recipes.test.tsx src/lib/validations/recipes.test.ts
```

Expected: PASS for query/mutation cache behavior and required-title validation.

- [ ] **Step 5: Commit**

```bash
cd /Users/joe.bor/code/family-hub/frontend
git add src/lib/types/recipes.ts \
  src/lib/types/index.ts \
  src/api/services/recipes.service.ts \
  src/api/services/index.ts \
  src/api/hooks/use-recipes.ts \
  src/api/hooks/index.ts \
  src/api/index.ts \
  src/lib/validations/recipes.ts \
  src/lib/validations/index.ts \
  src/lib/validations/recipes.test.ts \
  src/test/mocks/handlers.ts \
  src/test/mocks/server.ts \
  src/test/fixtures/recipes.ts \
  src/api/hooks/use-recipes.test.tsx
git commit -m "feat(recipes): add frontend recipe contracts"
```

## Task 4: Register Recipes In The App Shell And Build The Library / Detail / Add UX

**Files:**
- Modify: `frontend/src/App.tsx`
- Modify: `frontend/src/stores/app-store.ts`
- Modify: `frontend/src/stores/index.ts`
- Modify: `frontend/src/components/shared/mobile-bottom-nav.tsx`
- Modify: `frontend/src/components/shared/navigation-tabs.tsx`
- Modify: `frontend/src/components/shared/mobile-bottom-nav.test.tsx`
- Modify: `frontend/src/components/shared/navigation-tabs.test.tsx`
- Create: `frontend/src/components/recipes-view.tsx`
- Create: `frontend/src/components/recipes/recipe-library-card.tsx`
- Create: `frontend/src/components/recipes/recipe-filter-bar.tsx`
- Create: `frontend/src/components/recipes/recipe-form.tsx`
- Create: `frontend/src/components/recipes/recipe-create-sheet.tsx`
- Create: `frontend/src/components/recipes/recipe-edit-sheet.tsx`
- Create: `frontend/src/components/recipes/recipe-import-sheet.tsx`
- Create: `frontend/src/components/recipes/recipe-detail-view.tsx`
- Create: `frontend/src/components/recipes-view.test.tsx`
- Create: `frontend/e2e/mobile-recipes.spec.ts`

- [ ] **Step 1: Write the failing UI tests for shell registration, empty state, recipe detail, and add flows**

```tsx
it("shows Recipes in mobile bottom nav", () => {
  render(<MobileBottomNav />);
  expect(screen.getByLabelText("Recipes")).toBeInTheDocument();
});

it("renders the empty-state call to add the first recipe", async () => {
  server.use(http.get("*/api/recipes", () => HttpResponse.json({ data: [], message: "" })));

  render(<RecipesView />);

  expect(await screen.findByText("No recipes yet")).toBeInTheDocument();
  expect(screen.getByRole("button", { name: /add recipe/i })).toBeInTheDocument();
});

it("filters to favorites and keeps favorites ahead of non-favorites in filtered results", async () => {
  render(<RecipesView />);

  await user.click(await screen.findByRole("button", { name: /favorites/i }));
  const recipeCards = await screen.findAllByTestId("recipe-library-card");
  expect(recipeCards[0]).toHaveTextContent("Favorite Chili");
});

it("opens recipe detail from the library card", async () => {
  render(<RecipesView />);

  await user.click(await screen.findByRole("button", { name: /sheet pan gnocchi/i }));
  expect(await screen.findByText("Ingredients")).toBeInTheDocument();
  expect(screen.getByText("Instructions")).toBeInTheDocument();
});

it("toggles favorite from recipe detail", async () => {
  render(<RecipeDetailView recipeId="recipe-1" onBack={vi.fn()} />);

  await user.click(await screen.findByRole("button", { name: /favorite/i }));

  expect(await screen.findByRole("button", { name: /favorited/i })).toBeInTheDocument();
});

it("edits a saved recipe from recipe detail", async () => {
  render(<RecipeDetailView recipeId="recipe-1" onBack={vi.fn()} />);

  await user.click(await screen.findByRole("button", { name: /edit recipe/i }));
  await user.clear(screen.getByLabelText(/recipe name/i));
  await user.type(screen.getByLabelText(/recipe name/i), "Updated Gnocchi");
  await user.click(screen.getByRole("button", { name: /save changes/i }));

  expect(await screen.findByText("Updated Gnocchi")).toBeInTheDocument();
});
```

- [ ] **Step 2: Run the focused frontend tests to verify the shell and recipes screens are still missing**

Run:

```bash
cd /Users/joe.bor/code/family-hub/frontend
npm test -- --run src/components/recipes-view.test.tsx src/components/shared/mobile-bottom-nav.test.tsx src/components/shared/navigation-tabs.test.tsx
```

Expected: FAIL with missing `recipes` module wiring and missing screen components.

- [ ] **Step 3: Add shell registration and the recipes module screens**

```ts
export type ModuleType =
  | "calendar"
  | "lists"
  | "chores"
  | "recipes"
  | "meals"
  | "photos";
```

```tsx
const RecipesView = lazy(() =>
  import("@/components/recipes-view").then((m) => ({ default: m.RecipesView })),
);

case "recipes":
  return (
    <Suspense fallback={<ModuleLoader />}>
      <RecipesView />
    </Suspense>
  );
```

```tsx
const tabs: Array<{ id: ModuleType | null; label: string; icon: LucideIcon }> = [
  { id: null, label: "Home", icon: Home },
  { id: "calendar", label: "Calendar", icon: Calendar },
  { id: "lists", label: "Lists", icon: ListTodo },
  { id: "chores", label: "Chores", icon: CheckSquare },
  { id: "recipes", label: "Recipes", icon: BookOpenText },
  { id: "meals", label: "Meals", icon: UtensilsCrossed },
  { id: "photos", label: "Photos", icon: ImageIcon },
];
```

```tsx
export function RecipesView() {
  const [selectedRecipeId, setSelectedRecipeId] = useState<string | null>(null);
  const [createOpen, setCreateOpen] = useState(false);
  const [importOpen, setImportOpen] = useState(false);
  const [searchValue, setSearchValue] = useState("");
  const [favoritesOnly, setFavoritesOnly] = useState(false);
  const [selectedTag, setSelectedTag] = useState<string | null>(null);
  const recipes = useRecipes();
  const visibleRecipes = buildVisibleRecipes(recipes.data?.data ?? [], {
    searchValue,
    favoritesOnly,
    selectedTag,
  });

  if (selectedRecipeId) {
    return <RecipeDetailView recipeId={selectedRecipeId} onBack={() => setSelectedRecipeId(null)} />;
  }

  return (
    <div className="flex-1 overflow-y-auto p-4 sm:p-6">
      <div className="mx-auto max-w-5xl space-y-6">
        <div className="flex items-center justify-between gap-3">
          <h1 className="text-[24px] font-semibold leading-8 text-foreground">Recipes</h1>
          <Button type="button" onClick={() => setCreateOpen(true)}>
            <Plus className="h-4 w-4" />
            Add Recipe
          </Button>
        </div>

        <RecipeFilterBar
          searchValue={searchValue}
          favoritesOnly={favoritesOnly}
          selectedTag={selectedTag}
          onSearchChange={setSearchValue}
          onFavoritesOnlyChange={setFavoritesOnly}
          onSelectedTagChange={setSelectedTag}
        />

        {visibleRecipes.length ? (
          <div className="grid grid-cols-1 gap-3 sm:grid-cols-2 lg:grid-cols-3">
            {visibleRecipes.map((recipe) => (
              <RecipeLibraryCard key={recipe.id} recipe={recipe} onOpen={() => setSelectedRecipeId(recipe.id)} />
            ))}
          </div>
        ) : (
          <div className="rounded-lg border border-dashed border-border bg-card p-6 text-center shadow-sm">
            <h2 className="text-lg font-semibold text-foreground">No recipes yet</h2>
            <p className="mt-2 text-sm text-muted-foreground">
              Start with a manual recipe, then import from a URL inside the add flow when you want to grow the library faster.
            </p>
            <Button className="mt-4" onClick={() => setCreateOpen(true)}>Add recipe</Button>
          </div>
        )}
      </div>
    </div>
  );
}
```

```tsx
export function RecipeDetailView({ recipeId, onBack }: RecipeDetailViewProps) {
  const recipe = useRecipe(recipeId);
  const updateRecipe = useUpdateRecipe();
  const [editOpen, setEditOpen] = useState(false);

  const handleFavoriteToggle = () => {
    if (!recipe.data?.data) return;
    updateRecipe.mutate({
      id: recipeId,
      request: {
        ...toUpdateRecipeRequest(recipe.data.data),
        favorite: !recipe.data.data.favorite,
      },
    });
  };

  return (
    <>
      <Button type="button" variant="ghost" onClick={handleFavoriteToggle}>
        {recipe.data?.data.favorite ? "Favorited" : "Favorite"}
      </Button>
      <Button type="button" variant="outline" onClick={() => setEditOpen(true)}>
        Edit recipe
      </Button>
      <RecipeEditSheet
        open={editOpen}
        recipe={recipe.data?.data ?? null}
        onOpenChange={setEditOpen}
      />
    </>
  );
}
```

- [ ] **Step 4: Run unit and mobile E2E verification for the recipe module**

Run:

```bash
cd /Users/joe.bor/code/family-hub/frontend
npm test -- --run src/components/recipes-view.test.tsx src/components/shared/mobile-bottom-nav.test.tsx src/components/shared/navigation-tabs.test.tsx
npm run test:e2e -- e2e/mobile-recipes.spec.ts
```

Expected: PASS for shell registration, empty state, detail navigation, favorite toggle, edit flow, and the core mobile add/import flow.

- [ ] **Step 5: Commit**

```bash
cd /Users/joe.bor/code/family-hub/frontend
git add src/App.tsx \
  src/stores/app-store.ts \
  src/stores/index.ts \
  src/components/shared/mobile-bottom-nav.tsx \
  src/components/shared/navigation-tabs.tsx \
  src/components/shared/mobile-bottom-nav.test.tsx \
  src/components/shared/navigation-tabs.test.tsx \
  src/components/recipes-view.tsx \
  src/components/recipes/recipe-library-card.tsx \
  src/components/recipes/recipe-filter-bar.tsx \
  src/components/recipes/recipe-form.tsx \
  src/components/recipes/recipe-create-sheet.tsx \
  src/components/recipes/recipe-edit-sheet.tsx \
  src/components/recipes/recipe-import-sheet.tsx \
  src/components/recipes/recipe-detail-view.tsx \
  src/components/recipes-view.test.tsx \
  e2e/mobile-recipes.spec.ts
git commit -m "feat(recipes): add recipes module ui"
```

## Task 5: Add The Shared `Recipes <-> Meals` Intent Contract Without Building Meals

**Files:**
- Modify: `frontend/src/stores/app-store.ts`
- Modify: `frontend/src/stores/index.ts`
- Modify: `frontend/src/lib/time-utils.ts`
- Modify: `frontend/src/lib/time-utils.test.ts`
- Modify: `frontend/src/components/recipes/recipe-detail-view.tsx`
- Modify: `frontend/src/components/recipes-view.test.tsx`

- [ ] **Step 1: Write the failing test for the recipe detail handoff into the future Meals planner**

```tsx
it("stores a meal-placement draft and switches to Meals when Add to Meals is tapped", async () => {
  render(<RecipeDetailView recipeId="recipe-1" onBack={vi.fn()} />);

  await user.click(await screen.findByRole("button", { name: /add to meals/i }));

  expect(useAppStore.getState().activeModule).toBe("meals");
  expect(useAppStore.getState().mealPlacementDraft).toEqual({
    recipeId: "recipe-1",
    requestedAtWeekStartDate: expect.any(String),
    source: {
      kind: "recipes-library",
    },
  });
});

it("calculates a Sunday week start for the Add to Meals default week", () => {
  expect(formatLocalDate(getWeekStartSunday(new Date(2026, 5, 10)))).toBe("2026-06-07");
});

it("returns to Meals after recipe creation when the flow started from a meal slot", async () => {
  useAppStore.setState({
    recipeCreationDraft: {
      requestedAtWeekStartDate: "2026-06-07",
      dayIndex: 2,
      mealType: "dinner",
      typedTitle: "Leftovers",
    },
  });

  render(<RecipesView />);

  await user.click(await screen.findByRole("button", { name: /save recipe/i }));

  expect(useAppStore.getState().activeModule).toBe("meals");
  expect(useAppStore.getState().mealPlacementDraft).toEqual({
    recipeId: expect.any(String),
    requestedAtWeekStartDate: "2026-06-07",
    source: {
      kind: "meals-slot",
      dayIndex: 2,
      mealType: "dinner",
    },
  });
});
```

- [ ] **Step 2: Run the focused UI test to verify the cross-module handoff state does not exist yet**

Run:

```bash
cd /Users/joe.bor/code/family-hub/frontend
npm test -- --run src/components/recipes-view.test.tsx src/lib/time-utils.test.ts
```

Expected: FAIL with missing `mealPlacementDraft` state or missing `Add to Meals` handler.

- [ ] **Step 3: Add the shared handoff state and wire the recipe detail action**

```ts
export function getWeekStartSunday(date: Date): Date {
  const copy = startOfDay(date);
  copy.setDate(copy.getDate() - copy.getDay());
  return copy;
}
```

```ts
export interface MealPlacementDraft {
  recipeId: string;
  requestedAtWeekStartDate: string;
  source:
    | { kind: "recipes-library" }
    | { kind: "meals-slot"; dayIndex: number; mealType: "breakfast" | "lunch" | "dinner" };
}

export interface RecipeCreationDraft {
  requestedAtWeekStartDate: string;
  dayIndex: number;
  mealType: "breakfast" | "lunch" | "dinner";
  typedTitle: string;
}

interface AppState {
  activeModule: ModuleType | null;
  isSidebarOpen: boolean;
  mealPlacementDraft: MealPlacementDraft | null;
  recipeCreationDraft: RecipeCreationDraft | null;
  setActiveModule: (module: ModuleType | null) => void;
  startMealPlacementFromRecipe: (draft: MealPlacementDraft) => void;
  consumeMealPlacementDraft: () => MealPlacementDraft | null;
  startRecipeCreationFromMealSlot: (draft: RecipeCreationDraft) => void;
  consumeRecipeCreationDraft: () => RecipeCreationDraft | null;
  openSidebar: () => void;
  closeSidebar: () => void;
  toggleSidebar: () => void;
}

export const useAppStore = create<AppState>((set, get) => ({
  activeModule: null,
  isSidebarOpen: false,
  mealPlacementDraft: null,
  recipeCreationDraft: null,
  setActiveModule: (module) => set({ activeModule: module }),
  startMealPlacementFromRecipe: (draft) =>
    set({
      mealPlacementDraft: draft,
      activeModule: "meals",
    }),
  consumeMealPlacementDraft: () => {
    const current = get().mealPlacementDraft;
    set({ mealPlacementDraft: null });
    return current;
  },
  startRecipeCreationFromMealSlot: (draft) =>
    set({
      recipeCreationDraft: draft,
      activeModule: "recipes",
    }),
  consumeRecipeCreationDraft: () => {
    const current = get().recipeCreationDraft;
    set({ recipeCreationDraft: null });
    return current;
  },
  openSidebar: () => set({ isSidebarOpen: true }),
  closeSidebar: () => set({ isSidebarOpen: false }),
  toggleSidebar: () => set((state) => ({ isSidebarOpen: !state.isSidebarOpen })),
}));
```

```tsx
const startMealPlacementFromRecipe = useAppStore(
  (state) => state.startMealPlacementFromRecipe,
);
const recipeCreationDraft = useAppStore((state) => state.recipeCreationDraft);
const consumeRecipeCreationDraft = useAppStore(
  (state) => state.consumeRecipeCreationDraft,
);

<Button
  type="button"
  onClick={() =>
    startMealPlacementFromRecipe({
      recipeId,
      requestedAtWeekStartDate: formatLocalDate(getWeekStartSunday(new Date())),
      source: {
        kind: "recipes-library",
      },
    })
  }
>
  Add to Meals
</Button>

useEffect(() => {
  if (!recipeCreationDraft) return;
  setCreateOpen(true);
  setPrefillValues({
    title: recipeCreationDraft.typedTitle,
  });
}, [recipeCreationDraft]);

const handleCreateSuccess = (created: RecipeDetail) => {
  const draft = consumeRecipeCreationDraft();
  if (!draft) return;

  startMealPlacementFromRecipe({
    recipeId: created.id,
    requestedAtWeekStartDate: draft.requestedAtWeekStartDate,
    source: {
      kind: "meals-slot",
      dayIndex: draft.dayIndex,
      mealType: draft.mealType,
    },
  });
};
```

- [ ] **Step 4: Run the recipe-view test again and verify the handoff contract passes without implementing Meals yet**

Run:

```bash
cd /Users/joe.bor/code/family-hub/frontend
npm test -- --run src/components/recipes-view.test.tsx src/lib/time-utils.test.ts
```

Expected: PASS for the Sunday week-start helper, shared `Add to Meals` draft contract, and no new `Meals` board behavior in this plan.

- [ ] **Step 5: Commit**

```bash
cd /Users/joe.bor/code/family-hub/frontend
git add src/stores/app-store.ts \
  src/stores/index.ts \
  src/lib/time-utils.ts \
  src/lib/time-utils.test.ts \
  src/components/recipes/recipe-detail-view.tsx \
  src/components/recipes-view.test.tsx
git commit -m "feat(recipes): add meals handoff intent"
```

## Task 6: Review Recipes Delivery And Reconcile Contract Drift

**Files:**
- Read: `docs/superpowers/specs/2026-06-01-meals-and-recipes-foundation-design.md`
- Read: `docs/superpowers/plans/2026-06-01-recipes-module-foundation.md`
- Modify only if drifted: `docs/superpowers/specs/2026-06-01-meals-and-recipes-foundation-design.md`
- Modify only if drifted: `docs/superpowers/plans/2026-06-01-meals-weekly-planning.md`

- [ ] **Step 1: Review the completed Recipes implementation against the execution contract**

Confirm these non-negotiables before creating or merging the Recipes PR:

- standalone top-level `Recipes` module
- family-scoped backend persistence
- manual create with only name required
- guarded URL import with auto-save on success
- cook-first detail
- favorite toggle and edit from detail
- favorites/tags/light filters
- shared `Recipes -> Meals` and `Meals -> Recipes -> back to Meals` draft state
- Sunday week-start helper available before the Meals plan starts

- [ ] **Step 2: Apply review findings that are within the Recipes contract**

Fix only Recipes-owned issues in the delivery repos. If review discovers a cross-module contract change, update the root spec before proceeding to Meals.

- [ ] **Step 3: Update docs only if the implemented contract drifted**

If the implemented Recipes contract differs from this plan or the shared spec, update the root docs and commit those doc changes from the root workspace before issuing the Meals execution work.

## Spec Coverage Checklist

- Standalone top-level `Recipes` module: Tasks 3-4
- Manual recipe creation with name-only requirement: Tasks 2-4
- URL import with auto-save on success and import living inside the add flow: Task 2 and Task 4
- Cook-first recipe detail: Task 4
- Favorite toggle, edit, tags, light filters, and favorites surfacing higher in filtered/search contexts: Tasks 2-4
- Shared `Recipes -> Meals` and `Meals -> Recipes -> back to Meals` contract without building `Meals`: Task 5
- Recipes review and doc-drift reconciliation before Meals starts: Task 6

## Risks To Watch During Execution

- Adding `Recipes` increases module count; make sure the mobile bottom nav stays tappable and readable after shell registration.
- Keep the import parser resilient enough for MVP, but do not expand it into a full crawler or cleanup workflow.
- Do not hide recipe data behind the old `meals-store.ts` placeholder state; `Recipes` owns its own real API-backed contract now.
