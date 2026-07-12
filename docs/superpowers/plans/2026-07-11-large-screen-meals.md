# Large-Screen Meals Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** On large screens (1024px+), render the Meals week grid as a full-width board — all 7 day columns and 3 meal rows visible at once with no horizontal scrolling, today's column highlighted, one-line "Sun 5" day headers — and merge the module's second chrome row ("Add ingredients" / "Fill empty slots") into the `WeekHeader` toolbar so `lg+` spends one toolbar row on chrome (the item Foundations deferred to this story). Mobile and the 768–1023px tablet range keep today's stacked day-card layout and separate action row exactly as-is.

**Architecture:** Three small, focused changes plus one refactor, all inside the existing Meals tree — **no redesign, just layout math** (spec §1). (1) Extract the two board-action buttons into a shared `MealBoardActions` fragment. (2) Give `WeekHeader` an optional `actions` slot (rendered in its right cluster) and a `data-slot="week-header"` marker. (3) In `MealsView`, gate the grid **and** the toolbar merge on one `useIsLargeScreen()` boolean: at `lg+` pass the actions into `WeekHeader` and drop the standalone row; below `lg` keep the standalone row. Also widen the shared content cap so `lg+` fills the width instead of stranding a third of the screen. (4) In `MealGrid`, make the table `w-full` (drop the `min-w-[960px]` scroll floor), switch the day `<th>`s to one-line short-weekday + date-number headers that keep the full weekday as their accessible name, and tint + `aria-current="date"` today's column.

**Tech Stack:** React 19, TypeScript, Vite, TanStack Query, Zustand, Tailwind CSS v4, Vitest + Testing Library, Playwright, Biome. No new dependencies. Reuses `useIsLargeScreen()` (min-width 1024px), added by the Calendar story and exported from `@/hooks`.

