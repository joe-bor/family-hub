# Large-Screen Recipes Card Grid Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the large-screen Recipes index from a single stack of full-width "posters" into a browsable 2–4 column responsive card grid, without touching mobile, recipe detail, or add/edit flows.

**Architecture:** This is index-composition work across `recipes-view.tsx` and the existing `RecipeFilterBar`; `RecipeLibraryCard` already supplies the vertical `md:aspect-[4/3]` media treatment, so the card and detail components stay unchanged. The list gets responsive columns and a list-mode-only ~1200px container. On non-mobile list mode, the existing filter bar moves into the desktop header so title, bounded search, chips, and Add recipe form one non-wrapping `lg+` toolbar; mobile keeps the filter inside `ScreenTransition` exactly where it is today. A focused Playwright spec seeds deterministic recipe states through the released API, asserts computed geometry, and attaches the required screenshot matrix.

**Tech Stack:** React 19, TypeScript, Tailwind CSS v4 (arbitrary variants), Vitest + Testing Library, Biome.

---

## Source Documents

- **Story:** `docs/product/backlog/large-screen-ux/large-screen-recipes.md`
- **Spec:** `docs/superpowers/specs/2026-07-06-large-screen-recipes-design.md`
- This plan finishes the **large-screen-ux epic** (Recipes is the last of 7 surfaces).

## Key Decisions (read before coding)

1. **Detail stays untouched via a mode-conditional container width.** The container `<div>` in `recipes-view.tsx` wraps the header, the list, *and* the detail view (inside `ScreenTransition`). We widen it (`lg:max-w-[1200px]`) **only when `selectedRecipeId === null`** (list mode). In detail mode it keeps `max-w-3xl`, so `recipe-detail-view.tsx` renders pixel-identical to today. Trade-off: opening a recipe shrinks the container 1200→768 as the slide runs. This is intentional and acceptable — detail keeps its readable column. Do **not** add a max-width wrapper inside the detail view.

2. **Column tiers are keyed to the viewport to match the spec exactly:** 2 cols at 1024, 3 at 1280, 4 at 1440+. Because the container caps at ~1200px, the 3→4 step at 1440 makes cards denser (more recipes visible) at the same content width — this is the spec's intent, not a bug. Tailwind's `2xl` is 1536 and would miss the 1440 target, so the 4-col tier uses the arbitrary variant `min-[1440px]:`.

3. **The card's favorite heart stays display-only (unchanged).** The reviewed spec now matches the current `RecipeLibraryCard`: its heart is a non-interactive badge and favoriting happens in detail. The contract requires favorites *filtering* to stay unchanged. **Do not** add a per-card mutation in this pass.

4. **The foundations toolbar requirement is part of this story.** At `lg+`, the title, a `w-64` search field, horizontally scrolling filter chips, and Add recipe share one row. At 769-1023px the current title/action row above the filter bar remains. Filter inputs and buttons grow to 44px only at `lg+`. This closes the spec-to-plan gap without changing filter state or behavior.

5. **Mobile is untouched.** Base grid/container classes preserve today's card rows, and mobile keeps its single `RecipeFilterBar` inside `ScreenTransition`. Every new layout/touch-size treatment is gated at `lg`/`xl`/`min-[1440px]` or the existing `!isMobile` branch.

6. **Screenshot evidence is reproducible.** Add a real-backend Playwright spec rather than relying on an unspecified manually seeded database. It creates one recipe, expands the same family to twelve recipes (including favorites and image-less entries), verifies the grid at 1024/1280/1440, captures filter/no-results states, checks detail width, and captures the 375px mobile regression.

## File Structure

| File | Change | Responsibility |
|------|--------|----------------|
| `frontend/src/components/recipes-view.tsx` | Modify | Index composition: responsive grid, list-mode width, desktop toolbar ownership, centered states |
| `frontend/src/components/recipes/recipe-filter-bar.tsx` | Modify | `lg+` one-row filter composition, bounded search, 44px controls, optional placement class |
| `frontend/src/components/recipes-view.test.tsx` | Modify | Assert grid, detail width, desktop toolbar composition, and both centered states |
| `frontend/e2e/large-screen-recipes.spec.ts` | Create | Real-backend geometry assertions and screenshot matrix at acceptance widths/states |

No other files change. `recipe-library-card.tsx` and `recipe-detail-view.tsx` are **not** modified.

---

## Setup

