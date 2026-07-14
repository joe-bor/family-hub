# Large-Screen Chores Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** On large screens (`lg`, 1024px+), render Chores as a full-width, full-height three-column board — Today weighted as the primary column, This Week and This Month supporting — that fills the viewport with each column scrolling its own routines, instead of today's `max-w-6xl` centered grid that leaves dead side margins and a whitespace pool below the columns. Mobile (<768px) and portrait tablet (768–1023px) render exactly as they do today.

**Architecture:** Keep `ChoresView` as the single owner of data, mutation handlers, states, and the create sheet (no router, no duplicated wiring). Add `useIsLargeScreen()` and branch only two things on it: (1) the container classes — at `lg+` the region becomes a non-scrolling `flex` column capped at `max-w-[1600px]` so the board fills height; below `lg` it keeps today's `overflow-y-auto` / `max-w-6xl` byte-for-byte; and (2) the populated-board render — at `lg+` a new presentational `ChoresBoardLarge` (three fill-height `ChoreScopeColumn`s, Today weighted `flex-[1.4]` + emphasized), below `lg` the current `grid lg:grid-cols-3`. `ChoreScopeColumn` gains additive `fillHeight` / `emphasis` / `className` props (defaults preserve the current markup). `ChoreRow`'s checkoff grows to 44px via a `lg:` utility, which only takes effect where the large board renders. This is the same in-place, breakpoint-gated approach Meals v2 used (PR #287); Lists' master/detail structure does not map onto a board, but its principle — spend the whole viewport deliberately — is exactly what "fill the screen" means here.

**Tech Stack:** React 19, TypeScript, Vite, TanStack Query, Tailwind CSS v4 (`cn()` from `@/lib/utils`), Vitest + Testing Library, Playwright, Biome. No new dependencies. `useIsLargeScreen()` (min-width 1024px, exported from `@/hooks`, added by the Calendar story) is reused unchanged; its breakpoint equals Tailwind `lg`, so `lg:` utilities activate exactly when the large board mounts.

**Spec:** `docs/superpowers/specs/2026-07-06-large-screen-chores-design.md`
**Story:** `docs/product/backlog/large-screen-ux/large-screen-chores.md`
**Depends on (merged):** Large-screen foundations (FE PR #282 — slim header + single toolbar row), Large-screen Calendar (FE PR #285 — source of `useIsLargeScreen`), Large-screen Meals v2 (FE PR #287 — the full-height container pattern mirrored here).

**Repo:** All implementation work is in `frontend/` (separate git repo). This plan lives in the root docs repo for cross-repo planning. Branch: `feat/large-screen-chores`.

### Spec delta / decision (read before Task 1)

The spec (§3, "Refine, Don't Restructure") calls for widening beyond `max-w-6xl`, weighting Today, bigger checkoff targets, and member identity forward — but is silent on **vertical** space. The user directive for this story is to "use the screen real estate to its fullest," matching the Meals arc (conservative v1 → full-height v2). This plan therefore adds **full-height columns with independent internal scroll** on top of the spec's stated refinements. This stays inside the spec's "still three scope columns, no board-axis redesign" boundary (Today / This Week / This Month are unchanged; member-lane mode remains rejected/out of scope) — it is a vertical-fill refinement, not a restructure. No separate v2 spec doc is created; this section records the decision.

---

## Current Code (grounding — verified against the `frontend/` tree)

- **`src/components/chores-view.tsx`** — the module root and single data/handler owner. Uses `useIsMobile()`; holds `selectedScopeKey` + `isCreateOpen`; calls `useChoresBoard()` and the four mutation hooks (`useCreateChoreTemplate`, `useUpdateChoreTemplate`, `useCompleteChoreForCurrentPeriod`, `useUncompleteChoreForCurrentPeriod`) with `showStalePeriodRecovery` on completion errors. Derives `hasRoutines`, `visibleScopes` (mobile → one selected scope; desktop → all three), `activeFrom`, `canCreate`. Renders: outer `div.flex-1.overflow-y-auto.p-4.sm:p-6` (with mobile FAB scroll padding) → inner `div.mx-auto.max-w-6xl` → header row (`Chores` h1 + Add icon button on desktop; `ChoreScopeSwitcher` on mobile) → loading / error / offline / family-empty states → `div.grid.gap-4.lg:grid-cols-3` mapping `visibleScopes` to `ChoreScopeColumn`. Below: `ChoreFormSheet` + mobile-only `FloatingActionButton`.
- **`src/components/chores/chores-scope-column.tsx`** — `<section>` card (`rounded-lg border bg-card p-4 shadow-sm`) with an accessible name of the scope heading ("Today" / "This Week" / "This Month"), a heading+summary header (`showHeading` toggles `<header>` vs a summary-only `<p>`), an "All caught up" pill when fully complete, a per-timeframe empty `<p>`, and a `space-y-5` list of `ChoreAssigneeGroup`. Props today: `scope`, `showHeading?`, `onArchive?`, `onComplete?`, `onUncomplete?` (handlers are `(scope, chore) => void`).
- **`src/components/chores/chore-assignee-group.tsx`** — member header (colored `h-10 w-10` avatar circle with initial via `colorMap[color].bg`, name, "N left, M done", progress bar) + `space-y-2` list of `ChoreRow`. **Already renders avatar + name + member color** (spec AC-3); left unchanged by this plan (see self-review).
- **`src/components/chores/chore-row.tsx`** — `min-h-14` row; the checkoff is a raw `<button>` (`h-9 w-9` = 36px, `rounded-full border-2`) with a comment explaining it must stay a raw button (not `usePressable`) so the completion haptic isn't throttled away — **preserve that button and its click handler exactly**; only its size classes change. Title (`truncate`) + cadence label + Archive `<Button>`.
- **`src/components/chores/chores-scope-switcher.tsx`** — mobile Day/Week/Month tablist. Not touched.
- **`src/lib/types/chores.ts`** — `ChoresBoard { timezone, today, thisWeek, thisMonth: ChoreScopeBoard }`; `ChoreScopeBoard { scope, periodStartDate, periodEndDate, summary{total,completed,remaining}, assignees: ChoreAssigneeGroup[] }`; `ChoreAssigneeGroup { member{id,name,color}, summary, chores: ChoreBoardItem[] }`; `ChoreBoardItem { templateId, title, cadence, assignedToMemberId, completed, completedAt }`.
- **`src/hooks/use-is-large-screen.ts`** — `useIsLargeScreen()` → `useMediaQuery("(min-width: 1024px)")`; exported (with `LARGE_SCREEN_BREAKPOINT`) from `@/hooks`.
- **Shell height chain (enables full-height):** `src/App.tsx:176` renders the active module inside `<main className="flex-1 min-h-0 flex flex-col overflow-hidden">` via `ScreenTransition`, whose wrapper is `<div className="flex min-h-0 flex-1 flex-col">`. So a module root with `flex-1` fills the viewport height — this is exactly how Meals v2 (`meals-view.tsx:546`) fills height today.
- **Test infra:** `src/components/chores-view.test.tsx` mocks `@/hooks` via a hoisted `viewport` object (`useIsMobile: () => viewport.isMobile`), seeds with `seedMockChoresBoard(...)` + `seedFamilyStore(...)` over `setupMswServer()`, and defines `sampleChoresBoard()` / `emptyChoresBoard()` fixtures. `src/components/chores/chores-scope-column.test.tsx` and `src/components/chores/chore-row.test.tsx` are plain component tests (`render`/`screen` from `@/test/test-utils` and `@testing-library/react` respectively). `src/test/setup.ts` stubs `matchMedia` → `matches: false`, so `useIsLargeScreen()` is **false** in tests unless the hoisted mock overrides it.

## File Structure

**Create:**
- `src/components/chores/chores-board-large.tsx` (+ `.test.tsx`) — the `lg+` full-height, Today-weighted three-column board (presentational).

**Modify:**
- `src/components/chores/chore-row.tsx` (+ `.test.tsx`) — checkoff grows to ≥44px on `lg+`.
- `src/components/chores/chores-scope-column.tsx` (+ `.test.tsx`) — additive `fillHeight` / `emphasis` / `className` props; defaults unchanged.
- `src/components/chores-view.tsx` (+ `.test.tsx`) — add `useIsLargeScreen()`; branch container classes + populated board on it.

No barrel exports Chores components (imports are direct paths), so nothing to update there.

---

## Task 1: Grow the checkoff target to ≥44px on large screens

The spec AC requires ≥44px checkoff targets on large screens; the control is `h-9 w-9` (36px) today. A `lg:` bump satisfies it and only takes visual effect at ≥1024px, where the large board is the only Chores surface rendered (below `lg`, the class is inert, so mobile/tablet stay 36px).

**Files:**
- Modify: `src/components/chores/chore-row.tsx`
- Test: `src/components/chores/chore-row.test.tsx`

- [ ] **Step 1: Write the failing test**

Append to `src/components/chores/chore-row.test.tsx` (inside a new `describe` or after the haptics block — reuse the file's existing `baseChore` and imports):

```tsx
describe("ChoreRow large-screen touch target", () => {
  it("grows the checkoff control to at least 44px at lg+", () => {
    render(
      <ChoreRow chore={baseChore} onComplete={vi.fn()} onUncomplete={vi.fn()} />,
    );
    const toggle = screen.getByRole("button", {
      name: /mark dishes complete/i,
    });
    // Base stays 36px for mobile/tablet; lg bumps to 44px (h-11/w-11).
    expect(toggle.className).toContain("h-9");
    expect(toggle.className).toContain("w-9");
    expect(toggle.className).toContain("lg:h-11");
    expect(toggle.className).toContain("lg:w-11");
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npm test -- --run src/components/chores/chore-row.test.tsx`
Expected: FAIL — the toggle className has no `lg:h-11` / `lg:w-11`.

- [ ] **Step 3: Add the responsive size classes**

In `src/components/chores/chore-row.tsx`, in the checkoff `<button>`'s `className`, change the size utilities from:

```tsx
          "flex h-9 w-9 shrink-0 items-center justify-center rounded-full border-2 transition-colors",
```

to:

```tsx
          "flex h-9 w-9 shrink-0 items-center justify-center rounded-full border-2 transition-colors lg:h-11 lg:w-11",
```

Then bump the icons so they scale with the larger target. Change both the checked and unchecked glyphs from `className="h-4 w-4"` to `className="h-4 w-4 lg:h-5 lg:w-5"`:

```tsx
        {chore.completed ? (
          <Check className="h-4 w-4 lg:h-5 lg:w-5" />
        ) : (
          <Circle className="h-4 w-4 lg:h-5 lg:w-5" />
        )}
```

Leave the button's `onClick`, `aria-label`, and the raw-`<button>` structure exactly as they are (the haptics-throttle comment still applies).

- [ ] **Step 4: Run to verify it passes**

Run: `npm test -- --run src/components/chores/chore-row.test.tsx`
Expected: PASS (existing haptics tests + the new size test).

- [ ] **Step 5: Lint + commit**

```bash
npm run lint
git add src/components/chores/chore-row.tsx src/components/chores/chore-row.test.tsx
git commit -m "feat(chores): grow checkoff target to 44px on large screens"
```

---

## Task 2: `ChoreScopeColumn` gains `fillHeight`, `emphasis`, and `className`

Additive props so the large board can (a) make each column fill height and scroll its routines internally, and (b) render Today with primary emphasis. All three props default to the current behavior, so the mobile/tablet path and every existing test are unchanged.

**Files:**
- Modify: `src/components/chores/chores-scope-column.tsx`
- Test: `src/components/chores/chores-scope-column.test.tsx`

- [ ] **Step 1: Write the failing tests**

Append to `src/components/chores/chores-scope-column.test.tsx`:

```tsx
describe("ChoreScopeColumn large-screen props", () => {
  it("keeps the padded, non-fill layout by default", () => {
    render(<ChoreScopeColumn scope={board} />);
    const region = screen.getByRole("region", { name: "Today" });
    expect(region.className).toContain("p-4");
    expect(region.className).not.toContain("min-h-0");
    expect(region).not.toHaveAttribute("data-emphasis");
  });

  it("fills height and scrolls its body when fillHeight is set", () => {
    render(<ChoreScopeColumn scope={board} fillHeight />);
    const region = screen.getByRole("region", { name: "Today" });
    expect(region.className).toContain("flex");
    expect(region.className).toContain("min-h-0");
    expect(region.className).not.toContain("p-4"); // padding moves inside
    const body = region.querySelector('[data-slot="scope-body"]');
    expect(body).not.toBeNull();
    expect(body?.className).toContain("overflow-y-auto");
  });

  it("applies the primary emphasis treatment when emphasis is set", () => {
    render(<ChoreScopeColumn scope={board} emphasis />);
    const region = screen.getByRole("region", { name: "Today" });
    expect(region).toHaveAttribute("data-emphasis", "true");
    expect(region.className).toContain("bg-primary/5");
    expect(
      screen.getByRole("heading", { name: "Today" }).className,
    ).toContain("text-xl");
  });

  it("merges a passed className onto the section", () => {
    render(<ChoreScopeColumn scope={board} className="flex-[1.4]" />);
    expect(
      screen.getByRole("region", { name: "Today" }).className,
    ).toContain("flex-[1.4]");
  });
});
```

- [ ] **Step 2: Run to verify they fail**

Run: `npm test -- --run src/components/chores/chores-scope-column.test.tsx`
Expected: FAIL — unknown props; no `data-emphasis`, no `scope-body`, heading is `text-lg`.

- [ ] **Step 3: Rewrite `ChoreScopeColumn` with the additive props**

Replace the contents of `src/components/chores/chores-scope-column.tsx` with:

```tsx
import type { ChoreBoardItem, ChoreScopeBoard } from "@/lib/types";
import { cn } from "@/lib/utils";
import { ChoreAssigneeGroup } from "./chore-assignee-group";

interface ChoreScopeColumnProps {
  scope: ChoreScopeBoard;
  showHeading?: boolean;
  /** Fill the parent's height and scroll the routines list internally (lg board). */
  fillHeight?: boolean;
  /** Primary-column treatment for Today (stronger heading + subtle tint). */
  emphasis?: boolean;
  /** Extra classes merged onto the section (e.g. flex weighting from the board). */
  className?: string;
  onArchive?: (scope: ChoreScopeBoard, chore: ChoreBoardItem) => void;
  onComplete?: (scope: ChoreScopeBoard, chore: ChoreBoardItem) => void;
  onUncomplete?: (scope: ChoreScopeBoard, chore: ChoreBoardItem) => void;
}

function scopeHeading(scope: ChoreScopeBoard["scope"]): string {
  if (scope === "TODAY") return "Today";
  if (scope === "THIS_WEEK") return "This Week";
  return "This Month";
}

export function ChoreScopeColumn({
  scope,
  showHeading = true,
  fillHeight = false,
  emphasis = false,
  className,
  onArchive,
  onComplete,
  onUncomplete,
}: ChoreScopeColumnProps) {
  const heading = scopeHeading(scope.scope);
  const summary = `${scope.summary.remaining} left of ${scope.summary.total}`;
  const isFullyComplete =
    scope.summary.total > 0 && scope.summary.remaining === 0;

  const headerContent = showHeading ? (
    <header className={cn(!fillHeight && "mb-4")}>
      <h2
        id={`${scope.scope}-heading`}
        className={cn(
          "font-semibold text-foreground",
          emphasis ? "text-xl" : "text-lg",
        )}
      >
        {heading}
      </h2>
      <p className="mt-1 text-sm text-muted-foreground">{summary}</p>
    </header>
  ) : (
    <p className={cn("text-sm text-muted-foreground", !fillHeight && "mb-4")}>
      {summary}
    </p>
  );

  const bodyContent =
    scope.summary.total === 0 ? (
      <p className="rounded-lg border border-dashed border-border p-4 text-sm text-muted-foreground">
        No routines in this timeframe
      </p>
    ) : (
      <div className="space-y-5">
        {isFullyComplete && (
          <p className="rounded-lg bg-muted px-3 py-2 text-sm font-medium text-muted-foreground">
            All caught up
          </p>
        )}
        {scope.assignees.map((group) => (
          <ChoreAssigneeGroup
            key={group.member.id}
            group={group}
            onArchive={(chore) => onArchive?.(scope, chore)}
            onComplete={(chore) => onComplete?.(scope, chore)}
            onUncomplete={(chore) => onUncomplete?.(scope, chore)}
          />
        ))}
      </div>
    );

  return (
    <section
      aria-label={showHeading ? undefined : heading}
      aria-labelledby={showHeading ? `${scope.scope}-heading` : undefined}
      data-emphasis={emphasis ? "true" : undefined}
      className={cn(
        "rounded-lg border bg-card shadow-sm",
        emphasis && "border-primary/30 bg-primary/5",
        fillHeight ? "flex min-h-0 flex-col" : "p-4",
        className,
      )}
    >
      {fillHeight ? (
        <>
          <div className="shrink-0 p-4 pb-3">{headerContent}</div>
          <div
            data-slot="scope-body"
            className="min-h-0 flex-1 overflow-y-auto px-4 pb-4"
          >
            {bodyContent}
          </div>
        </>
      ) : (
        <>
          {headerContent}
          {bodyContent}
        </>
      )}
    </section>
  );
}
```

Notes for the implementer:
- The non-`fillHeight` branch renders `headerContent` then `bodyContent` directly inside the `p-4` section — byte-equivalent to the current markup (same `mb-4` on the header, same body), so the mobile/tablet path and existing tests are unchanged.
- In `fillHeight` mode the section drops `p-4` and becomes a flex column: a `shrink-0` header block and a `flex-1 overflow-y-auto` body (`data-slot="scope-body"`) that scrolls the routines. Padding moves inside so the border wraps the whole column and the scrollbar sits at the padded edge.
- `emphasis` adds `border-primary/30 bg-primary/5`, a `text-xl` heading, and `data-emphasis="true"` (a stable test/QA hook). It never fires on the mobile/tablet path (that path never passes `emphasis`).

- [ ] **Step 4: Run to verify the new + existing tests pass**

Run: `npm test -- --run src/components/chores/chores-scope-column.test.tsx`
Expected: PASS — the two original cases (`shows the period heading by default`, `hides the heading but keeps the accessible name`) plus the four new ones.

- [ ] **Step 5: Prove the ChoresView suite is still green (mobile/tablet path unchanged)**

Run: `npm test -- --run src/components/chores-view.test.tsx`
Expected: PASS — `ChoreScopeColumn`'s default output is unchanged, so every existing case (all-caught-up, scope empty state, complete/uncomplete, archive, mobile scope switch) is green.

- [ ] **Step 6: Lint + commit**

```bash
npm run lint
git add src/components/chores/chores-scope-column.tsx src/components/chores/chores-scope-column.test.tsx
git commit -m "feat(chores): add fillHeight/emphasis/className to ChoreScopeColumn"
```

---

## Task 3: `ChoresBoardLarge` — full-height, Today-weighted three-column board

The `lg+` board: three fill-height scope columns in a flex row, Today weighted `flex-[1.4]` and emphasized, This Week / This Month `flex-1`. Presentational — it forwards the same `(scope, chore) => void` handlers `ChoresView` already owns.

**Files:**
- Create: `src/components/chores/chores-board-large.tsx`
- Test: `src/components/chores/chores-board-large.test.tsx`

- [ ] **Step 1: Write the failing tests**

Create `src/components/chores/chores-board-large.test.tsx`:

```tsx
import { describe, expect, it, vi } from "vitest";
import type { ChoreScopeBoard } from "@/lib/types";
import { renderWithUser, screen } from "@/test/test-utils";
import { ChoresBoardLarge } from "./chores-board-large";

function scopeBoard(
  scope: ChoreScopeBoard["scope"],
  total: number,
): ChoreScopeBoard {
  return {
    scope,
    periodStartDate: "2026-06-11",
    periodEndDate: "2026-06-11",
    summary: { total, completed: 0, remaining: total },
    assignees:
      total === 0
        ? []
        : [
            {
              member: { id: "leo", name: "Leo", color: "coral" },
              summary: { total, completed: 0, remaining: total },
              chores: [
                {
                  templateId: `${scope}-c1`,
                  title: `${scope} chore`,
                  cadence: "DAILY",
                  assignedToMemberId: "leo",
                  completed: false,
                  completedAt: null,
                },
              ],
            },
          ],
  };
}

describe("ChoresBoardLarge", () => {
  it("renders all three scope columns as regions", () => {
    renderWithUser(
      <ChoresBoardLarge
        today={scopeBoard("TODAY", 1)}
        thisWeek={scopeBoard("THIS_WEEK", 1)}
        thisMonth={scopeBoard("THIS_MONTH", 1)}
      />,
    );
    expect(screen.getByRole("region", { name: "Today" })).toBeInTheDocument();
    expect(
      screen.getByRole("region", { name: "This Week" }),
    ).toBeInTheDocument();
    expect(
      screen.getByRole("region", { name: "This Month" }),
    ).toBeInTheDocument();
  });

  it("weights and emphasizes Today as the primary column", () => {
    renderWithUser(
      <ChoresBoardLarge
        today={scopeBoard("TODAY", 1)}
        thisWeek={scopeBoard("THIS_WEEK", 1)}
        thisMonth={scopeBoard("THIS_MONTH", 1)}
      />,
    );
    const today = screen.getByRole("region", { name: "Today" });
    expect(today).toHaveAttribute("data-emphasis", "true");
    expect(today.className).toContain("flex-[1.4]");
    const week = screen.getByRole("region", { name: "This Week" });
    expect(week).not.toHaveAttribute("data-emphasis");
    expect(week.className).toContain("flex-1");
  });

  it("forwards completion to the handler with the owning scope and chore", async () => {
    const onComplete = vi.fn();
    const { user } = renderWithUser(
      <ChoresBoardLarge
        today={scopeBoard("TODAY", 1)}
        thisWeek={scopeBoard("THIS_WEEK", 0)}
        thisMonth={scopeBoard("THIS_MONTH", 0)}
        onComplete={onComplete}
      />,
    );
    await user.click(
      screen.getByRole("button", { name: /mark TODAY chore complete/i }),
    );
    expect(onComplete).toHaveBeenCalledTimes(1);
    expect(onComplete.mock.calls[0][0].scope).toBe("TODAY");
    expect(onComplete.mock.calls[0][1].templateId).toBe("TODAY-c1");
  });
});
```

- [ ] **Step 2: Run to verify they fail**

Run: `npm test -- --run src/components/chores/chores-board-large.test.tsx`
Expected: FAIL with "Cannot find module './chores-board-large'".

- [ ] **Step 3: Implement `ChoresBoardLarge`**

Create `src/components/chores/chores-board-large.tsx`:

```tsx
import type { ChoreBoardItem, ChoreScopeBoard } from "@/lib/types";
import { ChoreScopeColumn } from "./chores-scope-column";

interface ChoresBoardLargeProps {
  today: ChoreScopeBoard;
  thisWeek: ChoreScopeBoard;
  thisMonth: ChoreScopeBoard;
  onArchive?: (scope: ChoreScopeBoard, chore: ChoreBoardItem) => void;
  onComplete?: (scope: ChoreScopeBoard, chore: ChoreBoardItem) => void;
  onUncomplete?: (scope: ChoreScopeBoard, chore: ChoreBoardItem) => void;
}

// Large-screen (lg+) Chores board: three scope columns filling the viewport
// height, Today weighted and emphasized as the primary column, each column
// scrolling its own routines. Mobile/tablet keep the stacked ChoreScopeColumn
// grid in ChoresView. `min-w-0` lets columns shrink so long chore titles never
// force horizontal overflow.
export function ChoresBoardLarge({
  today,
  thisWeek,
  thisMonth,
  onArchive,
  onComplete,
  onUncomplete,
}: ChoresBoardLargeProps) {
  const shared = {
    fillHeight: true as const,
    showHeading: true as const,
    onArchive,
    onComplete,
    onUncomplete,
  };

  return (
    <div className="flex min-h-0 flex-1 gap-4">
      <ChoreScopeColumn
        {...shared}
        scope={today}
        emphasis
        className="min-w-0 flex-[1.4]"
      />
      <ChoreScopeColumn {...shared} scope={thisWeek} className="min-w-0 flex-1" />
      <ChoreScopeColumn
        {...shared}
        scope={thisMonth}
        className="min-w-0 flex-1"
      />
    </div>
  );
}
```

Notes for the implementer:
- `flex min-h-0 flex-1 gap-4` makes the board a flex row that (a) takes the leftover height from its parent column via `flex-1`, and (b) lets its own children stretch to full height (default `align-items: stretch`). `min-h-0` allows it to shrink under a short viewport so each column's internal `overflow-y-auto` engages instead of the page growing.
- `flex-[1.4]` vs `flex-1`: both use `flex-basis: 0%`, so track width is pure grow ratio — Today ≈ 1.4/3.4 ≈ 41%, the other two ≈ 29.5% each. That reads as "Today primary" without a hard pixel width.

- [ ] **Step 4: Run to verify they pass**

Run: `npm test -- --run src/components/chores/chores-board-large.test.tsx`
Expected: PASS (3 tests).

- [ ] **Step 5: Lint + commit**

```bash
npm run lint
git add src/components/chores/chores-board-large.tsx src/components/chores/chores-board-large.test.tsx
git commit -m "feat(chores): add full-height Today-weighted large-screen board"
```

---

## Task 4: Route `ChoresView` to the large board at `lg+` + integration tests

Wire `useIsLargeScreen()` into the single `ChoresView`: branch the container classes and the populated board, keep everything else (data, handlers, states, create sheet, header, mobile FAB) exactly as-is.

**Files:**
- Modify: `src/components/chores-view.tsx`
- Test: `src/components/chores-view.test.tsx`

- [ ] **Step 1: Extend the test's hoisted viewport mock**

At the top of `src/components/chores-view.test.tsx`, extend the hoisted object and the `@/hooks` mock so the large-screen branch is controllable and defaults off:

```tsx
const viewport = vi.hoisted(() => ({ isMobile: false, isLargeScreen: false }));
```

```tsx
vi.mock("@/hooks", async (importOriginal) => {
  const actual = await importOriginal<typeof import("@/hooks")>();
  return {
    ...actual,
    useIsMobile: () => viewport.isMobile,
    useIsLargeScreen: () => viewport.isLargeScreen,
  };
});
```

In the top-level `beforeEach`, add `viewport.isLargeScreen = false;` next to `viewport.isMobile = false;` so every existing test keeps hitting the current (mobile/tablet) path.

- [ ] **Step 2: Run the existing suite — still green through the added mock**

Run: `npm test -- --run src/components/chores-view.test.tsx`
Expected: PASS (unchanged — `isLargeScreen` defaults false, so the current grid renders).

- [ ] **Step 3: Write the failing large-screen integration tests**

Append to `src/components/chores-view.test.tsx` (the file already imports `waitFor`, `renderWithUser`, `seedMockChoresBoard`, `sampleChoresBoard`, `server`, `http`, `HttpResponse`, `API_BASE`):

```tsx
describe("ChoresView large screen (full-height board)", () => {
  beforeEach(() => {
    viewport.isLargeScreen = true;
  });

  it("renders the three scope columns with Today as the primary column", async () => {
    seedMockChoresBoard(sampleChoresBoard());
    render(<ChoresView />);

    expect(await screen.findByRole("region", { name: "Today" })).toBeVisible();
    expect(screen.getByRole("region", { name: "This Week" })).toBeVisible();
    expect(screen.getByRole("region", { name: "This Month" })).toBeVisible();
    // Today carries the primary-column emphasis; the others do not.
    expect(screen.getByRole("region", { name: "Today" })).toHaveAttribute(
      "data-emphasis",
      "true",
    );
    expect(
      screen.getByRole("region", { name: "This Week" }),
    ).not.toHaveAttribute("data-emphasis");
  });

  it("completes a routine from the large-screen board", async () => {
    let capturedCompletionBody: UpdateCurrentPeriodCompletionRequest | null =
      null;
    server.use(
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
                title: "Brush teeth",
                cadence: "DAILY",
                assignedToMemberId: "leo",
                completed: true,
                completedAt: "2026-05-17T09:30:00Z",
              },
            },
          });
        },
      ),
    );
    seedMockChoresBoard(sampleChoresBoard());
    const { user } = renderWithUser(<ChoresView />);

    await user.click(
      await screen.findByRole("button", {
        name: /mark brush teeth complete/i,
      }),
    );

    await waitFor(() => {
      expect(capturedCompletionBody).toEqual({
        scope: "TODAY",
        periodStartDate: "2026-05-17",
      });
    });
  });
});
```

> `UpdateCurrentPeriodCompletionRequest` is already imported at the top of the file. If Biome flags a duplicate-import or ordering issue, keep the single existing import.

- [ ] **Step 4: Run to verify the new tests fail**

Run: `npm test -- --run src/components/chores-view.test.tsx`
Expected: FAIL — `ChoresView` still renders the `max-w-6xl` grid, which has no scope `region` emphasis wiring at `lg` and no `ChoresBoardLarge` (the "Today primary" assertion fails on `data-emphasis`).

- [ ] **Step 5: Implement the branch in `ChoresView`**

In `src/components/chores-view.tsx`:

Add to the imports:

```tsx
import { ChoresBoardLarge } from "@/components/chores/chores-board-large";
```

and extend the `@/hooks` import to include `useIsLargeScreen`:

```tsx
import { useIsLargeScreen, useIsMobile } from "@/hooks";
```

Inside `ChoresView`, add the hook next to `useIsMobile`:

```tsx
  const isMobile = useIsMobile();
  const isLargeScreen = useIsLargeScreen();
```

Replace the outer container `<div>` opening tag:

```tsx
      <div
        className="flex-1 overflow-y-auto p-4 sm:p-6"
        style={{
          paddingBottom: isMobile ? MOBILE_FAB_SCROLL_PADDING : undefined,
        }}
      >
        <div className="mx-auto max-w-6xl">
```

with the breakpoint-gated version:

```tsx
      <div
        className={cn(
          "flex-1 p-4 sm:p-6",
          isLargeScreen ? "flex min-h-0 flex-col" : "overflow-y-auto",
        )}
        style={{
          paddingBottom: isMobile ? MOBILE_FAB_SCROLL_PADDING : undefined,
        }}
      >
        <div
          className={cn(
            "mx-auto",
            isLargeScreen
              ? "flex min-h-0 w-full max-w-[1600px] flex-1 flex-col"
              : "max-w-6xl",
          )}
        >
```

Add `shrink-0` to the header row so it never compresses inside the `lg` flex column (inert below `lg`, where the inner div is not a flex container):

```tsx
          <div className="mb-6 flex shrink-0 items-start justify-between gap-3">
```

Replace the populated-board block:

```tsx
          {!isLoading && !isError && board && hasRoutines && (
            <div className={cn("grid gap-4", !isMobile && "lg:grid-cols-3")}>
              {visibleScopes.map((scope) => (
                <ChoreScopeColumn
                  key={scope.scope}
                  scope={scope}
                  showHeading={!isMobile}
                  onArchive={handleArchive}
                  onComplete={handleComplete}
                  onUncomplete={handleUncomplete}
                />
              ))}
            </div>
          )}
```

with the branched version:

```tsx
          {!isLoading && !isError && board && hasRoutines && (
            isLargeScreen ? (
              <ChoresBoardLarge
                today={board.today}
                thisWeek={board.thisWeek}
                thisMonth={board.thisMonth}
                onArchive={handleArchive}
                onComplete={handleComplete}
                onUncomplete={handleUncomplete}
              />
            ) : (
              <div className={cn("grid gap-4", !isMobile && "lg:grid-cols-3")}>
                {visibleScopes.map((scope) => (
                  <ChoreScopeColumn
                    key={scope.scope}
                    scope={scope}
                    showHeading={!isMobile}
                    onArchive={handleArchive}
                    onComplete={handleComplete}
                    onUncomplete={handleUncomplete}
                  />
                ))}
              </div>
            )
          )}
```

Leave everything else — the four handlers, `canCreate`, the states, `ChoreFormSheet`, and the mobile `FloatingActionButton` — unchanged. (`cn` is already imported.)

- [ ] **Step 6: Run to verify all pass**

Run: `npm test -- --run src/components/chores-view.test.tsx`
Expected: PASS — existing below-`lg` cases (default `isLargeScreen = false`) plus the two new large-screen cases.

- [ ] **Step 7: Typecheck, lint, commit**

```bash
npm run build   # tsc -b: the only type-check gate — do not skip it
npm run lint
git add src/components/chores-view.tsx src/components/chores-view.test.tsx
git commit -m "feat(chores): render full-height board on large screens"
```

---

## Task 5: Full verification + screenshot matrix

Mirror the Lists/Meals evidence discipline (spec §6 review cases). Screenshots require a real backend (root `MEMORY.md` → E2E against real BE; use the latest released `BE_IMAGE_TAG`).

- [ ] **Step 1: Whole-suite gate**

```bash
npm run build     # tsc -b + vite, exit 0
npm run lint      # Biome, exit 0
npm test -- --run # full unit/integration suite green
```

- [ ] **Step 2: Capture the spec's screenshot matrix**

Start the real backend, then capture Chores at **1024px and 1440px**, each across the spec's four cases:
- **Busy board** — several routines across all three scopes and multiple members.
- **Chores all done** — every routine complete ("All caught up" in each column).
- **Empty board** — no recurring routines (family-level empty state).
- **5+ members** — a family with five or more members with chores, to check assignee-group density and the "find your name" scan.

```bash
# from frontend/
BE_VERSION=$(curl -sf https://api.github.com/repos/joe-bor/family-hub-api/releases/latest | jq -r '.tag_name // empty' | sed 's/^v//')
BE_IMAGE_TAG=${BE_VERSION:-latest} docker compose -f docker-compose.e2e.yml up -d --wait   # real BE on :8080
npm run dev                                                                                # or the project's screenshot harness
# ...capture matrix...
docker compose -f docker-compose.e2e.yml down
```

Confirm at each size: the board fills the width up to the 1600px cap and centers beyond it (no dead side margins); the three columns fill the viewport height with the board's bottom edge at the section padding (no whitespace pool below); Today is visibly the primary column (wider + tint + stronger heading); each column scrolls its own routines independently; checkoff targets read as ≥44px; assignee headers show avatar + name + color.

- [ ] **Step 3: Prove below-1024 is unchanged**

Capture Chores at **900px** (portrait tablet) and **375px** (mobile) and diff against `origin/main`: the tablet three-stacked-cards layout, the mobile scope switcher + single column, the FAB, and the 36px checkoff must be visually identical.

- [ ] **Step 4: Iterate**

If any screenshot violates the spec (dead margins, board not filling height, Today not clearly primary, a column not scrolling, checkoff under 44px, or any below-1024 regression), fix and re-capture before proceeding. Commit fixes as `fix(chores): ...`.

> Confirm one behavior as **acceptable**, not a regression: crossing the 1024px boundary mid-session swaps between the stacked grid and the full-height board (they are separate render branches of one `ChoresView`); state (selected scope on the mobile switcher, create sheet) is not carried across the swap. Expected on the target devices and not a spec requirement.

---

## Task 6: PR + post-merge root docs

- [ ] **Step 1: Open the PR from `feat/large-screen-chores`**

Body must include, per the root execution-issue contract:
- **Story / Spec / Plan** links (this file).
- **Verification:** build/lint/test results + the screenshot matrix summary + the below-1024 unchanged evidence.
- **Execution contract checklist** mapping each spec AC to code + tests:
  - Widened container, Today primary at 1024/1440 → `ChoresView` `max-w-[1600px]` flex-fill branch + `ChoresBoardLarge` (`flex-[1.4]` + `emphasis`); tests "renders the three scope columns with Today as the primary column", "weights and emphasizes Today".
  - Checkoff ≥44px on large screens → `ChoreRow` `lg:h-11 lg:w-11`; test "grows the checkoff control to at least 44px at lg+".
  - Assignee headers avatar + name + member color consistent with Calendar Day lanes → existing `ChoreAssigneeGroup` (unchanged) rendered in the board; screenshot review (Step 2).
  - Single toolbar row, no stacked chrome → existing `ChoresView` header row preserved (foundations, already shipped).
  - All chore flows unchanged (create/edit/complete/uncomplete/archive/stale recovery) → handlers reused unchanged; `ChoresBoardLarge` forwards the same callbacks; test "completes a routine from the large-screen board" + full existing suite.
  - Mobile + tablet unchanged → below-`lg` branch is class-gated on `isLargeScreen`; full existing `chores-view.test.tsx` suite + 900/375px screenshots.
  - Screenshot review across busy / all-done / empty / 5+ members at 1024/1440 → Task 5 matrix.

- [ ] **Step 2: Merge, then update root docs (orchestrator, post-merge)**

After merge, in the **root** repo set `docs/product/backlog/large-screen-ux/large-screen-chores.md` `status: done`, bump `updated:`, record the PR under `prs:`, check off its acceptance criteria, and move its `roadmap.md` line from "Next up (spec written, plan not yet written)" into the large-screen **Shipped** list — exactly as the Meals story was closed out. If Recipes is still the only remaining large-screen story afterward, note that the epic is one story from complete.

---

## Self-Review (completed during planning)

**1. Spec coverage:**
- §3 "Let the board breathe" (widen beyond `max-w-6xl`) + §6 AC "widened container" → Task 4 (`max-w-[1600px]` full-width flex-fill container, replacing `max-w-6xl`).
- §3 "Weight Today as the primary column" + §6 AC "Today visually primary at 1024/1440" → Task 3 (`flex-[1.4]` + `emphasis`) and Task 2 (`emphasis` heading/tint), verified in Task 4 tests and Task 5 screenshots.
- §3 "Bigger checkoff targets" + §6 AC "≥44px on large screens" → Task 1 (`lg:h-11 lg:w-11`).
- §3 "Member identity forward" + §6 AC "avatar + name + member color consistent with Calendar Day lanes" → **already satisfied** by the current `ChoreAssigneeGroup` header (colored `colorMap` avatar + name + progress); it uses the same colored-initial-circle visual language as the Calendar day-lane `MemberAvatar`. Left unchanged to avoid touching the mobile path (spec §7 out of scope); confirmed in the Task 5 screenshot review. If the reviewer wants literal component reuse, promoting `MemberAvatar` to `@/components/shared` and swapping it in is a clean, separable follow-up.
- §3 "Foundations chrome (single toolbar row)" + §6 AC "single toolbar row, no stacked chrome" → **already shipped** in `ChoresView` (one header row: title + Add); preserved unchanged (the header only gains an inert `shrink-0`).
- §6 AC "all existing chore flows unchanged" → handlers and `ChoreFormSheet` reused verbatim; `ChoresBoardLarge` forwards the same `(scope, chore) => void` callbacks; Task 4 completion test + full existing suite.
- §6 AC "mobile Chores unchanged" → the entire below-`lg` render path is class-gated on `isLargeScreen` (container classes) and branch-gated (board); `ChoreScopeColumn` / `ChoreRow` defaults are byte-equivalent; existing suite + 900/375 screenshots (Tasks 2/4/5).
- §6 AC "screenshot review: busy / all-done / empty / 5+ members" → Task 5.
- §7 Out of scope (member-lane mode, new chore features, mobile behavior, backend) → respected; no board-axis change, no new features, no BE, mobile path untouched.

**2. Placeholder scan:** No TBD / "handle edge cases" / "similar to Task N" — every code step carries full code and exact commands with expected output.

**3. Type consistency:** Handler signatures are `(scope: ChoreScopeBoard, chore: ChoreBoardItem) => void` across `ChoresView`, `ChoresBoardLarge`, and `ChoreScopeColumn`. `ChoreScopeColumn`'s new props (`fillHeight`, `emphasis`, `className`) are used identically in `ChoresBoardLarge` and its tests. `ChoresBoardLarge` prop names (`today`, `thisWeek`, `thisMonth`, `onArchive`, `onComplete`, `onUncomplete`) match the `board.today/thisWeek/thisMonth` fields and the handler names in `ChoresView`. `useIsLargeScreen` is imported from `@/hooks`. The `data-emphasis` / `data-slot="scope-body"` hooks are asserted consistently in Tasks 2, 3, and 4.

**Key risk flagged:** full-height depends on the shell chain (`<main class="flex-1 min-h-0 flex flex-col">` → `ScreenTransition` `flex min-h-0 flex-1 flex-col` → module root `flex-1`). This is verified present in `App.tsx:176` and is the same chain Meals v2 fills today, so `ChoresView`'s `lg` flex-fill container reaches the bottom without extra shell changes. jsdom cannot measure layout, so height-fill / independent-scroll / no-horizontal-overflow are proven by the Task 5 real-browser screenshots, not unit tests.
