# Large-Screen Meals Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** On large screens (1024px+), render the Meals week grid as a full-width board — all 7 day columns and 3 meal rows visible at once with no horizontal scrolling, today's column highlighted, one-line "Sun 5" day headers — and merge the module's second chrome row ("Add ingredients" / "Fill empty slots") into the `WeekHeader` toolbar so `lg+` spends one toolbar row on chrome (the item Foundations deferred to this story). Mobile and the 768–1023px tablet range keep today's stacked day-card layout and separate action row exactly as-is.

**Architecture:** Three small, focused changes plus one refactor, all inside the existing Meals tree — **no redesign, just layout math** (spec §1). (1) Extract the two board-action buttons into a shared `MealBoardActions` fragment. (2) Give `WeekHeader` an optional `actions` slot (rendered in its right cluster) and a `data-slot="week-header"` marker. (3) In `MealsView`, gate the grid **and** the toolbar merge on one `useIsLargeScreen()` boolean: at `lg+` pass the actions into `WeekHeader` and drop the standalone row; below `lg` keep the standalone row. Also widen the shared content cap so `lg+` fills the width instead of stranding a third of the screen. (4) In `MealGrid`, make the table `w-full` (drop the `min-w-[960px]` scroll floor), switch the day `<th>`s to one-line short-weekday + date-number headers that keep the full weekday as their accessible name, tint + `aria-current="date"` today's column, and pass that full weekday into `MealSlotCard` so a focused slot includes its day context. A dedicated large-screen Playwright spec proves the no-overflow contract at every acceptance width and produces the required screenshot evidence.

**Tech Stack:** React 19, TypeScript, Vite, TanStack Query, Zustand, Tailwind CSS v4, Vitest + Testing Library, Playwright, Biome. No new dependencies. Reuses `useIsLargeScreen()` (min-width 1024px), added by the Calendar story and exported from `@/hooks`.