- [ ] **Branch from fresh `frontend/main`** (it must include Chores PR #289; the local checkout may be behind on another branch)

```bash
cd frontend
git checkout main
git pull origin main
git checkout -b feat/large-screen-recipes
```

Confirm the starting point matches what this plan assumes:

```bash
# Container is the shared max-w-3xl div; the loaded grid is a bare "grid gap-3".
rg -n 'mx-auto flex max-w-3xl|grid gap-3' src/components/recipes-view.tsx
# Filter bar is used only by RecipesView, so adding an optional className is local.
rg -n '<RecipeFilterBar' src
```

Expected: the first command finds one container plus loading/loaded grids; the second finds one call site in `recipes-view.tsx`. If they differ, re-read the files and adjust anchors before editing.

---

## Task 1: Responsive multi-column grid

**Files:**
- Modify: `frontend/src/components/recipes-view.tsx` (loaded grid ~329, loading skeleton grid ~250)
- Test: `frontend/src/components/recipes-view.test.tsx`

- [ ] **Step 1: Write the failing test**

Add this test inside the top-level `describe("RecipesView", …)` block (e.g. right after the `"renders summary cards…"` test near line 189). It seeds a recipe, waits for its card, then asserts the grid element carries the responsive column tokens.

```tsx
it("lays the large-screen library out as a responsive card grid", async () => {
  viewport.isMobile = false;
  seedMockRecipes([testRecipeDetail, importedRecipeDetail]);

  render(<RecipesView />);

  await screen.findByRole("article", {
    name: "Recipe card: Sheet Pan Salmon",
  });

  const grid = screen.getByTestId("recipe-library-grid");
  expect(grid).toHaveClass("grid-cols-1");
  expect(grid).toHaveClass("lg:grid-cols-2");
  expect(grid).toHaveClass("xl:grid-cols-3");
  expect(grid).toHaveClass("min-[1440px]:grid-cols-4");
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && npm test -- --run recipes-view.test.tsx -t "responsive card grid"`
Expected: FAIL — `Unable to find an element by: [data-testid="recipe-library-grid"]` (the grid has no test id / responsive classes yet).

- [ ] **Step 3: Implement the responsive grid**

In `recipes-view.tsx`, change the **loaded** grid wrapper (currently `<div className="grid gap-3">` at ~line 329) to add the test id and responsive columns:

```tsx
<div
  data-testid="recipe-library-grid"
  className="grid grid-cols-1 gap-3 lg:grid-cols-2 lg:gap-4 xl:grid-cols-3 min-[1440px]:grid-cols-4"
>
  {filteredRecipes.map((recipe) => (
    <div key={recipe.id} className={cn("min-w-0")}>
      <RecipeLibraryCard
        recipe={recipe}
        onSelect={(recipeId) => setSelectedRecipeId(recipeId)}
      />
    </div>
  ))}
</div>
```

Then make the **loading skeleton** grid (currently `<div className="grid gap-3">` at ~line 250) match, so the loading state mirrors the loaded layout:

```tsx
<div className="grid grid-cols-1 gap-3 lg:grid-cols-2 lg:gap-4 xl:grid-cols-3 min-[1440px]:grid-cols-4">
  {Array.from({ length: 3 }, (_, index) => (
```

(Leave the skeleton card markup inside unchanged.)

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend && npm test -- --run recipes-view.test.tsx -t "responsive card grid"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/components/recipes-view.tsx src/components/recipes-view.test.tsx
git commit -m "feat(recipes): responsive card grid on large screens"
```

---

## Task 2: Widen the index container in list mode only

**Files:**
- Modify: `frontend/src/components/recipes-view.tsx` (container div ~161)
- Test: `frontend/src/components/recipes-view.test.tsx`

- [ ] **Step 1: Write the failing test**

Add this test in the same `describe` block. It asserts the container is wide in list mode, then narrow after opening a recipe (detail mode).

```tsx
it("widens the container for the grid but keeps recipe detail at reading width", async () => {
  viewport.isMobile = false;
  seedMockRecipes([testRecipeDetail]);

  const { user } = renderWithUser(<RecipesView />);

  const openButton = await screen.findByRole("button", {
    name: `Open recipe: ${testRecipeDetail.title}`,
  });

  const container = screen.getByTestId("recipes-view-container");
  expect(container).toHaveClass("lg:max-w-[1200px]"); // list mode: wide

  await user.click(openButton);

  await screen.findByRole("button", { name: "Back to recipes" });
  expect(screen.getByTestId("recipes-view-container")).not.toHaveClass(
    "lg:max-w-[1200px]",
  ); // detail mode: reading width only
});
```

The accessible name is `Back to recipes` in current `frontend/main`; keep the test exact so contract drift is visible rather than hidden behind a fallback selector.

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && npm test -- --run recipes-view.test.tsx -t "reading width"`
Expected: FAIL — `Unable to find an element by: [data-testid="recipes-view-container"]`.

- [ ] **Step 3: Implement the mode-conditional width**

In `recipes-view.tsx`, change the container `<div>` (currently `<div className="mx-auto flex max-w-3xl flex-col gap-4">` at ~line 161) to add the test id and a list-mode-only wide cap via `cn()`:

```tsx
<div
  data-testid="recipes-view-container"
  className={cn(
    "mx-auto flex w-full max-w-3xl flex-col gap-4",
    selectedRecipeId === null && "lg:max-w-[1200px]",
  )}
>
```

`cn` is already imported from `@/lib/utils`. `max-w-3xl` (base) and `lg:max-w-[1200px]` are different breakpoints, so tailwind-merge keeps both in list mode; in detail mode only `max-w-3xl` remains.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend && npm test -- --run recipes-view.test.tsx -t "reading width"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/components/recipes-view.tsx src/components/recipes-view.test.tsx
git commit -m "feat(recipes): widen large-screen index without stretching detail"
```

---

## Task 3: Compose the foundations toolbar at `lg+`

**Files:**
- Modify: `frontend/src/components/recipes-view.tsx` (desktop heading and loaded-list filter placement)
- Modify: `frontend/src/components/recipes/recipe-filter-bar.tsx`
- Test: `frontend/src/components/recipes-view.test.tsx`

The current non-mobile heading and `RecipeFilterBar` are separate, and the filter itself stacks a full-width search above its chips. The approved spec requires title + bounded search + filters + Add recipe in one row at `lg+`, while the mobile filter must remain inside the sliding list screen.

- [ ] **Step 1: Write the failing toolbar-composition test**

```tsx
it("composes the large-screen controls into one foundations toolbar", async () => {
  viewport.isMobile = false;
  seedMockRecipes([testRecipeDetail, importedRecipeDetail]);

  render(<RecipesView />);

  await screen.findByRole("article", {
    name: "Recipe card: Sheet Pan Salmon",
  });

  const toolbar = screen.getByTestId("recipes-desktop-toolbar");
  const search = screen.getByRole("searchbox", { name: "Search recipes" });
  const favorites = screen.getByRole("button", { name: "Favorites only" });
  const add = screen.getByRole("button", { name: "Add recipe" });

  expect(toolbar).toHaveClass("lg:flex-nowrap");
  expect(toolbar).toContainElement(search);
  expect(toolbar).toContainElement(favorites);
  expect(toolbar).toContainElement(add);
  expect(screen.getByTestId("recipe-filter-bar")).toHaveClass("lg:flex");
  expect(search).toHaveClass("lg:h-11");
  expect(favorites).toHaveClass("lg:min-h-11");
  expect(add).toHaveClass("lg:min-h-11");
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `cd frontend && npm test -- --run src/components/recipes-view.test.tsx -t "foundations toolbar"`

Expected: FAIL — `recipes-desktop-toolbar` does not exist yet.

- [ ] **Step 3: Make `RecipeFilterBar` a one-row `lg+` building block**

In `recipe-filter-bar.tsx`, add `className?: string` to the props, destructure it, and replace the component return with the following. The base classes are unchanged in effect; only `lg+` becomes a bounded-search row with 44px controls.

```tsx
interface RecipeFilterBarProps {
  availableTags: RecipeTagFilterOption[];
  className?: string;
  favoritesOnly: boolean;
  onFavoritesOnlyChange: (favoritesOnly: boolean) => void;
  onSearchChange: (value: string) => void;
  onTagChange: (tag: string | null) => void;
  searchValue: string;
  selectedTag: string | null;
}

export function RecipeFilterBar({
  availableTags,
  className,
  favoritesOnly,
  onFavoritesOnlyChange,
  onSearchChange,
  onTagChange,
  searchValue,
  selectedTag,
}: RecipeFilterBarProps) {
  return (
    <div
      data-testid="recipe-filter-bar"
      className={cn(
        "space-y-3 lg:flex lg:min-w-0 lg:flex-1 lg:items-center lg:gap-3 lg:space-y-0",
        className,
      )}
    >
      <div className="relative lg:w-64 lg:shrink-0">
        <Search className="pointer-events-none absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-muted-foreground" />
        <Input
          type="search"
          aria-label="Search recipes"
          value={searchValue}
          onChange={(event) => onSearchChange(event.target.value)}
          placeholder="Search titles or tags"
          className="pl-9 lg:h-11"
        />
      </div>

      <div className="flex items-center gap-2 overflow-x-auto pb-1 scrollbar-hide lg:min-w-0 lg:flex-1 lg:pb-0">
        <Button
          type="button"
          variant={favoritesOnly ? "default" : "outline"}
          size="sm"
          onClick={() => onFavoritesOnlyChange(!favoritesOnly)}
          className="shrink-0 lg:min-h-11"
        >
          <Heart className={cn("h-4 w-4", favoritesOnly && "fill-current")} />
          Favorites only
        </Button>

        <Button
          type="button"
          variant={selectedTag === null ? "secondary" : "outline"}
          size="sm"
          onClick={() => onTagChange(null)}
          className="shrink-0 lg:min-h-11"
        >
          All
        </Button>

        {availableTags.map((tag) => (
          <Button
            key={tag.value}
            type="button"
            variant={selectedTag === tag.value ? "secondary" : "outline"}
            size="sm"
            onClick={() =>
              onTagChange(selectedTag === tag.value ? null : tag.value)
            }
            className="shrink-0 lg:min-h-11"
          >
            {tag.label}
          </Button>
        ))}
      </div>
    </div>
  );
}
```

- [ ] **Step 4: Give desktop/tablet ownership of the filter to the heading wrapper**

In `recipes-view.tsx`, after `updateSelectedRecipe`, add:

```tsx
  const showDesktopLibraryFilters =
    !isMobile &&
    selectedRecipeId === null &&
    !isLoading &&
    !isError &&
    data !== undefined;
```

Replace the existing `!isMobile` heading block with:

```tsx
{!isMobile && (
  <div
    data-testid="recipes-desktop-toolbar"
    className={cn(
      "flex flex-wrap items-start justify-between gap-3",
      selectedRecipeId === null && "lg:flex-nowrap lg:items-center",
    )}
  >
    <div className="min-w-0 shrink-0">
      <h1 className="text-2xl font-semibold text-foreground">Recipes</h1>
      <p
        className={cn(
          "text-sm text-muted-foreground",
          selectedRecipeId === null && "lg:hidden",
        )}
      >
        Save family favorites and discover what to cook next.
      </p>
    </div>

    {showDesktopLibraryFilters ? (
      <RecipeFilterBar
        availableTags={availableTags}
        className="order-3 w-full lg:order-none lg:w-auto"
        favoritesOnly={favoritesOnly}
        onFavoritesOnlyChange={setFavoritesOnly}
        onSearchChange={setSearchValue}
        onTagChange={setSelectedTag}
        searchValue={searchValue}
        selectedTag={selectedTag}
      />
    ) : null}

    {selectedRecipeId === null ? (
      <Button
        type="button"
        className="lg:min-h-11"
        onClick={() => setIsCreateSheetOpen(true)}
      >
        Add recipe
      </Button>
    ) : null}
  </div>
)}
```

Finally, keep the filter at its current mobile location by changing the loaded-list call site inside `ScreenTransition` to:

```tsx
{isMobile ? (
  <RecipeFilterBar
    availableTags={availableTags}
    favoritesOnly={favoritesOnly}
    onFavoritesOnlyChange={setFavoritesOnly}
    onSearchChange={setSearchValue}
    onTagChange={setSelectedTag}
    searchValue={searchValue}
    selectedTag={selectedTag}
  />
) : null}
```

Do not render two filter bars and hide one with CSS: Testing Library would still see duplicate controls, and hidden duplicate form controls are an avoidable accessibility risk.

- [ ] **Step 5: Run the focused and full RecipesView unit suites**

```bash
cd frontend
npm test -- --run src/components/recipes-view.test.tsx -t "foundations toolbar"
npm test -- --run src/components/recipes-view.test.tsx
```

Expected: both PASS; the pre-existing search, favorites, tag, detail, and mobile-FAB tests remain green.

- [ ] **Step 6: Commit**

```bash
git add src/components/recipes-view.tsx src/components/recipes/recipe-filter-bar.tsx src/components/recipes-view.test.tsx
git commit -m "feat(recipes): compose large-screen controls into one toolbar"
```

---

## Task 4: Keep empty and no-results states centered in the wide container

**Files:**
- Modify: `frontend/src/components/recipes-view.tsx` (empty state ~303, no-results ~320)
- Test: `frontend/src/components/recipes-view.test.tsx`

At 1200px the two dashed callout cards would stretch full width. Cap and center them so they read as calm centered panels (spec §3: "Empty and no-results states center within the grid area").

- [ ] **Step 1: Write the failing test**

```tsx
it("centers the empty state within the widened library", async () => {
  viewport.isMobile = false;
  seedMockRecipes([]);

  render(<RecipesView />);

  const heading = await screen.findByText("No recipes yet");
  const panel = heading.closest("div");
  expect(panel).toHaveClass("mx-auto");
  expect(panel).toHaveClass("max-w-xl");
});

it("centers the no-results state within the widened library", async () => {
  viewport.isMobile = false;
  seedMockRecipes([testRecipeDetail]);

  const { user } = renderWithUser(<RecipesView />);

  await screen.findByRole("article", {
    name: "Recipe card: Sheet Pan Salmon",
  });
  await user.type(
    screen.getByRole("searchbox", { name: "Search recipes" }),
    "definitely-not-a-recipe",
  );

  const heading = await screen.findByText("No recipes match those filters");
  const panel = heading.closest("div");
  expect(panel).toHaveClass("mx-auto");
  expect(panel).toHaveClass("max-w-xl");
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && npm test -- --run src/components/recipes-view.test.tsx -t "centers"`
Expected: both new tests FAIL — the panels are currently full-width.

- [ ] **Step 3: Implement centered callouts**

In `recipes-view.tsx`, add `mx-auto w-full max-w-xl` to **both** dashed callout containers.

Empty state (~line 303):

```tsx
{recipes.length === 0 ? (
  <div className="mx-auto w-full max-w-xl rounded-lg border border-dashed border-border bg-card p-6 text-center">
    <h2 className="text-lg font-semibold text-foreground">
      No recipes yet
    </h2>
```

No-results state (~line 320):

```tsx
) : filteredRecipes.length === 0 ? (
  <div className="mx-auto w-full max-w-xl rounded-lg border border-dashed border-border bg-card p-6 text-center">
    <h2 className="text-lg font-semibold text-foreground">
      No recipes match those filters
    </h2>
```

Leave the inner markup (headings, copy, the "Add your first recipe" button) unchanged.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend && npm test -- --run src/components/recipes-view.test.tsx -t "centers"`
Expected: both PASS.

- [ ] **Step 5: Commit**

```bash
git add src/components/recipes-view.tsx src/components/recipes-view.test.tsx
git commit -m "feat(recipes): center empty and no-results states in wide grid"
```

---

## Task 5: Add deterministic browser geometry and screenshot coverage

**Files:**
- Create: `frontend/e2e/large-screen-recipes.spec.ts`

Follow the established `large-screen-meals.spec.ts` pattern: seed through the real released backend, assert browser-computed geometry, and attach screenshots to the Playwright report. This test is evidence generation as well as regression coverage.

- [ ] **Step 1: Create the Playwright acceptance spec**

Create `e2e/large-screen-recipes.spec.ts`:

```tsx
import {
  type APIRequestContext,
  expect,
  type Page,
  test,
} from "@playwright/test";
import {
  createRecipe,
  registerFamily,
  seedBrowserAuth,
} from "./helpers/api-helpers";
import {
  clearStorage,
  safeClick,
  waitForHydration,
} from "./helpers/test-helpers";

const ACCEPTANCE_WIDTHS = [
  { width: 1024, columns: 2 },
  { width: 1280, columns: 3 },
  { width: 1440, columns: 4 },
] as const;
const LOCAL_RECIPE_IMAGE = "http://127.0.0.1:5173/icons/icon-512.png";

async function openRecipes(page: Page) {
  const nav = page.getByRole("navigation", { name: "Primary" });
  await safeClick(nav.getByRole("button", { name: "Recipes", exact: true }));
  await expect(
    page.getByRole("heading", { name: "Recipes", level: 1, exact: true }),
  ).toBeVisible();
}

async function authenticateAndOpenRecipes(
  page: Page,
  registration: Awaited<ReturnType<typeof registerFamily>>,
) {
  await clearStorage(page);
  await seedBrowserAuth(page, registration);
  await page.reload();
  await waitForHydration(page);
  await openRecipes(page);
}

async function attachRecipesScreenshot(
  page: Page,
  name: string,
  width: number,
) {
  await page.setViewportSize({
    width,
    height: width === 1024 ? 768 : 900,
  });
  const container = page.getByTestId("recipes-view-container");
  await expect(container).toBeVisible();
  await page.evaluate(() =>
    Promise.all(
      document
        .getAnimations({ subtree: true })
        .map((animation) => animation.finished.catch(() => undefined)),
    ).then(() => undefined),
  );
  const section = container.locator("xpath=ancestor::section");
  await test.info().attach(`${name}-${width}`, {
    body: await section.screenshot(),
    contentType: "image/png",
  });
}

async function expectGridColumns(
  page: Page,
  expectedColumns: number,
  expectedCards: number,
) {
  const grid = page.getByTestId("recipe-library-grid");
  await expect(grid).toBeVisible();
  await expect(grid.getByRole("article")).toHaveCount(expectedCards);

  const geometry = await grid.evaluate((element) => {
    const box = element.getBoundingClientRect();
    const columns = window
      .getComputedStyle(element)
      .gridTemplateColumns.trim()
      .split(/\s+/).length;
    return { columns, width: box.width };
  });

  expect(geometry.columns).toBe(expectedColumns);
  expect(geometry.width).toBeLessThanOrEqual(1201);
}

async function seedRemainingRecipes(
  request: APIRequestContext,
  token: string,
) {
  for (let index = 2; index <= 12; index += 1) {
    await createRecipe(request, token, {
      title: `Recipe ${String(index).padStart(2, "0")}`,
      imageUrl: index % 2 === 0 ? null : LOCAL_RECIPE_IMAGE,
      ingredients: [`Ingredient ${index}`],
      instructions: [`Cook recipe ${index}`],
      tags: [
        index % 3 === 0 ? "Favorite set" : "Weeknight",
        `Tag ${String(index).padStart(2, "0")}`,
      ],
      favorite: index % 3 === 0,
    });
  }
}

test.describe("Large-screen Recipes", () => {
  test.beforeEach(async ({ page, isMobile }) => {
    test.skip(
      isMobile,
      "Desktop geometry and controlled 375px regression only",
    );
    await page.setViewportSize({ width: 1440, height: 900 });
    await page.goto("/");
    await clearStorage(page);
  });

  test("verifies the grid, toolbar, detail width, and visual state matrix", async ({
    page,
    request,
  }) => {
    test.setTimeout(120000);

    const registration = await registerFamily(request, {
      familyName: "Large Recipes Matrix",
      members: [{ name: "Sam", color: "teal" }],
    });
    await createRecipe(request, registration.token, {
      title: "Recipe 01",
      imageUrl: LOCAL_RECIPE_IMAGE,
      ingredients: ["Ingredient 1"],
      instructions: ["Cook recipe 1"],
      tags: ["Weeknight", "Tag 01"],
      favorite: false,
    });
    await authenticateAndOpenRecipes(page, registration);

    for (const { width, columns } of ACCEPTANCE_WIDTHS) {
      await page.setViewportSize({
        width,
        height: width === 1024 ? 768 : 900,
      });
      await expectGridColumns(page, columns, 1);
      const grid = page.getByTestId("recipe-library-grid");
      const cardBox = await grid.getByRole("article").boundingBox();
      const gridBox = await grid.boundingBox();
      expect(cardBox).not.toBeNull();
      expect(gridBox).not.toBeNull();
      expect(cardBox!.width).toBeLessThan(gridBox!.width * 0.75);
      await attachRecipesScreenshot(page, "one-recipe", width);
    }

    await page.setViewportSize({ width: 1440, height: 900 });
    await page
      .getByRole("button", { name: "Open recipe: Recipe 01" })
      .click();
    await expect(
      page.getByRole("button", { name: "Back to recipes" }),
    ).toBeVisible();
    const detailContainerBox = await page
      .getByTestId("recipes-view-container")
      .boundingBox();
    expect(detailContainerBox).not.toBeNull();
    expect(detailContainerBox!.width).toBeLessThanOrEqual(769);
    await page.getByRole("button", { name: "Back to recipes" }).click();

    await seedRemainingRecipes(request, registration.token);
    await page
      .getByRole("button", { name: "Open recipe: Recipe 01" })
      .click();
    const favoriteButton = page.getByRole("button", {
      name: "Favorite recipe: Recipe 01",
    });
    await favoriteButton.click();
    await expect(favoriteButton).toHaveAttribute("aria-pressed", "true");
    await page.getByRole("button", { name: "Back to recipes" }).click();
    await expect(
      page.getByTestId("recipe-library-grid").getByRole("article"),
    ).toHaveCount(12);

    for (const { width, columns } of ACCEPTANCE_WIDTHS) {
      await page.setViewportSize({
        width,
        height: width === 1024 ? 768 : 900,
      });
      await expectGridColumns(page, columns, 12);

      const media = page
        .getByRole("article", { name: "Recipe card: Recipe 11" })
        .getByRole("img", { name: "Recipe 11" })
        .locator("..");
      const mediaBox = await media.boundingBox();
      expect(mediaBox).not.toBeNull();
      expect(mediaBox!.width / mediaBox!.height).toBeGreaterThan(1.3);
      expect(mediaBox!.width / mediaBox!.height).toBeLessThan(1.37);
      const noPhoto = page.getByText("No photo").first();
      await expect(noPhoto).toBeVisible();
      const placeholderBox = await noPhoto
        .locator("xpath=../..")
        .boundingBox();
      expect(placeholderBox).not.toBeNull();
      expect(placeholderBox!.width / placeholderBox!.height).toBeGreaterThan(
        1.3,
      );
      expect(placeholderBox!.width / placeholderBox!.height).toBeLessThan(
        1.37,
      );

      const toolbar = page.getByTestId("recipes-desktop-toolbar");
      const toolbarBox = await toolbar.boundingBox();
      const searchBox = await page
        .getByRole("searchbox", { name: "Search recipes" })
        .boundingBox();
      const favoritesBox = await page
        .getByRole("button", { name: "Favorites only" })
        .boundingBox();
      const addBox = await page
        .getByRole("button", { name: "Add recipe" })
        .boundingBox();
      expect(toolbarBox).not.toBeNull();
      expect(searchBox).not.toBeNull();
      expect(favoritesBox).not.toBeNull();
      expect(addBox).not.toBeNull();
      expect(toolbarBox!.height).toBeLessThanOrEqual(48);
      expect(searchBox!.width).toBeGreaterThanOrEqual(250);
      expect(searchBox!.width).toBeLessThanOrEqual(258);
      expect(searchBox!.height).toBeGreaterThanOrEqual(44);
      expect(favoritesBox!.height).toBeGreaterThanOrEqual(44);
      expect(addBox!.height).toBeGreaterThanOrEqual(44);

      await attachRecipesScreenshot(page, "twelve-recipes", width);
    }

    await page.getByRole("button", { name: "Favorites only" }).click();
    await expect(
      page.getByTestId("recipe-library-grid").getByRole("article"),
    ).toHaveCount(5);
    for (const { width } of ACCEPTANCE_WIDTHS) {
      await attachRecipesScreenshot(page, "favorites-only", width);
    }

    await page
      .getByRole("searchbox", { name: "Search recipes" })
      .fill("definitely-no-match");
    await expect(
      page.getByText("No recipes match those filters"),
    ).toBeVisible();
    for (const { width } of ACCEPTANCE_WIDTHS) {
      await attachRecipesScreenshot(page, "no-results", width);
    }
  });

  test("preserves the 375px horizontal cards and FAB", async ({
    page,
    request,
  }) => {
    const registration = await registerFamily(request, {
      familyName: "Recipes Mobile Regression",
      members: [{ name: "Sam", color: "teal" }],
    });
    await createRecipe(request, registration.token, {
      title: "Mobile Recipe",
      imageUrl: LOCAL_RECIPE_IMAGE,
      tags: ["Mobile"],
    });
    await authenticateAndOpenRecipes(page, registration);

    await page.setViewportSize({ width: 375, height: 812 });
    const card = page.getByRole("article", {
      name: "Recipe card: Mobile Recipe",
    });
    await expect(card).toBeVisible();
    const flexDirection = await card.evaluate(
      (element) => getComputedStyle(element).flexDirection,
    );
    expect(flexDirection).toBe("row");
    await expect(
      page.getByRole("button", { name: "Add recipe" }),
    ).toBeVisible();
    await attachRecipesScreenshot(page, "mobile-regression", 375);
  });
});
```

- [ ] **Step 2: Run the acceptance spec against Tasks 1-4**

Start the released backend exactly as CI does, then run Chromium only while iterating:

```bash
cd frontend
BE_IMAGE_TAG=$(GITHUB_TOKEN=$(gh auth token) bash .github/scripts/resolve-backend-version.sh) \
  docker compose -f docker-compose.e2e.yml up -d --wait
npx playwright test e2e/large-screen-recipes.spec.ts --project=chromium
```

Expected: PASS with the implemented grid/container/toolbar test ids and geometry. If it fails, treat the browser result as a contract failure and fix the implementation before reviewing screenshots.

- [ ] **Step 3: Re-run after implementation and inspect every attachment**

Run the same Playwright command. Expected: PASS with attached images for one recipe, twelve recipes (including image-less cards), favorites-only, and no-results at 1024/1280/1440, plus the 375px mobile regression.

Open the HTML report with `npx playwright show-report`. Check that no toolbar wraps, cards remain calm and uniform, horizontal chip overflow does not obscure Add recipe, the centered no-results panel reads well, and the 375px image/card/FAB composition is unchanged. If any state looks wrong, fix the CSS, rerun the unit suite and this spec, and inspect the replacement attachments before continuing.

- [ ] **Step 4: Commit**

```bash
git add e2e/large-screen-recipes.spec.ts
git commit -m "test(recipes): cover large-screen grid geometry and screenshots"
```

---

## Task 6: Full verification and PR

**Files:** none (verification + publish only)

- [ ] **Step 1: Run the full relevant gate**

```bash
cd frontend
npm run lint
npm test -- --run src/components/recipes-view.test.tsx
npx playwright test e2e/large-screen-recipes.spec.ts --project=chromium
npm run build
```

Expected: lint clean; the full RecipesView suite passes; the browser geometry/screenshot spec passes against the released backend; `tsc` + Vite build succeed. Stop the local backend afterward with `docker compose -f docker-compose.e2e.yml down`.

- [ ] **Step 2: Map the execution contract before opening the PR**

In the work log/PR draft, explicitly map each non-negotiable to evidence:

- index-only files; detail/card components untouched;
- 2/3/4 computed columns at 1024/1280/1440 and ~1200px cap;
- single `lg+` title/search/filter/Add row with 44px controls;
- display-only card heart, filter behavior unchanged;
- both centered states;
- detail container ≤769px;
- screenshot attachment names for one/twelve/favorites/no-results/mobile.

- [ ] **Step 3: Push and open the PR**

```bash
git push -u origin feat/large-screen-recipes
gh pr create --repo joe-bor/FamilyHub \
  --title "feat(recipes): large-screen responsive card grid" \
  --body "Closes #290

## Summary
- Large-screen Recipes becomes a 2/3/4-column grid at 1024/1280/1440 inside a ~1200px list-only container.
- Title, bounded search, filter chips, and Add recipe share one 44px-target toolbar row at lg+.
- Empty/no-results stay centered; detail width, display-only card hearts, and mobile behavior remain unchanged.

## Story / spec / plan
- Story: docs/product/backlog/large-screen-ux/large-screen-recipes.md
- Spec: docs/superpowers/specs/2026-07-06-large-screen-recipes-design.md
- Plan: docs/superpowers/plans/2026-07-15-large-screen-recipes.md

## Verification
- npm run lint
- npm test -- --run src/components/recipes-view.test.tsx
- npx playwright test e2e/large-screen-recipes.spec.ts --project=chromium
- npm run build
- Playwright attachments reviewed at 1024 / 1280 / 1440 plus 375px mobile"
```

Attach the reviewed Playwright images (or the report artifact link plus named attachments) to the PR description/comment before marking it ready.

---

## Self-Review (completed while writing this plan)

**1. Spec coverage:**
- 2–4 column grid at 1024/1280/1440 → Task 1 (`lg:grid-cols-2 xl:grid-cols-3 min-[1440px]:grid-cols-4`) plus computed-style assertions in Task 5.
- Capped-ratio images, no card fills viewport → existing `RecipeLibraryCard` media plus Task 5 ratio/card-width assertions; Task 2 caps the list container.
- One `lg+` foundations toolbar row with bounded search and 44px targets → Task 3 unit assertions plus Task 5 browser geometry.
- Search, favorites, and tag behavior unchanged → no filter state logic changes; the existing behavioral tests remain green in Tasks 3 and 6.
- Empty + no-results centered/intact → Task 4 tests both states.
- Mobile unchanged → mobile keeps the filter inside `ScreenTransition`; Task 5 checks the computed horizontal card direction, FAB, and 375px screenshot.
- Image-less placeholder uniform → existing card branch, visibility and screenshot evidence in Task 5.
- Screenshot matrix → deterministic real-backend attachments in Task 5, reviewed before Task 6 publishing.

**2. Placeholder scan:** No TBD/TODO/"handle edge cases"; every code step shows full class strings and real test bodies.

**3. Type / token consistency:** Test ids `recipe-library-grid`, `recipes-view-container`, `recipes-desktop-toolbar`, and `recipe-filter-bar` are defined and queried with identical strings. Column, width, toolbar, and touch-size tokens asserted in unit tests match the implementation verbatim; Playwright separately checks their computed result.

## Out of Scope (do not implement)
- Recipe detail composition; add/edit/import flows.
- Interactive per-card favorite toggle (Decision 3).
- New features: collections, sorting, non-tag filters.
- Mobile behavior; backend/API changes.
