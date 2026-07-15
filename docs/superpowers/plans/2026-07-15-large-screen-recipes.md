# Large-Screen Recipes Card Grid Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the large-screen Recipes index from a single stack of full-width "posters" into a browsable 2–4 column responsive card grid, without touching mobile, recipe detail, or add/edit flows.

**Architecture:** The work is CSS-composition only, in `recipes-view.tsx`. `RecipeLibraryCard` already flips to a vertical `md:aspect-[4/3]` poster, so no card changes are needed. Three surgical edits: (1) give the index grid responsive column classes, (2) widen the shared container to ~1200px **only in list mode** so recipe detail keeps its current reading width, and (3) keep the empty / no-results states centered inside the wider container. No new components, hooks, state, or backend changes.

**Tech Stack:** React 19, TypeScript, Tailwind CSS v4 (arbitrary variants), Vitest + Testing Library, Biome.

---

## Source Documents

- **Story:** `docs/product/backlog/large-screen-ux/large-screen-recipes.md`
- **Spec:** `docs/superpowers/specs/2026-07-06-large-screen-recipes-design.md`
- This plan finishes the **large-screen-ux epic** (Recipes is the last of 7 surfaces).

## Key Decisions (read before coding)

1. **Detail stays untouched via a mode-conditional container width.** The container `<div>` in `recipes-view.tsx` wraps the header, the list, *and* the detail view (inside `ScreenTransition`). We widen it (`lg:max-w-[1200px]`) **only when `selectedRecipeId === null`** (list mode). In detail mode it keeps `max-w-3xl`, so `recipe-detail-view.tsx` renders pixel-identical to today. Trade-off: opening a recipe shrinks the container 1200→768 as the slide runs. This is intentional and acceptable — detail keeps its readable column. Do **not** add a max-width wrapper inside the detail view.

2. **Column tiers are keyed to the viewport to match the spec exactly:** 2 cols at 1024, 3 at 1280, 4 at 1440+. Because the container caps at ~1200px, the 3→4 step at 1440 makes cards denser (more recipes visible) at the same content width — this is the spec's intent, not a bug. Tailwind's `2xl` is 1536 and would miss the 1440 target, so the 4-col tier uses the arbitrary variant `min-[1440px]:`.

3. **The card's favorite heart stays display-only (unchanged).** The spec §3/§5 mentions a "favorite toggle" on the card, but the current `RecipeLibraryCard` heart is a non-interactive badge and favoriting happens in detail. The story's acceptance criteria only require "favorites *filtering* works unchanged" and list new interactions as out of scope. Adding a per-card mutating toggle is deferred as a possible fast-follow; **do not** add it in this pass.

4. **Mobile is untouched.** Base classes (`grid-cols-1`, `max-w-3xl`) preserve today's mobile/tablet rendering; every new treatment is gated at `lg`/`xl`/`min-[1440px]`.

## File Structure

| File | Change | Responsibility |
|------|--------|----------------|
| `frontend/src/components/recipes-view.tsx` | Modify | Index composition: responsive grid columns, list-mode container width, centered empty/no-results |
| `frontend/src/components/recipes-view.test.tsx` | Modify | Add class-token assertions for grid columns + mode-conditional width; keep existing behavioral tests green |

No other files change. `recipe-library-card.tsx`, `recipe-filter-bar.tsx`, and `recipe-detail-view.tsx` are **not** modified.

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
grep -n 'mx-auto flex max-w-3xl' src/components/recipes-view.tsx   # expect line ~161
grep -n 'grid gap-3'            src/components/recipes-view.tsx    # expect ~250 (skeleton) and ~329 (loaded)
```

Expected: the `grep`s print the lines noted above. If they differ, re-read `recipes-view.tsx` and adjust anchors before editing.

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

  await screen.findByRole("button", { name: "Back" });
  expect(screen.getByTestId("recipes-view-container")).not.toHaveClass(
    "lg:max-w-[1200px]",
  ); // detail mode: reading width only
});
```

Note: the detail view's back control is an icon button labeled `Back` (see `recipe-detail-view.tsx`). If your local copy labels it differently, `await screen.findByRole("heading", { name: testRecipeDetail.title })` is an equivalent "detail is showing" wait.

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