**Spec:** `docs/superpowers/specs/2026-07-06-large-screen-meals-design.md`
**Story:** `docs/product/backlog/large-screen-ux/large-screen-meals.md`
**Depends on (merged):** Large-screen foundations (FE PR #282 — deferred the Meals "Fill empty slots" second-row merge to this story), Large-screen Calendar (FE PR #285 — source of `useIsLargeScreen` and the one-line "Sun 5" today-header pattern this board mirrors).

**Repo:** All implementation work is in `frontend/` (separate git repo). This plan lives in the root docs repo for cross-repo planning. Branch: `feat/large-screen-meals`.

---

## Pre-flight

- [ ] Read the spec end-to-end. Non-negotiables: **no horizontal scroll of the week grid at 1024/1280/1440**; all 7 days + 3 meal rows visible; grid centered under a generous cap on very wide screens; **today's column highlighted** with a non-color-only cue; weekday headers in Calendar's one-line "Sun 5" style; **week title + range navigation + "Fill empty slots" in one toolbar row** (the Foundations-deferred merge); every existing flow (add / edit / move / fill empty slots / recipe links / ingredients-to-grocery) works unchanged; **mobile Meals visually + behaviorally unchanged**; a screenshot-critique gate before done.
- [ ] Read `frontend/CLAUDE.md`. Follow the date/time rule: use `src/lib/time-utils.ts` helpers (`formatLocalDate`, `parseLocalDate`), never raw `new Date("yyyy-mm-dd")` / `toISOString()`. Use `cn()` from `@/lib/utils` for all className merging.
- [ ] Start from the assigned FE implementation Issue. Its body must link this Story / Spec / Plan, restate the execution contract above, and supply the Issue number used by the PR's `Closes #<issue>` line. If that Issue does not exist or any link is inaccessible, stop before branching and report the blocker; do not implement directly from this plan.
- [ ] Confirm `frontend/` is clean and on the feature branch: `cd frontend && git status -sb` (Task 0 creates `feat/large-screen-meals` from the current `origin/main`, or verifies an existing branch already contains that base).
- [ ] Do **not** touch `src/components/meals/meal-day-card.tsx` (the below-`lg` stacked view), the composer/editor/scope/planning sheets, or the ingredients-to-grocery flow. Do **not** change the Home→Meals routing effects in `meals-view.tsx` (`mealPlacementDraft`, `mealSlotIntent` consumers). No backend changes.

## Verified Codebase Facts

Everything below was re-verified on 2026-07-11 against FE `origin/main` at `b8a9db6` (the merge result of Calendar PR #285). The local FE checkout was still on the equivalent pre-merge Calendar tip `9001a27`, so Task 0 deliberately bases Meals on `origin/main`, not on the currently checked-out branch.

- **`src/components/meals-view.tsx`** — the module root. `const showGrid = useMediaQuery("(min-width: 1024px)")` (`:174`) switches between the mobile stack (`displayBoard && !showGrid` → `MealDayCard` list, `:612-626`) and the desktop board (`displayBoard && showGrid` → `MealGrid`, `:628-637`). Everything sits in `<section className="flex-1 overflow-y-auto p-4 sm:p-6">` › `<div className="mx-auto flex max-w-5xl flex-col gap-4">` (`:536-537`). The **second chrome row** is `:550-569`: `{persistedBoard && !readOnly ? (<div className="flex flex-wrap justify-end gap-2"> [Add ingredients?] [Fill empty slots] </div>) : null}` — rendered on **all** widths, independent of `showGrid`. `canAddIngredients` (`:178-181`) gates the "Add ingredients" button; "Fill empty slots" opens the scope dialog via `setScopeOpen(true)`. `Button` is still needed after this plan (the error-card Retry at `:593`).
- **`src/components/meals/week-header.tsx`** — `WeekHeader({ weekStartDate, readOnly, onWeekChange })`. Root `<div className="flex items-start justify-between gap-3">`: left = `Meals` title (hidden on mobile via `useIsMobile()`), optional "Review only" badge, and `formatWeekRange(...)`; right = `<div className="flex items-center gap-1">` with the Previous/Next week icon buttons (`aria-label="Previous week"` / `"Next week"`). `formatWeekRange` is exported and reused by `meals-view.tsx`.
- **`src/components/meals/meal-grid.tsx`** — `<div className="overflow-x-auto">` › `<table aria-label="Weekly meals" className="min-w-[960px] table-fixed …">`. Header row: an empty `w-[120px]` label `<th scope="col">` then one `<th scope="col">` per day whose text is `dayName(day.date)` = the **full** weekday ("Sunday"). Body: `MEAL_ROWS = ["breakfast","lunch","dinner"]`, each a `<tr>` with a `<th scope="row">` (`formatMealType`) then one `<td>` per day wrapping `<MealSlotCard>`. `parseLocalDate` is imported; `formatLocalDate` is **not** yet.
- **`src/components/meals/meal-slot-card.tsx`** — filled card truncates its title (`truncate`) and clamps notes (`line-clamp-2`) inside `min-w-0 flex-1`; the empty-slot button is `min-h-20` (80px ≥ 44px target). Its visible eyebrow is the meal type (not provenance), which the spec now names accurately. Its focused button label includes meal type + full title but omits the day, so Task 4 adds an optional grid-only `dayLabel` without changing below-`lg` accessible names.
- **`src/hooks`** — `useIsLargeScreen()` → `useMediaQuery("(min-width: 1024px)")`, exported from the `@/hooks` barrel (`src/hooks/index.ts:8`). Same 1024px boundary the Meals grid already uses.
- **`src/lib/time-utils.ts`** — `formatLocalDate(new Date())` gives today as `"yyyy-MM-dd"`; board `day.date` is the same format (`fixtures/meals.ts` builds it with `formatLocalDate`). So **today-of-column** = `day.date === formatLocalDate(new Date())` — no new helper needed.
- **Test harness — `src/components/meals-view.test.tsx`** (2466 lines): a hoisted `viewport = { showGrid: false }` plus `vi.mock("@/hooks", … useMediaQuery: () => viewport.showGrid)` (`:37-45`) drives the breakpoint; `beforeEach` (`:178-183`) resets `viewport.showGrid = false` and calls `mockCurrentWeek()` which stubs `Date` so **now = Wed 2026-06-10 09:15** every test. `createEmptyMealsBoard()` (`fixtures/meals.ts`) defaults to `testWeekStartDate = "2026-06-07"` (the Sunday of that week), so the seeded week is 2026-06-07…2026-06-13 and **today is the Wednesday column, `day.date = "2026-06-10"`, visible number `10`**. Grid tests set `viewport.showGrid = true` and assert `getByRole("table", { name: "Weekly meals" })`, `getByRole("rowheader", { name: "Breakfast" })`, and — critically — `getByRole("columnheader", { name: "Sunday" })` (`:1068`). All action-button assertions use role+name (`getByRole("button", { name: "Fill empty slots" })`), never DOM position, so a button that moves into the header still matches.
- **Consequence for the breakpoint hook:** `useIsLargeScreen()` internally imports `useMediaQuery` from `./use-media-query` (not the barrel), so the barrel mock does **not** control it — it would read the real `window.matchMedia` (globally stubbed to `matches:false` in `src/test/setup.ts`) and return `false` in every test. Therefore switching `MealsView` to `useIsLargeScreen()` **requires** adding `useIsLargeScreen: () => viewport.showGrid` to the hoisted mock (Task 3), reusing the same `viewport.showGrid` flag so grid and toolbar-merge share one boundary — exactly the real behavior.

## File Structure

**Create:**
- `src/components/meals/meal-board-actions.tsx` (+ `.test.tsx`) — the shared "Add ingredients" + "Fill empty slots" action fragment.
- `src/components/meals/week-header.test.tsx` — focused unit tests for the new `actions` slot (WeekHeader has no dedicated test today).
- `e2e/large-screen-meals.spec.ts` — real-backend geometry checks at 1024/1280/1440/1920 plus the four-case visual evidence matrix.

**Modify:**
- `src/components/meals/week-header.tsx` — add the `actions?: ReactNode` slot + `data-slot="week-header"`.
- `src/components/meals-view.tsx` — `useIsLargeScreen()`, route actions into the toolbar at `lg+`, widen the content cap, use `MealBoardActions`.
- `src/components/meals-view.test.tsx` — add `useIsLargeScreen` to the hoisted mock; add merge + grid-header/today assertions.
- `src/components/meals/meal-grid.tsx` — `w-full` table, one-line "Sun 5" headers with preserved accessible names, today column highlight.
- `src/components/meals/meal-slot-card.tsx` — optional `dayLabel` used only by the table grid so slot controls expose row + day + full meal name to assistive technology.

No barrel file exists under `src/components/meals/` (imports are direct paths), so nothing to update there.

Each task ends with a commit. Scope commits `feat(meals):` / `refactor(meals):`.

---

## Task 0: Branch setup

**Files:** none

- [ ] **Step 1: Create the FE feature branch from the merged Calendar base**

```bash
cd /Users/joe.bor/code/family-hub/frontend
git fetch origin
git show --no-patch --oneline origin/main
git show-ref --verify --quiet refs/heads/feat/large-screen-meals \
  && git switch feat/large-screen-meals \
  || git switch -c feat/large-screen-meals origin/main
git merge-base --is-ancestor origin/main HEAD
git status -sb
```

Expected: `origin/main` shows the Calendar merge (`b8a9db6` or a newer commit), `merge-base` exits 0, and status shows a clean `feat/large-screen-meals`. If the branch exists but the ancestry check fails, stop and reconcile the branch explicitly; do not silently reset, rebase, or implement on the stale Calendar checkout.

---

## Task 1: Extract `MealBoardActions`

A pure refactor so the standalone (below-`lg`) row and the merged (`lg+`) toolbar render the same two buttons from one source. No behavior change yet — `MealsView` keeps its own inline buttons until Task 3.

**Files:**
- Create: `src/components/meals/meal-board-actions.tsx`
- Test: `src/components/meals/meal-board-actions.test.tsx`

- [ ] **Step 1: Write the failing test**

`src/components/meals/meal-board-actions.test.tsx`:

```tsx
import { render, renderWithUser, screen } from "@/test/test-utils";
import { MealBoardActions } from "./meal-board-actions";

it("renders both actions when ingredients can be added", () => {
  render(
    <MealBoardActions
      canAddIngredients
      onAddIngredients={() => {}}
      onFillEmptySlots={() => {}}
    />,
  );
  expect(
    screen.getByRole("button", { name: "Add ingredients" }),
  ).toBeInTheDocument();
  expect(
    screen.getByRole("button", { name: "Fill empty slots" }),
  ).toBeInTheDocument();
});

it("hides Add ingredients when it is not available", () => {
  render(
    <MealBoardActions
      canAddIngredients={false}
      onAddIngredients={() => {}}
      onFillEmptySlots={() => {}}
    />,
  );
  expect(
    screen.queryByRole("button", { name: "Add ingredients" }),
  ).not.toBeInTheDocument();
  expect(
    screen.getByRole("button", { name: "Fill empty slots" }),
  ).toBeInTheDocument();
});

it("invokes the callbacks on click", async () => {
  const onAddIngredients = vi.fn();
  const onFillEmptySlots = vi.fn();
  const { user } = renderWithUser(
    <MealBoardActions
      canAddIngredients
      onAddIngredients={onAddIngredients}
      onFillEmptySlots={onFillEmptySlots}
    />,
  );
  await user.click(screen.getByRole("button", { name: "Add ingredients" }));
  await user.click(screen.getByRole("button", { name: "Fill empty slots" }));
  expect(onAddIngredients).toHaveBeenCalledTimes(1);
  expect(onFillEmptySlots).toHaveBeenCalledTimes(1);
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npm test -- --run src/components/meals/meal-board-actions.test.tsx`
Expected: FAIL with "Cannot find module './meal-board-actions'".

- [ ] **Step 3: Implement `MealBoardActions`**

`src/components/meals/meal-board-actions.tsx`:

```tsx
import { Button } from "@/components/ui/button";

interface MealBoardActionsProps {
  /** Whether the visible week has a recipe-backed meal to extract ingredients from. */
  canAddIngredients: boolean;
  onAddIngredients: () => void;
  onFillEmptySlots: () => void;
}

/**
 * Editable-week board actions (Add ingredients + Fill empty slots). Rendered in
 * the WeekHeader toolbar row on large screens and in a standalone row below the
 * lg breakpoint, so both layouts share one definition.
 */
export function MealBoardActions({
  canAddIngredients,
  onAddIngredients,
  onFillEmptySlots,
}: MealBoardActionsProps) {
  return (
    <>
      {canAddIngredients ? (
        <Button type="button" variant="outline" onClick={onAddIngredients}>
          Add ingredients
        </Button>
      ) : null}
      <Button type="button" variant="outline" onClick={onFillEmptySlots}>
        Fill empty slots
      </Button>
    </>
  );
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npm test -- --run src/components/meals/meal-board-actions.test.tsx`
Expected: PASS (3 tests).

- [ ] **Step 5: Lint + commit**

```bash
npm run lint
git add src/components/meals/meal-board-actions.tsx src/components/meals/meal-board-actions.test.tsx
git commit -m "refactor(meals): extract shared MealBoardActions"
```

---

## Task 2: `WeekHeader` gains an `actions` slot

The toolbar merge needs somewhere to put the actions. Add an optional `actions` slot rendered in the header's right cluster (before the week nav) and a `data-slot="week-header"` marker so tests can assert placement. Default (no actions) keeps mobile/tablet byte-for-byte identical.

**Files:**
- Modify: `src/components/meals/week-header.tsx`
- Test: `src/components/meals/week-header.test.tsx`

- [ ] **Step 1: Write the failing test**

`src/components/meals/week-header.test.tsx`:

```tsx
import { render, screen } from "@/test/test-utils";
import { WeekHeader } from "./week-header";

it("renders provided actions inside the header toolbar alongside the nav", () => {
  render(
    <WeekHeader
      weekStartDate="2026-06-07"
      readOnly={false}
      onWeekChange={() => {}}
      actions={
        <button type="button">Fill empty slots</button>
      }
    />,
  );
  const action = screen.getByRole("button", { name: "Fill empty slots" });
  // The action lives inside the WeekHeader row (its data-slot marker)…
  expect(action.closest('[data-slot="week-header"]')).not.toBeNull();
  // …next to the week navigation.
  expect(
    screen.getByRole("button", { name: "Previous week" }),
  ).toBeInTheDocument();
});

it("omits actions and keeps week navigation when no actions are provided", () => {
  render(
    <WeekHeader
      weekStartDate="2026-06-07"
      readOnly={false}
      onWeekChange={() => {}}
    />,
  );
  expect(
    screen.queryByRole("button", { name: "Fill empty slots" }),
  ).not.toBeInTheDocument();
  expect(
    screen.getByRole("button", { name: "Next week" }),
  ).toBeInTheDocument();
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npm test -- --run src/components/meals/week-header.test.tsx`
Expected: FAIL — the `actions` prop is unknown (TS error) and no `data-slot="week-header"` element exists.

- [ ] **Step 3: Add the `actions` slot + marker**

In `src/components/meals/week-header.tsx`, add the imports:

```tsx
import type { ReactNode } from "react";
import { cn } from "@/lib/utils";
```

Extend the props:

```tsx
interface WeekHeaderProps {
  weekStartDate: string;
  readOnly: boolean;
  onWeekChange: (weekStartDate: string) => void;
  /**
   * Board actions merged into the toolbar row on large screens (Add ingredients
   * / Fill empty slots). Omitted below the lg breakpoint, where the caller
   * renders them in their own row instead.
   */
  actions?: ReactNode;
}
```

Update the signature and root element (add `data-slot`, center the row only when actions are present so the mobile/tablet header — which never passes `actions` — stays `items-start` and pixel-identical):

```tsx
export function WeekHeader({
  weekStartDate,
  readOnly,
  onWeekChange,
  actions,
}: WeekHeaderProps) {
  const isMobile = useIsMobile();
  const currentStart = parseLocalDate(weekStartDate);

  return (
    <div
      data-slot="week-header"
      className={cn(
        "flex justify-between gap-3",
        actions ? "items-center" : "items-start",
      )}
    >
```

Leave the left block (title / badge / range) exactly as-is. Replace the right-hand week-nav wrapper — change:

```tsx
      <div className="flex items-center gap-1">
        <Button
          type="button"
          variant="ghost"
          size="icon-sm"
          aria-label="Previous week"
```

to wrap the nav in an outer cluster that also holds the actions:

```tsx
      <div className="flex items-center gap-2">
        {actions}
        <div className="flex items-center gap-1">
          <Button
            type="button"
            variant="ghost"
            size="icon-sm"
            aria-label="Previous week"
```

and close the extra wrapper after the Next-week button (the existing `</div>` closes the inner `gap-1` nav; add one more `</div>` to close the new `gap-2` cluster):

```tsx
          </Button>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npm test -- --run src/components/meals/week-header.test.tsx`
Expected: PASS (2 tests).

- [ ] **Step 5: Lint + commit**

```bash
npm run lint
git add src/components/meals/week-header.tsx src/components/meals/week-header.test.tsx
git commit -m "feat(meals): add actions slot to WeekHeader toolbar"
```

---

## Task 3: `MealsView` — one large-screen boundary, merged toolbar, wider cap

Switch the module to `useIsLargeScreen()` (driving both the grid and the merge from one boundary), route the board actions into `WeekHeader` at `lg+` while keeping the standalone row below `lg`, and widen the content cap so `lg+` fills the width instead of stranding a third of the screen (spec §3, Foundations §3.3).

**Files:**
- Modify: `src/components/meals-view.tsx`
- Test: `src/components/meals-view.test.tsx`

- [ ] **Step 1: Add `useIsLargeScreen` to the hoisted `@/hooks` mock**

In `src/components/meals-view.test.tsx`, extend the mock at `:39-45` so the large-screen branch is driven by the **same** `viewport.showGrid` flag every grid test already sets (grid and merge share the 1024px boundary in reality):

```tsx
vi.mock("@/hooks", async (importOriginal) => {
  const actual = await importOriginal<typeof import("@/hooks")>();
  return {
    ...actual,
    useMediaQuery: () => viewport.showGrid,
    useIsLargeScreen: () => viewport.showGrid,
  };
});
```

- [ ] **Step 2: Run the existing suite — still green (mock is a superset)**

Run: `npm test -- --run src/components/meals-view.test.tsx`
Expected: PASS (unchanged — `MealsView` still calls `useMediaQuery`; the extra mock key is inert until Step 4).

- [ ] **Step 3: Write the failing merge tests**

Append near the existing grid tests in `src/components/meals-view.test.tsx` (they share the file's `seedMockMealsBoard`, `createEmptyMealsBoard`, `renderWithUser` helpers and the `beforeEach` current-week/`showGrid=false` setup):

```tsx
it("merges the board actions into the week toolbar on large screens", async () => {
  viewport.showGrid = true;
  seedMockMealsBoard(createEmptyMealsBoard());
  renderWithUser(<MealsView />);

  const fill = await screen.findByRole("button", { name: "Fill empty slots" });
  // Exactly one instance, living inside the WeekHeader toolbar row.
  expect(
    screen.getAllByRole("button", { name: "Fill empty slots" }),
  ).toHaveLength(1);
  expect(fill.closest('[data-slot="week-header"]')).not.toBeNull();
});

it("keeps the board actions in their own row below the lg breakpoint", async () => {
  // beforeEach leaves viewport.showGrid = false (mobile/tablet).
  seedMockMealsBoard(createEmptyMealsBoard());
  renderWithUser(<MealsView />);

  const fill = await screen.findByRole("button", { name: "Fill empty slots" });
  // Present, but NOT inside the WeekHeader — it is the standalone row.
  expect(fill.closest('[data-slot="week-header"]')).toBeNull();
});
```

- [ ] **Step 4: Run to verify they fail**

Run: `npm test -- --run src/components/meals-view.test.tsx -t "board actions"`
Expected: FAIL — today the standalone row renders on every breakpoint, so at `showGrid=true` the button is not inside `[data-slot="week-header"]`.

- [ ] **Step 5: Wire the merge in `MealsView`**

In `src/components/meals-view.tsx`:

(a) Swap the hook import — change `import { useMediaQuery } from "@/hooks";` to:

```tsx
import { useIsLargeScreen } from "@/hooks";
```

(b) Add the `MealBoardActions` import alongside the other `@/components/meals/*` imports:

```tsx
import { MealBoardActions } from "@/components/meals/meal-board-actions";
```

(c) Replace the breakpoint line (`:174`):

```tsx
  const showGrid = useMediaQuery("(min-width: 1024px)");
```

with:

```tsx
  const isLargeScreen = useIsLargeScreen();
```

(d) Build the shared actions node just before the `return` (after `addExtraFromEditor`, near `:534`):

```tsx
  const boardActions =
    persistedBoard && !readOnly ? (
      <MealBoardActions
        canAddIngredients={canAddIngredients}
        onAddIngredients={() => setAddIngredientsOpen(true)}
        onFillEmptySlots={() => setScopeOpen(true)}
      />
    ) : null;
```

(e) Widen the content cap (`:537`) — change:

```tsx
      <div className="mx-auto flex max-w-5xl flex-col gap-4">
```

to the fixed initial cap under test (change it only if Task 5's 1920px evidence rejects it):

```tsx
      <div className="mx-auto flex max-w-[1600px] flex-col gap-4">
```

(f) Pass actions into the toolbar at `lg+` — change the `WeekHeader` usage (`:538-548`) to add:

```tsx
        <WeekHeader
          weekStartDate={visibleWeekStartDate}
          readOnly={readOnly}
          onWeekChange={(weekStartDate) => {
            setVisibleWeekStartDate(weekStartDate);
            setSelectedSlot(null);
            setEditingSlotId(null);
            setAddIngredientsOpen(false);
            resetPlanningSession();
          }}
          actions={isLargeScreen ? boardActions : undefined}
        />
```

(g) Delete the existing `persistedBoard && !readOnly` standalone action block at `:550-569` in full, then insert the below-`lg` render of the shared actions in the same location:

```tsx
        {!isLargeScreen && boardActions ? (
          <div className="flex flex-wrap justify-end gap-2">{boardActions}</div>
        ) : null}
```

(h) Rename only the two remaining `showGrid` identifiers at the `MealDayCard` and `MealGrid` branch conditions (`:612`, `:628`) to `isLargeScreen`. Do not change either branch's child markup or props.

> `Button` stays imported (used by the error-card Retry at `:593`). `useMediaQuery` is no longer used in this file — remove it from the import if it was the only symbol taken from `@/hooks`; here the import becomes `useIsLargeScreen` only.

- [ ] **Step 6: Run to verify all pass**

Run: `npm test -- --run src/components/meals-view.test.tsx`
Expected: PASS — the two new merge tests plus the entire existing suite (all "Fill empty slots" / "Add ingredients" role-name queries still match; at `showGrid=true` the button is now in the header, at `showGrid=false` it is in the standalone row, exactly one instance either way).

- [ ] **Step 7: Typecheck + lint + commit**

```bash
npm run build
npm run lint
git add src/components/meals-view.tsx src/components/meals-view.test.tsx
git commit -m "feat(meals): merge board actions into toolbar and widen large-screen cap"
```

---

## Task 4: `MealGrid` — full-width board, one-line headers, today highlight, day-aware slot names

Make the grid fill its width with no scroll floor, switch day headers to the one-line "Sun 5" style (keeping the full weekday as the accessible column name), and highlight today's column with a tint + filled date pill + `aria-current="date"` (a non-color cue, spec §5). Native table headers help table navigation, but the interactive card supplies its own accessible name when it receives focus; pass the full weekday into `MealSlotCard` so that name also satisfies the spec's "Dinner, Wednesday: …" contract.

**Files:**
- Modify: `src/components/meals/meal-grid.tsx`
- Modify: `src/components/meals/meal-slot-card.tsx`
- Test: `src/components/meals-view.test.tsx` (integration, where `mockCurrentWeek()` fixes today to Wed 2026-06-10)

- [ ] **Step 1: Write the failing header/today tests**

Append near the existing grid test (`:1055`) in `src/components/meals-view.test.tsx`:

```tsx
it("uses one-line weekday headers while keeping full-day accessible names", async () => {
  viewport.showGrid = true;
  seedMockMealsBoard(createEmptyMealsBoard());
  renderWithUser(<MealsView />);
  await screen.findByRole("table", { name: "Weekly meals" });

  // Visible label is the short weekday, not the full "Wednesday".
  expect(screen.getByText("Wed")).toBeInTheDocument();
  expect(screen.queryByText("Wednesday")).not.toBeInTheDocument();
  // …but the column header's accessible name stays the full weekday (spec §5),
  // so the pre-existing `columnheader { name: "Sunday" }` assertion still holds.
  expect(
    screen.getByRole("columnheader", { name: "Sunday" }),
  ).toBeInTheDocument();
});

it("highlights today's column (Wednesday) on the large-screen board", async () => {
  viewport.showGrid = true; // now = Wed 2026-06-10 (mockCurrentWeek in beforeEach)
  seedMockMealsBoard(createEmptyMealsBoard());
  renderWithUser(<MealsView />);
  await screen.findByRole("table", { name: "Weekly meals" });

  const today = screen.getByRole("columnheader", { current: "date" });
  expect(today).toHaveAttribute("aria-label", "Wednesday");
  // Exactly one column is marked as today.
  const todayColumns = screen
    .getAllByRole("columnheader")
    .filter((header) => header.getAttribute("aria-current") === "date");
  expect(todayColumns).toHaveLength(1);
});

it("includes the full weekday in a focused grid slot's accessible name", async () => {
  viewport.showGrid = true;
  seedMockMealsBoard(
    withOccupiedMealSlot(
      createEmptyMealsBoard(),
      3,
      "dinner",
      "Turbong Manok",
    ),
  );
  renderWithUser(<MealsView />);

  expect(
    await screen.findByRole("button", {
      name: "Open dinner, Wednesday: Turbong Manok",
    }),
  ).toBeInTheDocument();
});
```

- [ ] **Step 2: Run to verify they fail**

Run: `npm test -- --run src/components/meals-view.test.tsx -t "weekday headers|highlights today|full weekday"`
Expected: FAIL — headers currently render the full "Wednesday" text, no `aria-current="date"` exists, and the focused card is named `Open dinner: Turbong Manok` without its day.

- [ ] **Step 3: Rewrite the `MealGrid` table for width + headers + today**

Replace the top of `src/components/meals/meal-grid.tsx` — change:

```tsx
import { parseLocalDate } from "@/lib/time-utils";
import type { MealBoard, MealSlot } from "@/lib/types";
import type {
  MealPlanningDraft,
  MealPlanningTarget,
} from "./meal-planning-session";
import { MealSlotCard } from "./meal-slot-card";
import { formatMealType } from "./meal-type-utils";

function dayName(date: string) {
  return parseLocalDate(date).toLocaleDateString("en-US", {
    weekday: "long",
  });
}
```

to:

```tsx
import { formatLocalDate, parseLocalDate } from "@/lib/time-utils";
import type { MealBoard, MealSlot } from "@/lib/types";
import { cn } from "@/lib/utils";
import type {
  MealPlanningDraft,
  MealPlanningTarget,
} from "./meal-planning-session";
import { MealSlotCard } from "./meal-slot-card";
import { formatMealType } from "./meal-type-utils";

function shortWeekday(date: string) {
  return parseLocalDate(date).toLocaleDateString("en-US", { weekday: "short" });
}

function fullWeekday(date: string) {
  return parseLocalDate(date).toLocaleDateString("en-US", { weekday: "long" });
}
```

Inside the component, compute today once (just after the `MealGrid({ … })` destructure, before `return`):

```tsx
  const todayIso = formatLocalDate(new Date());
```

Add the grid-only day context to `MealSlotCard`. In `src/components/meals/meal-slot-card.tsx`, add `dayLabel?: string` immediately before `onSelectSlot` in `MealSlotCardProps`, and add `dayLabel,` immediately before `onSelectSlot` in the existing function destructuring. Do not rename or reorder any other prop. Immediately after `const label = formatMealType(slot.mealType);`, add:

```tsx
  const dayContext = dayLabel ? `, ${dayLabel}` : "";
```

Replace the filled/extras button's existing `aria-label` expression with:

```tsx
        aria-label={
          draft
            ? `Draft ${slot.mealType}${dayContext}: ${draft.displayTitle}`
            : primary
              ? `Open ${slot.mealType}${dayContext}: ${primary.title}`
              : firstExtraTitle
                ? `Open ${slot.mealType}${dayContext}: extras - ${firstExtraTitle}`
                : `Open ${slot.mealType}${dayContext}: extras`
        }
```

Replace the empty-slot button's existing `aria-label` expression with:

```tsx
      aria-label={
        pendingRecipeId
          ? `Add recipe to ${slot.mealType}${dayContext}`
          : `Add ${slot.mealType} meal${dayContext}`
      }
```

Because `dayLabel` defaults to `undefined`, the stacked `MealDayCard` callers retain their current exact names (`Open dinner: …`, `Add dinner meal`) and all mobile E2E locators remain unchanged.

Replace the scrolling table wrapper + header row — change:

```tsx
  return (
    <div className="overflow-x-auto">
      <table
        aria-label="Weekly meals"
        className="min-w-[960px] table-fixed overflow-hidden rounded-lg border border-border"
      >
        <thead>
          <tr>
            <th
              scope="col"
              className="w-[120px] border-b border-r border-border bg-muted/40 p-3"
            />
            {board.days.map((day) => (
              <th
                key={day.date}
                scope="col"
                className="border-b border-r border-border bg-muted/40 p-3 text-center text-sm font-semibold text-foreground last:border-r-0"
              >
                {dayName(day.date)}
              </th>
            ))}
          </tr>
        </thead>
```

to (drop `min-w-[960px]`, add `w-full`; one-line header with preserved accessible name + today accent):

```tsx
  return (
    <div className="overflow-x-auto">
      <table
        aria-label="Weekly meals"
        className="w-full table-fixed overflow-hidden rounded-lg border border-border"
      >
        <thead>
          <tr>
            <th
              scope="col"
              className="w-[120px] border-b border-r border-border bg-muted/40 p-3"
            />
            {board.days.map((day) => {
              const isToday = day.date === todayIso;
              const dayNumber = parseLocalDate(day.date).getDate();
              return (
                <th
                  key={day.date}
                  scope="col"
                  aria-label={fullWeekday(day.date)}
                  aria-current={isToday ? "date" : undefined}
                  className={cn(
                    "border-b border-r border-border p-2 text-center last:border-r-0",
                    isToday ? "bg-primary/10" : "bg-muted/40",
                  )}
                >
                  <span className="flex items-center justify-center gap-1.5">
                    <span
                      className={cn(
                        "text-xs font-semibold uppercase tracking-wide",
                        isToday ? "text-primary" : "text-muted-foreground",
                      )}
                    >
                      {shortWeekday(day.date)}
                    </span>
                    <span
                      className={cn(
                        "flex h-6 min-w-6 items-center justify-center rounded-full px-1 text-sm font-bold tabular-nums",
                        isToday
                          ? "bg-primary text-primary-foreground"
                          : "text-foreground",
                      )}
                    >
                      {dayNumber}
                    </span>
                  </span>
                </th>
              );
            })}
          </tr>
        </thead>
```

Tint today's body cells — change the `<td>`:

```tsx
                  <td
                    key={`${day.date}-${mealType}`}
                    className="border-b border-r border-border p-2 align-top last:border-r-0"
                  >
```

to:

```tsx
                  <td
                    key={`${day.date}-${mealType}`}
                    className={cn(
                      "border-b border-r border-border p-2 align-top last:border-r-0",
                      day.date === todayIso ? "bg-primary/5" : null,
                    )}
                  >
```

Leave the row headers (`<th scope="row">`) and the `MEAL_ROWS` loop unchanged. In the existing `<MealSlotCard>` call, add exactly one prop immediately before `onSelectSlot`:

```tsx
                      dayLabel={fullWeekday(day.date)}
                      onSelectSlot={onSelectSlot}
```

- [ ] **Step 4: Run the new tests + the full Meals suite**

```bash
npm test -- --run src/components/meals-view.test.tsx
```

Expected: PASS — the two new tests, the pre-existing `columnheader { name: "Sunday" }` / `rowheader { name: "Breakfast" }` / `table { name: "Weekly meals" }` assertions, and every planning/ingredients test.

- [ ] **Step 5: Typecheck + lint + commit**

```bash
npm run build
npm run lint
git add src/components/meals/meal-grid.tsx src/components/meals/meal-slot-card.tsx src/components/meals-view.test.tsx
git commit -m "feat(meals): full-width week board with one-line headers and today highlight"
```

---

## Task 5: Real-browser geometry contract + screenshot matrix

The unit suite proves branching and semantics; it cannot prove layout. Add one real-backend Playwright spec that measures overflow at every acceptance width and attaches the exact visual matrix to the test report. This replaces the ambiguous manual "use a screenshot harness" step.

**Files:**
- Create: `e2e/large-screen-meals.spec.ts`

- [ ] **Step 1: Build the deterministic large-screen fixture**

In `e2e/large-screen-meals.spec.ts`, reuse the established helpers already used by `large-screen-home.spec.ts`: `registerFamily`, `seedBrowserAuth`, `upsertMealSlot`, `createRecipe`, `clearStorage`, `safeClick`, and `waitForHydration`. Pin the browser clock to Sunday 2026-07-05 08:00 local time, derive the week start with `formatLocalDate(getWeekStartSunday(FIXED_NOW))`, and open Meals through the desktop Primary navigation button named `Meals`.

Implement these helpers with the following exact contracts:

```tsx
const FIXED_NOW = new Date(2026, 6, 5, 8, 0, 0);
const ACCEPTANCE_WIDTHS = [1024, 1280, 1440, 1920] as const;
const SCREENSHOT_WIDTHS = [1024, 1440, 1920] as const;
const MEAL_TYPES = ["breakfast", "lunch", "dinner"] as const;

async function openMealsBoard(page: Page) {
  await safeClick(
    page
      .getByRole("navigation", { name: "Primary" })
      .getByRole("button", { name: "Meals" }),
  );
  await expect(
    page.getByRole("heading", { name: "Meals", level: 1, exact: true }),
  ).toBeVisible();
  await expect(page.getByRole("table", { name: "Weekly meals" })).toBeVisible();
}

async function seedFilledWeek(
  request: APIRequestContext,
  token: string,
  weekStartDate: string,
  titleFor: (dayIndex: number, mealType: (typeof MEAL_TYPES)[number]) => string,
) {
  for (let dayIndex = 0; dayIndex < 7; dayIndex += 1) {
    for (const mealType of MEAL_TYPES) {
      await upsertMealSlot(request, token, {
        weekStartDate,
        dayIndex,
        mealType,
        primary: {
          sourceType: "quick",
          recipeId: null,
          title: titleFor(dayIndex, mealType),
          imageUrl: null,
          note: null,
        },
        extras: [],
        note: null,
        collisionMode: null,
      });
    }
  }
}

async function attachBoardScreenshot(page: Page, name: string, width: number) {
  await page.setViewportSize({ width, height: width === 1024 ? 768 : 900 });
  await expect(page.getByRole("table", { name: "Weekly meals" })).toBeVisible();
  await test.info().attach(`${name}-${width}`, {
    body: await page.screenshot({ fullPage: true }),
    contentType: "image/png",
  });
}
```

For the **full-week** fixture, seed all 21 slots, create a recipe named `Sheet Pan Chicken` with `ingredients: ["1 lb chicken"]`, `instructions: ["Roast until cooked through."]`, `tags: ["Dinner"]`, and `favorite: false`; then upsert Sunday dinner a second time with `sourceType: "recipe"`, the returned recipe `id` / `title`, `imageUrl: null`, `extras: []`, `note: null`, and `collisionMode: "replace_primary"`. (`createRecipe` returns only `{ id, title }` in the current helper.) This makes both `Add ingredients` and `Fill empty slots` visible in the merged toolbar. For the **long-name** fixture, use `A deliberately very long ${mealType} name for day ${dayIndex + 1} that must truncate inside one fixed-height card` for all 21 slots. Empty and past cases need no meal seeds.

- [ ] **Step 2: Add objective geometry and breakpoint assertions**

Add one test that registers an empty family, opens the current Meals week, and loops over `ACCEPTANCE_WIDTHS`. At every width assert:

```tsx
const table = page.getByRole("table", { name: "Weekly meals" });
await expect(table.getByRole("columnheader")).toHaveCount(8); // blank row-label header + 7 days
await expect(table.getByRole("rowheader")).toHaveCount(3);
await expect(table.getByRole("columnheader", { current: "date" })).toHaveCount(1);

const geometry = await table.evaluate((element) => {
  const scroller = element.parentElement;
  const rect = element.getBoundingClientRect();
  if (!scroller) throw new Error("MealGrid table must have a wrapper");
  return {
    tableRight: rect.right,
    viewportWidth: document.documentElement.clientWidth,
    scrollerClientWidth: scroller.clientWidth,
    scrollerScrollWidth: scroller.scrollWidth,
    documentClientWidth: document.documentElement.clientWidth,
    documentScrollWidth: document.documentElement.scrollWidth,
  };
});
expect(geometry.scrollerScrollWidth).toBeLessThanOrEqual(
  geometry.scrollerClientWidth + 1,
);
expect(geometry.documentScrollWidth).toBeLessThanOrEqual(
  geometry.documentClientWidth + 1,
);
expect(geometry.tableRight).toBeLessThanOrEqual(geometry.viewportWidth + 1);
```

At 1024px, also assert the table's bottom edge is within the 768px viewport so "all 3 rows visible" is literal, not merely present in the DOM. At 1920px, compare the table and section bounding boxes: table width must be at most 1600px and its left/right free space within the section must differ by no more than 1px, proving the cap is centered.

Add a second breakpoint test that sets 900×800, opens Meals, and asserts there is no `Weekly meals` table, there are seven `Add dinner meal` controls, and `Fill empty slots` is outside `[data-slot="week-header"]`. Re-run the existing `Mobile Chrome` Meals E2E suite at 375px-equivalent through its configured device rather than duplicating mobile flows here.

- [ ] **Step 3: Add the four-case screenshot evidence test**

Create four separately registered families so state cannot leak between cases: `full-week`, `empty-week`, `long-names`, and `past-read-only`. For each case, seed as specified in Step 1, authenticate, reload, open Meals, and call `attachBoardScreenshot` for every `SCREENSHOT_WIDTHS` value. Before capture, assert:

- current-week cases: exactly one `aria-current="date"` column header;
- full-week: both toolbar actions exist inside `[data-slot="week-header"]` and all 21 slot buttons are present;
- empty-week: 21 `Add * meal, <weekday>` buttons are present;
- long-names at 1024px: the title element inside the first occupied slot has `scrollWidth > clientWidth` and one computed line height, proving truncation rather than wrapping;
- past-read-only: click `Previous week`, then assert `Review only` is visible, both board actions are absent, and no column header has `aria-current="date"`.

Visually review all 12 report attachments before marking the task complete. Reject and iterate if a column is clipped, the toolbar wraps, the today cue is unclear, cards become uneven, or the 1920px board is not centered. The fixed 1600px cap may be changed only in response to this evidence; if changed, update the geometry assertion and plan self-review together.

- [ ] **Step 4: Run the focused browser gates**

```bash
npm run test:e2e -- e2e/large-screen-meals.spec.ts --project=chromium
npm run test:e2e -- e2e/mobile-meals.spec.ts --project="Mobile Chrome"
```

Expected: the large-screen spec passes its 1024/1280/1440/1920 geometry loop and emits 12 named screenshot attachments; the existing mobile Meals suite passes unchanged.

- [ ] **Step 5: Whole-suite gate + commit**

```bash
npm run build
npm run lint
npm test -- --run
git add e2e/large-screen-meals.spec.ts
git commit -m "test(meals): verify large-screen board geometry and visual matrix"
```

Expected: build, lint, and the full unit/integration suite exit 0. Commit any visual correction separately with a message that names the correction (for example, `fix(meals): prevent toolbar wrap at laptop width`), then re-run Steps 3–5.

---

## Task 6: PR + post-merge housekeeping

- [ ] **Step 1: Open the PR from `feat/large-screen-meals`**

Body must include, per the root execution-issue contract:
- `Closes #<assigned FE Issue number>` as a standalone line.
- **Story / Spec / Plan** links (this file).
- **Verification:** build/lint/test results + the screenshot matrix summary + the below-1024 unchanged evidence.
- **Resolves the Foundations-deferred item:** the Meals second "Fill empty slots" chrome row is now merged into the `WeekHeader` toolbar, so every module renders one toolbar row on `lg+` (closes the deferral called out in Foundations PR #282's body).
- **Execution contract checklist** mapping each spec AC to code + tests:
  - No horizontal scroll at 1024/1280/1440; all 7 days + 3 rows visible; centered under cap → `MealGrid` `w-full table-fixed` (no `min-w`) + `MealsView` `max-w-[1600px]`; Task 5 geometry assertions and screenshot matrix.
  - Today's column highlighted → `MealGrid` today tint + filled pill + `aria-current="date"`; test "highlights today's column".
  - One-line "Sun 5" headers, full-day accessible name preserved → `MealGrid` short/`aria-label` headers; tests "one-line weekday headers" + retained `columnheader { name: "Sunday" }`.
  - Focused slots include meal type + weekday + full title → grid-only `MealSlotCard.dayLabel`; test "includes the full weekday in a focused grid slot's accessible name".
  - Week title + range nav + Fill empty slots in one toolbar row → `WeekHeader` `actions` slot + `MealsView` `lg+` routing; tests "merges the board actions into the week toolbar" / "keeps the board actions in their own row below the lg breakpoint".
  - All existing flows unchanged (add/edit/move/fill/recipe links/ingredients-to-grocery) → reused `MealBoardActions`, `MealSlotCard`, planning/editor/ingredients wiring; full existing `meals-view.test.tsx` suite green.
  - Mobile Meals unchanged → `MealDayCard` stack + standalone row untouched below `lg`; Task 5 900px breakpoint assertion + existing Mobile Chrome Meals E2E suite.
  - Screenshot review across the four cases at three widths → Task 5.

- [ ] **Step 2: Merge, then update root docs (orchestrator, post-merge)**

After merge, in the **root** repo set `docs/product/backlog/large-screen-ux/large-screen-meals.md` `status: done`, bump `updated:`, record the Issue and PR under `issues:` / `prs:`, and move its `roadmap.md` line from "Next up (plan written, ready to implement)" into "Shipped" — exactly as the Calendar and Lists stories were closed out.

---

## Self-Review (completed during planning)

**1. Spec coverage:**
- §3 Full-width board — 7 columns, no h-scroll, flex to share width, centered under cap → Task 4 (`w-full table-fixed`, dropped `min-w-[960px]`) + Task 3 (`max-w-[1600px]` cap, `mx-auto` centering). Pixel proof in Task 5.
- §3 Today's column highlighted (Calendar's treatment) → Task 4 (`bg-primary/10` header + `text-primary` + filled `bg-primary text-primary-foreground` pill + `bg-primary/5` body tint), mirroring the Calendar weekly today header.
- §3 Slot cards adapt / truncate at narrow columns → visual card CSS stays unchanged (`truncate`/`line-clamp-2` inside `min-w-0 flex-1`); Task 4 changes only the grid-only accessible name, and Task 5 proves long titles remain one line at 1024px.
- §3 One-line "Sun 5" weekday headers → Task 4 (`shortWeekday` + date pill).
- §3 Foundations chrome — week title, range navigation, and Fill empty slots in one toolbar row → Tasks 1–3 (`MealBoardActions`, `WeekHeader.actions`, `lg+` routing).
- §5 Accessibility — column headers retain the full weekday; focused grid slots add `dayLabel` so the control itself reads meal type + weekday + full title; empty targets remain `min-h-20`; and `aria-current="date"` + the filled date pill make today non-color-only. Task 4 tests each changed semantic.
- §6 AC screenshot matrix → Task 5. §7 Out of scope (no workflow/nutrition/mobile/BE changes) → respected; only board layout + toolbar placement change.

**2. Placeholder scan:** No TBD / "handle edge cases" / "similar to Task N" / open-ended screenshot-harness step remains. Runtime values (assigned Issue/PR numbers and the recipe ID returned by the fixture API) are explicitly sourced rather than guessed.

**3. Type consistency:** `MealBoardActions` props (`canAddIngredients`, `onAddIngredients`, `onFillEmptySlots`) are identical across Task 1 (definition), Task 3 (usage). `WeekHeader` `actions?: ReactNode` prop matches between Task 2 (definition) and Task 3 (usage). The single breakpoint boolean is `isLargeScreen` (from `useIsLargeScreen()`) everywhere in `MealsView` after Task 3 — no lingering `showGrid`. `MealGrid` helpers `shortWeekday` / `fullWeekday` / `todayIso` and `day.date` (yyyy-MM-dd) are used consistently; `fullWeekday(day.date)` feeds both the column label and `MealSlotCard.dayLabel`; today identity is the same string compare in the header and body cell.

**Key risks flagged:**
- The 2466-line `meals-view.test.tsx` mocks `useMediaQuery` but not `useIsLargeScreen`; switching the module to the hook **requires** the Task 3 Step 1 mock addition (documented) or every grid test silently renders the non-large branch. All action-button assertions are role/name based, so the button moving into the header does not break them.
- The visible-label change ("Sunday" → "Sun 5") would break the existing `columnheader { name: "Sunday" }` assertion **unless** `aria-label={fullWeekday}` preserves the accessible name — which Task 4 does and Step 1 re-asserts.
- A table header does not automatically become part of a focused descendant button's announced name. Task 4 therefore passes an optional day label only from `MealGrid`; below-`lg` names stay byte-for-byte compatible with existing mobile E2E locators.
- Screenshots alone do not reliably expose overflow on platforms with overlay scrollbars. Task 5 measures both the grid wrapper and document `scrollWidth` at 1024/1280/1440/1920, then uses screenshots for visual quality rather than geometry proof.