**Spec:** `docs/superpowers/specs/2026-07-06-large-screen-meals-design.md`
**Story:** `docs/product/backlog/large-screen-ux/large-screen-meals.md`
**Depends on (merged):** Large-screen foundations (FE PR #282 — deferred the Meals "Fill empty slots" second-row merge to this story), Large-screen Calendar (FE PR #285 — source of `useIsLargeScreen` and the one-line "Sun 5" today-header pattern this board mirrors).

**Repo:** All implementation work is in `frontend/` (separate git repo). This plan lives in the root docs repo for cross-repo planning. Branch: `feat/large-screen-meals`.

---

## Pre-flight

- [ ] Read the spec end-to-end. Non-negotiables: **no horizontal scroll of the week grid at 1024/1280/1440**; all 7 days + 3 meal rows visible; grid centered under a generous cap on very wide screens; **today's column highlighted** with a non-color-only cue; weekday headers in Calendar's one-line "Sun 5" style; **week title + range navigation + "Fill empty slots" in one toolbar row** (the Foundations-deferred merge); every existing flow (add / edit / move / fill empty slots / recipe links / ingredients-to-grocery) works unchanged; **mobile Meals visually + behaviorally unchanged**; a screenshot-critique gate before done.
- [ ] Read `frontend/CLAUDE.md`. Follow the date/time rule: use `src/lib/time-utils.ts` helpers (`formatLocalDate`, `parseLocalDate`), never raw `new Date("yyyy-mm-dd")` / `toISOString()`. Use `cn()` from `@/lib/utils` for all className merging.
- [ ] Confirm `frontend/` is clean and on the feature branch: `cd frontend && git status -sb` (Task 0 creates/reuses `feat/large-screen-meals`).
- [ ] Do **not** touch `src/components/meals/meal-day-card.tsx` (the below-`lg` stacked view), the composer/editor/scope/planning sheets, or the ingredients-to-grocery flow. Do **not** change the Home→Meals routing effects in `meals-view.tsx` (`mealPlacementDraft`, `mealSlotIntent` consumers). No backend changes.

## Verified Codebase Facts

Everything below is verified against the current `frontend/` tree.

- **`src/components/meals-view.tsx`** — the module root. `const showGrid = useMediaQuery("(min-width: 1024px)")` (`:174`) switches between the mobile stack (`displayBoard && !showGrid` → `MealDayCard` list, `:612-626`) and the desktop board (`displayBoard && showGrid` → `MealGrid`, `:628-637`). Everything sits in `<section className="flex-1 overflow-y-auto p-4 sm:p-6">` › `<div className="mx-auto flex max-w-5xl flex-col gap-4">` (`:536-537`). The **second chrome row** is `:550-569`: `{persistedBoard && !readOnly ? (<div className="flex flex-wrap justify-end gap-2"> [Add ingredients?] [Fill empty slots] </div>) : null}` — rendered on **all** widths, independent of `showGrid`. `canAddIngredients` (`:178-181`) gates the "Add ingredients" button; "Fill empty slots" opens the scope dialog via `setScopeOpen(true)`. `Button` is still needed after this plan (the error-card Retry at `:593`).
- **`src/components/meals/week-header.tsx`** — `WeekHeader({ weekStartDate, readOnly, onWeekChange })`. Root `<div className="flex items-start justify-between gap-3">`: left = `Meals` title (hidden on mobile via `useIsMobile()`), optional "Review only" badge, and `formatWeekRange(...)`; right = `<div className="flex items-center gap-1">` with the Previous/Next week icon buttons (`aria-label="Previous week"` / `"Next week"`). `formatWeekRange` is exported and reused by `meals-view.tsx`.
- **`src/components/meals/meal-grid.tsx`** — `<div className="overflow-x-auto">` › `<table aria-label="Weekly meals" className="min-w-[960px] table-fixed …">`. Header row: an empty `w-[120px]` label `<th scope="col">` then one `<th scope="col">` per day whose text is `dayName(day.date)` = the **full** weekday ("Sunday"). Body: `MEAL_ROWS = ["breakfast","lunch","dinner"]`, each a `<tr>` with a `<th scope="row">` (`formatMealType`) then one `<td>` per day wrapping `<MealSlotCard>`. `parseLocalDate` is imported; `formatLocalDate` is **not** yet.
- **`src/components/meals/meal-slot-card.tsx`** — filled card truncates its title (`truncate`) and clamps notes (`line-clamp-2`) inside `min-w-0 flex-1`; the empty-slot button is `min-h-20` (80px ≥ 44px target). No changes needed — it already truncates for narrow columns (spec §3/§5).
- **`src/hooks`** — `useIsLargeScreen()` → `useMediaQuery("(min-width: 1024px)")`, exported from the `@/hooks` barrel (`src/hooks/index.ts:8`). Same 1024px boundary the Meals grid already uses.
- **`src/lib/time-utils.ts`** — `formatLocalDate(new Date())` gives today as `"yyyy-MM-dd"`; board `day.date` is the same format (`fixtures/meals.ts` builds it with `formatLocalDate`). So **today-of-column** = `day.date === formatLocalDate(new Date())` — no new helper needed.
- **Test harness — `src/components/meals-view.test.tsx`** (2466 lines): a hoisted `viewport = { showGrid: false }` plus `vi.mock("@/hooks", … useMediaQuery: () => viewport.showGrid)` (`:37-45`) drives the breakpoint; `beforeEach` (`:178-183`) resets `viewport.showGrid = false` and calls `mockCurrentWeek()` which stubs `Date` so **now = Wed 2026-06-10 09:15** every test. `createEmptyMealsBoard()` (`fixtures/meals.ts`) defaults to `testWeekStartDate = "2026-06-07"` (the Sunday of that week), so the seeded week is 2026-06-07…2026-06-13 and **today is the Wednesday column, `day.date = "2026-06-10"`, visible number `10`**. Grid tests set `viewport.showGrid = true` and assert `getByRole("table", { name: "Weekly meals" })`, `getByRole("rowheader", { name: "Breakfast" })`, and — critically — `getByRole("columnheader", { name: "Sunday" })` (`:1068`). All action-button assertions use role+name (`getByRole("button", { name: "Fill empty slots" })`), never DOM position, so a button that moves into the header still matches.
- **Consequence for the breakpoint hook:** `useIsLargeScreen()` internally imports `useMediaQuery` from `./use-media-query` (not the barrel), so the barrel mock does **not** control it — it would read the real `window.matchMedia` (globally stubbed to `matches:false` in `src/test/setup.ts`) and return `false` in every test. Therefore switching `MealsView` to `useIsLargeScreen()` **requires** adding `useIsLargeScreen: () => viewport.showGrid` to the hoisted mock (Task 3), reusing the same `viewport.showGrid` flag so grid and toolbar-merge share one boundary — exactly the real behavior.

## File Structure

**Create:**
- `src/components/meals/meal-board-actions.tsx` (+ `.test.tsx`) — the shared "Add ingredients" + "Fill empty slots" action fragment.
- `src/components/meals/week-header.test.tsx` — focused unit tests for the new `actions` slot (WeekHeader has no dedicated test today).

**Modify:**
- `src/components/meals/week-header.tsx` — add the `actions?: ReactNode` slot + `data-slot="week-header"`.
- `src/components/meals-view.tsx` — `useIsLargeScreen()`, route actions into the toolbar at `lg+`, widen the content cap, use `MealBoardActions`.
- `src/components/meals-view.test.tsx` — add `useIsLargeScreen` to the hoisted mock; add merge + grid-header/today assertions.
- `src/components/meals/meal-grid.tsx` — `w-full` table, one-line "Sun 5" headers with preserved accessible names, today column highlight.

No barrel file exists under `src/components/meals/` (imports are direct paths), so nothing to update there.

Each task ends with a commit. Scope commits `feat(meals):` / `refactor(meals):`.

---

## Task 0: Branch setup

**Files:** none

- [ ] **Step 1: Ensure the FE feature branch exists and is current**

```bash
cd /Users/joe.bor/code/family-hub/frontend
git fetch origin
git checkout feat/large-screen-meals 2>/dev/null || git checkout -b feat/large-screen-meals origin/main
git status -sb
```

Expected: on `feat/large-screen-meals`, clean tree.

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

to (a generous cap that fits seven comfortable day columns and centers on very wide displays; tunable in the screenshot gate):

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

(g) Replace the standalone action row (`:550-569`) — the whole `{persistedBoard && !readOnly ? (<div className="flex flex-wrap justify-end gap-2">…</div>) : null}` block — with the below-`lg` render of the shared actions:

```tsx
        {!isLargeScreen && boardActions ? (
          <div className="flex flex-wrap justify-end gap-2">{boardActions}</div>
        ) : null}
```

(h) Rename the two remaining `showGrid` reads (`:612` and `:628`) to `isLargeScreen`:

```tsx
        {displayBoard && !isLargeScreen ? (
          <div className="space-y-4">
            {/* …MealDayCard stack unchanged… */}
```

```tsx
        {displayBoard && isLargeScreen ? (
          <MealGrid
            {/* …props unchanged… */}
```

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

## Task 4: `MealGrid` — full-width board, one-line headers, today highlight

Make the grid fill its width with no scroll floor, switch day headers to the one-line "Sun 5" style (keeping the full weekday as the accessible name so screen-reader column association is unchanged), and highlight today's column with a tint + filled date pill + `aria-current="date"` (a non-color cue, spec §5).

**Files:**
- Modify: `src/components/meals/meal-grid.tsx`
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
```

- [ ] **Step 2: Run to verify they fail**

Run: `npm test -- --run src/components/meals-view.test.tsx -t "weekday headers"`
and `npm test -- --run src/components/meals-view.test.tsx -t "highlights today"`
Expected: FAIL — headers currently render the full "Wednesday" text and no `aria-current="date"` exists.

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

Leave the row headers (`<th scope="row">`), the `MEAL_ROWS` loop, and the `<MealSlotCard>` props unchanged.

- [ ] **Step 4: Run the new tests + the full Meals suite**

```bash
npm test -- --run src/components/meals-view.test.tsx
```

Expected: PASS — the two new tests, the pre-existing `columnheader { name: "Sunday" }` / `rowheader { name: "Breakfast" }` / `table { name: "Weekly meals" }` assertions, and every planning/ingredients test.

- [ ] **Step 5: Typecheck + lint + commit**

```bash
npm run build
npm run lint
git add src/components/meals/meal-grid.tsx src/components/meals-view.test.tsx
git commit -m "feat(meals): full-width week board with one-line headers and today highlight"
```

---

## Task 5: Full verification + screenshot matrix

Mirror the Calendar/Lists evidence discipline. Screenshots require a real backend (root `MEMORY.md` → E2E against real BE).

- [ ] **Step 1: Whole-suite gate**

```bash
npm run build     # tsc + vite, exit 0
npm run lint      # Biome, exit 0
npm test -- --run # full unit/integration suite green
```

- [ ] **Step 2: Capture the spec's screenshot matrix (spec §6)**

Start the real backend, seed a week, then capture at **1024px, 1440px, and a large-display width** (e.g. 1920px), each across the four cases:
- **full week planned** (every slot filled),
- **empty week** (add affordances only),
- **long meal names** (confirm slot text truncates instead of wrapping into tall cells),
- **a past (read-only) week** (actions hidden → the toolbar is title + range + nav only; no today highlight).

Confirm for each: **no horizontal scroll of the grid**, all 7 day columns + 3 rows visible, the grid centered under its cap on the widest width, today's column visibly highlighted (accent header + filled date pill), and the merged toolbar is **one row** (title/range left, Add ingredients / Fill empty slots + week nav right).

```bash
# from frontend/
docker compose -f docker-compose.e2e.yml up -d --wait   # real BE on :8080
npm run dev                                              # or the project's screenshot harness
# …capture matrix…
docker compose -f docker-compose.e2e.yml down
```

- [ ] **Step 3: Prove below-1024 is unchanged**

Capture Meals at **900px** (tablet) and **375px** (mobile) and diff against `origin/main`: the stacked `MealDayCard` list, the module-aware header, and the standalone "Add ingredients / Fill empty slots" action row must be visually identical (the merge and the wider cap only bite at `lg+`).

- [ ] **Step 4: Iterate**

If any screenshot violates the spec — grid scrolls, a column is cut off, today accent is color-only, toolbar wraps to two rows at `lg+`, slot text wraps instead of truncating at 1024px, or a below-`lg` regression — fix and re-capture before proceeding. If the 1600px cap looks too wide/narrow for comfortable columns, tune the `max-w-[…]` value here. Commit fixes with `fix(meals): …`.

> Known acceptable behavior to confirm, not treat as a regression: at the narrowest 1024px columns the filled slot card is tight (56px thumbnail + truncated title). Confirm it reads well in the screenshots; if the reviewer wants more room, dropping the thumbnail at the narrowest columns is a follow-up, not part of this story's "layout math" scope.

---

## Task 6: PR + post-merge housekeeping

- [ ] **Step 1: Open the PR from `feat/large-screen-meals`**

Body must include, per the root execution-issue contract:
- **Story / Spec / Plan** links (this file).
- **Verification:** build/lint/test results + the screenshot matrix summary + the below-1024 unchanged evidence.
- **Resolves the Foundations-deferred item:** the Meals second "Fill empty slots" chrome row is now merged into the `WeekHeader` toolbar, so every module renders one toolbar row on `lg+` (closes the deferral called out in Foundations PR #282's body).
- **Execution contract checklist** mapping each spec AC to code + tests:
  - No horizontal scroll at 1024/1280/1440; all 7 days + 3 rows visible; centered under cap → `MealGrid` `w-full table-fixed` (no `min-w`) + `MealsView` `max-w-[1600px]`; Task 5 screenshot matrix.
  - Today's column highlighted → `MealGrid` today tint + filled pill + `aria-current="date"`; test "highlights today's column".
  - One-line "Sun 5" headers, full-day accessible name preserved → `MealGrid` short/`aria-label` headers; tests "one-line weekday headers" + retained `columnheader { name: "Sunday" }`.
  - Week title + range nav + Fill empty slots in one toolbar row → `WeekHeader` `actions` slot + `MealsView` `lg+` routing; tests "merges the board actions into the week toolbar" / "keeps the board actions in their own row below the lg breakpoint".
  - All existing flows unchanged (add/edit/move/fill/recipe links/ingredients-to-grocery) → reused `MealBoardActions`, `MealSlotCard`, planning/editor/ingredients wiring; full existing `meals-view.test.tsx` suite green.
  - Mobile Meals unchanged → `MealDayCard` stack + standalone row untouched below `lg`; 900/375px screenshots.
  - Screenshot review across the four cases at three widths → Task 5.

- [ ] **Step 2: Merge, then update root docs (orchestrator, post-merge)**

After merge, in the **root** repo set `docs/product/backlog/large-screen-ux/large-screen-meals.md` `status: done`, bump `updated:`, record the PR under `prs:`, and move its `roadmap.md` line from "Next up (spec written, plan not yet written)" into "Shipped" — exactly as the Calendar and Lists stories were closed out.

---

## Self-Review (completed during planning)

**1. Spec coverage:**
- §3 Full-width board — 7 columns, no h-scroll, flex to share width, centered under cap → Task 4 (`w-full table-fixed`, dropped `min-w-[960px]`) + Task 3 (`max-w-[1600px]` cap, `mx-auto` centering). Pixel proof in Task 5.
- §3 Today's column highlighted (Calendar's treatment) → Task 4 (`bg-primary/10` header + `text-primary` + filled `bg-primary text-primary-foreground` pill + `bg-primary/5` body tint), mirroring the Calendar weekly today header.
- §3 Slot cards adapt / truncate at narrow columns → no code change needed (`MealSlotCard` already `truncate`/`line-clamp-2` inside `min-w-0 flex-1`); confirmed in Verified Facts + Task 5 long-name case.
- §3 One-line "Sun 5" weekday headers → Task 4 (`shortWeekday` + date pill).
- §3 Foundations chrome — week title, range navigation, and Fill empty slots in one toolbar row → Tasks 1–3 (`MealBoardActions`, `WeekHeader.actions`, `lg+` routing).
- §5 Accessibility — header associations keep full weekday, ≥44px add targets, truncation keeps full names, today not color-only → Task 4 (`aria-label={fullWeekday}` preserves association + retains the `columnheader { name: "Sunday" }` test; `aria-current="date"` + filled pill are non-color cues); empty-slot button stays `min-h-20`; `MealSlotCard` title `aria-label` already carries the full name.
- §6 AC screenshot matrix → Task 5. §7 Out of scope (no workflow/nutrition/mobile/BE changes) → respected; only board layout + toolbar placement change.

**2. Placeholder scan:** No TBD / "handle edge cases" / "similar to Task N" — every code step carries the actual before/after. The one intentional deferral (thumbnail at 1024px) is flagged as out-of-scope, not a gap.

**3. Type consistency:** `MealBoardActions` props (`canAddIngredients`, `onAddIngredients`, `onFillEmptySlots`) are identical across Task 1 (definition), Task 3 (usage). `WeekHeader` `actions?: ReactNode` prop matches between Task 2 (definition) and Task 3 (usage). The single breakpoint boolean is `isLargeScreen` (from `useIsLargeScreen()`) everywhere in `MealsView` after Task 3 — no lingering `showGrid`. `MealGrid` helpers `shortWeekday` / `fullWeekday` / `todayIso` and `day.date` (yyyy-MM-dd) are used consistently; today identity is the same string compare (`day.date === todayIso`) in both the header and the body cell.

**Key risks flagged:**
- The 2466-line `meals-view.test.tsx` mocks `useMediaQuery` but not `useIsLargeScreen`; switching the module to the hook **requires** the Task 3 Step 1 mock addition (documented) or every grid test silently renders the non-large branch. All action-button assertions are role/name based, so the button moving into the header does not break them.
- The visible-label change ("Sunday" → "Sun 5") would break the existing `columnheader { name: "Sunday" }` assertion **unless** `aria-label={fullWeekday}` preserves the accessible name — which Task 4 does and Step 1 re-asserts.