## Task 3: Keep empty and no-results states centered in the wide container

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
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && npm test -- --run recipes-view.test.tsx -t "centers the empty state"`
Expected: FAIL — `expect(element).toHaveClass("mx-auto")` (panel is currently full-width).

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

Run: `cd frontend && npm test -- --run recipes-view.test.tsx -t "centers the empty state"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/components/recipes-view.tsx src/components/recipes-view.test.tsx
git commit -m "feat(recipes): center empty and no-results states in wide grid"
```

---

## Task 4: Full verification, screenshot matrix, and PR

**Files:** none (verification + PR only)

- [ ] **Step 1: Run the full check suite**

```bash
cd frontend
npm run lint
npm test -- --run src/components/recipes-view.test.tsx
npm run build
```

Expected: lint clean; the full `recipes-view` suite (existing behavioral tests for filtering, empty/no-results, FAB placement, detail open/close **plus** the three new tests) passes; `tsc` + Vite build succeed.

- [ ] **Step 2: Manual screenshot matrix (story requires "Screenshot review per spec matrix, iterated before done")**

```bash
cd frontend && npm run dev   # http://localhost:5173  (needs a running BE + seeded recipes)
```

Capture and eyeball each spec §6 case at **1024, 1280, and 1440px** widths:
- [ ] 1 recipe (no single card fills the viewport)
- [ ] ~a dozen recipes (2 cols @1024, 3 @1280, 4 @1440; images at the capped ~4:3 ratio)
- [ ] favorites-only filter applied
- [ ] no-results search (centered panel)
- [ ] image-less recipes (calm "No photo" placeholder at the same ratio, grid stays uniform)
- [ ] **Mobile regression check** at 375px: unchanged horizontal row cards, FAB present
- [ ] Open a recipe from the grid: detail renders at its current reading width (not stretched)

Iterate on spacing/column feel if any case looks off, then re-run Step 1.

- [ ] **Step 3: Open the PR**

```bash
git push -u origin feat/large-screen-recipes
gh pr create --repo joe-bor/FamilyHub \
  --title "feat(recipes): large-screen responsive card grid" \
  --body "Closes #<ISSUE_NUMBER>

## Summary
- Large-screen Recipes index becomes a 2–4 column responsive card grid (2 @1024, 3 @1280, 4 @1440+) inside a ~1200px container.
- Container widens in list mode only, so recipe detail keeps its current reading width.
- Empty and no-results states stay centered; mobile and the card component are unchanged.

## Story / spec / plan
- Story: docs/product/backlog/large-screen-ux/large-screen-recipes.md
- Spec: docs/superpowers/specs/2026-07-06-large-screen-recipes-design.md
- Plan: docs/superpowers/plans/2026-07-15-large-screen-recipes.md

## Verification
- npm run lint / npm test -- --run recipes-view.test.tsx / npm run build all pass
- Screenshot matrix at 1024 / 1280 / 1440 + 375px mobile regression"
```

Replace `<ISSUE_NUMBER>` with the FE issue this plan is executed under.

---

## Self-Review (completed while writing this plan)

**1. Spec coverage:**
- 2–4 column grid at 1024/1280/1440 → Task 1 (`lg:grid-cols-2 xl:grid-cols-3 min-[1440px]:grid-cols-4`).
- Capped-ratio images, no card fills viewport → already handled by `RecipeLibraryCard` (`md:aspect-[4/3]`), verified in Task 4 screenshots; container cap (Task 2) prevents full-viewport cards.
- Search + favorites filtering unchanged → no filter logic touched; existing tests (`"matches recipes by title and tags"`, `"shows only favorites…"`) remain green (Task 4 Step 1).
- Empty + no-results centered/intact → Task 3.
- Mobile unchanged → base classes preserved; Task 4 Step 2 mobile regression check.
- Image-less placeholder uniform → existing card "No photo" branch; verified Task 4.
- Screenshot matrix → Task 4 Step 2.

**2. Placeholder scan:** No TBD/TODO/"handle edge cases"; every code step shows full class strings and real test bodies.

**3. Type / token consistency:** Test ids `recipe-library-grid` (Task 1) and `recipes-view-container` (Task 2) are each defined where added and queried by the same string. Column/width tokens asserted in tests match the tokens implemented verbatim.

## Out of Scope (do not implement)
- Recipe detail composition; add/edit/import flows.
- Interactive per-card favorite toggle (Decision 3).
- New features: collections, sorting, non-tag filters.
- Mobile behavior; backend/API changes.
