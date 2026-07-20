# Large-Screen Calendar Month and Schedule Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** At 1024px and above, make Calendar Month fill its viewport with density derived from measured row height and a working overflow path, and give Schedule a date-gutter composition that uses the width it has — without changing mobile, the 769-1023px range, Week, or Day.

**Architecture:** All layout arithmetic lives in pure, separately-tested helpers (`month-capacity.ts`, `month-slots.ts`, `schedule-rows.ts`) because the repo's Vitest environment cannot exercise measured geometry. Components consume those helpers and are responsible only for rendering and interaction. Every behavioural change is gated on `useIsLargeScreen()`; below 1024px both views take their existing code paths untouched.

**Tech Stack:** React 19, TypeScript, Vite, Tailwind CSS v4, Radix UI (`@radix-ui/react-popover`), TanStack Query, Zustand, Vitest + Testing Library, Playwright.

**Spec:** `docs/superpowers/specs/2026-07-18-large-screen-calendar-month-schedule-design.md`
**Story:** `docs/product/backlog/large-screen-ux/large-screen-calendar-month-schedule.md`
**Review baseline:** frontend `origin/main`
`f9dc7e8070457f965b823baee4e3746486afa438` and released backend `v1.9.0`,
verified 2026-07-20. The only frontend movement after the initial source audit
was a `package-lock.json` dependency update; Calendar and test sources did not
change.

---

## Critical environment facts

Read these before Task 1. They change how tests must be written.

1. **`matchMedia` is mocked to `matches: false` for every query** in `src/test/setup.ts`. `useIsLargeScreen()` and `useIsMobile()` are therefore both `false` in every unit test unless the test overrides it. Task 1 promotes an existing working override into a shared helper.
2. **`ResizeObserver` is a no-op mock** and jsdom has no layout engine. Any assertion that depends on a measured height is unreachable in Vitest. Capacity and slot planning are proven on pure helpers; observer-to-render wiring is proven in Playwright only.
3. **Type errors are caught only by `npm run build`.** `npm run lint` and `npx tsc --noEmit` both miss them (the root tsconfig has `files: []`, and `tsconfig.app` excludes tests). Run `npm run build` before every commit.
4. **`resetAllStores()` in `src/test/setup.ts` resets stores by name.** No new store is added by this plan, so nothing needs adding there.
5. **Run every command from a dedicated frontend worktree**, never by switching
   the user's active frontend checkout. Work stays on one branch in **two
   phases** — Month, then Schedule — each ending at its own screenshot gate.
   Commit frequently within a phase; Schedule does not begin until the Month
   gate (Task 12) passes.
6. **The plan is an ordered implementation contract, not permission to paste
   blindly.** It was reconciled with frontend `origin/main` at review time. If
   a named symbol or snippet conflicts with the current fetched baseline, stop
   that task, record the drift, and preserve the spec/Issue requirement rather
   than weakening it.

---

## File Structure

**Create:**

| Path | Responsibility |
| --- | --- |
| `src/components/calendar/utils/month-matrix.ts` | `buildMonthMatrix`, `selectMonthDayDots` (moved from `day-rail.ts`) plus `selectMonthDayMembers` |
| `src/components/calendar/utils/month-matrix.test.ts` | Tests moved from `day-rail.test.ts` |
| `src/components/calendar/utils/month-capacity.ts` | Layout constants, `monthRowHeight`, `monthSlotCapacity` |
| `src/components/calendar/utils/month-capacity.test.ts` | Capacity formula + invariants |
| `src/components/calendar/utils/month-slots.ts` | `isMultiDay`, `multiDayEdge`, `orderRowMultiDay`, `planCellSlots` |
| `src/components/calendar/utils/month-slots.test.ts` | Ordering, reservation, overflow counting |
| `src/components/calendar/utils/schedule-rows.ts` | `buildScheduleRows` — day rows and gap runs |
| `src/components/calendar/utils/schedule-rows.test.ts` | Gap detection, boundaries, filtering |
| `src/components/calendar/components/month-event-chip.tsx` | Presentational chip: exact weld geometry and visual markers |
| `src/components/calendar/components/month-event-chip.test.tsx` | Presentational semantics, exact weld and fallback contrast |
| `src/components/calendar/components/month-day-cell.tsx` | One cell: numeral, dot summary, slots, `+N more` |
| `src/components/calendar/components/month-overflow-popover.tsx` | Per-day full event list |
| `src/components/calendar/components/calendar-view-states.tsx` | Shared lg+ skeleton/error/empty state primitives |
| `src/components/calendar/components/month-day-cell.test.tsx` | Cell rendering + a11y names |
| `src/components/calendar/components/month-overflow-popover.test.tsx` | Popover contents + focus return |
| `src/components/calendar/views/monthly-calendar.test.tsx` | lg+ grid structure, keyboard and breakpoint integration |
| `e2e/large-screen-calendar-month.spec.ts` | Month E2E against the real backend |
| `e2e/large-screen-calendar-schedule.spec.ts` | Schedule E2E against the real backend |
| `e2e/large-screen-calendar-visual.spec.ts` | Opt-in deterministic Month/Schedule visual evidence harness |
| `e2e/calendar-preservation-visual.spec.ts` | Opt-in origin/current preservation captures for all Calendar views |
| `scripts/run-calendar-e2e.sh` | Fail-safe E2E runner pinned to the released backend with guaranteed teardown |

**Modify:**

| Path | Change |
| --- | --- |
| `src/test/test-utils.tsx` | Add `setViewportWidth` helper |
| `src/components/calendar/utils/day-rail.ts` | Remove the two moved functions; keep rail constants |
| `src/components/calendar/utils/day-rail.test.ts` | Remove the two moved test cases |
| `src/components/calendar/components/day-mini-month-rail.tsx` | Update import path |
| `src/components/calendar/components/index.ts` | Export new components |
| `src/components/calendar/views/monthly-calendar.tsx` | Full lg+ rewrite; tablet path preserved |
| `src/components/calendar/views/schedule-calendar.tsx` | lg+ gutter composition; border fix; states |
| `src/components/calendar/views/schedule-calendar.test.tsx` | Migrate banner assertions to lg+/mobile blocks |
| `src/components/calendar/calendar-module.tsx` | lg+-gated Month query range |
| `.gitignore` | Ignore local `.calendar-evidence/` screenshot output |
| `e2e/offline-persistence.spec.ts` | Real persisted-query restoration and visual proof for Month |

---

## Task 0: Isolated worktree setup

- [ ] **Step 1: Create the branch in an isolated worktree**

```bash
# The linked plan lives in the architect repo, but every command below owns the
# FE delivery repo. This conditional handles either starting location.
if [ -f frontend/package.json ]; then cd frontend; fi
test -f package.json
test -f AGENTS.md
git fetch origin
git check-ignore -q .worktrees
git worktree add .worktrees/large-screen-month-schedule \
  -b feat/large-screen-month-schedule origin/main
cd .worktrees/large-screen-month-schedule
git log --oneline -1
```

Expected: `.worktrees` is ignored, the new worktree is clean, and HEAD matches
fresh `origin/main`. If the branch or worktree already exists, inspect it and
reuse it only when it is clean and points at the intended work; do not delete or
overwrite it. `git remote get-url origin` must identify the FE `FamilyHub`
repository, not the lowercase architect/docs repository.

- [ ] **Step 2: Confirm a clean baseline**

```bash
npm ci
npm run lint && npm test -- --run && npm run build
```

Expected: all three exit 0. If anything fails here it is pre-existing; stop and report rather than absorbing it into this work.

---

## Task 1: Shared viewport test helper

`schedule-calendar.test.tsx` already contains a working `matchMedia` override that handles both `min-width` and `max-width` queries. Promote it so every later test can select a breakpoint.

**Files:**
- Modify: `src/test/test-utils.tsx`
- Modify: `src/components/calendar/views/schedule-calendar.test.tsx:19-48`

- [ ] **Step 1: Add the helper to test-utils**

Append to `src/test/test-utils.tsx`:

```tsx
/**
 * Drive the mocked `matchMedia` from a single viewport width so breakpoint
 * hooks resolve correctly. `src/test/setup.ts` mocks `matchMedia` to return
 * `matches: false` for every query, which would otherwise make `useIsMobile()`
 * and `useIsLargeScreen()` both false in every test.
 *
 * 390 -> mobile. 800 -> tablet (769-1023). 1024+ -> large screen.
 */
export function setViewportWidth(width: number): void {
  Object.defineProperty(window, "innerWidth", {
    configurable: true,
    value: width,
  });
  vi.mocked(window.matchMedia).mockImplementation((query: string) => ({
    matches: (() => {
      const maxWidth = Number.parseInt(
        query.match(/max-width:\s*(\d+)px/)?.[1] ?? "",
        10,
      );
      const minWidth = Number.parseInt(
        query.match(/min-width:\s*(\d+)px/)?.[1] ?? "",
        10,
      );
      const matchesMax = Number.isNaN(maxWidth) || width <= maxWidth;
      const matchesMin = Number.isNaN(minWidth) || width >= minWidth;
      return matchesMax && matchesMin;
    })(),
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  }));
}
```

Ensure `vi` is imported at the top of the file: `import { vi } from "vitest";`

**Also add the reset counterpart.** `src/test/setup.ts` uses `vi.clearAllMocks()` in `afterEach`, which calls `mockClear()` — it does **not** reset `mockImplementation`. Without an explicit reset, a viewport set by one test silently persists into every later test in the file, and tests written to cover the compact path would quietly start exercising the large one while still passing.

```tsx
/**
 * Restore the default `matchMedia` behaviour (`matches: false` for every
 * query). Required in an `afterEach` of any describe block that calls
 * `setViewportWidth`, because setup.ts's `vi.clearAllMocks()` clears call
 * history but leaves the implementation in place.
 */
export function resetViewportWidth(): void {
  Object.defineProperty(window, "innerWidth", {
    configurable: true,
    value: DEFAULT_VIEWPORT_WIDTH,
  });
  vi.mocked(window.matchMedia).mockImplementation((query: string) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  }));
}
```

Define `const DEFAULT_VIEWPORT_WIDTH = window.innerWidth;` once at module scope
before either helper. Resetting only `matchMedia` is insufficient because
`useIsMobile()` initializes from `window.innerWidth` before its effect runs.

Every describe block in this plan that calls `setViewportWidth` must pair it with `afterEach(resetViewportWidth)`. `calendar-module.test.tsx` already does this by hand at `:768` and `:843`; the new helper generalises that pattern.

- [ ] **Step 2: Delegate the existing local helper**

In `src/components/calendar/views/schedule-calendar.test.tsx`, delete the local `setMobile` function body and replace it with a delegation, importing both `setViewportWidth` and `resetViewportWidth` from `@/test/test-utils`:

```tsx
function setMobile(isMobile: boolean) {
  setViewportWidth(isMobile ? 390 : 1024);
}
```

Add `resetViewportWidth();` to the file's existing outer `afterEach` in this
same task. Do not wait for Task 15's describe restructuring: the Task 1 commit
must already obey the no-leak rule.

This shim is temporary. Task 15 wraps all seven tests in a `describe("compact")` block with its own `beforeEach(() => setViewportWidth(390))`, at which point the only remaining caller is the FAB-clearance test's `setMobile(true)`. Inline that call and delete the shim then — `noUnusedLocals` does not cover test files, so nothing will flag it for you.

- [ ] **Step 3: Verify existing tests still pass**

```bash
npm test -- --run src/components/calendar/views/schedule-calendar.test.tsx
```

Expected: 7 passed. Behaviour is identical — this is a pure extraction.

- [ ] **Step 4: Commit**

```bash
npm run build
git add src/test/test-utils.tsx src/components/calendar/views/schedule-calendar.test.tsx
git commit -m "test(calendar): extract shared setViewportWidth helper"
```

---

## Task 2: Move the month matrix utilities

Pure move. `MonthlyCalendar` will consume `buildMonthMatrix` instead of duplicating it, so the function no longer belongs in a Day-rail-named module.

**Files:**
- Create: `src/components/calendar/utils/month-matrix.ts`
- Create: `src/components/calendar/utils/month-matrix.test.ts`
- Modify: `src/components/calendar/utils/day-rail.ts`
- Modify: `src/components/calendar/utils/day-rail.test.ts`
- Modify: `src/components/calendar/components/day-mini-month-rail.tsx:7-11`

- [ ] **Step 1: Create the new module**

Create `src/components/calendar/utils/month-matrix.ts` containing `buildMonthMatrix`, `selectMonthDayDots` and the private `uniqueEventDays`, moved verbatim from `day-rail.ts:33-87`. Keep the existing imports it needs:

```ts
import { isEventOnDate } from "@/lib/time-utils";
import type { CalendarEvent, FamilyColor, FamilyMember } from "@/lib/types";
```

Do not change any logic. This must be a byte-equivalent move of the function bodies.

- [ ] **Step 2: Trim day-rail.ts**

Delete `buildMonthMatrix`, `selectMonthDayDots` and `uniqueEventDays` from `src/components/calendar/utils/day-rail.ts`. It keeps only `RAIL_WIDTH`, `MIN_LANE_WIDTH`, `TIME_AXIS_WIDTH`, `DESKTOP_NAV_WIDTH`, `RAIL_LAYOUT_SLACK` and `railThresholdPx`. Remove the now-unused `isEventOnDate` / `CalendarEvent` / `FamilyColor` / `FamilyMember` imports.

- [ ] **Step 3: Update the rail's import**

In `src/components/calendar/components/day-mini-month-rail.tsx`, split the import:

```tsx
import { RAIL_WIDTH } from "../utils/day-rail";
import { buildMonthMatrix, selectMonthDayDots } from "../utils/month-matrix";
```

- [ ] **Step 4: Move the tests**

Also delete whatever the move orphans in `day-rail.test.ts` — the `member()` and `ev()` builders and the `CalendarEvent` / `FamilyMember` type imports become unused once the two cases leave. Biome reports `noUnusedVariables` as a warning here and `biome check` still exits 0, so lint will not catch it.

Move the two cases at `src/components/calendar/utils/day-rail.test.ts:44` (`"builds a 6x7 (or 5x7) month matrix covering the current month"`) and `:55` (the `selectMonthDayDots` case) into a new `src/components/calendar/utils/month-matrix.test.ts`, importing from `./month-matrix`. Leave the `railThresholdPx` cases in `day-rail.test.ts`.

Start the new test file with the complete import block. The member/event
builders used by the moved cases come from the shared fixtures, so do not copy
the soon-to-be-orphaned local `member()` / `ev()` helpers into the new file:

```ts
import { expect, it } from "vitest";
import { createTestEvent, testMembers } from "@/test/fixtures";
import {
  buildMonthMatrix,
  selectMonthDayDots,
} from "./month-matrix";

it("builds a 4x7, 5x7 or 6x7 month matrix covering the current month", () => {
  const matrix = buildMonthMatrix(new Date(2026, 6, 15));

  expect(matrix.length % 7).toBe(0);
  expect(matrix).toHaveLength(35);
  expect(
    matrix.some((day) => day.getMonth() === 6 && day.getDate() === 1),
  ).toBe(true);
  expect(
    matrix.some((day) => day.getMonth() === 6 && day.getDate() === 31),
  ).toBe(true);
});

it("maps each day to unique member colours in family order", () => {
  const july6 = new Date(2026, 6, 6);
  const dots = selectMonthDayDots(
    [
      createTestEvent({
        id: "first",
        date: july6,
        memberId: testMembers[0].id,
      }),
      createTestEvent({
        id: "duplicate-member",
        date: july6,
        memberId: testMembers[0].id,
      }),
      createTestEvent({
        id: "second",
        date: july6,
        memberId: testMembers[1].id,
      }),
    ],
    testMembers,
  );

  expect(dots.get(july6.toDateString())).toEqual([
    testMembers[0].color,
    testMembers[1].color,
  ]);
});
```

This is the complete initial `month-matrix.test.ts`, not a fragment. Steps 5
and 6 append their member-consistency and exact four-row-February cases to it.
It remains self-contained after the old builders and their type imports are
deleted from `day-rail.test.ts`.

- [ ] **Step 5: Add `selectMonthDayMembers`**

Month needs per-day member **names** as well as colours, for the visually-hidden dot summary in Task 7. `selectMonthDayDots` returns colours only, so add the richer function and reduce the existing one to a wrapper over it — this is also what justifies the move, since otherwise Month would consume neither function.

First add `selectMonthDayMembers` to the import in
`month-matrix.test.ts`, append the consistency test below, and run the focused
file. Expected: FAIL because `./month-matrix` does not export
`selectMonthDayMembers` yet. This preserves the red-first order; only then add
the implementation and wrapper.

Append to `src/components/calendar/utils/month-matrix.ts`:

```ts
/** Distinct members with an event on each day, in family order. */
export function selectMonthDayMembers(
  events: CalendarEvent[],
  members: FamilyMember[],
): Map<string, FamilyMember[]> {
  const dayMemberIds = new Map<string, Set<string>>();
  for (const event of events) {
    for (const day of uniqueEventDays(event)) {
      const key = day.toDateString();
      if (!dayMemberIds.has(key)) dayMemberIds.set(key, new Set());
      dayMemberIds.get(key)?.add(event.memberId);
    }
  }

  const result = new Map<string, FamilyMember[]>();
  for (const [key, ids] of dayMemberIds) {
    const dayMembers = members.filter((member) => ids.has(member.id));
    if (dayMembers.length > 0) result.set(key, dayMembers);
  }
  return result;
}
```

Then rewrite `selectMonthDayDots` as a wrapper so the two cannot drift:

```ts
export function selectMonthDayDots(
  events: CalendarEvent[],
  members: FamilyMember[],
): Map<string, FamilyColor[]> {
  const result = new Map<string, FamilyColor[]>();
  for (const [key, dayMembers] of selectMonthDayMembers(events, members)) {
    result.set(key, dayMembers.map((member) => member.color));
  }
  return result;
}
```

Add a test asserting the two stay consistent:

```ts
it("derives dot colours from the same members it reports", () => {
  const events = [
    createTestEvent({ id: "a", date: new Date(2026, 2, 8), memberId: testMembers[0].id }),
    createTestEvent({ id: "b", date: new Date(2026, 2, 8), memberId: testMembers[1].id }),
  ];
  const key = new Date(2026, 2, 8).toDateString();

  const memberList = selectMonthDayMembers(events, testMembers).get(key) ?? [];
  const colors = selectMonthDayDots(events, testMembers).get(key) ?? [];

  expect(colors).toEqual(memberList.map((m) => m.color));
  expect(memberList.map((m) => m.name)).toEqual(["John", "Jane"]);
});
```

- [ ] **Step 6: Add the four-row February case**

Append to `src/components/calendar/utils/month-matrix.test.ts`:

```ts
it("builds exactly 4 rows for a 28-day February starting on Sunday", () => {
  // Feb 2026 starts on a Sunday and 2026 is not a leap year, so the month
  // fills exactly four weeks with no leading or trailing padding. It is the
  // only such month between 2024 and 2030.
  const matrix = buildMonthMatrix(new Date(2026, 1, 15));

  expect(matrix).toHaveLength(28);
  expect(matrix[0]).toEqual(new Date(2026, 1, 1));
  expect(matrix[27]).toEqual(new Date(2026, 1, 28));
});
```

- [ ] **Step 7: Run tests and typecheck**

```bash
npm test -- --run src/components/calendar/utils/ src/components/calendar/components/day-mini-month-rail.test.tsx
npm run build
```

Expected: all pass, build exits 0. The Day rail is untouched behaviourally.

- [ ] **Step 8: Commit**

```bash
git add src/components/calendar/utils src/components/calendar/components/day-mini-month-rail.tsx
git commit -m "refactor(calendar): move month matrix helpers out of day-rail"
```

---

## Task 3: Month capacity helper

**Files:**
- Create: `src/components/calendar/utils/month-capacity.ts`
- Create: `src/components/calendar/utils/month-capacity.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `src/components/calendar/utils/month-capacity.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import {
  MONTH_MIN_ROW_HEIGHT,
  MONTH_ROW_GAP,
  monthRowHeight,
  monthSlotCapacity,
} from "./month-capacity";

describe("monthSlotCapacity", () => {
  it("guarantees at least 2 slots at the minimum row height", () => {
    expect(monthSlotCapacity(MONTH_MIN_ROW_HEIGHT)).toBeGreaterThanOrEqual(2);
  });

  it("is monotonic in row height", () => {
    let previous = 0;
    for (let h = MONTH_MIN_ROW_HEIGHT; h <= 260; h += 4) {
      const capacity = monthSlotCapacity(h);
      expect(capacity).toBeGreaterThanOrEqual(previous);
      previous = capacity;
    }
  });

  it("yields strictly more slots for a tall row than a short one", () => {
    // Representative of a five-week month at 1440x900 vs a six-week month
    // at 1024x768. Absolute counts are confirmed at the screenshot gate.
    expect(monthSlotCapacity(148)).toBeGreaterThan(monthSlotCapacity(102));
  });

  it("never returns a negative capacity for degenerate heights", () => {
    expect(monthSlotCapacity(0)).toBe(0);
    expect(monthSlotCapacity(-50)).toBe(0);
  });
});

describe("monthRowHeight", () => {
  it("divides the container across the week count", () => {
    expect(monthRowHeight(744, 6)).toBe(124);
  });

  it("applies the minimum row height floor", () => {
    expect(monthRowHeight(300, 6)).toBe(MONTH_MIN_ROW_HEIGHT);
  });

  it("handles a four-row month", () => {
    const rowHeight = monthRowHeight(744, 4, MONTH_ROW_GAP);
    expect(rowHeight).toBe(Math.floor((744 - 3 * MONTH_ROW_GAP) / 4));
    expect(monthSlotCapacity(rowHeight)).toBeGreaterThan(
      monthSlotCapacity(monthRowHeight(744, 6, MONTH_ROW_GAP)),
    );
  });

  it("falls back to the floor for a zero week count", () => {
    expect(monthRowHeight(744, 0)).toBe(MONTH_MIN_ROW_HEIGHT);
  });

  it("subtracts inter-row gaps from the available height", () => {
    // 6 rows have 5 gaps between them. Without this the rows overflow their
    // container by the total gap height and the grid scrolls permanently.
    expect(monthRowHeight(744, 6, MONTH_ROW_GAP)).toBe(
      Math.floor((744 - 5 * MONTH_ROW_GAP) / 6),
    );
  });
});
```

**Important:** `containerHeight` is the height of the **weeks container only**, not the whole grid. The weekday header row is a sibling of the weeks container, outside the observed element, so it must not be subtracted here — Task 8 attaches the observer accordingly. Getting this wrong produces a permanent scrollbar and fails the "no dead space below the final week row" criterion, and no unit test would catch it.

- [ ] **Step 2: Run to verify it fails**

```bash
npm test -- --run src/components/calendar/utils/month-capacity.test.ts
```

Expected: FAIL — cannot resolve `./month-capacity`.

- [ ] **Step 3: Implement**

Create `src/components/calendar/utils/month-capacity.ts`:

```ts
/**
 * Month cell geometry at lg+. Capacity is counted in *slots*, not events:
 * `planCellSlots` can reserve a slot that holds no event so that a multi-day
 * run keeps the same vertical position across the cells it touches.
 *
 * These constants are tunable at the screenshot gate, in the same way
 * DENSE_HOUR_ROW_HEIGHT was tuned for the shipped Week view — with one
 * exception: MONTH_CHIP_HEIGHT is a visual floor, not a dial. The cell is the
 * interactive target; dense slots are presentational. See spec Section 7.
 */

/** Two 1px cell borders. */
export const MONTH_CELL_BORDER_Y = 2;
/** Two 4px vertical paddings; the cell renders this exact value. */
export const MONTH_CELL_PADDING_Y = 8;
/** Fixed date-numeral/header row; the cell renders this exact height. */
export const MONTH_NUMERAL_BLOCK = 20;
/** Gap between the header row and the first visual slot. */
export const MONTH_HEADER_SLOT_GAP = 2;
/**
 * Visual slot height floor. Chips and the `+N` summary are not controls; the
 * 96px-or-taller gridcell is the one >=44px target and opens the popover.
 */
export const MONTH_CHIP_HEIGHT = 28;
/** Vertical gap between chips. */
export const MONTH_CHIP_GAP = 2;
/** Row height floor; guarantees monthSlotCapacity() >= 2. */
export const MONTH_MIN_ROW_HEIGHT = 96;
/** Vertical gap between week rows; adjacent targets require at least 8px. */
export const MONTH_ROW_GAP = 8;
/** Horizontal gap between day columns; adjacent targets require at least 8px. */
export const MONTH_COLUMN_GAP = 8;
/** Horizontal padding inside a day cell. */
export const MONTH_CELL_PADDING_X = 4;
/** One horizontal border plus padding plus half the inter-cell gap. */
export const MONTH_CHIP_BLEED_X =
  1 + MONTH_CELL_PADDING_X + MONTH_COLUMN_GAP / 2;

/**
 * Row height for `weekCount` rows inside `containerHeight` px.
 *
 * `containerHeight` must be the height of the **weeks container**, not the
 * whole grid — the weekday header sits outside it. `rowGap` accounts for the
 * `weekCount - 1` gaps between rows; omitting it overflows the container.
 */
export function monthRowHeight(
  containerHeight: number,
  weekCount: number,
  rowGap = 0,
): number {
  if (weekCount <= 0) return MONTH_MIN_ROW_HEIGHT;
  const available = containerHeight - rowGap * (weekCount - 1);
  return Math.max(MONTH_MIN_ROW_HEIGHT, Math.floor(available / weekCount));
}

/** How many chip-sized slots fit in a cell of the given row height. */
export function monthSlotCapacity(rowHeight: number): number {
  const usable =
    rowHeight -
    MONTH_CELL_BORDER_Y -
    MONTH_CELL_PADDING_Y -
    MONTH_NUMERAL_BLOCK -
    MONTH_HEADER_SLOT_GAP;
  if (usable <= 0) return 0;
  return Math.max(
    0,
    Math.floor((usable + MONTH_CHIP_GAP) / (MONTH_CHIP_HEIGHT + MONTH_CHIP_GAP)),
  );
}
```

- [ ] **Step 4: Run to verify it passes**

```bash
npm test -- --run src/components/calendar/utils/month-capacity.test.ts
```

Expected: PASS, 9 tests.

- [ ] **Step 5: Commit**

```bash
npm run build
git add src/components/calendar/utils/month-capacity.ts src/components/calendar/utils/month-capacity.test.ts
git commit -m "feat(calendar): add month slot capacity helper"
```

---

## Task 4: Month slot planning helper

This is the heart of the multi-day design. Spec Sections 4.2 and 4.3.

**Files:**
- Create: `src/components/calendar/utils/month-slots.ts`
- Create: `src/components/calendar/utils/month-slots.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `src/components/calendar/utils/month-slots.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createTestEvent, testMembers } from "@/test/fixtures";
import {
  isMultiDay,
  multiDayEdge,
  orderRowMultiDay,
  planCellSlots,
} from "./month-slots";

const d = (day: number) => new Date(2026, 2, day); // March 2026
const weekOne = [1, 2, 3, 4, 5, 6, 7].map(d);

const trip = (title: string, from: number, to: number) =>
  createTestEvent({
    id: title,
    title,
    date: d(from),
    endDate: d(to),
    isAllDay: true,
    memberId: testMembers[0].id,
  });

const single = (title: string, day: number, startTime = "9:00 AM") =>
  createTestEvent({ id: title, title, date: d(day), startTime });

describe("isMultiDay", () => {
  it("is true when endDate is after date", () => {
    expect(isMultiDay(trip("Spring break", 2, 6))).toBe(true);
  });

  it("is false with no endDate", () => {
    expect(isMultiDay(single("Piano", 3))).toBe(false);
  });

  it("is false when endDate equals date", () => {
    expect(isMultiDay(trip("One day", 3, 3))).toBe(false);
  });
});

describe("multiDayEdge", () => {
  const run = trip("Spring break", 2, 6);

  it("marks the true start", () => {
    expect(multiDayEdge(run, d(2))).toBe("start");
  });

  it("marks interior days", () => {
    expect(multiDayEdge(run, d(4))).toBe("middle");
  });

  it("marks the true end", () => {
    expect(multiDayEdge(run, d(6))).toBe("end");
  });

  it("marks a single-day event solo", () => {
    expect(multiDayEdge(single("Piano", 3), d(3))).toBe("solo");
  });

  it("keeps a middle edge at a week boundary", () => {
    // Sat 14 is the last cell of its row but not the run's end, so it must
    // stay square-edged for the weld to continue into the next row.
    const acrossBoundary = trip("Grandma visits", 11, 16);
    expect(multiDayEdge(acrossBoundary, d(14))).toBe("middle");
    expect(multiDayEdge(acrossBoundary, d(15))).toBe("middle");
    expect(multiDayEdge(acrossBoundary, d(16))).toBe("end");
  });
});

describe("orderRowMultiDay", () => {
  it("orders by start ascending, then end descending, then title", () => {
    const later = trip("Later", 4, 5);
    const earlyShort = trip("Early short", 2, 3);
    const earlyLong = trip("Early long", 2, 6);

    expect(
      orderRowMultiDay([later, earlyShort, earlyLong], weekOne).map(
        (e) => e.title,
      ),
    ).toEqual(["Early long", "Early short", "Later"]);
  });

  it("excludes single-day events", () => {
    expect(
      orderRowMultiDay([single("Piano", 3), trip("Trip", 2, 4)], weekOne).map(
        (e) => e.title,
      ),
    ).toEqual(["Trip"]);
  });

  it("excludes runs that do not touch the row", () => {
    expect(orderRowMultiDay([trip("Far", 20, 24)], weekOne)).toHaveLength(0);
  });
});

describe("planCellSlots", () => {
  it("places a lone run at slot 0 on every day it covers", () => {
    const run = trip("Spring break", 2, 6);
    const rowMultiDay = orderRowMultiDay([run], weekOne);

    for (const day of [2, 3, 4, 5, 6]) {
      const plan = planCellSlots({
        rowMultiDay,
        day: d(day),
        singleDayEvents: [],
        capacity: 4,
      });
      expect(plan.slots[0]).toMatchObject({ kind: "event" });
      expect(plan.slots[0].event?.title).toBe("Spring break");
    }
  });

  it("keeps 3 overlapping runs in stable slots across every covered cell", () => {
    const runs = [
      trip("Outer", 1, 7),
      trip("Middle", 2, 6),
      trip("Inner", 3, 5),
    ];
    const rowMultiDay = orderRowMultiDay(runs, weekOne);

    for (const [expectedSlot, run] of runs.entries()) {
      const start = run.date.getDate();
      const end = run.endDate?.getDate() ?? start;
      for (let day = start; day <= end; day++) {
        const plan = planCellSlots({
          rowMultiDay,
          day: d(day),
          singleDayEvents: [],
          capacity: 4,
        });
        expect(
          plan.slots.findIndex((slot) => slot.event?.id === run.id),
        ).toBe(expectedSlot);
      }
    }
  });

  it("reserves a blank slot above a lower-ordered run", () => {
    const first = trip("First", 2, 3);
    const second = trip("Second", 4, 6);
    const rowMultiDay = orderRowMultiDay([first, second], weekOne);

    // Mar 5 is covered by Second (index 1) but not First (index 0).
    const plan = planCellSlots({
      rowMultiDay,
      day: d(5),
      singleDayEvents: [],
      capacity: 4,
    });

    expect(plan.slots[0]).toMatchObject({ kind: "blank" });
    expect(plan.slots[1].event?.title).toBe("Second");
  });

  it("reserves nothing on a day no run covers", () => {
    const rowMultiDay = orderRowMultiDay([trip("Trip", 2, 3)], weekOne);
    const plan = planCellSlots({
      rowMultiDay,
      day: d(6),
      singleDayEvents: [single("Dentist", 6)],
      capacity: 4,
    });

    expect(plan.slots).toHaveLength(1);
    expect(plan.slots[0].event?.title).toBe("Dentist");
  });

  it("never exceeds capacity even when blanks are reserved", () => {
    const first = trip("First", 2, 3);
    const second = trip("Second", 4, 6);
    const rowMultiDay = orderRowMultiDay([first, second], weekOne);

    // Mar 5 needs slot 0 blank + slot 1 for Second, plus two singles = 4 slots
    // in a 3-slot cell. A naive eventCount comparison (3 events <= 3 capacity)
    // would render all of them and overflow the cell.
    const plan = planCellSlots({
      rowMultiDay,
      day: d(5),
      singleDayEvents: [single("A", 5), single("B", 5, "10:00 AM")],
      capacity: 3,
    });

    expect(plan.slots.length).toBeLessThanOrEqual(3);
  });

  it("counts only events in the overflow total, never blanks", () => {
    const first = trip("First", 2, 3);
    const second = trip("Second", 4, 6);
    const rowMultiDay = orderRowMultiDay([first, second], weekOne);

    const plan = planCellSlots({
      rowMultiDay,
      day: d(5),
      singleDayEvents: [single("A", 5), single("B", 5, "10:00 AM")],
      capacity: 3,
    });

    // Rendered: slot 0 blank, slot 1 "Second". Hidden: A and B.
    expect(plan.slots).toHaveLength(2);
    expect(plan.overflowCount).toBe(2);
  });

  it("documents the reserved-lane overflow-only edge case", () => {
    const first = trip("First", 2, 3);
    const second = trip("Second", 4, 6);
    const rowMultiDay = orderRowMultiDay([first, second], weekOne);

    // Mar 5 needs [blank, Second, Single]. At capacity 2 the final visual slot
    // is the +N summary, so exact lane alignment deliberately leaves no event
    // chip visible. The gridcell count and full-day popover remain complete.
    const plan = planCellSlots({
      rowMultiDay,
      day: d(5),
      singleDayEvents: [single("Single", 5)],
      capacity: 2,
    });

    expect(plan.slots).toEqual([{ kind: "blank" }]);
    expect(plan.overflowCount).toBe(2);
  });

  it("reports no overflow when everything fits", () => {
    const plan = planCellSlots({
      rowMultiDay: [],
      day: d(3),
      singleDayEvents: [single("A", 3), single("B", 3, "10:00 AM")],
      capacity: 4,
    });

    expect(plan.slots).toHaveLength(2);
    expect(plan.overflowCount).toBe(0);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

```bash
npm test -- --run src/components/calendar/utils/month-slots.test.ts
```

Expected: FAIL — cannot resolve `./month-slots`.

- [ ] **Step 3: Implement**

Create `src/components/calendar/utils/month-slots.ts`:

```ts
import { isEventOnDate } from "@/lib/time-utils";
import type { CalendarEvent } from "@/lib/types";

/** Corner geometry for a chip within a multi-day run. */
export type MonthChipEdge = "start" | "middle" | "end" | "solo";

export interface MonthSlot {
  kind: "event" | "blank";
  event?: CalendarEvent;
  edge?: MonthChipEdge;
}

export interface MonthCellPlan {
  /** Slots to render, already truncated to capacity. */
  slots: MonthSlot[];
  /** Events not rendered. Excludes reserved blank slots. */
  overflowCount: number;
}

function sameDay(a: Date, b: Date): boolean {
  return a.toDateString() === b.toDateString();
}

/** True when the event spans more than one calendar day. */
export function isMultiDay(event: CalendarEvent): boolean {
  return Boolean(event.endDate) && !sameDay(event.date, event.endDate as Date);
}

/**
 * Corner geometry for `event` as rendered on `day`. Rounded corners appear
 * only at the run's true start and true end, so a run clipped by the visible
 * grid keeps square edges at the grid boundary — and a week boundary needs no
 * special case.
 */
export function multiDayEdge(event: CalendarEvent, day: Date): MonthChipEdge {
  if (!isMultiDay(event)) return "solo";
  const isStart = sameDay(event.date, day);
  const isEnd = sameDay(event.endDate as Date, day);
  if (isStart && isEnd) return "solo";
  if (isStart) return "start";
  if (isEnd) return "end";
  return "middle";
}

/**
 * Multi-day events touching this week row, in the stable order every cell in
 * the row uses: start ascending, then end descending (longest first), then
 * title. Computed once per row so a run holds one slot index across the row.
 *
 * Known limitation (spec Section 4.3): this is row-scoped, so a run can hold a
 * different index in the next row. Alignment is exact within a row only.
 */
export function orderRowMultiDay(
  events: CalendarEvent[],
  weekDays: Date[],
): CalendarEvent[] {
  return events
    .filter(
      (event) =>
        isMultiDay(event) && weekDays.some((day) => isEventOnDate(event, day)),
    )
    .sort((a, b) => {
      const byStart = a.date.getTime() - b.date.getTime();
      if (byStart !== 0) return byStart;
      const aEnd = (a.endDate ?? a.date).getTime();
      const bEnd = (b.endDate ?? b.date).getTime();
      if (aEnd !== bEnd) return bEnd - aEnd;
      return a.title.localeCompare(b.title);
    });
}

/**
 * Ordered slot plan for one day cell.
 *
 * Multi-day runs occupy slots `0..k`, where `k` is the highest index in
 * `rowMultiDay` covering this day; slots below `k` that do not cover this day
 * are reserved blank so the runs above stay aligned. Single-day events follow.
 * The result is truncated to `capacity`, reserving the last visual slot for
 * the `+N` summary when anything is hidden. A reserved blank can be the only
 * visible prefix in a pathological overlap; this preserves exact lane
 * alignment and is an explicit, tested limitation rather than a false
 * guarantee that one chip always renders.
 */
export function planCellSlots({
  rowMultiDay,
  day,
  singleDayEvents,
  capacity,
}: {
  rowMultiDay: CalendarEvent[];
  day: Date;
  singleDayEvents: CalendarEvent[];
  capacity: number;
}): MonthCellPlan {
  let lastCovering = -1;
  for (let i = 0; i < rowMultiDay.length; i++) {
    if (isEventOnDate(rowMultiDay[i], day)) lastCovering = i;
  }

  const slots: MonthSlot[] = [];
  for (let i = 0; i <= lastCovering; i++) {
    const event = rowMultiDay[i];
    if (isEventOnDate(event, day)) {
      slots.push({ kind: "event", event, edge: multiDayEdge(event, day) });
    } else {
      slots.push({ kind: "blank" });
    }
  }
  for (const event of singleDayEvents) {
    slots.push({ kind: "event", event, edge: "solo" });
  }

  if (slots.length <= capacity) {
    return { slots, overflowCount: 0 };
  }

  const visibleCount = Math.max(0, capacity - 1);
  const visible = slots.slice(0, visibleCount);
  const overflowCount = slots
    .slice(visibleCount)
    .filter((slot) => slot.kind === "event").length;

  return { slots: visible, overflowCount };
}
```

- [ ] **Step 4: Run to verify it passes**

```bash
npm test -- --run src/components/calendar/utils/month-slots.test.ts
npm run build
```

Expected: PASS, 19 tests. Build exits 0.

- [ ] **Step 5: Commit**

```bash
git add src/components/calendar/utils/month-slots.ts src/components/calendar/utils/month-slots.test.ts
git commit -m "feat(calendar): add month slot planning with multi-day reservation"
```

---

## Task 5: Month event chip

**Files:**
- Create: `src/components/calendar/components/month-event-chip.tsx`
- Create: `src/components/calendar/components/month-event-chip.test.tsx`
- Modify: `src/components/calendar/components/index.ts`

- [ ] **Step 1: Write the failing tests**

Create `src/components/calendar/components/month-event-chip.test.tsx`:

```tsx
import { describe, expect, it } from "vitest";
import { colorMap } from "@/lib/types";
import { createTestEvent, testMembers } from "@/test/fixtures";
import { render, screen, seedFamilyStore } from "@/test/test-utils";
import { MonthEventChip } from "./month-event-chip";

const event = createTestEvent({
  id: "trip",
  title: "Spring break",
  memberId: "missing-member",
});

function relativeLuminance(hex: string): number {
  const channels = hex.match(/[0-9a-f]{2}/gi);
  if (!channels || channels.length !== 3) throw new Error(`Invalid hex: ${hex}`);
  const [red, green, blue] = channels.map((channel) => {
    const value = Number.parseInt(channel, 16) / 255;
    return value <= 0.04045
      ? value / 12.92
      : ((value + 0.055) / 1.055) ** 2.4;
  });
  return 0.2126 * red + 0.7152 * green + 0.0722 * blue;
}

function contrastRatio(first: string, second: string): number {
  const lighter = Math.max(relativeLuminance(first), relativeLuminance(second));
  const darker = Math.min(relativeLuminance(first), relativeLuminance(second));
  return (lighter + 0.05) / (darker + 0.05);
}

describe("MonthEventChip", () => {
  it("is visual content rather than a nested control", () => {
    render(
      <MonthEventChip
        event={event}
        edge="solo"
        weldLeft={false}
        weldRight={false}
      />,
    );
    expect(screen.queryByRole("button")).not.toBeInTheDocument();
    expect(screen.getByTestId("month-event-chip")).toHaveAttribute(
      "aria-hidden",
      "true",
    );
  });

  it("expands its box by the exact left and right weld", () => {
    render(
      <MonthEventChip
        event={event}
        edge="middle"
        weldLeft
        weldRight
      />,
    );
    expect(screen.getByTestId("month-event-chip")).toHaveStyle({
      height: "28px",
      minHeight: "28px",
      marginLeft: "-9px",
      width: "calc(100% + 18px)",
    });
  });

  it("suppresses outward bleed at a row edge", () => {
    render(
      <MonthEventChip
        event={event}
        edge="middle"
        weldLeft={false}
        weldRight
      />,
    );
    expect(screen.getByTestId("month-event-chip")).toHaveStyle({
      marginLeft: "0px",
      width: "calc(100% + 9px)",
    });
  });

  it("uses foreground text on the light missing-member fallback", () => {
    render(
      <MonthEventChip
        event={event}
        edge="solo"
        weldLeft={false}
        weldRight={false}
      />,
    );
    const chip = screen.getByTestId("month-event-chip");
    expect(chip).toHaveClass("bg-muted", "text-foreground", "shrink-0");
    expect(chip).not.toHaveClass("text-white");
    expect(chip).toHaveTextContent(/Unknown.*Spring break/);
  });

  it.each(Object.entries(colorMap))(
    "%s solid member background meets AA with white text",
    (_name, colors) => {
      expect(contrastRatio("#ffffff", colors.hex)).toBeGreaterThanOrEqual(4.5);
    },
  );

  it("renders all-day and recurrence markers as visual content", () => {
    render(
      <MonthEventChip
        event={{ ...event, isAllDay: true, isRecurring: true }}
        edge="solo"
        weldLeft={false}
        weldRight={false}
      />,
    );
    expect(screen.getByTestId("month-all-day-marker")).toHaveAttribute(
      "aria-hidden",
      "true",
    );
    expect(screen.getByTestId("month-all-day-marker")).toHaveClass(
      "bg-current",
    );
    expect(screen.getByTestId("month-recurring-marker")).toHaveAttribute(
      "aria-hidden",
      "true",
    );
  });

  it("shows the member name so colour is not the only visible identity cue", () => {
    const namedMember = { ...testMembers[0], name: "John Smith" };
    seedFamilyStore({ name: "Test Family", members: [namedMember] });
    render(
      <MonthEventChip
        event={{ ...event, memberId: namedMember.id }}
        edge="solo"
        weldLeft={false}
        weldRight={false}
      />,
    );
    expect(screen.getByTestId("month-chip-member")).toHaveTextContent("John");
    expect(screen.getByTestId("month-chip-member")).not.toHaveTextContent(
      "Smith",
    );
  });
});
```

- [ ] **Step 2: Run to verify it fails**

```bash
npm test -- --run src/components/calendar/components/month-event-chip.test.tsx
```

Expected: FAIL — cannot resolve `./month-event-chip`.

- [ ] **Step 3: Implement the chip**

Create `src/components/calendar/components/month-event-chip.tsx`:

```tsx
import { Repeat } from "lucide-react";
import { useFamilyMembers } from "@/api";
import { type CalendarEvent, colorMap, getFamilyMember } from "@/lib/types";
import { cn } from "@/lib/utils";
import {
  MONTH_CHIP_BLEED_X,
  MONTH_CHIP_HEIGHT,
} from "../utils/month-capacity";
import type { MonthChipEdge } from "../utils/month-slots";
import { GoogleBadge, isGoogleEvent } from "./calendar-event";

interface MonthEventChipProps {
  event: CalendarEvent;
  edge: MonthChipEdge;
  /** False at Sunday so nothing bleeds outside the row. */
  weldLeft: boolean;
  /** False at Saturday so nothing bleeds outside the row. */
  weldRight: boolean;
}

export function MonthEventChip({
  event,
  edge,
  weldLeft,
  weldRight,
}: MonthEventChipProps) {
  const familyMembers = useFamilyMembers();
  const member = getFamilyMember(familyMembers, event.memberId);
  const colors = member ? colorMap[member.color] : undefined;
  const memberFirstName = member?.name.trim().split(/\s+/)[0] ?? "Unknown";
  const left = weldLeft ? MONTH_CHIP_BLEED_X : 0;
  const right = weldRight ? MONTH_CHIP_BLEED_X : 0;

  return (
    <div
      data-testid="month-event-chip"
      aria-hidden="true"
      className={cn(
        "pointer-events-none relative z-10 flex w-full shrink-0 items-center gap-1 px-1.5 text-left text-sm font-medium",
        colors ? [colors.bg, "text-white"] : ["bg-muted", "text-foreground"],
        edge === "solo" && "rounded",
        edge === "start" && "rounded-l",
        edge === "end" && "rounded-r",
      )}
      style={{
        height: MONTH_CHIP_HEIGHT,
        minHeight: MONTH_CHIP_HEIGHT,
        marginLeft: -left,
        width: `calc(100% + ${left + right}px)`,
      }}
    >
      {event.isAllDay && (
        <span
          data-testid="month-all-day-marker"
          aria-hidden="true"
          className="size-1.5 shrink-0 rounded-full bg-current opacity-80"
        />
      )}
      {isGoogleEvent(event) && (
        <span className="shrink-0" aria-hidden="true">
          <GoogleBadge size={8} />
        </span>
      )}
      <span
        data-testid="month-chip-member"
        className="max-w-[35%] shrink-0 truncate font-semibold"
      >
        {memberFirstName}
      </span>
      <span aria-hidden="true">·</span>
      <span className="truncate">{event.title}</span>
      {event.isRecurring && (
        <Repeat
          data-testid="month-recurring-marker"
          aria-hidden="true"
          className="ml-auto size-3 shrink-0"
        />
      )}
    </div>
  );
}
```

The all-day dot replaces the `"● "` string prefix. All dense chip content is
`aria-hidden`; the day cell announces date/count, and the popover action names
title, time/all-day, member, recurrence and span without duplicate marker
announcements.

- [ ] **Step 4: Run the tests and build**

```bash
npm test -- --run src/components/calendar/components/month-event-chip.test.tsx
npm run build
```

Expected: 13 cases pass (6 behavioural cases plus all 7 member colours); build
exits 0.

- [ ] **Step 5: Export it**

Add to `src/components/calendar/components/index.ts`, keeping alphabetical order:

```ts
export { MonthEventChip } from "./month-event-chip";
```

- [ ] **Step 6: Typecheck the final barrel edit**

```bash
npm run build
```

Expected: exits 0.

- [ ] **Step 7: Commit**

```bash
git add src/components/calendar/components/month-event-chip.tsx src/components/calendar/components/month-event-chip.test.tsx src/components/calendar/components/index.ts
git commit -m "feat(calendar): add month event chip with run edge geometry"
```

---

## Task 6: Month overflow popover

**Files:**
- Create: `src/components/calendar/components/month-overflow-popover.tsx`
- Create: `src/components/calendar/components/month-overflow-popover.test.tsx`
- Modify: `src/components/calendar/components/index.ts`

- [ ] **Step 1: Write the failing test**

Create `src/components/calendar/components/month-overflow-popover.test.tsx`:

```tsx
import userEvent from "@testing-library/user-event";
import { addDays } from "date-fns";
import { useState } from "react";
import { describe, expect, it, vi } from "vitest";
import type { CalendarEvent } from "@/lib/types";
import { createTestEvent, testMembers } from "@/test/fixtures";
import { render, screen, seedFamilyStore } from "@/test/test-utils";
import { MonthOverflowPopover } from "./month-overflow-popover";

const date = new Date(2026, 2, 8);

function setup(
  eventCount: number,
  open = true,
  eventOverrides: Partial<CalendarEvent> = {},
) {
  seedFamilyStore({ name: "Test Family", members: testMembers });
  const events = Array.from({ length: eventCount }, (_, i) =>
    createTestEvent({
      id: `e${i}`,
      title: `Event ${i}`,
      date,
      memberId: testMembers[0].id,
      ...eventOverrides,
    }),
  );
  const onEventClick = vi.fn();
  const onOpenDay = vi.fn();
  const onOpenChange = vi.fn();
  const onCloseFocus = vi.fn();

  function Harness() {
    const [isOpen, setIsOpen] = useState(open);
    return (
      <>
        <MonthOverflowPopover
          date={date}
          events={events}
          open={isOpen}
          onOpenChange={(next) => {
            onOpenChange(next);
            setIsOpen(next);
          }}
          onEventClick={onEventClick}
          onOpenDay={onOpenDay}
          onCloseFocus={onCloseFocus}
        >
          <div data-testid="anchor" tabIndex={-1}>anchor</div>
        </MonthOverflowPopover>
        <button type="button" data-testid="outside-target">Outside</button>
      </>
    );
  }

  render(<Harness />);
  return { onEventClick, onOpenDay, onOpenChange, onCloseFocus, events };
}

describe("MonthOverflowPopover", () => {
  it("lists every event for the day, not only the hidden ones", () => {
    setup(6);
    for (let i = 0; i < 6; i++) {
      expect(
        screen.getByRole("button", { name: new RegExp(`Event ${i}`) }),
      ).toBeInTheDocument();
    }
  });

  it("names the popover with the full date", () => {
    setup(6);
    const dialog = screen.getByRole("dialog", {
      name: /events for march 8, 2026/i,
    });
    expect(dialog).toBeInTheDocument();
    expect(dialog).toHaveClass("motion-reduce:animate-none");
  });

  it("names all-day, recurrence and multi-day state on event actions", () => {
    setup(1, true, {
      isAllDay: true,
      isRecurring: true,
      endDate: addDays(date, 4),
    });
    expect(
      screen.getByRole("button", {
        name: /Event 0, all day, John, day 1 of 5, repeats/i,
      }),
    ).toBeInTheDocument();
  });

  it("labels a stale event whose member no longer exists", () => {
    setup(1, true, { memberId: "missing-member" });
    const action = screen.getByRole("button", {
      name: /Event 0, 9:00 AM to 10:00 AM, Unknown member/i,
    });
    expect(action).toHaveTextContent("Unknown member");
  });

  it("offers an open-in-day-view action", async () => {
    const user = userEvent.setup();
    const { onOpenDay, onOpenChange, onCloseFocus } = setup(6);

    await user.click(screen.getByRole("button", { name: /open in day view/i }));

    expect(onOpenChange).toHaveBeenCalledWith(false);
    expect(onOpenDay).toHaveBeenCalledWith(date);
    expect(onOpenChange.mock.invocationCallOrder[0]).toBeLessThan(
      onOpenDay.mock.invocationCallOrder[0],
    );
    // Day navigation unmounts the grid; do not restore focus into it.
    expect(onCloseFocus).not.toHaveBeenCalled();
    expect(screen.queryByRole("dialog")).not.toBeInTheDocument();
  });

  it("opens the detail modal for a selected event", async () => {
    const user = userEvent.setup();
    const { onEventClick, onOpenChange, onCloseFocus } = setup(6);

    await user.click(screen.getByRole("button", { name: /Event 3/ }));

    expect(onOpenChange).toHaveBeenCalledWith(false);
    expect(onEventClick).toHaveBeenCalledTimes(1);
    expect(onEventClick.mock.calls[0][0].title).toBe("Event 3");
    expect(onOpenChange.mock.invocationCallOrder[0]).toBeLessThan(
      onEventClick.mock.invocationCallOrder[0],
    );
    // EventDetailModal owns focus next; the popover must not fight it.
    expect(onCloseFocus).not.toHaveBeenCalled();
  });

  it("renders nothing when closed", () => {
    setup(6, false);
    expect(screen.queryByRole("dialog")).not.toBeInTheDocument();
  });

  it("returns focus to the anchor on Escape rather than to a trigger", async () => {
    const user = userEvent.setup();
    const { onOpenChange, onCloseFocus } = setup(6);

    await user.keyboard("{Escape}");

    expect(onOpenChange).toHaveBeenCalledWith(false);
    // Radix would otherwise focus its Trigger; this component has none, so
    // the cell must be focused explicitly via onCloseAutoFocus.
    expect(onCloseFocus).toHaveBeenCalled();
  });

  it("leaves focus on a newly clicked outside control", async () => {
    const user = userEvent.setup();
    const { onCloseFocus } = setup(6);
    const outside = screen.getByTestId("outside-target");

    await user.click(outside);

    expect(outside).toHaveFocus();
    expect(onCloseFocus).not.toHaveBeenCalled();
  });

  it("keeps dense event lists scrollable", () => {
    setup(20);
    expect(screen.getByRole("list")).toHaveClass("max-h-72", "overflow-y-auto");
  });
});
```

- [ ] **Step 2: Run to verify it fails**

```bash
npm test -- --run src/components/calendar/components/month-overflow-popover.test.tsx
```

Expected: FAIL — cannot resolve `./month-overflow-popover`.

- [ ] **Step 3: Implement**

Create `src/components/calendar/components/month-overflow-popover.tsx`:

```tsx
import { differenceInCalendarDays, format } from "date-fns";
import { type ReactNode, useRef } from "react";
import { useFamilyMembers } from "@/api";
import { Popover, PopoverAnchor, PopoverContent } from "@/components/ui/popover";
import { getEventKey } from "@/lib/time-utils";
import { type CalendarEvent, colorMap, getFamilyMember } from "@/lib/types";
import { cn } from "@/lib/utils";

interface MonthOverflowPopoverProps {
  date: Date;
  /** Every event for the day, not only the hidden ones. */
  events: CalendarEvent[];
  /** Always a boolean — the popover is unconditionally controlled. */
  open: boolean;
  onOpenChange: (open: boolean) => void;
  onEventClick: (event: CalendarEvent) => void;
  onOpenDay: (date: Date) => void;
  /** Focus target for dismissals; actions transfer focus elsewhere. */
  onCloseFocus?: () => void;
  /** The element the popover anchors to — the day cell. */
  children: ReactNode;
}

export function MonthOverflowPopover({
  date,
  events,
  open,
  onOpenChange,
  onEventClick,
  onOpenDay,
  onCloseFocus,
  children,
}: MonthOverflowPopoverProps) {
  const familyMembers = useFamilyMembers();
  const skipCloseFocus = useRef(false);

  const closeForAction = (action: () => void) => {
    skipCloseFocus.current = true;
    onOpenChange(false);
    action();
  };

  return (
    // Anchor-based, deliberately trigger-less. The popover must be openable on
    // ANY day that has events. `open` is always a boolean, never undefined, so
    // the Radix instance is unambiguously controlled.
    <Popover
      open={open}
      onOpenChange={(next) => {
        if (next) skipCloseFocus.current = false;
        onOpenChange(next);
      }}
    >
      <PopoverAnchor asChild>{children}</PopoverAnchor>
      <PopoverContent
        align="start"
        aria-label={`Events for ${format(date, "MMMM d, yyyy")}`}
        className="motion-reduce:animate-none"
        onInteractOutside={() => {
          // Let a pointer/focus dismissal keep the newly chosen outside target.
          // Escape has no interact-outside event and restores the cell below.
          skipCloseFocus.current = true;
        }}
        onCloseAutoFocus={(event) => {
          // Escape returns to the cell. Outside pointer dismissal keeps the
          // selected target; actions transfer focus or unmount this grid.
          event.preventDefault();
          if (skipCloseFocus.current) {
            skipCloseFocus.current = false;
            return;
          }
          onCloseFocus?.();
        }}
      >
        <p className="mb-2 text-sm font-semibold">
          {format(date, "EEEE, MMMM d")}
        </p>
        <ul className="flex max-h-72 flex-col gap-1 overflow-y-auto pr-1">
          {events.map((event) => {
            const member = getFamilyMember(familyMembers, event.memberId);
            const memberName = member?.name ?? "Unknown member";
            const colors = member ? colorMap[member.color] : undefined;
            const spanTotal = event.endDate
              ? differenceInCalendarDays(event.endDate, event.date) + 1
              : 1;
            const spanDay = differenceInCalendarDays(date, event.date) + 1;
            const actionLabel = [
              event.title,
              event.isAllDay
                ? "all day"
                : `${event.startTime} to ${event.endTime}`,
              memberName,
              spanTotal > 1 ? `day ${spanDay} of ${spanTotal}` : undefined,
              event.isRecurring ? "repeats" : undefined,
            ]
              .filter(Boolean)
              .join(", ");
            return (
              <li key={getEventKey(event)}>
                <button
                  type="button"
                  aria-label={actionLabel}
                  onClick={() => closeForAction(() => onEventClick(event))}
                  className="flex min-h-11 w-full items-center gap-2 rounded-lg px-2 text-left hover:bg-accent"
                >
                  <span
                    aria-hidden="true"
                    className={cn(
                      "h-8 w-1 shrink-0 rounded-full",
                      colors?.bg ?? "bg-muted",
                    )}
                  />
                  <span className="min-w-0 flex-1">
                    <span className="block truncate text-sm font-medium">
                      {event.title}
                    </span>
                    <span className="block text-sm text-muted-foreground">
                      {event.isAllDay
                        ? "All day"
                        : `${event.startTime} – ${event.endTime}`}
                      {` · ${memberName}`}
                    </span>
                  </span>
                </button>
              </li>
            );
          })}
        </ul>
        <button
          type="button"
          onClick={() => closeForAction(() => onOpenDay(date))}
          className="mt-2 min-h-11 w-full rounded-lg border border-border text-sm font-medium hover:bg-accent"
        >
          Open in Day view
        </button>
      </PopoverContent>
    </Popover>
  );
}
```

The `+N more` summary is not part of this component and is not a button. The
day gridcell owns activation for both overflowing and non-overflowing days.

- [ ] **Step 4: Run to verify it passes**

```bash
npm test -- --run src/components/calendar/components/month-overflow-popover.test.tsx
npm run build
```

Expected: PASS, 10 tests. Build exits 0.

- [ ] **Step 5: Export and commit**

Add `export { MonthOverflowPopover } from "./month-overflow-popover";` to `src/components/calendar/components/index.ts`.

```bash
npm run build
git add src/components/calendar/components
git commit -m "feat(calendar): add month overflow popover"
```

---

## Task 7: Month day cell

**Files:**
- Create: `src/components/calendar/components/month-day-cell.tsx`
- Create: `src/components/calendar/components/month-day-cell.test.tsx`
- Modify: `src/components/calendar/components/index.ts`

- [ ] **Step 1: Write the failing test**

Create `src/components/calendar/components/month-day-cell.test.tsx`:

```tsx
import userEvent from "@testing-library/user-event";
import { describe, expect, it, vi } from "vitest";
import { createTestEvent, testMembers } from "@/test/fixtures";
import { render, screen, seedFamilyStore } from "@/test/test-utils";
import { MonthDayCell } from "./month-day-cell";

const date = new Date(2026, 2, 8);

function renderCell(overrides: Partial<Parameters<typeof MonthDayCell>[0]> = {}) {
  seedFamilyStore({ name: "Test Family", members: testMembers });
  const props = {
    date,
    visibleMonthName: "March",
    columnIndex: 0,
    plan: { slots: [], overflowCount: 0 },
    allEvents: [],
    memberColors: [],
    memberNames: [],
    isToday: false,
    isFocused: false,
    isOutsideMonth: false,
    isWeekend: false,
    rowHeight: 124,
    onActivateDay: vi.fn(),
    onSelectDay: vi.fn(),
    onEventClick: vi.fn(),
    onFocusDay: vi.fn(),
    onKeyDown: vi.fn(),
    popoverOpen: false,
    onPopoverOpenChange: vi.fn(),
    ...overrides,
  };
  render(<MonthDayCell {...props} />);
  return props;
}

describe("MonthDayCell", () => {
  it("is one gridcell target with no nested controls while closed", () => {
    renderCell();
    const cell = screen.getByRole("gridcell");
    expect(cell).toBeInTheDocument();
    expect(cell.tagName).not.toBe("BUTTON");
    expect(screen.queryByRole("button")).not.toBeInTheDocument();
  });

  it("names the day with its event count", () => {
    renderCell({
      plan: {
        slots: [
          {
            kind: "event",
            edge: "solo",
            event: createTestEvent({ id: "a", title: "Soccer", date }),
          },
        ],
        overflowCount: 0,
      },
      allEvents: [createTestEvent({ id: "a", title: "Soccer", date })],
    });
    expect(screen.getByRole("gridcell")).toHaveAccessibleName(
      /March 8, 2026, 1 event/i,
    );
  });

  it("names an empty day as having no events", () => {
    renderCell();
    expect(screen.getByRole("gridcell")).toHaveAccessibleName(
      /March 8, 2026, no events/i,
    );
  });

  it("announces days outside the visible month", () => {
    renderCell({ date: new Date(2026, 1, 26), isOutsideMonth: true });
    const cell = screen.getByRole("gridcell");
    expect(cell).toHaveAccessibleName(
      "February 26, 2026, no events, outside March",
    );
    expect(cell.className).not.toMatch(/opacity/);
    expect(screen.getByText("26")).toHaveClass("text-muted-foreground");
  });

  it("marks today with aria-current", () => {
    renderCell({ isToday: true });
    expect(screen.getByRole("gridcell")).toHaveAttribute(
      "aria-current",
      "date",
    );
  });

  it("carries the roving tabindex only when focused", () => {
    renderCell({ isFocused: true });
    expect(screen.getByRole("gridcell")).toHaveAttribute("tabindex", "0");
  });

  it("takes itself out of the tab order when not focused", () => {
    renderCell();
    expect(screen.getByRole("gridcell")).toHaveAttribute("tabindex", "-1");
  });

  it("summarises members for screen readers instead of relying on title", () => {
    // Colours must be real FamilyColor values ("coral" | "teal" | "green" |
    // "purple" | "yellow" | "pink" | "orange") and names must match the repo's
    // fixtures (John / Jane / Alex). Note tsconfig.app.json excludes test
    // files, so `npm run build` would NOT catch an invalid colour here — an
    // unknown key silently yields `colorMap[key]?.bg === undefined`.
    renderCell({
      memberColors: ["coral", "teal"],
      memberNames: ["John", "Jane"],
    });
    expect(screen.getByText("John and Jane have events")).toBeInTheDocument();
    expect(screen.getByRole("gridcell")).toHaveAccessibleDescription(
      "John and Jane have events",
    );
  });

  it("uses singular phrasing for one member", () => {
    renderCell({ memberColors: ["coral"], memberNames: ["John"] });
    expect(screen.getByText("John has events")).toBeInTheDocument();
  });

  it("renders a blank placeholder for a reserved slot", () => {
    renderCell({
      plan: { slots: [{ kind: "blank" }], overflowCount: 0 },
    });
    expect(screen.getByTestId("month-slot-blank")).toHaveStyle({
      height: "28px",
      minHeight: "28px",
    });
  });

  it("renders +N as a fixed visual slot, not a second control", () => {
    renderCell({
      plan: { slots: [], overflowCount: 3 },
      allEvents: [createTestEvent({ id: "a", title: "Soccer", date })],
    });

    expect(screen.queryByRole("button", { name: /show all/i })).not.toBeInTheDocument();
    expect(screen.getByTestId("month-overflow-summary")).toHaveStyle({
      height: "28px",
      minHeight: "28px",
    });
  });

  it("delegates the entire cell target to the parent activation policy", async () => {
    const user = userEvent.setup();
    const props = renderCell({
      allEvents: [createTestEvent({ id: "a", title: "Soccer", date })],
    });

    await user.click(screen.getByRole("gridcell"));
    expect(props.onActivateDay).toHaveBeenCalledWith(date);
    expect(props.onFocusDay).toHaveBeenCalledWith(date);
    expect(screen.getByRole("gridcell")).toHaveFocus();
  });

  it("keeps the selected ring visible when the focused day is also today", () => {
    // Spec 4.7 requires today and selected to be separable when they are the
    // same day, so the selected ring must not be suppressed by isToday.
    renderCell({ isToday: true, isFocused: true });
    const cell = screen.getByRole("gridcell");
    expect(cell.className).toMatch(/ring-primary/); // today
    expect(cell.className).toMatch(/outline-ring/); // selected
  });
});
```

- [ ] **Step 2: Run to verify it fails**

```bash
npm test -- --run src/components/calendar/components/month-day-cell.test.tsx
```

Expected: FAIL — cannot resolve `./month-day-cell`.

- [ ] **Step 3: Implement**

Create `src/components/calendar/components/month-day-cell.tsx`:

```tsx
import { format } from "date-fns";
import type React from "react";
import { useRef } from "react";
import { type CalendarEvent, colorMap, type FamilyColor } from "@/lib/types";
import { cn } from "@/lib/utils";
import { formatLocalDate, getEventKey } from "@/lib/time-utils";
import {
  MONTH_CELL_PADDING_X,
  MONTH_CELL_PADDING_Y,
  MONTH_CHIP_GAP,
  MONTH_CHIP_HEIGHT,
  MONTH_HEADER_SLOT_GAP,
  MONTH_NUMERAL_BLOCK,
} from "../utils/month-capacity";
import type { MonthCellPlan } from "../utils/month-slots";
import { MonthEventChip } from "./month-event-chip";
import { MonthOverflowPopover } from "./month-overflow-popover";

interface MonthDayCellProps {
  date: Date;
  /** Visible grid month, used by adjacent-day accessible names. */
  visibleMonthName: string;
  /** Sunday=0 through Saturday=6; suppresses outward weld bleed. */
  columnIndex: number;
  plan: MonthCellPlan;
  /** Every event on this day, for the popover and the accessible name. */
  allEvents: CalendarEvent[];
  memberColors: FamilyColor[];
  memberNames: string[];
  isToday: boolean;
  isFocused: boolean;
  isOutsideMonth: boolean;
  isWeekend: boolean;
  rowHeight: number;
  /** Parent policy: populated day opens popover; empty day opens Day view. */
  onActivateDay: (date: Date) => void;
  /** Explicit action inside the popover. */
  onSelectDay: (date: Date) => void;
  onEventClick: (event: CalendarEvent) => void;
  onFocusDay: (date: Date) => void;
  onKeyDown: (event: React.KeyboardEvent<HTMLDivElement>, date: Date) => void;
  popoverOpen: boolean;
  onPopoverOpenChange: (open: boolean) => void;
}

function memberSummary(names: string[]): string {
  if (names.length === 1) return `${names[0]} has events`;
  const joined = `${names.slice(0, -1).join(", ")} and ${names[names.length - 1]}`;
  return `${joined} have events`;
}

export function MonthDayCell({
  date,
  visibleMonthName,
  columnIndex,
  plan,
  allEvents,
  memberColors,
  memberNames,
  isToday,
  isFocused,
  isOutsideMonth,
  isWeekend,
  rowHeight,
  onActivateDay,
  onSelectDay,
  onEventClick,
  onFocusDay,
  onKeyDown,
  popoverOpen,
  onPopoverOpenChange,
}: MonthDayCellProps) {
  const cellRef = useRef<HTMLDivElement>(null);

  const eventCount = allEvents.length;
  const countLabel =
    eventCount === 0
      ? "no events"
      : `${eventCount} ${eventCount === 1 ? "event" : "events"}`;
  const outsideLabel = isOutsideMonth
    ? `, outside ${visibleMonthName}`
    : "";
  const memberSummaryId =
    memberNames.length > 0
      ? `month-members-${formatLocalDate(date)}`
      : undefined;

  const cell = (
    <div
      ref={cellRef}
      // One >=44px target. Dense chip/+N children are presentational.
      role="gridcell"
      tabIndex={isFocused ? 0 : -1}
      aria-current={isToday ? "date" : undefined}
      aria-haspopup={allEvents.length > 0 ? "dialog" : undefined}
      aria-label={`${format(date, "MMMM d, yyyy")}, ${countLabel}${outsideLabel}`}
      aria-describedby={memberSummaryId}
      data-date={formatLocalDate(date)}
      onClick={() => {
        cellRef.current?.focus();
        onFocusDay(date);
        onActivateDay(date);
      }}
      onFocus={() => onFocusDay(date)}
      onKeyDown={(event) => onKeyDown(event, date)}
      className={cn(
        "box-border flex cursor-pointer flex-col overflow-visible rounded-lg border transition-colors motion-reduce:transition-none",
        "focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary focus-visible:ring-offset-1",
        isWeekend ? "bg-muted/40" : "bg-card",
        isOutsideMonth && "bg-muted/20",
        isToday && "border-primary bg-primary/10",
        !isToday && isOutsideMonth && "border-border/30 hover:bg-muted/30",
        !isToday && !isOutsideMonth && "border-border/50 hover:bg-accent/50",
        // Today's own ring, and the selected ring, must both be able to show
        // on the same cell — spec 4.7 requires them separable when they
        // coincide. Today uses an inset ring, selected an offset outline.
        isToday && "ring-1 ring-primary ring-inset",
        isFocused && "outline outline-2 outline-offset-[-3px] outline-ring",
      )}
      style={{
        height: rowHeight,
        paddingTop: MONTH_CELL_PADDING_Y / 2,
        paddingBottom: MONTH_CELL_PADDING_Y / 2,
        paddingLeft: MONTH_CELL_PADDING_X,
        paddingRight: MONTH_CELL_PADDING_X,
        gap: MONTH_HEADER_SLOT_GAP,
      }}
    >
      <div
        className="flex shrink-0 items-center justify-between"
        style={{ height: MONTH_NUMERAL_BLOCK, minHeight: MONTH_NUMERAL_BLOCK }}
      >
        <span
          className={cn(
            "text-sm font-semibold tabular-nums",
            isOutsideMonth && "text-muted-foreground",
            isToday && "text-primary",
          )}
        >
          {date.getDate()}
        </span>
        {memberColors.length > 0 && (
          <span className="flex items-center gap-0.5">
            {memberColors.slice(0, 4).map((color, index) => (
              <span
                key={`${color}-${index}`}
                aria-hidden="true"
                className={cn("size-1.5 rounded-full", colorMap[color]?.bg)}
              />
            ))}
            <span id={memberSummaryId} className="sr-only">
              {memberSummary(memberNames)}
            </span>
          </span>
        )}
      </div>

      <div
        className="pointer-events-none flex min-h-0 flex-col"
        style={{ gap: MONTH_CHIP_GAP }}
      >
        {plan.slots.map((slot, index) =>
          slot.kind === "blank" || !slot.event ? (
            <div
              // biome-ignore lint/suspicious/noArrayIndexKey: blanks are positional
              key={`blank-${index}`}
              data-testid="month-slot-blank"
              aria-hidden="true"
              className="shrink-0"
              style={{
                height: MONTH_CHIP_HEIGHT,
                minHeight: MONTH_CHIP_HEIGHT,
              }}
            />
          ) : (
            <MonthEventChip
              key={getEventKey(slot.event)}
              event={slot.event}
              edge={slot.edge ?? "solo"}
              weldLeft={
                columnIndex > 0 &&
                (slot.edge === "middle" || slot.edge === "end")
              }
              weldRight={
                columnIndex < 6 &&
                (slot.edge === "middle" || slot.edge === "start")
              }
            />
          ),
        )}

        {plan.overflowCount > 0 && (
          <div
            data-testid="month-overflow-summary"
            aria-hidden="true"
            className="w-full shrink-0 rounded px-1.5 text-left text-sm font-semibold text-primary"
            style={{
              height: MONTH_CHIP_HEIGHT,
              minHeight: MONTH_CHIP_HEIGHT,
            }}
          >
            +{plan.overflowCount} more
          </div>
        )}
      </div>
    </div>
  );

  if (allEvents.length === 0) return cell;

  return (
    <MonthOverflowPopover
      date={date}
      events={allEvents}
      open={popoverOpen}
      onOpenChange={onPopoverOpenChange}
      onEventClick={onEventClick}
      onOpenDay={onSelectDay}
      onCloseFocus={() => cellRef.current?.focus()}
    >
      {cell}
    </MonthOverflowPopover>
  );
}
```

`PopoverAnchor asChild` clones the cell and merges refs, so `cellRef` keeps working for focus return. Mounting the popover on every day with events — not only overflowing ones — is what makes `Enter` work uniformly (spec Section 4.4).

- [ ] **Step 4: Run to verify it passes**

```bash
npm test -- --run src/components/calendar/components/month-day-cell.test.tsx
npm run build
```

Expected: PASS, 13 tests. Build exits 0.

- [ ] **Step 5: Export and commit**

Add `export { MonthDayCell } from "./month-day-cell";` to the components barrel.

```bash
npm run build
git add src/components/calendar/components
git commit -m "feat(calendar): add month day cell with gridcell semantics"
```

---

## Task 8: Month grid — sizing, ARIA, keyboard

Rewrite `MonthlyCalendar` so that at `lg+` it renders the ARIA grid with measured row heights, and below `lg` it renders exactly what it renders today.

**Files:**
- Modify: `src/components/calendar/views/monthly-calendar.tsx`
- Create: `src/components/calendar/views/monthly-calendar.test.tsx`

- [ ] **Step 1: Preserve the tablet path**

Extract the current JSX body (everything returned today) into a local component `MonthlyCalendarCompact` in the same file, taking the same props. Do not change its markup, classes, or the `eventsToShow` value of 3.

Replace `eventsToShow` with the literal `3` and **delete the `useIsMobile` import as well as the call** — `tsconfig.app.json` sets `"noUnusedLocals": true`, so leaving an unused import fails `npm run build`. The `isMobile` branch is dead code because this component never renders at or below 768px.

- [ ] **Step 2: Write the large-screen integration tests**

Create `src/components/calendar/views/monthly-calendar.test.tsx`:

```tsx
import userEvent from "@testing-library/user-event";
import { fireEvent } from "@testing-library/react";
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import type { CalendarEvent } from "@/lib/types";
import { createTestEvent, testMembers } from "@/test/fixtures";
import {
  render,
  resetViewportWidth,
  screen,
  seedFamilyStore,
  setViewportWidth,
} from "@/test/test-utils";
import { MonthlyCalendar } from "./monthly-calendar";

const filter = {
  selectedMembers: testMembers.map((member) => member.id),
  showAllDayEvents: true,
};

function setup(
  currentDate = new Date(2026, 2, 8),
  suppliedEvents?: CalendarEvent[],
) {
  const events = suppliedEvents ?? [
    createTestEvent({
      id: "soccer",
      title: "Soccer",
      date: new Date(2026, 2, 8),
      memberId: testMembers[0].id,
    }),
  ];
  const onDateSelect = vi.fn();
  const onMonthChange = vi.fn();
  render(
    <MonthlyCalendar
      events={events}
      currentDate={currentDate}
      filter={filter}
      onDateSelect={onDateSelect}
      onMonthChange={onMonthChange}
      onEventClick={vi.fn()}
    />,
  );
  return { onDateSelect, onMonthChange };
}

describe("MonthlyCalendar large screen", () => {
  beforeEach(() => {
    setViewportWidth(1280);
    seedFamilyStore({ name: "Test Family", members: testMembers });
  });
  afterEach(resetViewportWidth);

  it("owns only a weekday row and an observed weeks rowgroup", () => {
    setup();
    const grid = screen.getByRole("grid");
    expect(Array.from(grid.children).map((child) => child.getAttribute("role")))
      .toEqual(["row", "rowgroup"]);
    expect(grid.querySelectorAll('[role="rowgroup"] > [role="row"]'))
      .toHaveLength(5);
    expect(screen.getAllByRole("columnheader")).toHaveLength(7);
  });

  it("has exactly one roving gridcell tab stop and Tab leaves the grid", async () => {
    const user = userEvent.setup();
    const crowded = Array.from({ length: 12 }, (_, index) =>
      createTestEvent({
        id: `crowded-${index}`,
        title: `Crowded ${index}`,
        date: new Date(2026, 2, index < 6 ? 8 : 9),
        memberId: testMembers[index % testMembers.length].id,
      }),
    );
    setup(new Date(2026, 2, 8), crowded);
    const cells = screen.getAllByRole("gridcell");
    expect(cells.filter((cell) => cell.tabIndex === 0)).toHaveLength(1);
    expect(screen.getAllByTestId("month-overflow-summary").length)
      .toBeGreaterThanOrEqual(2);
    expect(screen.queryAllByRole("button", { name: /show all/i })).toHaveLength(0);
    await user.tab();
    expect(document.activeElement).toHaveAttribute("role", "gridcell");
    await user.tab();
    expect(document.activeElement).not.toHaveAttribute("role", "gridcell");
  });

  it("opens the popover for a populated day and Day view for an empty day", async () => {
    const user = userEvent.setup();
    const { onDateSelect } = setup();
    await user.click(screen.getByRole("gridcell", { name: /March 8, 2026, 1 event/i }));
    expect(screen.getByRole("dialog", { name: /events for march 8/i }))
      .toBeInTheDocument();
    await user.keyboard("{Escape}");
    await user.click(screen.getByRole("gridcell", { name: /March 9, 2026, no events/i }));
    expect(onDateSelect).toHaveBeenCalledWith(new Date(2026, 2, 9));
  });

  it("moves by exact day/week boundaries with arrows, Home and End", async () => {
    const user = userEvent.setup();
    setup();
    const start = screen.getByRole("gridcell", { name: /March 8, 2026/ });
    start.focus();
    await user.keyboard("{ArrowRight}");
    expect(document.activeElement).toHaveAttribute("data-date", "2026-03-09");
    await user.keyboard("{End}");
    expect(document.activeElement).toHaveAttribute("data-date", "2026-03-14");
    await user.keyboard("{Home}");
    expect(document.activeElement).toHaveAttribute("data-date", "2026-03-08");
    await user.keyboard("{ArrowDown}");
    expect(document.activeElement).toHaveAttribute("data-date", "2026-03-15");
    await user.keyboard("{ArrowUp}");
    expect(document.activeElement).toHaveAttribute("data-date", "2026-03-08");
    await user.keyboard("{ArrowLeft}");
    expect(document.activeElement).toHaveAttribute("data-date", "2026-03-07");
  });

  it("keeps ArrowRight on an adjacent-month cell already owned by the grid", async () => {
    const user = userEvent.setup();
    const { onMonthChange } = setup(new Date(2026, 2, 31));
    const cell = screen.getByRole("gridcell", { name: /March 31, 2026/ });
    cell.focus();

    await user.keyboard("{ArrowRight}");

    expect(document.activeElement).toHaveAttribute("data-date", "2026-04-01");
    expect(onMonthChange).not.toHaveBeenCalled();
  });

  it("forces PageDown to change month even when April 1 is already in the matrix", () => {
    const { onMonthChange } = setup(new Date(2026, 2, 1));
    const cell = screen.getByRole("gridcell", { name: /March 1, 2026/ });
    cell.focus();
    fireEvent.keyDown(cell, { key: "PageDown" });
    expect(onMonthChange).toHaveBeenCalledWith(new Date(2026, 3, 1));
  });

  it("forces PageUp to change to the previous visible month", () => {
    const { onMonthChange } = setup(new Date(2026, 3, 1));
    const cell = screen.getByRole("gridcell", { name: /April 1, 2026/ });
    cell.focus();
    fireEvent.keyDown(cell, { key: "PageUp" });
    expect(onMonthChange).toHaveBeenCalledWith(new Date(2026, 2, 1));
  });

  it("treats Space like Enter and prevents page scrolling", () => {
    setup();
    const cell = screen.getByRole("gridcell", { name: /March 8, 2026/ });
    expect(fireEvent.keyDown(cell, { key: " " })).toBe(false);
    expect(screen.getByRole("dialog", { name: /events for march 8/i }))
      .toBeInTheDocument();
  });

  it("routes Enter and Space on an empty day to Day view", () => {
    const { onDateSelect } = setup();
    const cell = screen.getByRole("gridcell", {
      name: /March 9, 2026, no events/i,
    });
    expect(fireEvent.keyDown(cell, { key: "Enter" })).toBe(false);
    expect(fireEvent.keyDown(cell, { key: " " })).toBe(false);
    expect(onDateSelect).toHaveBeenNthCalledWith(1, new Date(2026, 2, 9));
    expect(onDateSelect).toHaveBeenNthCalledWith(2, new Date(2026, 2, 9));
  });

  it("renders and opens events on a leading adjacent-month day", () => {
    const adjacent = createTestEvent({
      id: "march-adjacent",
      title: "March handoff",
      date: new Date(2026, 2, 31),
      memberId: testMembers[0].id,
    });
    setup(new Date(2026, 3, 15), [adjacent]);
    const cell = screen.getByRole("gridcell", {
      name: "March 31, 2026, 1 event, outside April",
    });
    expect(cell.className).not.toMatch(/opacity/);
    fireEvent.keyDown(cell, { key: "Enter" });
    expect(
      screen.getByRole("button", { name: /March handoff/i }),
    ).toBeInTheDocument();
  });
});
```

- [ ] **Step 3: Run to verify it fails**

```bash
npm test -- --run src/components/calendar/views/monthly-calendar.test.tsx
```

Expected: FAIL because the shipped component has no lg+ grid semantics.

- [ ] **Step 4: Replace the import section with the merged imports**

Do not append imports to the shipped block: the compact extraction still uses
the baseline symbols, while several large-branch symbols come from the same
modules. Replace the file's import section with this complete merged result:

```tsx
import {
  addDays,
  addMonths,
  endOfWeek,
  format,
  startOfWeek,
  subMonths,
} from "date-fns";
import type React from "react";
import { useEffect, useMemo, useRef, useState } from "react";
import { useFamilyMembers } from "@/api";
import { useIsLargeScreen } from "@/hooks";
import {
  compareEventsAllDayFirst,
  formatLocalDate,
  getEventKey,
  isEventOnDate,
} from "@/lib/time-utils";
import {
  type CalendarEvent,
  colorMap,
  type FamilyMember,
  getFamilyMember,
} from "@/lib/types";
import { cn } from "@/lib/utils";
import { GoogleBadge, isGoogleEvent } from "../components/calendar-event";
import type { FilterState } from "../components/calendar-filter";
import { MonthDayCell } from "../components/month-day-cell";
import {
  MONTH_COLUMN_GAP,
  MONTH_MIN_ROW_HEIGHT,
  MONTH_ROW_GAP,
  monthRowHeight,
  monthSlotCapacity,
} from "../utils/month-capacity";
import {
  buildMonthMatrix,
  selectMonthDayMembers,
} from "../utils/month-matrix";
import { isMultiDay, orderRowMultiDay, planCellSlots } from "../utils/month-slots";
```

- [ ] **Step 5: Add the large-screen grid**

```tsx
export function MonthlyCalendar(props: MonthlyCalendarProps) {
  const isLargeScreen = useIsLargeScreen();
  if (!isLargeScreen) return <MonthlyCalendarCompact {...props} />;
  return <MonthlyCalendarLarge {...props} />;
}
```

The fragments below explain the measured layout and focus mechanics, but are
not independent paste targets. Implementers must use the complete authoritative
`MonthlyCalendarLarge` body at the end of Step 6 so declarations, hooks and the
return stay in one valid function.

In `MonthlyCalendarLarge`:

```tsx
const familyMembers = useFamilyMembers();

const visibleEvents = useMemo(
  () =>
    events.filter(
      (event) =>
        filter.selectedMembers.includes(event.memberId) &&
        (filter.showAllDayEvents || !event.isAllDay),
    ),
  [events, filter.selectedMembers, filter.showAllDayEvents],
);

const days = useMemo(() => buildMonthMatrix(currentDate), [currentDate]);
const weekCount = days.length / 7;
const weeks = useMemo(
  () =>
    Array.from({ length: weekCount }, (_, i) => days.slice(i * 7, i * 7 + 7)),
  [days, weekCount],
);
const dayMembers = useMemo(
  () => selectMonthDayMembers(visibleEvents, familyMembers),
  [visibleEvents, familyMembers],
);

// Callback ref, not useRef. The weeks container unmounts while the loading
// skeleton is shown, and `weekCount` does not change when it remounts — an
// effect keyed on [weekCount] with a useRef would never attach the observer,
// leaving rowHeight pinned at the floor forever. A callback ref re-runs the
// effect on every mount and unmount.
const [weeksEl, setWeeksEl] = useState<HTMLDivElement | null>(null);
const [rowHeight, setRowHeight] = useState(MONTH_MIN_ROW_HEIGHT);

useEffect(() => {
  if (!weeksEl) return;
  const observer = new ResizeObserver((entries) => {
    const height = entries[0]?.contentRect.height ?? 0;
    const next = monthRowHeight(height, weekCount, MONTH_ROW_GAP);
    // Write only on change so observation cannot re-trigger itself.
    setRowHeight((previous) => (previous === next ? previous : next));
  });
  observer.observe(weeksEl);
  return () => observer.disconnect();
}, [weeksEl, weekCount]);

const capacity = monthSlotCapacity(rowHeight);
```

Structure. Note that the observer is attached to the **weeks container**, which excludes the weekday header — `monthRowHeight` must not be handed a height that includes chrome it does not lay out:

```tsx
<div role="grid" aria-label={format(currentDate, "MMMM yyyy")}
     className="flex min-h-0 flex-1 flex-col overflow-x-clip overflow-y-auto p-4">
  <div role="row" className="grid shrink-0 grid-cols-7"
       style={{ columnGap: MONTH_COLUMN_GAP }}>
    {WEEKDAY_LABELS.map((label) => (
      <div key={label} role="columnheader"
           className="py-1 text-center text-sm font-medium text-muted-foreground">
        {label}
      </div>
    ))}
  </div>

  <div ref={setWeeksEl} role="rowgroup"
       className="flex min-h-0 flex-1 flex-col"
       style={{ gap: MONTH_ROW_GAP }}>
    {weeks.map((week) => {
      const rowMultiDay = orderRowMultiDay(visibleEvents, week);
      return (
        <div key={formatLocalDate(week[0])} role="row"
             className="grid shrink-0 grid-cols-7"
             style={{ columnGap: MONTH_COLUMN_GAP }}>
          {week.map((day, columnIndex) => {
            const dayEvents = visibleEvents
              .filter((event) => isEventOnDate(event, day))
              .sort(compareEventsAllDayFirst);
            const singleDayEvents = dayEvents.filter(
              (event) => !isMultiDay(event),
            );
            const plan = planCellSlots({
              rowMultiDay,
              day,
              singleDayEvents,
              capacity,
            });
            const members = dayMembers.get(day.toDateString()) ?? [];

            return (
              <MonthDayCell
                key={formatLocalDate(day)}
                date={day}
                visibleMonthName={format(currentDate, "MMMM")}
                columnIndex={columnIndex}
                plan={plan}
                allEvents={dayEvents}
                memberColors={members.map((member) => member.color)}
                memberNames={members.map((member) => member.name)}
                isToday={day.toDateString() === new Date().toDateString()}
                isFocused={
                  formatLocalDate(day) === formatLocalDate(focusedDate)
                }
                isOutsideMonth={day.getMonth() !== currentDate.getMonth()}
                isWeekend={day.getDay() === 0 || day.getDay() === 6}
                rowHeight={rowHeight}
                onActivateDay={handleActivateDay}
                onSelectDay={handleSelectDay}
                onEventClick={(event) => handleEventClick(event, day)}
                onFocusDay={setFocusedDate}
                onKeyDown={handleKeyDown}
                popoverOpen={
                  openPopoverDate !== null &&
                  formatLocalDate(openPopoverDate) === formatLocalDate(day)
                }
                onPopoverOpenChange={(open) =>
                  setOpenPopoverDate(open ? day : null)
                }
              />
            );
          })}
        </div>
      );
    })}
  </div>
</div>
```

Define `const WEEKDAY_LABELS = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"] as const;` at module scope.

`onDateSelect` is optional on `MonthlyCalendarProps` but `onSelectDay` is required on `MonthDayCell`, so wrap rather than passing through directly:

```tsx
const handleSelectDay = (date: Date) => onDateSelect?.(date);
const pendingModalReturnDate = useRef<Date | null>(null);
const handleEventClick = (event: CalendarEvent, originDate: Date) => {
  pendingModalReturnDate.current = originDate;
  onEventClick?.(event);
};
const handleActivateDay = (date: Date) => {
  const hasEvents = visibleEvents.some((event) => isEventOnDate(event, date));
  if (hasEvents) setOpenPopoverDate(date);
  else handleSelectDay(date);
};

const wasDetailOpen = useRef(isEventDetailOpen);
useEffect(() => {
  const justClosed = wasDetailOpen.current && !isEventDetailOpen;
  wasDetailOpen.current = isEventDetailOpen;
  if (!justClosed || !pendingModalReturnDate.current) return;
  const target = weeksEl?.querySelector<HTMLElement>(
    `[data-date="${formatLocalDate(pendingModalReturnDate.current)}"]`,
  );
  if (!target) return;
  target.focus();
  pendingModalReturnDate.current = null;
}, [isEventDetailOpen, weeksEl]);
```

- [ ] **Step 6: Add roving tabindex and keyboard navigation**

```tsx
const [focusedDate, setFocusedDate] = useState<Date>(currentDate);
const [openPopoverDate, setOpenPopoverDate] = useState<Date | null>(null);
const pendingFocus = useRef(false);
const pendingVisibleMonth = useRef<string | null>(null);

// Toolbar Previous/Next is not a keyboard edge traversal: reset the roving
// cell to the parent's new date. Edge traversal sets pendingFocus first and
// preserves its exact landed-on date through the re-render.
useEffect(() => {
  if (pendingFocus.current) return;
  setFocusedDate(currentDate);
  setOpenPopoverDate(null);
}, [currentDate]);

// Restore focus after a month change re-renders the grid, so crossing a grid
// edge does not dump focus back to the first cell.
useEffect(() => {
  if (!pendingFocus.current) return;
  if (
    pendingVisibleMonth.current !== null &&
    pendingVisibleMonth.current !== format(currentDate, "yyyy-MM")
  ) {
    return;
  }
  const target = weeksEl?.querySelector<HTMLElement>(
    `[data-date="${formatLocalDate(focusedDate)}"]`,
  );
  // The state update can render once against the old matrix before the parent
  // month change lands. Do not clear the request until the target exists.
  if (!target) return;
  target.focus();
  pendingFocus.current = false;
  pendingVisibleMonth.current = null;
}, [currentDate, days, focusedDate, weeksEl]);

// Membership in the rendered matrix, not month equality. March 2026 renders
// Apr 1-4 as trailing cells; arrowing onto one of those must move focus within
// the existing grid, not re-page the whole month — otherwise the adjacent-month
// events fetched in Task 9 are unreachable by keyboard.
const matrixKeys = useMemo(
  () => new Set(days.map((day) => formatLocalDate(day))),
  [days],
);

const moveFocus = (
  next: Date,
  forceMonthChange = !matrixKeys.has(formatLocalDate(next)),
) => {
  pendingFocus.current = true;
  pendingVisibleMonth.current = forceMonthChange
    ? format(next, "yyyy-MM")
    : null;
  setFocusedDate(next);
  if (forceMonthChange) onMonthChange?.(next);
};

const DAY_OFFSET: Record<string, number> = {
  ArrowLeft: -1,
  ArrowRight: 1,
  ArrowUp: -7,
  ArrowDown: 7,
};

const handleKeyDown = (
  event: React.KeyboardEvent<HTMLDivElement>,
  date: Date,
) => {
  if (event.key === "Enter" || event.key === " ") {
    event.preventDefault();
    // Any day with events opens the popover — not only overflowing days.
    // The popover is the keyboard path to every event, so gating it on
    // overflow would make Enter dead on a day with one event.
    handleActivateDay(date);
    return;
  }

  const offset = DAY_OFFSET[event.key];
  if (offset !== undefined) {
    event.preventDefault();
    moveFocus(addDays(date, offset));
    return;
  }

  if (event.key === "Home") {
    event.preventDefault();
    moveFocus(startOfWeek(date, { weekStartsOn: 0 }));
  } else if (event.key === "End") {
    event.preventDefault();
    moveFocus(endOfWeek(date, { weekStartsOn: 0 }));
  } else if (event.key === "PageUp") {
    event.preventDefault();
    moveFocus(subMonths(date, 1), true);
  } else if (event.key === "PageDown") {
    event.preventDefault();
    moveFocus(addMonths(date, 1), true);
  }
};
```

Add `onMonthChange?: (date: Date) => void` and
`isEventDetailOpen?: boolean` to `MonthlyCalendarProps`; default
`isEventDetailOpen = false` in the large component. The latter is the signal
that lets popover event selection transfer focus into the modal and restore the
originating cell only when that modal closes.

Use this complete body as the authoritative result of Steps 5-6. Do not paste
the explanatory fragments above in addition to it. Task 10 extends this same
function with query-state props and branches; it does not create a second large
component.

```tsx
const WEEKDAY_LABELS = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"] as const;

const DAY_OFFSET: Record<string, number> = {
  ArrowLeft: -1,
  ArrowRight: 1,
  ArrowUp: -7,
  ArrowDown: 7,
};

function MonthlyCalendarLarge({
  events,
  currentDate,
  onEventClick,
  filter,
  onDateSelect,
  onMonthChange,
  isEventDetailOpen = false,
}: MonthlyCalendarProps) {
  const familyMembers = useFamilyMembers();
  const visibleEvents = useMemo(
    () =>
      events.filter(
        (event) =>
          filter.selectedMembers.includes(event.memberId) &&
          (filter.showAllDayEvents || !event.isAllDay),
      ),
    [events, filter.selectedMembers, filter.showAllDayEvents],
  );
  const days = useMemo(() => buildMonthMatrix(currentDate), [currentDate]);
  const weekCount = days.length / 7;
  const weeks = useMemo(
    () =>
      Array.from({ length: weekCount }, (_, index) =>
        days.slice(index * 7, index * 7 + 7),
      ),
    [days, weekCount],
  );
  const dayMembers = useMemo(
    () => selectMonthDayMembers(visibleEvents, familyMembers),
    [visibleEvents, familyMembers],
  );

  const [weeksEl, setWeeksEl] = useState<HTMLDivElement | null>(null);
  const [rowHeight, setRowHeight] = useState(MONTH_MIN_ROW_HEIGHT);
  const [focusedDate, setFocusedDate] = useState<Date>(currentDate);
  const [openPopoverDate, setOpenPopoverDate] = useState<Date | null>(null);
  const pendingFocus = useRef(false);
  const pendingVisibleMonth = useRef<string | null>(null);
  const pendingModalReturnDate = useRef<Date | null>(null);
  const wasDetailOpen = useRef(isEventDetailOpen);

  useEffect(() => {
    if (!weeksEl) return;
    const observer = new ResizeObserver((entries) => {
      const height = entries[0]?.contentRect.height ?? 0;
      const next = monthRowHeight(height, weekCount, MONTH_ROW_GAP);
      setRowHeight((previous) => (previous === next ? previous : next));
    });
    observer.observe(weeksEl);
    return () => observer.disconnect();
  }, [weeksEl, weekCount]);

  const capacity = monthSlotCapacity(rowHeight);
  const matrixKeys = useMemo(
    () => new Set(days.map((day) => formatLocalDate(day))),
    [days],
  );

  const handleSelectDay = (date: Date) => onDateSelect?.(date);

  const handleEventClick = (event: CalendarEvent, originDate: Date) => {
    if (!onEventClick) return;
    pendingModalReturnDate.current = originDate;
    onEventClick(event);
  };

  const handleActivateDay = (date: Date) => {
    const hasEvents = visibleEvents.some((event) => isEventOnDate(event, date));
    if (hasEvents) setOpenPopoverDate(date);
    else handleSelectDay(date);
  };

  const moveFocus = (
    next: Date,
    forceMonthChange = !matrixKeys.has(formatLocalDate(next)),
  ) => {
    pendingFocus.current = true;
    pendingVisibleMonth.current = forceMonthChange
      ? format(next, "yyyy-MM")
      : null;
    setFocusedDate(next);
    if (forceMonthChange) onMonthChange?.(next);
  };

  const handleKeyDown = (
    event: React.KeyboardEvent<HTMLDivElement>,
    date: Date,
  ) => {
    if (event.key === "Enter" || event.key === " ") {
      event.preventDefault();
      handleActivateDay(date);
      return;
    }

    const offset = DAY_OFFSET[event.key];
    if (offset !== undefined) {
      event.preventDefault();
      moveFocus(addDays(date, offset));
      return;
    }

    if (event.key === "Home") {
      event.preventDefault();
      moveFocus(startOfWeek(date, { weekStartsOn: 0 }));
    } else if (event.key === "End") {
      event.preventDefault();
      moveFocus(endOfWeek(date, { weekStartsOn: 0 }));
    } else if (event.key === "PageUp") {
      event.preventDefault();
      moveFocus(subMonths(date, 1), true);
    } else if (event.key === "PageDown") {
      event.preventDefault();
      moveFocus(addMonths(date, 1), true);
    }
  };

  // Toolbar paging resets the roving date. Keyboard edge paging sets the
  // pending flag first and preserves its exact destination across the render.
  useEffect(() => {
    if (pendingFocus.current) return;
    setFocusedDate(currentDate);
    setOpenPopoverDate(null);
  }, [currentDate]);

  useEffect(() => {
    if (!pendingFocus.current) return;
    if (
      pendingVisibleMonth.current !== null &&
      pendingVisibleMonth.current !== format(currentDate, "yyyy-MM")
    ) {
      return;
    }
    const target = weeksEl?.querySelector<HTMLElement>(
      `[data-date="${formatLocalDate(focusedDate)}"]`,
    );
    if (!target) return;
    target.focus();
    pendingFocus.current = false;
    pendingVisibleMonth.current = null;
  }, [currentDate, days, focusedDate, weeksEl]);

  useEffect(() => {
    const justClosed = wasDetailOpen.current && !isEventDetailOpen;
    wasDetailOpen.current = isEventDetailOpen;
    if (!justClosed || !pendingModalReturnDate.current) return;
    const target = weeksEl?.querySelector<HTMLElement>(
      `[data-date="${formatLocalDate(pendingModalReturnDate.current)}"]`,
    );
    if (!target) return;
    target.focus();
    pendingModalReturnDate.current = null;
  }, [isEventDetailOpen, weeksEl]);

  return (
    <div
      role="grid"
      aria-label={format(currentDate, "MMMM yyyy")}
      className="flex min-h-0 flex-1 flex-col overflow-x-clip overflow-y-auto p-4"
    >
      <div
        role="row"
        className="grid shrink-0 grid-cols-7"
        style={{ columnGap: MONTH_COLUMN_GAP }}
      >
        {WEEKDAY_LABELS.map((label) => (
          <div
            key={label}
            role="columnheader"
            className="py-1 text-center text-sm font-medium text-muted-foreground"
          >
            {label}
          </div>
        ))}
      </div>

      <div
        ref={setWeeksEl}
        role="rowgroup"
        className="flex min-h-0 flex-1 flex-col"
        style={{ gap: MONTH_ROW_GAP }}
      >
        {weeks.map((week) => {
          const rowMultiDay = orderRowMultiDay(visibleEvents, week);
          return (
            <div
              key={formatLocalDate(week[0])}
              role="row"
              className="grid shrink-0 grid-cols-7"
              style={{ columnGap: MONTH_COLUMN_GAP }}
            >
              {week.map((day, columnIndex) => {
                const dayEvents = visibleEvents
                  .filter((event) => isEventOnDate(event, day))
                  .sort(compareEventsAllDayFirst);
                const singleDayEvents = dayEvents.filter(
                  (event) => !isMultiDay(event),
                );
                const plan = planCellSlots({
                  rowMultiDay,
                  day,
                  singleDayEvents,
                  capacity,
                });
                const members = dayMembers.get(day.toDateString()) ?? [];

                return (
                  <MonthDayCell
                    key={formatLocalDate(day)}
                    date={day}
                    visibleMonthName={format(currentDate, "MMMM")}
                    columnIndex={columnIndex}
                    plan={plan}
                    allEvents={dayEvents}
                    memberColors={members.map((member) => member.color)}
                    memberNames={members.map((member) => member.name)}
                    isToday={
                      day.toDateString() === new Date().toDateString()
                    }
                    isFocused={
                      formatLocalDate(day) === formatLocalDate(focusedDate)
                    }
                    isOutsideMonth={
                      day.getMonth() !== currentDate.getMonth()
                    }
                    isWeekend={day.getDay() === 0 || day.getDay() === 6}
                    rowHeight={rowHeight}
                    onActivateDay={handleActivateDay}
                    onSelectDay={handleSelectDay}
                    onEventClick={(event) => handleEventClick(event, day)}
                    onFocusDay={setFocusedDate}
                    onKeyDown={handleKeyDown}
                    popoverOpen={
                      openPopoverDate !== null &&
                      formatLocalDate(openPopoverDate) === formatLocalDate(day)
                    }
                    onPopoverOpenChange={(open) =>
                      setOpenPopoverDate(open ? day : null)
                    }
                  />
                );
              })}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

- [ ] **Step 7: Typecheck and run the full suite**

```bash
npm run build
npm test -- --run
```

Expected: build exits 0; no previously-passing test regresses.

- [ ] **Step 8: Commit**

```bash
git add src/components/calendar/views/monthly-calendar.tsx src/components/calendar/views/monthly-calendar.test.tsx
git commit -m "feat(calendar): fill viewport in large-screen month grid"
```

---

## Task 9: Large-screen Month query range

**Files:**
- Modify: `src/components/calendar/calendar-module.tsx:173-203`

- [ ] **Step 0: Write and run the range regression tests first**

Add the complete `describe("Month query range")` block from Step 3 now, before
editing `dateRange`, and add its imports. Run the focused file:

```bash
npm test -- --run src/components/calendar/calendar-module.test.tsx
```

Expected: the below-lg case passes, while the lg+ April range fails because the
shipped query still requests April 1-30. Do not add the block a second time in
Step 3.

- [ ] **Step 1: Gate the widened range**

In the `dateRange` `useMemo`, change the `monthly` case and add `isLargeScreen` to the dependency array:

```tsx
case "monthly":
  if (isLargeScreen) {
    // The lg+ grid renders leading and trailing days from adjacent months,
    // so fetch the range it actually draws. Gated because dateRange is
    // module-wide and MobileMonthlyView also renders adjacent-month days —
    // an ungated change would alter mobile. See spec Section 4.6.
    start = startOfWeek(startOfMonth(currentDate), { weekStartsOn: 0 });
    end = endOfWeek(endOfMonth(currentDate), { weekStartsOn: 0 });
  } else {
    start = startOfMonth(currentDate);
    end = endOfMonth(currentDate);
  }
  break;
```

Add `startOfWeek`/`endOfWeek` to the `date-fns` import if not already present (they are, for the weekly case). Change the memo dependencies to `[currentDate, calendarView, isLargeScreen]`.

- [ ] **Step 2: Pass the month-change callback**

In `renderCalendarView`'s `monthly` case:

```tsx
case "monthly":
  return (
    <MonthlyCalendar
      {...commonProps}
      onDateSelect={selectDateAndSwitchToDaily}
      onMonthChange={setDate}
      isEventDetailOpen={isDetailModalOpen}
    />
  );
```

- [ ] **Step 3: Confirm the range regression tests are now green**

The block added in Step 0 is reproduced here as the authoritative reference:

```tsx
describe("Month query range", () => {
  const requestedRanges: { startDate: string; endDate: string }[] = [];

  beforeEach(() => {
    requestedRanges.length = 0;
    server.events.removeAllListeners("request:start");
    server.events.on("request:start", ({ request }) => {
      const url = new URL(request.url);
      if (!url.pathname.includes("/calendar/events")) return;
      requestedRanges.push({
        startDate: url.searchParams.get("startDate") ?? "",
        endDate: url.searchParams.get("endDate") ?? "",
      });
    });
    // April 2026 deliberately: Apr 1 is a Wednesday, so the grid has three
    // leading days from March and the two ranges genuinely differ. March 2026
    // would be a vacuous test — Mar 1 is itself a Sunday, so the grid needs no
    // leading days and both ranges would share a start date.
    seedCalendarStore({ currentDate: new Date(2026, 3, 15) });
    useCalendarStore.setState({ calendarView: "monthly" });
  });

  afterEach(() => {
    server.events.removeAllListeners("request:start");
    resetViewportWidth();
  });

  it("requests only the calendar month below lg", async () => {
    setViewportWidth(800);
    render(<CalendarModule />);

    await waitFor(() => expect(requestedRanges.length).toBeGreaterThan(0));
    expect(requestedRanges.at(-1)).toEqual({
      startDate: "2026-04-01",
      endDate: "2026-04-30",
    });
  });

  it("requests the full rendered grid range at lg+", async () => {
    setViewportWidth(1280);
    render(<CalendarModule />);

    await waitFor(() => expect(requestedRanges.length).toBeGreaterThan(0));
    expect(requestedRanges.at(-1)).toEqual({
      startDate: "2026-03-29",
      endDate: "2026-05-02",
    });
  });

  it("reuses the calendar-month parameters for grid-aligned February 2026", async () => {
    seedCalendarStore({ currentDate: new Date(2026, 1, 15) });
    setViewportWidth(1280);
    render(<CalendarModule />);

    await waitFor(() => expect(requestedRanges.length).toBeGreaterThan(0));
    expect(requestedRanges.at(-1)).toEqual({
      startDate: "2026-02-01",
      endDate: "2026-02-28",
    });
  });
});
```

Import `server` from the file's existing MSW setup — reuse whichever import
`calendar-module.test.tsx` already uses at the top rather than adding a new one.
`seedCalendarStore`, `setViewportWidth` and `resetViewportWidth` come from
`@/test/test-utils`.

The expected dates are derived, not guessed: Apr 1 2026 is a Wednesday, so `startOfWeek` walks back to Sun Mar 29. April has 30 days ending Thu Apr 30, so `endOfWeek` walks forward to Sat May 2. Confirm both against `buildMonthMatrix(new Date(2026, 3, 15))` in a scratch test before relying on them.

- [ ] **Step 4: Verify**

```bash
npm test -- --run src/components/calendar/calendar-module.test.tsx
npm run build
```

Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add src/components/calendar/calendar-module.tsx src/components/calendar/calendar-module.test.tsx
git commit -m "feat(calendar): fetch the rendered grid range for large-screen month"
```

---

## Task 10: Large-screen Month view states

`renderCalendarView()` currently returns the shared centred "Loading events..." / error text **before** the view switch is reached, so a per-view state inside `MonthlyCalendar` would never render. The early return has to become conditional first.

**Files:**
- Modify: `src/components/calendar/calendar-module.tsx:465-484`
- Modify: `src/components/calendar/views/monthly-calendar.tsx`
- Modify: `src/components/calendar/views/monthly-calendar.test.tsx`
- Create: `src/components/calendar/components/calendar-view-states.tsx`

- [ ] **Step 0: Write and run the query-state tests first**

Add the full `MonthlyCalendar large query states` block from Step 4 to
`monthly-calendar.test.tsx` before changing production code. Run it now:

```bash
npm test -- --run src/components/calendar/views/monthly-calendar.test.tsx
```

Expected: FAIL because the shipped props/branches and shared state components
do not exist. Step 4 is the authoritative copy of the tests; do not append it a
second time later.

- [ ] **Step 1: Make the shared fallback conditional**

In `calendar-module.tsx`, replace the unconditional early returns with ones that skip the views that own their states:

**Month only, for now.** Schedule is added to this gate in Task 15, in the same commit that gives Schedule its states. Gating it here would leave large-screen Schedule with no loading and no error state for several commits.

```tsx
// Only the lg+ compositions own their states. This gate is load-bearing:
// without `isLargeScreen`, a mobile "monthly" view would skip the shared
// loading state and render MobileMonthlyView with zero events mid-fetch.
const ownsItsStates = isLargeScreen && calendarView === "monthly";

if (isLoading && !ownsItsStates) {
  return (
    <div className="flex-1 flex items-center justify-center">
      <div className="text-muted-foreground">Loading events...</div>
    </div>
  );
}

if (isError && !ownsItsStates) {
  return (
    <div className="flex-1 flex items-center justify-center">
      <div className="text-destructive">
        Error loading events: {error?.message ?? "Unknown error"}
      </div>
    </div>
  );
}
```

Week and Day keep exactly today's behaviour.

Destructure `refetch` from the `useCalendarEvents(dateRange)` result alongside the existing fields, then pass the query state **to the `MonthlyCalendar` call site only** — not through `commonProps`:

Import `useIsRestoring` from `@tanstack/react-query` in
`calendar-module.tsx` and call `const isRestoring = useIsRestoring();` beside
the other top-level hooks. Persisted query data hydrates asynchronously; an
undefined response while this flag is true is not a cold cache.

```tsx
case "monthly":
  return (
    <MonthlyCalendar
      {...commonProps}
      onDateSelect={selectDateAndSwitchToDaily}
      onMonthChange={setDate}
      isEventDetailOpen={isDetailModalOpen}
      isLoading={isLoading}
      isError={isError}
      errorMessage={error?.message}
      onRetry={refetch}
      hasQueryData={eventsResponse !== undefined}
      isQueryRestoring={isRestoring}
    />
  );
```

`commonProps` is spread into all three `ScheduleCalendar` call sites (`calendar-module.tsx:524`, `:526`, `:552`), two of which are the mobile branch — routing state props through it would push new props onto the mobile component this story must leave untouched. Do not introduce an unused intermediate object either: `tsconfig.app.json` sets `"noUnusedLocals": true`, so a declared-but-unwired `viewStateProps` would fail `npm run build`.

Add these fields to `MonthlyCalendarProps`, and default them only in
`MonthlyCalendarLarge` so the compact extraction remains unchanged:

```tsx
interface MonthlyCalendarQueryStateProps {
  isLoading?: boolean;
  isError?: boolean;
  errorMessage?: string;
  onRetry?: () => void;
  hasQueryData?: boolean;
  isQueryRestoring?: boolean;
}

interface MonthlyCalendarProps extends MonthlyCalendarQueryStateProps {
  events: CalendarEvent[];
  currentDate: Date;
  onEventClick?: (event: CalendarEvent) => void;
  filter: FilterState;
  onDateSelect?: (date: Date) => void;
  onMonthChange?: (date: Date) => void;
  isEventDetailOpen?: boolean;
}
```

Extend the authoritative Task 8 function's existing parameter destructuring
with this narrow diff. Do not add a second component definition or replace its
body:

```diff
 function MonthlyCalendarLarge({
   events,
   currentDate,
   onEventClick,
   filter,
   onDateSelect,
   onMonthChange,
   isEventDetailOpen = false,
+  isLoading = false,
+  isError = false,
+  errorMessage,
+  onRetry,
+  hasQueryData = true,
+  isQueryRestoring = false,
 }: MonthlyCalendarProps) {
```

- [ ] **Step 2: Create the shared state components**

Create `src/components/calendar/components/calendar-view-states.tsx`:

```tsx
import { Calendar, WifiOff } from "lucide-react";

export function CalendarErrorState({
  message,
  onRetry,
}: {
  message?: string;
  onRetry?: () => void;
}) {
  return (
    <div className="flex flex-1 flex-col items-center justify-center gap-3 p-8 text-center">
      <p className="text-destructive">
        Error loading events: {message ?? "Unknown error"}
      </p>
      {onRetry && (
        <button
          type="button"
          onClick={onRetry}
          className="min-h-11 rounded-lg border border-border px-4 text-sm font-medium hover:bg-accent"
        >
          Try again
        </button>
      )}
    </div>
  );
}

export function CalendarOfflineState({ message }: { message: string }) {
  return (
    <div className="flex flex-1 flex-col items-center justify-center gap-2 p-8 text-center text-muted-foreground">
      <WifiOff aria-hidden="true" className="size-12 opacity-50" />
      <p className="text-lg font-medium">You're offline</p>
      <p className="text-sm">{message}</p>
    </div>
  );
}

export function CalendarEmptyState({
  title,
  description,
}: {
  title: string;
  description: string;
}) {
  return (
    <div className="flex flex-1 flex-col items-center justify-center gap-2 p-8 text-center text-muted-foreground">
      <Calendar aria-hidden="true" className="size-12 opacity-50" />
      <p className="text-lg font-medium">{title}</p>
      <p className="text-sm">{description}</p>
    </div>
  );
}
```

Export all three from the components barrel.

- [ ] **Step 3: Add the Month skeleton**

In `monthly-calendar.tsx`, add `useOnlineStatus` to the existing `@/hooks`
import, call `const isOnline = useOnlineStatus();` with the other hooks, and
import `CalendarErrorState` and `CalendarOfflineState` from
`../components/calendar-view-states`. `MonthGridSkeleton` stays local to this
file. Then
render in this priority order inside `MonthlyCalendarLarge`:

1. `isError` → `<CalendarErrorState message={errorMessage} onRetry={onRetry} />`
2. `isLoading || isQueryRestoring` → `<MonthGridSkeleton weekCount={weekCount} />` below. Restoration owns the surface until persisted queries finish, so the uncached message cannot flash during hydration
3. `!hasQueryData && !isOnline` → `<CalendarOfflineState message="This month isn't cached yet." />`. `events.length === 0` is **not** a cache-presence test: a successfully cached empty response is valid data and must render the normal empty grid. `useOnlineStatus()` is reactive, whereas a direct `navigator.onLine` read would not re-render when connectivity returns
4. otherwise the grid

Insert the branches after all hooks (including the observer/focus effects) and
immediately before the grid return, preserving React's hook order:

```tsx
if (isError) {
  return <CalendarErrorState message={errorMessage} onRetry={onRetry} />;
}
if (isLoading || isQueryRestoring) {
  return <MonthGridSkeleton weekCount={weekCount} />;
}
if (!hasQueryData && !isOnline) {
  return (
    <CalendarOfflineState message="This month isn't cached yet." />
  );
}
```

```tsx
function MonthGridSkeleton({
  weekCount,
}: {
  weekCount: number;
}) {
  return (
    <div
      role="status"
      aria-label="Loading month"
      className="flex min-h-0 flex-1 flex-col overflow-y-auto p-4"
      style={{ gap: MONTH_ROW_GAP }}
    >
      {Array.from({ length: weekCount }).map((_, week) => (
        // biome-ignore lint/suspicious/noArrayIndexKey: static skeleton
        <div
          key={week}
          className="grid min-h-24 flex-1 shrink-0 grid-cols-7"
          style={{ columnGap: MONTH_COLUMN_GAP }}
        >
          {Array.from({ length: 7 }).map((__, day) => (
            <div
              // biome-ignore lint/suspicious/noArrayIndexKey: static skeleton
              key={day}
              className="animate-pulse rounded-lg border border-border/50 bg-muted/40 motion-reduce:animate-none"
            />
          ))}
        </div>
      ))}
    </div>
  );
}
```

- [ ] **Step 4: Confirm cache-presence and state-priority tests are green**

The block added in Step 0 is reproduced here for reference (reuse the file's
`filter` and large-screen setup):

```tsx
describe("MonthlyCalendar large query states", () => {
  const baseProps = {
    events: [],
    currentDate: new Date(2026, 3, 15),
    filter,
    onDateSelect: vi.fn(),
    onMonthChange: vi.fn(),
  };

  function setOnline(value: boolean) {
    Object.defineProperty(navigator, "onLine", {
      configurable: true,
      value,
    });
    window.dispatchEvent(new Event(value ? "online" : "offline"));
  }

  beforeEach(() => {
    setViewportWidth(1280);
    seedFamilyStore({ name: "Test Family", members: testMembers });
  });

  afterEach(() => {
    setOnline(true);
    resetViewportWidth();
  });

  it("shows offline-cold-cache only when query data is absent", () => {
    setOnline(false);
    render(<MonthlyCalendar {...baseProps} hasQueryData={false} />);
    expect(screen.getByText("This month isn't cached yet.")).toBeInTheDocument();
  });

  it("renders a cached empty month instead of calling it uncached", () => {
    setOnline(false);
    render(<MonthlyCalendar {...baseProps} hasQueryData />);
    expect(screen.queryByText(/isn't cached/i)).not.toBeInTheDocument();
    expect(screen.getByRole("grid", { name: /april 2026/i })).toBeInTheDocument();
  });

  it("treats a cached empty grid-aligned February as valid data", () => {
    setOnline(false);
    render(
      <MonthlyCalendar
        {...baseProps}
        currentDate={new Date(2026, 1, 15)}
        hasQueryData
      />,
    );
    expect(screen.queryByText(/isn't cached/i)).not.toBeInTheDocument();
    expect(screen.getByRole("grid", { name: /february 2026/i }))
      .toBeInTheDocument();
  });

  it("shows restoration skeleton without flashing cold-cache copy", () => {
    setOnline(false);
    const { rerender } = render(
      <MonthlyCalendar
        {...baseProps}
        hasQueryData={false}
        isQueryRestoring
      />,
    );
    expect(screen.getByRole("status", { name: /loading month/i }))
      .toBeInTheDocument();
    expect(screen.queryByText(/isn't cached/i)).not.toBeInTheDocument();

    rerender(
      <MonthlyCalendar
        {...baseProps}
        hasQueryData={false}
        isQueryRestoring={false}
      />,
    );
    expect(screen.getByText("This month isn't cached yet.")).toBeInTheDocument();
  });

  it("renders loading before content", () => {
    render(<MonthlyCalendar {...baseProps} isLoading hasQueryData={false} />);
    expect(screen.getByRole("status", { name: /loading month/i }))
      .toBeInTheDocument();
  });

  it("renders error before loading and retries", async () => {
    const user = userEvent.setup();
    const onRetry = vi.fn();
    render(
      <MonthlyCalendar
        {...baseProps}
        isLoading
        isError
        errorMessage="No route"
        onRetry={onRetry}
      />,
    );
    expect(screen.queryByRole("status", { name: /loading month/i }))
      .not.toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: /try again/i }));
    expect(onRetry).toHaveBeenCalledOnce();
  });
});
```

- [ ] **Step 5: Verify and commit**

```bash
npm test -- --run && npm run build
git add src/components/calendar
git commit -m "feat(calendar): add large-screen month view states"
```

---

## Task 11: Month E2E

**Files:**
- Create: `scripts/run-calendar-e2e.sh`
- Create: `e2e/large-screen-calendar-month.spec.ts`
- Modify: `e2e/offline-persistence.spec.ts`

- [ ] **Step 1: Add a fail-safe released-backend runner**

FE E2E hits the real backend through the Vite proxy, not MSW. The compose
default `latest` is stale and 404s newer endpoints. Do not keep Compose running
across task shells or rely on exported variables surviving between agent calls.
Create `scripts/run-calendar-e2e.sh` and make it executable:

```bash
#!/usr/bin/env bash
set -euo pipefail

export GITHUB_TOKEN="${GITHUB_TOKEN:-$(gh auth token)}"
export BE_IMAGE_TAG="${BE_IMAGE_TAG:-$(.github/scripts/resolve-backend-version.sh)}"
if [[ ! "$BE_IMAGE_TAG" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
  echo "Expected a bare-semver backend tag, got: $BE_IMAGE_TAG" >&2
  exit 1
fi

compose=(docker compose -f docker-compose.e2e.yml)

cleanup() {
  local command_status=$?
  trap - EXIT INT TERM
  set +e
  "${compose[@]}" down
  local cleanup_status=$?
  if ((command_status != 0)); then exit "$command_status"; fi
  exit "$cleanup_status"
}
trap cleanup EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

"${compose[@]}" up -d --wait
"$@"
```

Expected: `BE_IMAGE_TAG` is bare semver (for example `1.9.0`, not `v1.9.0`)
and Compose reports the backend healthy. The `EXIT` trap always runs `down` in
the same shell with the same tag. It preserves the test command's failure and
also fails an otherwise-successful run when teardown fails.

```bash
chmod +x scripts/run-calendar-e2e.sh
```

- [ ] **Step 2: Write the spec**

Use the repo's existing helpers rather than raw selectors. `switchCalendarView` handles the desktop/mobile switcher difference, which matters because the view switcher is **icon-only below xl (1280px)** — a `name: /month/i` selector would not match there.

Follow the repo's canonical bootstrap exactly — `registerFamily` takes an `APIRequestContext` and only hits the backend; the browser must then be seeded and reloaded. See `e2e/calendar-navigation.spec.ts:11-27` for the pattern this mirrors.

```ts
import { addDays, addMonths, startOfMonth } from "date-fns";
import { expect, test } from "@playwright/test";
import { formatLocalDate, parseLocalDate } from "../src/lib/time-utils";
import {
  createCalendarEvent,
  registerFamily,
  seedBrowserAuth,
} from "./helpers/api-helpers";
import {
  clearStorage,
  getTodayDateString,
  switchCalendarView,
  waitForCalendarReady,
  waitForHydration,
} from "./helpers/test-helpers";

test.describe("Large-screen Calendar Month", () => {
  test.beforeEach(async ({ page, request, isMobile }) => {
    test.skip(isMobile, "Large-screen only");
    await page.setViewportSize({ width: 1280, height: 800 });
    await page.goto("/");
    await clearStorage(page);

    const reg = await registerFamily(request, {
      familyName: "Test Family",
      members: [
        { name: "Alice", color: "coral" },
        { name: "Bob", color: "teal" },
      ],
    });

    // Six events on one day guarantees overflow at any capacity this
    // viewport yields (max 5), so the popover tests have data to work with.
    // Seed through the API rather than the Add Event modal: the `createEvent`
    // UI helper accepts only { title, recurrence, allDay } and cannot set a
    // date or time.
    const today = getTodayDateString();
    for (let i = 0; i < 6; i++) {
      await createCalendarEvent(request, reg.token, {
        title: `Overflow event ${i}`,
        date: today,
        startTime: "9:00 AM",
        endTime: "10:00 AM",
        memberId: reg.family.members[0].id,
        isAllDay: false,
      });
    }

    const parsedToday = parseLocalDate(today);
    const monthStart = startOfMonth(parsedToday);

    // A fixed in-month day, kept distinct from today and the run below, proves
    // the non-overflow activation path even when today is month-end.
    const singleDate = new Date(
      monthStart.getFullYear(),
      monthStart.getMonth(),
      parsedToday.getDate() === 15 ? 16 : 15,
    );
    await createCalendarEvent(request, reg.token, {
      title: "Single event day",
      date: formatLocalDate(singleDate),
      startTime: "2:00 PM",
      endTime: "3:00 PM",
      memberId: reg.family.members[1].id,
      isAllDay: false,
    });

    // The first Friday of the displayed month through Tuesday guarantees a
    // five-segment run crossing the Sat/Sun row edge, fully inside the Month
    // matrix regardless of today's position near month-end.
    const daysUntilFriday = (5 - monthStart.getDay() + 7) % 7;
    const runStart = addDays(monthStart, daysUntilFriday);
    const runEnd = addDays(runStart, 4);
    await createCalendarEvent(request, reg.token, {
      title: "Family trip",
      date: formatLocalDate(runStart),
      endDate: formatLocalDate(runEnd),
      startTime: "12:00 AM",
      endTime: "11:59 PM",
      memberId: reg.family.members[0].id,
      isAllDay: true,
    });

    await seedBrowserAuth(page, reg);
    await page.reload();
    await waitForHydration(page);
    await waitForCalendarReady(page);
    await switchCalendarView(page, "monthly");
  });

  test("grid fills the viewport with no dead space below the last week", async ({
    page,
  }) => {
    const grid = page.getByRole("grid");
    const gridBox = await grid.boundingBox();
    const lastRow = grid.getByRole("row").last();
    const lastBox = await lastRow.boundingBox();

    expect(gridBox).not.toBeNull();
    expect(lastBox).not.toBeNull();
    const lastBottom =
      (lastBox as { y: number; height: number }).y +
      (lastBox as { height: number }).height;
    const gridBottom =
      (gridBox as { y: number; height: number }).y +
      (gridBox as { height: number }).height;
    // Two-sided: dead space and an overfull/permanently scrolling grid fail.
    expect(lastBottom).toBeGreaterThanOrEqual(gridBottom - 24);
    expect(lastBottom).toBeLessThanOrEqual(gridBottom + 2);
  });

  test("the grid is a single tab stop", async ({ page }) => {
    // Count tabbable cells directly rather than tabbing from a fixed control.
    // The "Today" button is `disabled` whenever `isViewingToday` is true
    // (calendar-navigation.tsx), so focusing it on a fresh load is a no-op and
    // a tab-from-Today test would silently start from <body>.
    const grid = page.getByRole("grid");
    await expect(grid.locator('[role="gridcell"][tabindex="0"]')).toHaveCount(1);
    await expect(
      grid.locator('[role="gridcell"]:not([tabindex="-1"]):not([tabindex="0"])'),
    ).toHaveCount(0);
    // Dense visual slots must not introduce any button at all.
    await expect(
      grid.locator('button:not([tabindex="-1"])'),
    ).toHaveCount(0);
  });

  test("Enter opens the popover on a day with events but no overflow", async ({
    page,
  }) => {
    // Guards the case where the popover is gated on overflow: a day with
    // 1..capacity events must still open it, since the popover is the
    // keyboard path to every event.
    const singleEventDay = page
      .getByRole("gridcell")
      .filter({ hasText: "Single event day" });
    await expect(singleEventDay).toHaveCount(1);
    await singleEventDay.focus();
    await page.keyboard.press("Enter");

    await expect(page.getByRole("dialog")).toBeVisible();
  });

  test("a pointer click through a normal chip opens that day's popover", async ({
    page,
  }) => {
    const singleEventDay = page
      .getByRole("gridcell")
      .filter({ hasText: "Single event day" });
    const chip = singleEventDay
      .getByTestId("month-event-chip")
      .filter({ hasText: "Single event day" });
    await expect(chip).toBeVisible();
    const box = await chip.boundingBox();
    expect(box).not.toBeNull();

    await page.mouse.click(
      (box as { x: number; width: number }).x +
        (box as { width: number }).width / 2,
      (box as { y: number; height: number }).y +
        (box as { height: number }).height / 2,
    );

    const dialog = page.getByRole("dialog", { name: /events for/i });
    await expect(dialog).toBeVisible();
    await expect(dialog).toContainText("Single event day");
  });

  test("a pointer click through +N opens the complete busy-day popover", async ({
    page,
  }) => {
    const busyDay = page.locator(
      `[role="gridcell"][data-date="${getTodayDateString()}"]`,
    );
    const summary = busyDay.getByTestId("month-overflow-summary");
    await expect(summary).toBeVisible();
    const box = await summary.boundingBox();
    expect(box).not.toBeNull();

    await page.mouse.click(
      (box as { x: number; width: number }).x +
        (box as { width: number }).width / 2,
      (box as { y: number; height: number }).y +
        (box as { height: number }).height / 2,
    );

    const dialog = page.getByRole("dialog", { name: /events for/i });
    await expect(dialog).toBeVisible();
    await expect(
      dialog.getByRole("button", { name: /Overflow event/ }),
    ).toHaveCount(6);
  });

  test("ArrowRight crosses the grid edge and restores the exact landed day", async ({
    page,
  }) => {
    const cells = page.getByRole("gridcell");
    const last = cells.last();
    const before = await last.getAttribute("data-date");
    if (!before) throw new Error("Last gridcell has no data-date");
    await last.focus();

    await page.keyboard.press("ArrowRight");
    const expected = formatLocalDate(addDays(parseLocalDate(before), 1));
    await expect.poll(() =>
      page.evaluate(() =>
        document.activeElement?.getAttribute("data-date"),
      ),
    ).toBe(expected);
  });

  test("PageDown changes the visible month and restores the exact date", async ({
    page,
  }) => {
    const cell = page.getByRole("gridcell").nth(10);
    const before = await cell.getAttribute("data-date");
    if (!before) throw new Error("Gridcell has no data-date");
    await cell.focus();
    await page.keyboard.press("PageDown");
    const expected = formatLocalDate(addMonths(parseLocalDate(before), 1));
    await expect.poll(() =>
      page.evaluate(() =>
        document.activeElement?.getAttribute("data-date"),
      ),
    ).toBe(expected);
  });

  test("Enter on a day with events opens the overflow popover", async ({
    page,
  }) => {
    // The six overflow events are seeded in beforeEach via the API.
    const busyDay = page.locator(
      `[role="gridcell"][data-date="${getTodayDateString()}"]`,
    );
    await busyDay.focus();
    await page.keyboard.press("Enter");

    await expect(page.getByRole("dialog")).toBeVisible();
    await page.keyboard.press("Escape");
    await expect(page.getByRole("dialog")).not.toBeVisible();

    const focusedRole = await page.evaluate(() =>
      document.activeElement?.getAttribute("role"),
    );
    expect(focusedRole).toBe("gridcell");
  });

  test("selecting a popover event closes it and opens EventDetailModal", async ({
    page,
  }) => {
    const busyDay = page.locator(
      `[role="gridcell"][data-date="${getTodayDateString()}"]`,
    );
    const originDate = await busyDay.getAttribute("data-date");
    await busyDay.focus();
    await page.keyboard.press("Enter");
    const dayPopover = page.getByRole("dialog", { name: /events for/i });
    await dayPopover.getByRole("button", { name: /Overflow event 3/i }).click();
    await expect(dayPopover).not.toBeVisible();
    const detail = page
      .getByRole("dialog")
      .filter({ hasText: "Overflow event 3" });
    await expect(detail).toBeVisible();
    await detail.getByRole("button", { name: /close/i }).click();
    await expect(detail).not.toBeVisible();
    expect(
      await page.evaluate(() =>
        document.activeElement?.getAttribute("data-date"),
      ),
    ).toBe(originDate);
  });

  test("toolbar navigation dismisses the popover and keeps focus on the toolbar", async ({
    page,
  }) => {
    const busyDay = page.locator(
      `[role="gridcell"][data-date="${getTodayDateString()}"]`,
    );
    await busyDay.focus();
    await page.keyboard.press("Enter");
    await expect(page.getByRole("dialog", { name: /events for/i })).toBeVisible();

    const next = page.getByRole("button", { name: "Next" });
    await next.click();

    await expect(page.getByRole("dialog", { name: /events for/i }))
      .not.toBeVisible();
    await expect(next).toBeFocused();
  });

  test("weekday header is a row of columnheaders inside the grid", async ({
    page,
  }) => {
    const grid = page.getByRole("grid");
    await expect(grid.getByRole("columnheader")).toHaveCount(7);
    await expect(grid.locator(':scope > [role="row"]')).toHaveCount(1);
    await expect(grid.locator(':scope > [role="rowgroup"]')).toHaveCount(1);
    const weekRows = grid.locator(
      ':scope > [role="rowgroup"] > [role="row"]',
    );
    expect(await weekRows.count()).toBeGreaterThanOrEqual(4);
    expect(await weekRows.count()).toBeLessThanOrEqual(6);
  });

  test("multi-day segments weld, keep row-edge corners square and never overflow the page", async ({
    page,
  }) => {
    const chips = page
      .getByTestId("month-event-chip")
      .filter({ hasText: "Family trip" });
    await expect(chips).toHaveCount(5);
    await expect(chips.nth(0)).toHaveClass(/rounded-l/);
    await expect(chips.nth(0)).not.toHaveClass(/rounded-r/);
    await expect(chips.nth(4)).toHaveClass(/rounded-r/);
    await expect(chips.nth(4)).not.toHaveClass(/rounded-l/);

    const friday = await chips.nth(0).boundingBox();
    const saturday = await chips.nth(1).boundingBox();
    expect(friday).not.toBeNull();
    expect(saturday).not.toBeNull();
    expect(
      Math.abs(
        (friday as { x: number; width: number }).x +
          (friday as { width: number }).width -
          (saturday as { x: number }).x,
      ),
    ).toBeLessThanOrEqual(1);

    // Saturday and Sunday are interior segments at opposite row edges. They
    // stay square even though outward bleed is suppressed there.
    await expect(chips.nth(1)).not.toHaveClass(/rounded-r/);
    await expect(chips.nth(2)).not.toHaveClass(/rounded-l/);
    expect(
      await page.evaluate(
        () =>
          document.documentElement.scrollWidth <=
          document.documentElement.clientWidth,
      ),
    ).toBe(true);
  });
});
```

- [ ] **Step 3: Add real persisted-restoration coverage**

The leaf tests in Task 10 prove state priority but cannot prove TanStack's
IndexedDB provider restores before Month decides that a range is uncached. Add
this case inside the existing `Offline read persistence (Option C)` describe in
`e2e/offline-persistence.spec.ts`. Add `mkdir` from `node:fs/promises`, plus
`readPersistedQueryData` and `switchCalendarView` to the existing
`./helpers/test-helpers` import. This project already runs the production
build/preview and real service worker.

```ts
test("large Month restores cached empty April and February without a cold-cache flash", async ({
  page,
  context,
  request,
}) => {
  await page.setViewportSize({ width: 1280, height: 800 });
  await page.emulateMedia({ reducedMotion: "reduce" });
  await page.clock.setFixedTime(new Date(2026, 3, 15, 12, 0, 0));
  await clearStorage(page);

  // Observe every text mutation from before application scripts run. A final
  // absence assertion alone could miss a one-frame uncached-message flash.
  await page.addInitScript(() => {
    (window as Window & { __sawMonthColdCopy?: boolean })
      .__sawMonthColdCopy = false;
    new MutationObserver(() => {
      if (document.body?.textContent?.includes("This month isn't cached yet.")) {
        (window as Window & { __sawMonthColdCopy?: boolean })
          .__sawMonthColdCopy = true;
      }
    }).observe(document, { childList: true, subtree: true, characterData: true });
  });

  const reg = await registerFamily(request, {
    familyName: "Empty Month Cache",
    members: [{ name: "Alice", color: "coral" }],
  });
  await seedBrowserAuth(page, reg);
  await page.reload();
  await waitForHydration(page);
  await waitForCalendarReady(page);
  await switchCalendarView(page, "monthly");

  // April has a grid-expanded key (Mar 29-May 2). February 2026 is exactly
  // four Sunday-start weeks, so its expanded and month-only keys are equal.
  await expect(page.getByRole("grid", { name: "April 2026" })).toBeVisible();
  await page.getByRole("button", { name: "Previous" }).click();
  await page.getByRole("button", { name: "Previous" }).click();
  await expect(page.getByRole("grid", { name: "February 2026" })).toBeVisible();
  await page.getByRole("button", { name: "Next" }).click();
  await page.getByRole("button", { name: "Next" }).click();
  await expect(page.getByRole("grid", { name: "April 2026" })).toBeVisible();

  // Persistence is only ready when both exact Month query keys exist with
  // cached empty payloads. A generic "some cache exists" wait can pass on the
  // bootstrap/preferences queries and leave this test racing IndexedDB.
  await expect.poll(async () => {
    const [april, february] = await Promise.all([
      readPersistedQueryData<{ data?: unknown[] }>(page, [
        "calendar",
        "events",
        { startDate: "2026-03-29", endDate: "2026-05-02" },
      ]),
      readPersistedQueryData<{ data?: unknown[] }>(page, [
        "calendar",
        "events",
        { startDate: "2026-02-01", endDate: "2026-02-28" },
      ]),
    ]);
    return [april?.data?.length, february?.data?.length];
  }, { timeout: 10_000 }).toEqual([0, 0]);
  await waitForServiceWorkerReady(page);
  await context.setOffline(true);
  await page.reload();
  await waitForHydration(page);
  await waitForCalendarReady(page);
  await switchCalendarView(page, "monthly");

  await expect(page.getByRole("grid", { name: "April 2026" })).toBeVisible();
  expect(
    await page.evaluate(() =>
      Boolean(
        (window as Window & { __sawMonthColdCopy?: boolean })
          .__sawMonthColdCopy,
      ),
    ),
  ).toBe(false);

  // Task 12 opts into one deterministic offline-restoration screenshot. The
  // normal offline-persistence project remains artifact-free.
  const visualOut = process.env.CALENDAR_VISUAL_OUT_DIR;
  if (visualOut) {
    const directory = `${visualOut}/month/1280x800`;
    await mkdir(directory, { recursive: true });
    await page.evaluate(() => document.fonts.ready);
    await page.evaluate(
      () => new Promise<void>((resolve) =>
        requestAnimationFrame(() => requestAnimationFrame(() => resolve())),
      ),
    );
    await page.screenshot({
      path: `${directory}/offline-restored.png`,
      fullPage: true,
    });
  }

  await page.getByRole("button", { name: "Previous" }).click();
  await page.getByRole("button", { name: "Previous" }).click();
  await expect(page.getByRole("grid", { name: "February 2026" })).toBeVisible();
  await expect(page.getByText(/isn't cached/i)).toHaveCount(0);

  // March and April were loaded during navigation; May was never loaded. The
  // honest cold-cache state therefore appears only on the third Next click.
  await page.getByRole("button", { name: "Next" }).click();
  await page.getByRole("button", { name: "Next" }).click();
  await page.getByRole("button", { name: "Next" }).click();
  await expect(page.getByText("This month isn't cached yet.")).toBeVisible();
});
```

- [ ] **Step 4: Run both Month browser gates**

```bash
mkdir -p .calendar-evidence/current
CALENDAR_VISUAL_OUT_DIR="$PWD/.calendar-evidence/current" \
scripts/run-calendar-e2e.sh bash -c '
  set -euo pipefail
  npm run test:e2e -- e2e/large-screen-calendar-month.spec.ts
  npx playwright test e2e/offline-persistence.spec.ts --project=offline-persistence
'
```

Expected: all pass. Fix real failures; do not weaken assertions to make them pass.

- [ ] **Step 5: Commit**

```bash
npm run build
git add scripts/run-calendar-e2e.sh e2e/large-screen-calendar-month.spec.ts e2e/offline-persistence.spec.ts
git commit -m "test(calendar): add large-screen month E2E coverage"
```

---

## Task 12: Month phase gate

- [ ] **Step 1: Add the deterministic visual harness**

Add `.calendar-evidence/` to `.gitignore`; screenshots are local PR evidence,
not source files. Create `e2e/large-screen-calendar-visual.spec.ts`. It is
opt-in so the normal full E2E suite does not generate files. Do not fabricate a
zero-member browser fixture: released registration requires a member and the
authenticated shell sends a family with no remaining members to onboarding
before Calendar mounts. The defensive Schedule branches retain direct
component-test evidence. The reachable visual fallback case keeps a valid
family mounted and intercepts one Calendar response to substitute an unknown
`memberId`.

```ts
import { mkdir, writeFile } from "node:fs/promises";
import { expect, test } from "@playwright/test";
import {
  createCalendarEvent,
  registerFamily,
  seedBrowserAuth,
} from "./helpers/api-helpers";
import {
  clearStorage,
  switchCalendarView,
  waitForCalendarReady,
  waitForHydration,
} from "./helpers/test-helpers";

const evidenceRoot = process.env.CALENDAR_VISUAL_OUT_DIR;
test.skip(!evidenceRoot, "Set CALENDAR_VISUAL_OUT_DIR for visual evidence");

const monthViewports = [
  { width: 375, height: 812 },
  { width: 768, height: 1024 },
  { width: 769, height: 1024 },
  { width: 1024, height: 768 },
  { width: 1280, height: 800 },
  { width: 1440, height: 900 },
  { width: 1920, height: 1080 },
  { width: 2560, height: 1440 },
] as const;

async function settleAndCapture(
  page: import("@playwright/test").Page,
  relativePath: string,
) {
  if (!evidenceRoot) throw new Error("CALENDAR_VISUAL_OUT_DIR is required");
  const directory = `${evidenceRoot}/${relativePath.substring(0, relativePath.lastIndexOf("/"))}`;
  await mkdir(directory, { recursive: true });
  await page.evaluate(() => document.fonts.ready);
  await page.evaluate(
    () => new Promise<void>((resolve) =>
      requestAnimationFrame(() => requestAnimationFrame(() => resolve())),
    ),
  );
  await page.screenshot({
    path: `${evidenceRoot}/${relativePath}`,
    fullPage: true,
  });
}

async function renderedTextContrast(
  locator: import("@playwright/test").Locator,
  backgroundAncestor?: string,
): Promise<number> {
  return locator.evaluate((element, ancestorSelector) => {
    const requestedBackground = ancestorSelector
      ? element.closest(ancestorSelector)
      : element;
    if (!requestedBackground) {
      throw new Error("Contrast background was not found");
    }

    type Rgba = [number, number, number, number];
    const rgba = (cssColor: string): Rgba => {
      // Canvas converts rgb()/oklch()/other supported CSS colours to sRGB and
      // preserves alpha. Normalize all four channels before compositing.
      const canvas = document.createElement("canvas");
      canvas.width = 1;
      canvas.height = 1;
      const context = canvas.getContext("2d");
      if (!context) throw new Error("2D canvas is unavailable");
      context.clearRect(0, 0, 1, 1);
      context.fillStyle = cssColor;
      context.fillRect(0, 0, 1, 1);
      const [red, green, blue, alpha] = Array.from(
        context.getImageData(0, 0, 1, 1).data,
      );
      return [red / 255, green / 255, blue / 255, alpha / 255];
    };
    const over = (foreground: Rgba, backdrop: Rgba): Rgba => {
      const alpha = foreground[3] + backdrop[3] * (1 - foreground[3]);
      if (alpha === 0) return [0, 0, 0, 0];
      return [
        (foreground[0] * foreground[3] +
          backdrop[0] * backdrop[3] * (1 - foreground[3])) / alpha,
        (foreground[1] * foreground[3] +
          backdrop[1] * backdrop[3] * (1 - foreground[3])) / alpha,
        (foreground[2] * foreground[3] +
          backdrop[2] * backdrop[3] * (1 - foreground[3])) / alpha,
        alpha,
      ];
    };
    const luminance = (color: Rgba): number => {
      const linear = color.slice(0, 3).map((value) =>
        value <= 0.04045
          ? value / 12.92
          : ((value + 0.055) / 1.055) ** 2.4,
      );
      return 0.2126 * linear[0] + 0.7152 * linear[1] + 0.0722 * linear[2];
    };

    // Resolve translucent surfaces against every ancestor. The app currently
    // supports a light canvas, so opaque white is the final page backdrop.
    const layers: Rgba[] = [];
    // Start at the text element so any translucent intermediate wrapper before
    // the requested surface is included too.
    for (let node: Element | null = element; node; node = node.parentElement) {
      layers.push(rgba(getComputedStyle(node).backgroundColor));
    }
    let renderedBackground: Rgba = [1, 1, 1, 1];
    for (let index = layers.length - 1; index >= 0; index--) {
      renderedBackground = over(layers[index], renderedBackground);
    }
    const renderedForeground = over(
      rgba(getComputedStyle(element).color),
      renderedBackground,
    );
    const foregroundLuminance = luminance(renderedForeground);
    const backgroundLuminance = luminance(renderedBackground);
    return (
      (Math.max(foregroundLuminance, backgroundLuminance) + 0.05) /
      (Math.min(foregroundLuminance, backgroundLuminance) + 0.05)
    );
  }, backgroundAncestor);
}

async function renderedOpacity(
  locator: import("@playwright/test").Locator,
): Promise<number> {
  return locator.evaluate((element) => {
    let opacity = 1;
    for (let node: Element | null = element; node; node = node.parentElement) {
      opacity *= Number.parseFloat(getComputedStyle(node).opacity);
    }
    return opacity;
  });
}

test("Month viewport matrix and measured capacity proof", async ({
  page,
  request,
}) => {
  await page.clock.setFixedTime(new Date(2026, 7, 15, 12, 0, 0));
  await page.emulateMedia({ reducedMotion: "reduce" });
  await page.setViewportSize({ width: 1440, height: 900 });
  await page.goto("/");
  await clearStorage(page);

  const reg = await registerFamily(request, {
    familyName: "Calendar Visual Matrix",
    members: [
      { name: "Alice", color: "coral" },
      { name: "Bob", color: "teal" },
      { name: "Charlie", color: "purple" },
    ],
  });

  for (const date of ["2026-08-15", "2026-04-15"]) {
    for (let index = 0; index < 10; index++) {
      await createCalendarEvent(request, reg.token, {
        title:
          index === 0
            ? "A deliberately very long family event title for visual truncation"
            : `Busy event ${index}`,
        date,
        startTime: "9:00 AM",
        endTime: "10:00 AM",
        memberId: reg.family.members[index % 3].id,
        isAllDay: false,
      });
    }
  }
  await createCalendarEvent(request, reg.token, {
    title: "Family trip",
    date: "2026-08-07",
    endDate: "2026-08-11",
    startTime: "12:00 AM",
    endTime: "11:59 PM",
    memberId: reg.family.members[0].id,
    isAllDay: true,
  });
  await createCalendarEvent(request, reg.token, {
    title: "Daily medicine",
    date: "2026-08-03",
    startTime: "8:00 AM",
    endTime: "8:15 AM",
    memberId: reg.family.members[1].id,
    isAllDay: false,
    // Released backend v1.9.0 accepts UNTIL but not COUNT.
    recurrenceRule: "FREQ=DAILY;UNTIL=20260805",
  });
  await createCalendarEvent(request, reg.token, {
    title: "Adjacent month event",
    date: "2026-07-31",
    startTime: "4:00 PM",
    endTime: "5:00 PM",
    memberId: reg.family.members[0].id,
    isAllDay: false,
  });

  await seedBrowserAuth(page, reg);
  await page.reload();
  await waitForHydration(page);
  await waitForCalendarReady(page);
  await switchCalendarView(page, "monthly");

  for (const viewport of monthViewports) {
    await page.setViewportSize(viewport);
    await settleAndCapture(
      page,
      `month/${viewport.width}x${viewport.height}/busy.png`,
    );
  }

  await page.setViewportSize({ width: 1024, height: 768 });
  const augustGrid = page.getByRole("grid", { name: "August 2026" });
  await expect(augustGrid.locator('[role="rowgroup"] > [role="row"]'))
    .toHaveCount(6);
  const augustCell = augustGrid.locator('[data-date="2026-08-15"]');
  const adjacentChip = augustGrid
    .locator('[data-date="2026-07-31"]')
    .getByTestId("month-event-chip")
    .filter({ hasText: "Adjacent month event" });
  await expect(adjacentChip).toBeVisible();
  expect(await renderedOpacity(adjacentChip)).toBeCloseTo(1, 5);
  const sixWeekCapacity = await augustCell
    .locator(
      '[data-testid="month-event-chip"], [data-testid="month-slot-blank"], [data-testid="month-overflow-summary"]',
    )
    .count();
  const overflowSummary = augustCell.getByTestId("month-overflow-summary");
  await expect(overflowSummary).toBeVisible();
  expect(
    await renderedTextContrast(overflowSummary, '[role="gridcell"]'),
  ).toBeGreaterThanOrEqual(4.5);

  for (let step = 0; step < 4; step++) {
    await page.getByRole("button", { name: "Previous" }).click();
  }
  await page.setViewportSize({ width: 1440, height: 900 });
  const aprilGrid = page.getByRole("grid", { name: "April 2026" });
  await expect(aprilGrid.locator('[role="rowgroup"] > [role="row"]'))
    .toHaveCount(5);
  const aprilCell = aprilGrid.locator('[data-date="2026-04-15"]');
  const fiveWeekCapacity = await aprilCell
    .locator(
      '[data-testid="month-event-chip"], [data-testid="month-slot-blank"], [data-testid="month-overflow-summary"]',
    )
    .count();
  expect(fiveWeekCapacity).toBeGreaterThan(sixWeekCapacity);
  await settleAndCapture(page, "month/1440x900/five-week-capacity.png");

  if (!evidenceRoot) throw new Error("CALENDAR_VISUAL_OUT_DIR is required");
  await writeFile(
    `${evidenceRoot}/month/capacity.json`,
    `${JSON.stringify({
      sixWeek: { month: "August 2026", viewport: "1024x768", slots: sixWeekCapacity },
      fiveWeek: { month: "April 2026", viewport: "1440x900", slots: fiveWeekCapacity },
    }, null, 2)}\n`,
  );
});
```

Add named tests in the same file for the remaining Month evidence. Every test
must seed its own family/query state, assert the expected UI before capture,
call `page.emulateMedia({ reducedMotion: "reduce" })`, and use
`settleAndCapture`; screenshot-only tests with no assertion are invalid:

| Test / output | Deterministic fixture and pre-capture assertion |
| --- | --- |
| `month sparse` at 1280/1440 | fixed 2026-08-15, one timed event; assert one chip and no `+N` |
| `month four-row February` at 1280/1440 | fixed 2026-02-15; assert exactly four week rows |
| `month overflow popover` at 1280/1440 | ten Aug 15 events; activate that gridcell; assert the complete 10-action dialog |
| `month missing-member fallback` at 1280/1440 | register Bob and seed one event; intercept its successful Calendar response and replace only that event's `memberId` with a fixed unknown UUID; assert the chip uses `bg-muted text-foreground` and remains readable while Calendar stays mounted |
| `month empty` at 1280/1440 | three-member family, zero events; assert normal grid, not offline copy |
| `month loading` at 1280/1440 | route `**/api/calendar/events?**` through a held promise; assert `Loading month` status, capture, then release in `finally` |
| `month error` at 1280/1440 | fulfill that route with HTTP 500; assert retry button |
| `month offline-restored` | capture from the Task 11 production-preview restoration test after the cached April grid is visible |

Use this response transform in the missing-member test after the valid event is
created and before reloading the authenticated page; do not hand-author the
rest of the released API response:

```ts
const unknownMemberId = "00000000-0000-0000-0000-000000000001";
await page.route("**/api/calendar/events?**", async (route) => {
  const response = await route.fetch();
  const body = (await response.json()) as {
    data: Array<Record<string, unknown>>;
  };
  body.data = body.data.map((event, index) =>
    index === 0 ? { ...event, memberId: unknownMemberId } : event,
  );
  await route.fulfill({ response, json: body });
});
const chip = page
  .getByTestId("month-event-chip")
  .filter({ hasText: "Missing member fixture" });
await expect(chip).toHaveClass(/bg-muted/);
await expect(chip).toHaveClass(/text-foreground/);
expect(await renderedTextContrast(chip)).toBeGreaterThanOrEqual(4.5);
```

The busy fixture already covers a long title, three visible member names,
multi-day week-boundary geometry, recurrence, overflow, and three solid member
colours. The chip unit test records numeric AA ratios for all seven solid member
colours; browser checks record the rendered ratios for token-based text and
confirm no ancestor opacity changes adjacent-month event content.

Run the harness through the released-backend wrapper:

```bash
mkdir -p .calendar-evidence/current
CALENDAR_VISUAL_OUT_DIR="$PWD/.calendar-evidence/current" \
  scripts/run-calendar-e2e.sh npx playwright test \
  e2e/large-screen-calendar-visual.spec.ts --project=chromium --workers=1
```

- [ ] **Step 2: Prove preserved views against `origin/main`**

Create `e2e/calendar-preservation-visual.spec.ts`. It is opt-in, self-seeding,
and contains the assertions as well as the capture matrix, so the file copied
to `origin/main` exercises the exact same test logic:

```ts
import { mkdir } from "node:fs/promises";
import { expect, test } from "@playwright/test";
import {
  createCalendarEvent,
  registerFamily,
  seedBrowserAuth,
} from "./helpers/api-helpers";
import {
  clearStorage,
  switchCalendarView,
  waitForCalendarReady,
  waitForHydration,
} from "./helpers/test-helpers";

const preservationOut = process.env.PRESERVATION_OUT_DIR;
test.skip(!preservationOut, "Set PRESERVATION_OUT_DIR for parity evidence");

const preservedCases = [
  ["monthly", 375, 812],
  ["monthly", 768, 1024],
  ["monthly", 769, 1024],
  ["schedule", 375, 812],
  ["schedule", 768, 1024],
  ["schedule", 769, 1024],
  ["weekly", 375, 812],
  ["weekly", 768, 1024],
  ["weekly", 769, 1024],
  ["daily", 375, 812],
  ["daily", 768, 1024],
  ["daily", 769, 1024],
] as const;

const desktopLabels = {
  monthly: "Month",
  schedule: "Schedule",
  weekly: "Week",
  daily: "Day",
} as const;
const mobileLabels = {
  monthly: "Monthly view",
  schedule: "Schedule view",
  weekly: "Weekly view",
  daily: "Daily view",
} as const;

for (const [view, width, height] of preservedCases) {
  test(`${view} is pixel-identical at ${width}x${height}`, async ({
    page,
    request,
  }) => {
    if (!preservationOut) {
      throw new Error("PRESERVATION_OUT_DIR is required");
    }

    await page.clock.setFixedTime(new Date(2026, 7, 15, 12, 0, 0));
    await page.emulateMedia({ reducedMotion: "reduce" });
    await page.setViewportSize({ width, height });
    await page.goto("/");
    await clearStorage(page);

    const reg = await registerFamily(request, {
      familyName: "Calendar Preservation",
      members: [{ name: "Alice", color: "coral" }],
    });
    await createCalendarEvent(request, reg.token, {
      title: "Baseline event",
      date: "2026-08-15",
      startTime: "9:00 AM",
      endTime: "10:00 AM",
      memberId: reg.family.members[0].id,
      isAllDay: false,
    });

    await seedBrowserAuth(page, reg);
    await page.reload();
    await waitForHydration(page);
    await waitForCalendarReady(page);
    await switchCalendarView(page, view);

    const desktopButton = page
      .getByTestId("view-switcher")
      .getByRole("button", { name: desktopLabels[view] });
    if (await desktopButton.isVisible().catch(() => false)) {
      await expect(desktopButton).toHaveClass(/bg-background/);
    } else {
      await expect(
        page.getByRole("button", { name: mobileLabels[view] }),
      ).toHaveClass(/bg-primary/);
    }
    await expect(page.getByText("Baseline event").first()).toBeVisible();

    await page.evaluate(() => document.fonts.ready);
    await page.evaluate(
      () => new Promise<void>((resolve) =>
        requestAnimationFrame(() => requestAnimationFrame(() => resolve())),
      ),
    );
    await mkdir(preservationOut, { recursive: true });
    await page.screenshot({
      path: `${preservationOut}/${view}-${width}x${height}.png`,
      fullPage: true,
    });
  });
}
```

Run the identical file at `origin/main` and the implementation commit under one
released backend, and aggregate all twelve comparisons:

```bash
scripts/run-calendar-e2e.sh bash -c '
  set -euo pipefail
  baseline="../calendar-preservation-baseline"
  root="$PWD/.calendar-evidence/preservation"
  test ! -e "$baseline"
  mkdir -p "$root/baseline" "$root/current"

  cleanup_preservation() {
    command_status=$?
    trap - EXIT INT TERM
    set +e
    cleanup_status=0
    rm -f "$baseline/e2e/calendar-preservation-visual.spec.ts" || cleanup_status=$?
    if [ -d "$baseline" ]; then
      git worktree remove "$baseline" || cleanup_status=$?
    fi
    if [ "$command_status" -ne 0 ]; then exit "$command_status"; fi
    exit "$cleanup_status"
  }
  trap cleanup_preservation EXIT
  trap 'exit 130' INT
  trap 'exit 143' TERM

  git worktree add "$baseline" origin/main
  cp e2e/calendar-preservation-visual.spec.ts "$baseline/e2e/"
  (
    cd "$baseline"
    npm ci
    CI=1 PRESERVATION_OUT_DIR="$root/baseline" \
      npx playwright test e2e/calendar-preservation-visual.spec.ts \
      --project=chromium --workers=1
  )
  CI=1 PRESERVATION_OUT_DIR="$root/current" \
    npx playwright test e2e/calendar-preservation-visual.spec.ts \
    --project=chromium --workers=1

  test "$(find "$root/baseline" -name "*.png" | wc -l | tr -d " ")" -eq 12
  test "$(find "$root/current" -name "*.png" | wc -l | tr -d " ")" -eq 12
  compare_status=0
  for baseline_file in "$root"/baseline/*.png; do
    name="${baseline_file##*/}"
    shasum -a 256 "$baseline_file" "$root/current/$name"
    cmp -s "$baseline_file" "$root/current/$name" || compare_status=1
  done
  test "$compare_status" -eq 0
'
```

Task 17 reruns this exact preservation gate after Schedule changes. The hashes,
not current-only screenshots, prove that mobile Month/Schedule at 375 and the
768 boundary, tablet Month/Schedule at 769, and Week/Day at all three widths did
not change.

- [ ] **Step 3: Review the captured Month matrix**

With the real backend running and a family of three members seeded, capture
Month at: **375x812**, **768x1024** (must show the mobile view), **769x1024**,
**1024x768**, **1280x800**, **1440x900**, **1920x1080** and **2560x1440**.
The harness stores deterministic files under
`.calendar-evidence/current/month/<width>x<height>/<scenario>.png`. Attach them
to the PR; the directory is ignored and is removed only after attachment.

At 1280x800 and 1440x900 also capture: a busy month, a sparse month,
**February 2026** (four rows), a month with a multi-day run crossing a week
boundary, a day with overflow and its open popover, long event titles, the
reachable missing-member fallback, loading, empty, error, and offline.

- [ ] **Step 4: Review against the spec**

Confirm: no dead space or permanent scrollbar below the last week row; visual
slots are exactly 28px and the cell is the only target; runs weld with rounded
corners only at true start and end; weekend, today and focused day are distinct;
`+N more` is legible. In browser geometry, assert adjacent continuation-chip
rectangles meet within 1px, Sunday/Saturday do not bleed outward, and
`document.documentElement.scrollWidth <= document.documentElement.clientWidth`.

If capacity at 1024x768 for a six-week month reads too tight, apply the spec
Section 4.2 levers **in order**: reduce the rendered
`MONTH_NUMERAL_BLOCK`, reduce the rendered `MONTH_CELL_PADDING_Y`, then raise
`MONTH_MIN_ROW_HEIGHT` and accept scrolling. Do not reduce
`MONTH_CHIP_HEIGHT` below 28 or change arithmetic without the matching DOM
dimension.

- [ ] **Step 5: Confirm Month is complete**

```bash
npm run lint && npm test -- --run && npm run build
```

Expected: all exit 0. Month is done and gated; Schedule begins. Do not start Task 13 until this passes — the phase boundary is what keeps Schedule's mobile risk from contaminating a Month regression hunt.

If the gate changed any tracked file, run the three commands again and commit
those corrections before Task 13:

```bash
npm run lint
npm test -- --run
npm run build
git add src e2e
git commit -m "fix(calendar): address month screenshot gate"
```

If either visual harness or the ignore rule is new/changed, include it in that
verified commit: `git add .gitignore e2e/large-screen-calendar-visual.spec.ts e2e/calendar-preservation-visual.spec.ts`.

---

## Task 13: Schedule row grouping helper

**Files:**
- Create: `src/components/calendar/utils/schedule-rows.ts`
- Create: `src/components/calendar/utils/schedule-rows.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `src/components/calendar/utils/schedule-rows.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createTestEvent, testMembers } from "@/test/fixtures";
import {
  buildScheduleRows,
  hasScheduleWindowEvents,
} from "./schedule-rows";

const d = (day: number) => new Date(2026, 2, day);
const filter = {
  selectedMembers: testMembers.map((m) => m.id),
  showAllDayEvents: true,
};
const at = (day: number, title = `E${day}`) =>
  createTestEvent({ id: title, title, date: d(day), memberId: testMembers[0].id });

describe("buildScheduleRows", () => {
  it("emits a day row per populated day", () => {
    const rows = buildScheduleRows({
      events: [at(8), at(9)],
      startDate: d(8),
      dayCount: 2,
      filter,
    });

    expect(rows).toHaveLength(2);
    expect(rows.every((r) => r.kind === "day")).toBe(true);
  });

  it("collapses a run of empty days into one gap row", () => {
    const rows = buildScheduleRows({
      events: [at(8), at(13)],
      startDate: d(8),
      dayCount: 14,
      filter,
    });

    const gaps = rows.filter((r) => r.kind === "gap");
    expect(gaps).toHaveLength(2); // 9-12, then 14-21
    expect(gaps[0]).toMatchObject({ start: d(9), end: d(12) });
  });

  it("emits a gap row for a single empty day", () => {
    const rows = buildScheduleRows({
      events: [at(8), at(10)],
      startDate: d(8),
      dayCount: 3,
      filter,
    });

    expect(rows[1]).toMatchObject({ kind: "gap", start: d(9), end: d(9) });
  });

  it("emits a leading gap when the window opens empty", () => {
    const rows = buildScheduleRows({
      events: [at(10)],
      startDate: d(8),
      dayCount: 3,
      filter,
    });

    expect(rows[0]).toMatchObject({ kind: "gap", start: d(8), end: d(9) });
  });

  it("clips a trailing gap to the window", () => {
    const rows = buildScheduleRows({
      events: [at(8)],
      startDate: d(8),
      dayCount: 3,
      filter,
    });

    expect(rows[1]).toMatchObject({ kind: "gap", start: d(9), end: d(10) });
  });

  it("returns no rows when the whole window is empty", () => {
    // The caller renders the whole-view empty state in this case rather than
    // one 14-day gap row. See spec Section 5.2.
    expect(
      buildScheduleRows({ events: [], startDate: d(8), dayCount: 14, filter }),
    ).toHaveLength(0);
  });

  it("treats filtered-out events as empty days", () => {
    const rows = buildScheduleRows({
      events: [at(9)],
      startDate: d(8),
      dayCount: 2,
      filter: { selectedMembers: [], showAllDayEvents: true },
    });

    expect(rows).toHaveLength(0);
  });

  it("detects raw events only inside the rendered window", () => {
    expect(hasScheduleWindowEvents([at(21)], d(8), 14)).toBe(true);
    // Offset 14 is fetched by the existing inclusive query but not rendered.
    expect(hasScheduleWindowEvents([at(22)], d(8), 14)).toBe(false);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

```bash
npm test -- --run src/components/calendar/utils/schedule-rows.test.ts
```

Expected: FAIL — cannot resolve `./schedule-rows`.

- [ ] **Step 3: Implement**

Create `src/components/calendar/utils/schedule-rows.ts`:

```ts
import { addDays } from "date-fns";
import { compareEventsAllDayFirst, isEventOnDate } from "@/lib/time-utils";
import type { CalendarEvent } from "@/lib/types";
import type { FilterState } from "../components/calendar-filter";

export type ScheduleRow =
  | { kind: "day"; date: Date; events: CalendarEvent[] }
  | { kind: "gap"; start: Date; end: Date };

/** Whether any unfiltered event intersects offsets 0..dayCount-1. */
export function hasScheduleWindowEvents(
  events: CalendarEvent[],
  startDate: Date,
  dayCount: number,
): boolean {
  return Array.from({ length: dayCount }, (_, offset) =>
    addDays(startDate, offset),
  ).some((day) => events.some((event) => isEventOnDate(event, day)));
}

/**
 * Day rows for populated days, and one gap row per consecutive run of
 * event-free days. Runs are computed only within the window and clipped to it,
 * with no special leading or trailing case. Returns an empty array when every
 * day is empty, so the caller can render the whole-view empty state instead.
 */
export function buildScheduleRows({
  events,
  startDate,
  dayCount,
  filter,
}: {
  events: CalendarEvent[];
  startDate: Date;
  dayCount: number;
  filter: FilterState;
}): ScheduleRow[] {
  const days: { date: Date; events: CalendarEvent[] }[] = [];

  for (let offset = 0; offset < dayCount; offset++) {
    const date = addDays(startDate, offset);

    const dayEvents = events
      .filter(
        (event) =>
          isEventOnDate(event, date) &&
          filter.selectedMembers.includes(event.memberId) &&
          (filter.showAllDayEvents || !event.isAllDay),
      )
      .sort(compareEventsAllDayFirst);

    days.push({ date, events: dayEvents });
  }

  if (days.every((day) => day.events.length === 0)) return [];

  const rows: ScheduleRow[] = [];
  let gapStart: Date | null = null;

  for (const day of days) {
    if (day.events.length === 0) {
      if (!gapStart) gapStart = day.date;
      continue;
    }
    if (gapStart) {
      const previous = addDays(day.date, -1);
      rows.push({ kind: "gap", start: gapStart, end: previous });
      gapStart = null;
    }
    rows.push({ kind: "day", date: day.date, events: day.events });
  }

  if (gapStart) {
    rows.push({ kind: "gap", start: gapStart, end: days[days.length - 1].date });
  }

  return rows;
}
```

- [ ] **Step 4: Run to verify it passes**

```bash
npm test -- --run src/components/calendar/utils/schedule-rows.test.ts
npm run build
```

Expected: PASS, 8 tests.

- [ ] **Step 5: Commit**

```bash
git add src/components/calendar/utils/schedule-rows.ts src/components/calendar/utils/schedule-rows.test.ts
git commit -m "feat(calendar): add schedule row grouping with gap runs"
```

---

## Task 14: Schedule large-screen composition

**Files:**
- Modify: `src/components/calendar/views/schedule-calendar.tsx`
- Modify: `src/components/calendar/views/schedule-calendar.test.tsx`

- [ ] **Step 0: Write the large-composition red tests**

Before changing production code, add a `describe("large composition")` block
to `schedule-calendar.test.tsx`. Its nested `beforeEach` sets 1280px; the outer
`afterEach` already calls `resetViewportWidth()` from Task 1. Add `within`,
`createTestEvent`, and `setViewportWidth` to the file's existing imports.

```tsx
describe("large composition", () => {
  const fixedNow = new Date(2026, 2, 18, 12, 0, 0);

  beforeEach(() => {
    vi.useFakeTimers();
    vi.setSystemTime(fixedNow);
    setViewportWidth(1280);
  });

  function renderLargeSchedule() {
    return render(
      <ScheduleCalendar
        events={[
          createTestEvent({
            id: "e1",
            title: "Test Event",
            date: fixedNow,
            memberId: testMembers[0].id,
          }),
          createTestEvent({
            id: "e2",
            title: "Later Event",
            date: new Date(2026, 2, 21),
            memberId: testMembers[0].id,
          }),
        ]}
        currentDate={fixedNow}
        filter={defaultFilter}
      />,
    );
  }

  it("uses the full surface with a date gutter", () => {
    const { container } = renderLargeSchedule();
    const region = screen.getByRole("region", {
      name: /Today, Wednesday March 18, 2026, 1 event/i,
    });
    expect(within(region).getByText("Today")).toBeInTheDocument();
    expect(container.querySelector(".w-full.space-y-1")).not.toBeNull();
    expect(container.querySelector(".mx-auto.max-w-3xl")).toBeNull();
  });

  it("compresses an empty run and keeps member/title information visible", () => {
    renderLargeSchedule();
    expect(
      screen.getByRole("group", {
        name: /Thursday March 19, 2026 to Friday March 20, 2026, nothing scheduled/i,
      }),
    ).toBeInTheDocument();
    expect(screen.getAllByText(testMembers[0].name).length).toBeGreaterThan(0);
    expect(screen.getByRole("heading", { name: "Test Event" }))
      .toHaveClass("text-xl");
  });

  it("distinguishes defensive no-selection from a genuinely empty window", () => {
    const { rerender } = render(
      <ScheduleCalendar
        events={[]}
        currentDate={fixedNow}
        filter={{ selectedMembers: [], showAllDayEvents: true }}
      />,
    );
    expect(
      screen.getByText("Select at least one profile to view events"),
    ).toBeInTheDocument();
    rerender(
      <ScheduleCalendar
        events={[]}
        currentDate={fixedNow}
        filter={defaultFilter}
      />,
    );
    expect(screen.getByText("No upcoming events")).toBeInTheDocument();
  });

  it("uses a truthful zero-member empty state", () => {
    resetFamilyStore();
    seedFamilyStore({ name: "Empty Family", members: [] });
    render(
      <ScheduleCalendar
        events={[]}
        currentDate={fixedNow}
        filter={{ selectedMembers: [], showAllDayEvents: true }}
      />,
    );
    expect(screen.getByText("No family members yet")).toBeInTheDocument();
  });

  it("identifies events hidden by an active all-day filter", () => {
    const allDay = createTestEvent({
      id: "hidden-all-day",
      title: "Hidden all-day",
      date: fixedNow,
      memberId: testMembers[0].id,
      isAllDay: true,
    });
    render(
      <ScheduleCalendar
        events={[allDay]}
        currentDate={fixedNow}
        filter={{ ...defaultFilter, showAllDayEvents: false }}
      />,
    );
    expect(screen.getByText("No events match your filters")).toBeInTheDocument();
  });
});
```

Run the focused file now. Expected: FAIL because the shipped component has no
large branch, date-gutter regions, gap rows, truthful empty reasons or 20px
titles. These five tests stay
in the file; Task 15 expands the large-screen matrix with state, boundary,
colour and past-day cases.

```bash
npm test -- --run src/components/calendar/views/schedule-calendar.test.tsx
```

- [ ] **Step 1: Split the component by breakpoint**

Rename the current exported body to `ScheduleCalendarCompact`, unchanged. Add:

Also add an optional `hasUnfilteredEventsInWindow?: boolean` field to
`ScheduleCalendarProps`. It is a large-branch reason signal; the compact branch
ignores it. Task 15 wires the real raw-query value from `calendar-module.tsx`.

```tsx
export function ScheduleCalendar(props: ScheduleCalendarProps) {
  const isLargeScreen = useIsLargeScreen();
  if (!isLargeScreen) return <ScheduleCalendarCompact {...props} />;
  return <ScheduleCalendarLarge {...props} />;
}
```

This is the mobile-parity gate. `ScheduleCalendarCompact` must remain byte-for-byte the markup that ships today, including `mx-auto max-w-3xl` and the `MOBILE_FAB_SCROLL_PADDING` handling.

- [ ] **Step 2: Fix the coloured left border — in the large branch only**

Today the component passes both `colorMap[member.color]?.bg` and `colorMap[member.color]?.light` into `cn()`. Both are `bg-*` utilities, so `twMerge` treats them as conflicting, keeps only the last, and `border-l-4` renders in the default border colour for every member.

**Apply this fix in `ScheduleCalendarLarge` only. Leave `ScheduleCalendarCompact` exactly as it is.**

The fix is correct, but applying it to the compact path would change mobile rendering — a member-coloured border would appear where a grey one ships today — and mobile parity is a hard contract for this story. The compact path keeps the bug, recorded as a follow-up in the spec's known limitations. Fixing it on mobile deserves its own small change where the visual delta is the point of the review, not a side effect of a large-screen pass.

Use the member's hex via an inline style, which `twMerge` cannot collapse:

```tsx
style={{ borderLeftColor: member ? colorMap[member.color].hex : undefined }}
className={cn(
  "border-l-4",
  member ? colorMap[member.color]?.light : "bg-muted",
)}
```

- [ ] **Step 3: Build the complete large composition**

Replace the compact component's import section with this complete merged
result. `useIsMobile` must remain for `ScheduleCalendarCompact`; do not append a
second `date-fns` or `@/hooks` import:

```tsx
import {
  addDays,
  format,
  isBefore,
  isSameMonth,
  isSameYear,
  startOfToday,
} from "date-fns";
import { Clock, MapPin } from "lucide-react";
import type React from "react";
import { useMemo } from "react";
import { useFamilyMembers } from "@/api";
import { useIsLargeScreen, useIsMobile } from "@/hooks";
import {
  compareEventsAllDayFirst,
  formatLocalDate,
  getEventKey,
  isEventOnDate,
} from "@/lib/time-utils";
import { type CalendarEvent, colorMap, getFamilyMember } from "@/lib/types";
import { cn } from "@/lib/utils";
import type { FilterState } from "../components/calendar-filter";
import { MOBILE_FAB_SCROLL_PADDING } from "../components/floating-action-layout";
import { MemberAvatar } from "../components/member-avatar";
import { CalendarEmptyState } from "../components/calendar-view-states";
import {
  buildScheduleRows,
  hasScheduleWindowEvents,
} from "../utils/schedule-rows";
```

Define the helpers at module scope:

```tsx
const SCHEDULE_GUTTER_WIDTH = 168;

const sameDate = (a: Date, b: Date) =>
  formatLocalDate(a) === formatLocalDate(b);

function relativeDayLabel(date: Date, today: Date): string {
  if (sameDate(date, today)) return "Today";
  if (sameDate(date, addDays(today, 1))) return "Tomorrow";
  return format(date, "EEEE");
}

function dayRegionLabel(date: Date, today: Date, count: number): string {
  const relative = relativeDayLabel(date, today);
  const dateLabel = format(date, "EEEE MMMM d, yyyy");
  const prefix =
    relative === "Today" || relative === "Tomorrow"
      ? `${relative}, ${dateLabel}`
      : dateLabel;
  return `${prefix}, ${count} ${count === 1 ? "event" : "events"}`;
}

function gapLabel(start: Date, end: Date): string {
  if (sameDate(start, end)) return format(start, "EEE, MMM d");
  if (!isSameYear(start, end)) {
    return `${format(start, "EEE, MMM d, yyyy")} – ${format(end, "EEE, MMM d, yyyy")}`;
  }
  if (!isSameMonth(start, end)) {
    return `${format(start, "EEE, MMM d")} – ${format(end, "EEE, MMM d")}`;
  }
  return `${format(start, "EEE d")} – ${format(end, "EEE d")}`;
}

function gapAriaLabel(start: Date, end: Date): string {
  if (sameDate(start, end)) {
    return `${format(start, "EEEE MMMM d, yyyy")}, nothing scheduled`;
  }
  return `${format(start, "EEEE MMMM d, yyyy")} to ${format(end, "EEEE MMMM d, yyyy")}, nothing scheduled`;
}
```

Implement the complete large branch. The outer surface has **no max width**;
readability is constrained by the event text's `max-w-[72ch]`:

```tsx
function ScheduleCalendarLarge({
  events,
  currentDate,
  onEventClick,
  filter,
  hasUnfilteredEventsInWindow = hasScheduleWindowEvents(
    events,
    currentDate,
    14,
  ),
}: ScheduleCalendarProps) {
  const familyMembers = useFamilyMembers();
  const today = startOfToday();
  const rows = useMemo(
    () =>
      buildScheduleRows({
        events,
        startDate: currentDate,
        dayCount: 14,
        filter,
      }),
    [events, currentDate, filter.selectedMembers, filter.showAllDayEvents],
  );

  if (rows.length === 0) {
    if (familyMembers.length === 0) {
      return (
        <CalendarEmptyState
          title="No family members yet"
          description="Add a family member to start seeing their events."
        />
      );
    }
    if (filter.selectedMembers.length === 0) {
      return (
        <CalendarEmptyState
          title="Select at least one profile to view events"
          description="Choose a family member to see their events."
        />
      );
    }
    if (hasUnfilteredEventsInWindow) {
      return (
        <CalendarEmptyState
          title="No events match your filters"
          description="Adjust the member or all-day filters to see more events."
        />
      );
    }
    return (
      <CalendarEmptyState
        title="No upcoming events"
        description="Nothing scheduled in the next 2 weeks."
      />
    );
  }

  return (
    <div
      data-testid="schedule-scroll-surface"
      className="flex min-h-0 flex-1 flex-col overflow-y-auto bg-background p-4"
    >
      <div className="w-full space-y-1">
        {rows.map((row) => {
          if (row.kind === "gap") {
            return (
              <div
                key={`gap-${formatLocalDate(row.start)}`}
                role="group"
                aria-label={gapAriaLabel(row.start, row.end)}
                className="grid gap-4 border-t border-border py-3"
                style={{
                  gridTemplateColumns: `${SCHEDULE_GUTTER_WIDTH}px minmax(0, 1fr)`,
                }}
              >
                <p
                  aria-hidden="true"
                  className="text-sm font-medium text-muted-foreground"
                >
                  {gapLabel(row.start, row.end)}
                </p>
                <p
                  aria-hidden="true"
                  className="text-sm italic text-muted-foreground"
                >
                  Nothing scheduled
                </p>
              </div>
            );
          }

          const relative = relativeDayLabel(row.date, today);
          const isPast = isBefore(row.date, today);
          return (
            <section
              key={formatLocalDate(row.date)}
              aria-label={dayRegionLabel(row.date, today, row.events.length)}
              className={cn(
                "grid gap-4 border-t py-3",
                isPast ? "border-border/40" : "border-border",
              )}
              style={{
                gridTemplateColumns: `${SCHEDULE_GUTTER_WIDTH}px minmax(0, 1fr)`,
              }}
            >
              <div
                data-testid="schedule-date-gutter"
                className="sticky top-0 z-10 self-start bg-background/95 py-1"
              >
                <p
                  className={cn(
                    "text-sm font-bold uppercase tracking-wide",
                    sameDate(row.date, today)
                      ? "text-primary"
                      : "text-muted-foreground",
                    isPast && "font-medium",
                  )}
                >
                  {relative}
                </p>
                <p className="text-sm font-semibold">
                  {format(row.date, "EEE, MMM d")}
                </p>
                <p className="text-sm text-muted-foreground">
                  {row.events.length} {row.events.length === 1 ? "event" : "events"}
                </p>
              </div>

              <div className="flex min-w-0 flex-col gap-2">
                {row.events.map((event) => {
                  const member = getFamilyMember(familyMembers, event.memberId);
                  return (
                    <button
                      type="button"
                      key={getEventKey(event)}
                      onClick={() => onEventClick?.(event)}
                      style={{
                        borderLeftColor: member
                          ? colorMap[member.color].hex
                          : undefined,
                      }}
                      className={cn(
                        "flex min-h-14 w-full cursor-pointer items-center gap-4 rounded-xl border-l-4 p-3 text-left",
                        "ring-1 ring-inset ring-black/5 transition-all hover:scale-[1.005] hover:shadow-md motion-reduce:transition-none motion-reduce:hover:scale-100",
                        "focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary focus-visible:ring-offset-2",
                        member ? colorMap[member.color].light : "bg-muted",
                      )}
                    >
                      <div className="min-w-0 max-w-[72ch] flex-1">
                        <h3 className="truncate text-xl leading-6 font-semibold text-foreground">
                          {event.title}
                        </h3>
                        <div
                          data-testid="schedule-event-metadata"
                          className="mt-1 flex min-w-0 flex-wrap items-center gap-2 text-sm leading-5 text-muted-foreground"
                        >
                          <span className="flex items-center gap-1">
                            <Clock aria-hidden="true" className="size-3.5" />
                            {event.isAllDay
                              ? "All day"
                              : `${event.startTime} - ${event.endTime}`}
                          </span>
                          {event.location && (
                            <span className="flex min-w-0 items-center gap-1">
                              <MapPin aria-hidden="true" className="size-3.5 shrink-0" />
                              <span className="truncate">{event.location}</span>
                            </span>
                          )}
                        </div>
                      </div>
                      {member && (
                        <span className="ml-auto flex shrink-0 items-center gap-2">
                          <span className="text-sm font-medium text-foreground">
                            {member.name}
                          </span>
                          <MemberAvatar
                            name={member.name}
                            color={member.color}
                            size="md"
                          />
                        </span>
                      )}
                    </button>
                  );
                })}
              </div>
            </section>
          );
        })}
      </div>
    </div>
  );
}
```

- [ ] **Step 4: De-emphasise past days**

For `row.date < startOfToday()`, apply `text-muted-foreground` to the gutter label and `border-border/40` to the section border. **Do not** reduce event text contrast and do not use opacity — spec Sections 4.3 and 5.3.

- [ ] **Step 5: Verify and commit**

```bash
npm test -- --run && npm run build
git add src/components/calendar/views/schedule-calendar.tsx src/components/calendar/views/schedule-calendar.test.tsx
git commit -m "feat(calendar): add large-screen schedule gutter composition"
```

---

## Task 15: Schedule states and test migration

**Files:**
- Modify: `src/components/calendar/calendar-module.tsx`
- Modify: `src/components/calendar/calendar-module.test.tsx`
- Modify: `src/components/calendar/views/schedule-calendar.tsx`
- Modify: `src/components/calendar/views/schedule-calendar.test.tsx`

- [ ] **Step 0: Write and run the state/integration regressions first**

First perform the compact viewport pinning described in Step 3 (a passing
test-only refactor). Then add the large-screen loading/error cases and the
`calendar-module.test.tsx` raw-event filter-reason case reproduced in Step 3.
Run both files before changing production code:

```bash
npm test -- --run src/components/calendar/views/schedule-calendar.test.tsx src/components/calendar/calendar-module.test.tsx
```

Expected: the compact block stays green; the new large state and raw-query
reason tests fail. Do not add these cases a second time in Step 3.

- [ ] **Step 1: Preserve compact empty state; verify large empty states**

Do **not** delete or replace the locally duplicated inline `Calendar` SVG. It
belongs to `ScheduleCalendarCompact`, and replacing it would change the compact
DOM/pixels. Task 14's large branch already distinguishes all four empty causes:

```tsx
if (rows.length === 0) {
  if (familyMembers.length === 0) {
    return <CalendarEmptyState title="No family members yet" description="Add a family member to start seeing their events." />;
  }
  if (filter.selectedMembers.length === 0) {
    return (
      <CalendarEmptyState
        title="Select at least one profile to view events"
        description="Choose a family member to see their events."
      />
    );
  }
  if (hasUnfilteredEventsInWindow) {
    return <CalendarEmptyState title="No events match your filters" description="Adjust the member or all-day filters to see more events." />;
  }
  return <CalendarEmptyState title="No upcoming events" description="Nothing scheduled in the next 2 weeks." />;
}
```

`hasUnfilteredEventsInWindow` is deliberately computed over offsets 0 through
13 from `rawEvents` before filters are applied. The existing query ends at
`currentDate + 14` inclusively; an event on that boundary day must not turn the
rendered 14-day window into a false filter-empty state.

- [ ] **Step 2: Add the Schedule loading and error states — `lg+` only**

**`ScheduleCalendarCompact` takes no new props and renders no new states.**
Below 1024px the shared centred text in `calendar-module.tsx` continues to
handle loading and error exactly as it ships today. Adding a skeleton there
would be a visible change on the mobile smart-default surface, which the
contract forbids.

Add the optional query-state fields to the shared prop type. The compact branch
ignores them; `ScheduleCalendarLarge` renders error, then loading, then content.
There is **no Schedule offline-with-no-data branch**: the spec deliberately
leaves Schedule's existing offline reads unchanged.

```tsx
interface ScheduleCalendarProps {
  events: CalendarEvent[];
  currentDate: Date;
  onEventClick?: (event: CalendarEvent) => void;
  filter: FilterState;
  isLoading?: boolean;
  isError?: boolean;
  errorMessage?: string;
  onRetry?: () => void;
  /** Raw-query reason signal; the compact branch ignores it. */
  hasUnfilteredEventsInWindow?: boolean;
}
```

Wire them at the **desktop** `ScheduleCalendar` call site only (`calendar-module.tsx:552`), leaving the two mobile call sites (`:524`, `:526`) spreading `commonProps` alone. Then extend the Task 10 gate to cover Schedule, in this same commit so lg+ Schedule never ships without states:

Import `hasScheduleWindowEvents` from `./utils/schedule-rows` in
`calendar-module.tsx`. The module's `events` value is already filtered, so the
reason signal must come from `rawEvents`; deriving it from Schedule's `events`
prop would make the real filter-specific state look genuinely empty.

```tsx
const ownsItsStates =
  isLargeScreen &&
  (calendarView === "monthly" || calendarView === "schedule");
```

At the non-mobile `case "schedule"` only:

```tsx
case "schedule":
  return (
    <ScheduleCalendar
      {...commonProps}
      isLoading={isLoading}
      isError={isError}
      errorMessage={error?.message}
      onRetry={refetch}
      hasUnfilteredEventsInWindow={hasScheduleWindowEvents(
        rawEvents,
        currentDate,
        14,
      )}
    />
  );
```

Add `isLoading = false`, `isError = false`, `errorMessage`, and `onRetry` to
the existing `ScheduleCalendarLarge` parameter destructuring. Put these exact
branches after its hooks/rows calculation and before `rows.length === 0`:

```tsx
if (isError) {
  return <CalendarErrorState message={errorMessage} onRetry={onRetry} />;
}
if (isLoading) return <ScheduleSkeleton />;
```

```tsx
function ScheduleSkeleton() {
  return (
    <div
      role="status"
      aria-label="Loading schedule"
      className="flex w-full flex-col gap-4 p-4"
    >
      {Array.from({ length: 4 }).map((_, group) => (
        // biome-ignore lint/suspicious/noArrayIndexKey: static skeleton
        <div
          key={group}
          className="grid gap-4"
          style={{
            gridTemplateColumns: `${SCHEDULE_GUTTER_WIDTH}px minmax(0, 1fr)`,
          }}
        >
          <div className="h-10 animate-pulse rounded-lg bg-muted/60 motion-reduce:animate-none" />
          <div className="flex flex-col gap-2">
            <div className="h-14 animate-pulse rounded-xl bg-muted/40 motion-reduce:animate-none" />
            <div className="h-14 animate-pulse rounded-xl bg-muted/40 motion-reduce:animate-none" />
          </div>
        </div>
      ))}
    </div>
  );
}
```

Same for the empty states in Step 1: they apply **only in the large branch**. `ScheduleCalendarCompact` keeps its existing empty-state markup and copy, so its rendering is unchanged.

Import `CalendarErrorState` beside `CalendarEmptyState`. Net effect:
`ScheduleCalendarCompact` is untouched by this entire task. Its only edit in
the plan is the mechanical extraction in Task 14 Step 1.

- [ ] **Step 3: Complete the test migration and confirm the red tests are green**

**Pin every existing test to a viewport.** All 7 current tests run at whatever `matchMedia` happens to be set to. Once a `describe` in this file calls `setViewportWidth(1280)`, the leak described in Task 1 means unpinned tests may run against `ScheduleCalendarLarge` — and they would still pass, silently ceasing to guard the compact path they exist for.

If not already done in Step 0, wrap **all seven** existing test bodies,
unchanged, inside a describe named
`compact`. Add
`beforeEach(() => setViewportWidth(390));` and
`afterEach(resetViewportWidth);` inside that describe. Do not copy them into a
second file or change any assertion.

Their assertions must not change — including the three banner strings (`"Today — Wed, Mar 18"`, `"Tomorrow — Thu, Mar 19"`, `"Friday, Mar 20"`) and `"reserves bottom clearance on mobile for the floating action button"`. They are the regression guard for the mobile path.

Retain Task 14's five red/green composition cases, then add the comprehensive
parallel large-screen block below. It adds state, colour, boundary and past-day
coverage without replacing the earlier smoke tests. It needs a shared render
helper and pinned time:

```tsx
describe("large screen", () => {
  const fixedNow = new Date(2026, 2, 18, 12, 0, 0); // Wed Mar 18 2026

  beforeEach(() => {
    vi.useFakeTimers();
    vi.setSystemTime(fixedNow);
    setViewportWidth(1280);
  });

  afterEach(() => {
    vi.useRealTimers();
    resetViewportWidth();
  });

  function renderSchedule() {
    // Mar 18 populated, Mar 19-20 empty (the gap), Mar 21 populated.
    const events = [
      createTestEvent({
        id: "e1",
        title: "Test Event",
        date: new Date(2026, 2, 18),
        memberId: testMembers[0].id,
      }),
      createTestEvent({
        id: "e2",
        title: "Later Event",
        date: new Date(2026, 2, 21),
        memberId: testMembers[0].id,
      }),
    ];
    return render(
      <ScheduleCalendar
        events={events}
        currentDate={fixedNow}
        filter={defaultFilter}
      />,
    );
  }

  it("splits the date into gutter label, date and count", () => {
    renderSchedule();
    const region = screen.getByRole("region", {
      name: /Today, Wednesday March 18, 2026, 1 event/i,
    });
    expect(region).toBeInTheDocument();
    expect(within(region).getByText("Today")).toBeInTheDocument();
    expect(within(region).getByText("Wed, Mar 18")).toBeInTheDocument();
  });

  it("renders a gap row for an event-free stretch", () => {
    renderSchedule();
    const gap = screen.getByRole("group", {
      name: /Thursday March 19, 2026 to Friday March 20, 2026, nothing scheduled/i,
    });
    expect(gap).not.toHaveAttribute("tabindex");
    expect(within(gap).getByText("Nothing scheduled")).toBeInTheDocument();
  });

  it("disambiguates a gap that crosses a year boundary", () => {
    const events = [
      createTestEvent({
        id: "dec",
        title: "December event",
        date: new Date(2026, 11, 30),
        memberId: testMembers[0].id,
      }),
      createTestEvent({
        id: "jan",
        title: "January event",
        date: new Date(2027, 0, 2),
        memberId: testMembers[0].id,
      }),
    ];
    render(
      <ScheduleCalendar
        events={events}
        currentDate={new Date(2026, 11, 30)}
        filter={defaultFilter}
      />,
    );
    const gap = screen.getByRole("group", {
      name: /Thursday December 31, 2026 to Friday January 1, 2027, nothing scheduled/i,
    });
    expect(gap).toBeInTheDocument();
    expect(
      within(gap).getByText("Thu, Dec 31, 2026 – Fri, Jan 1, 2027"),
    ).toBeInTheDocument();
  });

  it("disambiguates a gap that crosses a month boundary", () => {
    const events = [
      createTestEvent({
        id: "march",
        title: "March event",
        date: new Date(2026, 2, 30),
        memberId: testMembers[0].id,
      }),
      createTestEvent({
        id: "april",
        title: "April event",
        date: new Date(2026, 3, 2),
        memberId: testMembers[0].id,
      }),
    ];
    render(
      <ScheduleCalendar
        events={events}
        currentDate={new Date(2026, 2, 30)}
        filter={defaultFilter}
      />,
    );
    const gap = screen.getByRole("group", {
      name: /Tuesday March 31, 2026 to Wednesday April 1, 2026, nothing scheduled/i,
    });
    expect(
      within(gap).getByText("Tue, Mar 31 – Wed, Apr 1"),
    ).toBeInTheDocument();
  });

  it("shows the member name as visible text", () => {
    renderSchedule();
    expect(screen.getAllByText(testMembers[0].name).length).toBeGreaterThan(0);
  });

  it("colours the left border with the member's colour", () => {
    renderSchedule();
    const row = screen.getByRole("button", { name: /Test Event/ });
    expect(row).toHaveStyle({
      borderLeftColor: colorMap[testMembers[0].color].hex,
    });
  });

  it("uses the full surface without a fixed outer max width", () => {
    const { container } = renderSchedule();
    const content = container.querySelector(".w-full.space-y-1");
    expect(content).not.toBeNull();
    expect(content).not.toHaveStyle({ maxWidth: "1400px" });
  });

  it("de-emphasises a past group without opacity on event text", () => {
    const past = createTestEvent({
      id: "past",
      title: "Past Event",
      date: new Date(2026, 2, 17),
      memberId: testMembers[0].id,
    });
    render(
      <ScheduleCalendar
        events={[past]}
        currentDate={new Date(2026, 2, 17)}
        filter={defaultFilter}
      />,
    );
    const region = screen.getByRole("region", { name: /Tuesday March 17/ });
    expect(region).toHaveClass("border-border/40");
    expect(screen.getByRole("button", { name: /Past Event/ }).className)
      .not.toMatch(/opacity/);
  });

  it("distinguishes no selection from a genuinely empty window", () => {
    const { rerender } = render(
      <ScheduleCalendar
        events={[]}
        currentDate={fixedNow}
        filter={{ selectedMembers: [], showAllDayEvents: true }}
      />,
    );
    expect(
      screen.getByText("Select at least one profile to view events"),
    ).toBeInTheDocument();
    rerender(
      <ScheduleCalendar
        events={[]}
        currentDate={fixedNow}
        filter={defaultFilter}
      />,
    );
    expect(screen.getByText("No upcoming events")).toBeInTheDocument();
  });

  it("uses a truthful zero-member empty state", () => {
    resetFamilyStore();
    seedFamilyStore({ name: "Empty Family", members: [] });
    render(
      <ScheduleCalendar
        events={[]}
        currentDate={fixedNow}
        filter={{ selectedMembers: [], showAllDayEvents: true }}
      />,
    );
    expect(screen.getByText("No family members yet")).toBeInTheDocument();
  });

  it("identifies events hidden by active filters", () => {
    const allDay = createTestEvent({
      id: "all-day",
      title: "All-day event",
      date: fixedNow,
      memberId: testMembers[0].id,
      isAllDay: true,
    });
    render(
      <ScheduleCalendar
        events={[allDay]}
        currentDate={fixedNow}
        filter={{ ...defaultFilter, showAllDayEvents: false }}
      />,
    );
    expect(screen.getByText("No events match your filters")).toBeInTheDocument();
  });

  it("does not count the query-only fifteenth day as part of the window", () => {
    const boundary = createTestEvent({
      id: "boundary",
      title: "Boundary event",
      date: addDays(fixedNow, 14),
      memberId: testMembers[0].id,
    });
    render(
      <ScheduleCalendar
        events={[boundary]}
        currentDate={fixedNow}
        filter={defaultFilter}
      />,
    );
    expect(screen.getByText("No upcoming events")).toBeInTheDocument();
  });

  it("renders loading and error/retry in priority order", async () => {
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    const onRetry = vi.fn();
    const { rerender } = render(
      <ScheduleCalendar
        events={[]}
        currentDate={fixedNow}
        filter={defaultFilter}
        isLoading
      />,
    );
    expect(screen.getByRole("status", { name: /loading schedule/i }))
      .toBeInTheDocument();
    rerender(
      <ScheduleCalendar
        events={[]}
        currentDate={fixedNow}
        filter={defaultFilter}
        isLoading
        isError
        errorMessage="No route"
        onRetry={onRetry}
      />,
    );
    expect(screen.queryByRole("status", { name: /loading schedule/i }))
      .not.toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: /try again/i }));
    expect(onRetry).toHaveBeenCalledOnce();
  });
});
```

Add the imports this block needs and the file currently lacks: `userEvent` from
`@testing-library/user-event`, `within` from `@testing-library/react`,
`addDays` from `date-fns`,
`createTestEvent` from `@/test/fixtures`, `colorMap` from `@/lib/types`, and
`setViewportWidth` / `resetViewportWidth` from `@/test/test-utils`.

`testMembers[0]` is John with colour `coral`; `colorMap.coral.hex` is the value the border must resolve to.

The isolated component test receives raw events directly, so also add this
integration case to `calendar-module.test.tsx`. It proves the module derives the
reason from `rawEvents` before its existing client-side filter:

```tsx
describe("large Schedule empty reason", () => {
  afterEach(resetViewportWidth);

  it("reports active filters when raw in-window events are filtered out", async () => {
    setViewportWidth(1280);
    seedCalendarStore({
      calendarView: "schedule",
      filter: {
        selectedMembers: testMembers.map((member) => member.id),
        showAllDayEvents: false,
      },
    });
    seedMockEvents([
      createTestEventResponse({
        id: "hidden-all-day",
        title: "Hidden all-day event",
        memberId: testMembers[0].id,
        isAllDay: true,
      }),
    ]);

    render(<CalendarModule />);

    expect(
      await screen.findByText("No events match your filters"),
    ).toBeInTheDocument();
    expect(screen.queryByText("Hidden all-day event")).not.toBeInTheDocument();
  });
});
```

- [ ] **Step 4: Verify and commit**

```bash
npm test -- --run src/components/calendar/views/schedule-calendar.test.tsx src/components/calendar/calendar-module.test.tsx
npm run build
git add src/components/calendar/calendar-module.tsx src/components/calendar/calendar-module.test.tsx src/components/calendar/views/schedule-calendar.tsx src/components/calendar/views/schedule-calendar.test.tsx
git commit -m "feat(calendar): correct schedule empty states and migrate tests"
```

---

## Task 16: Schedule E2E and mobile parity proof

**Files:**
- Create: `e2e/large-screen-calendar-schedule.spec.ts`

- [ ] **Step 1: Write the spec**

The window must be **seeded with a gap** — one populated day, several empty, another populated. With no events at all, `buildScheduleRows` short-circuits to `[]` and the view renders the whole-view empty state, so every assertion below would fail. A gap row cannot exist unless at least one day is populated.

```ts
import { addDays } from "date-fns";
import { expect, test } from "@playwright/test";
import { formatLocalDate, parseLocalDate } from "../src/lib/time-utils";
import { colorMap } from "../src/lib/types";
import {
  createCalendarEvent,
  registerFamily,
  seedBrowserAuth,
} from "./helpers/api-helpers";
import {
  clearStorage,
  getTodayDateString,
  switchCalendarView,
  waitForCalendarReady,
  waitForHydration,
} from "./helpers/test-helpers";

test.describe("Large-screen Calendar Schedule", () => {
  test.beforeEach(async ({ page, request, isMobile }) => {
    test.skip(isMobile, "Large-screen only");
    await page.setViewportSize({ width: 1920, height: 1080 });
    await page.goto("/");
    await clearStorage(page);

    const reg = await registerFamily(request, {
      familyName: "Test Family",
      members: [{ name: "Alice", color: "coral" }],
    });

    const today = parseLocalDate(getTodayDateString());
    const dateFor = (offset: number) => formatLocalDate(addDays(today, offset));

    // Today and day +13 define the initial window and leave exactly one
    // 12-day interior gap. Day +20 is outside offsets 0..13 but enters after
    // the preserved 7-day paging step, making the 50% overlap testable.
    for (const offset of [0, 13, 20]) {
      await createCalendarEvent(request, reg.token, {
        title: `Event day ${offset}`,
        date: dateFor(offset),
        startTime: "9:00 AM",
        endTime: "10:00 AM",
        memberId: reg.family.members[0].id,
        isAllDay: false,
      });
    }

    // Keep Today's section taller than the scroll offset so browser geometry
    // can prove the date gutter actually sticks within its day group.
    for (let index = 1; index <= 4; index++) {
      await createCalendarEvent(request, reg.token, {
        title: `Today extra ${index}`,
        date: dateFor(0),
        startTime: "10:00 AM",
        endTime: "11:00 AM",
        memberId: reg.family.members[0].id,
        isAllDay: false,
      });
    }

    await seedBrowserAuth(page, reg);
    await page.reload();
    await waitForHydration(page);
    await waitForCalendarReady(page);
    await switchCalendarView(page, "schedule");
  });

  test("content spans the width with no centred narrow column", async ({
    page,
  }) => {
    const section = page.getByRole("region", { name: /Today.*5 events/i });
    const box = await section.boundingBox();
    expect(box).not.toBeNull();
    // A 1400px cap fails here; 1920 minus nav/padding leaves about 1800px.
    expect((box as { width: number }).width).toBeGreaterThan(1700);
  });

  test("the date gutter carries label, date and count", async ({ page }) => {
    const todayRegion = page.getByRole("region", { name: /Today.*5 events/i });
    await expect(todayRegion.getByText("Today", { exact: true })).toBeVisible();
    await expect(todayRegion.getByText("5 events", { exact: true })).toBeVisible();
  });

  test("the date gutter sticks while its day group scrolls", async ({ page }) => {
    // Force a short but still large-screen viewport so the fixture genuinely
    // overflows; sticky geometry is meaningless on a non-scrollable surface.
    await page.setViewportSize({ width: 1920, height: 480 });
    const scroller = page.getByTestId("schedule-scroll-surface");
    const region = page.getByRole("region", { name: /Today.*5 events/i });
    const gutter = region.getByTestId("schedule-date-gutter");
    const regionBefore = await region.boundingBox();

    expect(
      await scroller.evaluate(
        (element) => element.scrollHeight > element.clientHeight,
      ),
    ).toBe(true);

    await scroller.evaluate((element) => {
      element.scrollTop = 120;
    });

    const [scrollBox, regionAfter, gutterAfter] = await Promise.all([
      scroller.boundingBox(),
      region.boundingBox(),
      gutter.boundingBox(),
    ]);
    expect(scrollBox).not.toBeNull();
    expect(regionBefore).not.toBeNull();
    expect(regionAfter).not.toBeNull();
    expect(gutterAfter).not.toBeNull();
    expect((regionAfter as { y: number }).y).toBeLessThan(
      (regionBefore as { y: number }).y - 50,
    );
    expect((gutterAfter as { y: number }).y).toBeGreaterThanOrEqual(
      (scrollBox as { y: number }).y,
    );
    expect((gutterAfter as { y: number }).y).toBeLessThanOrEqual(
      (scrollBox as { y: number }).y + 20,
    );
  });

  test("an event-free stretch renders one gap row", async ({ page }) => {
    const gaps = page.getByRole("group", { name: /nothing scheduled/i });
    await expect(gaps).toHaveCount(1);
    await expect(gaps).not.toHaveAttribute("tabindex");
  });

  test("event rows show the member and resolve the coloured border", async ({
    page,
  }) => {
    const row = page.getByRole("button", { name: /Event day 0/ });
    await expect(row).toContainText("Alice");
    const actual = await row.evaluate(
      (element) => (element as HTMLElement).style.borderLeftColor,
    );
    const expected = await page.evaluate((hex) => {
      const probe = document.createElement("div");
      probe.style.borderLeftColor = hex;
      return probe.style.borderLeftColor;
    }, colorMap.coral.hex);
    expect(actual).toBe(expected);
  });

  test("selecting a Schedule row opens EventDetailModal", async ({ page }) => {
    await page.getByRole("button", { name: /Event day 0/ }).click();
    await expect(
      page.getByRole("dialog").filter({ hasText: "Event day 0" }),
    ).toBeVisible();
  });

  test("pages a 14-day window by exactly 7 days and back", async ({ page }) => {
    // Initial offsets are 0..13: both boundary events render and +20 does not.
    await expect(page.getByText("Event day 0", { exact: true })).toBeVisible();
    await expect(page.getByText("Event day 13", { exact: true })).toBeVisible();
    await expect(page.getByText("Event day 20", { exact: true })).toHaveCount(0);

    await page.getByRole("button", { name: "Next" }).click();

    // The next window is offsets 7..20. Day +13 proves the seven-day overlap;
    // the old lower boundary leaves and the new upper boundary enters.
    await expect(page.getByText("Event day 0", { exact: true })).toHaveCount(0);
    await expect(page.getByText("Event day 13", { exact: true })).toBeVisible();
    await expect(page.getByText("Event day 20", { exact: true })).toBeVisible();

    await page.getByRole("button", { name: "Previous" }).click();

    await expect(page.getByText("Event day 0", { exact: true })).toBeVisible();
    await expect(page.getByText("Event day 13", { exact: true })).toBeVisible();
    await expect(page.getByText("Event day 20", { exact: true })).toHaveCount(0);
  });
});
```

- [ ] **Step 2: Run**

```bash
scripts/run-calendar-e2e.sh bash -c '
  set -euo pipefail
  npm run test:e2e -- e2e/large-screen-calendar-schedule.spec.ts
  npm run test:e2e -- e2e/mobile-calendar.spec.ts
'
```

Expected: both pass. The mobile spec is the regression guard.

- [ ] **Step 3: Prove mobile parity by screenshot hash**

Add a throwaway Playwright spec that captures mobile Schedule deterministically, so both runs produce comparable bytes:

```ts
// e2e/tmp-schedule-parity.spec.ts — delete after the gate passes
import { test } from "@playwright/test";
import {
  createCalendarEvent,
  registerFamily,
  seedBrowserAuth,
} from "./helpers/api-helpers";
import {
  clearStorage,
  switchCalendarView,
  waitForCalendarReady,
  waitForHydration,
} from "./helpers/test-helpers";

async function captureSchedule({
  page,
  request,
  populated,
}: {
  page: import("@playwright/test").Page;
  request: import("@playwright/test").APIRequestContext;
  populated: boolean;
}) {
  const outputDir = process.env.PARITY_OUT_DIR;
  if (!outputDir) throw new Error("PARITY_OUT_DIR is required");
  await page.clock.setFixedTime(new Date(2026, 7, 15, 12, 0, 0));
  await page.setViewportSize({ width: 375, height: 812 });
  await page.emulateMedia({ reducedMotion: "reduce" });
  await page.goto("/");
  await clearStorage(page);

  // Each state gets a fresh browser context and family. Reusing the empty
  // query key after an API write can rehydrate its still-fresh persisted empty
  // response and produce a false populated capture.
  const reg = await registerFamily(request, {
    familyName: populated ? "Parity Populated" : "Parity Empty",
    members: [{ name: "Alice", color: "coral" }],
  });

  if (populated) {
    await createCalendarEvent(request, reg.token, {
      title: "Parity event",
      date: "2026-08-15",
      startTime: "9:00 AM",
      endTime: "10:00 AM",
      memberId: reg.family.members[0].id,
      isAllDay: false,
      location: "Kitchen",
    });
  }
  await seedBrowserAuth(page, reg);
  await page.reload();
  await waitForHydration(page);
  await waitForCalendarReady(page);
  await switchCalendarView(page, "schedule");
  if (populated) {
    await page.getByText("Parity event").waitFor();
  } else {
    await page.getByText("No upcoming events").waitFor();
  }
  await page.evaluate(() => document.fonts.ready);
  await page.evaluate(
    () => new Promise<void>((resolve) =>
      requestAnimationFrame(() => requestAnimationFrame(() => resolve())),
    ),
  );
  await page.screenshot({
    path: `${outputDir}/${populated ? "populated" : "empty"}.png`,
    fullPage: true,
  });
}

test("capture empty mobile schedule", async ({ page, request }) => {
  await captureSchedule({ page, request, populated: false });
});

test("capture populated mobile schedule", async ({ page, request }) => {
  await captureSchedule({ page, request, populated: true });
});
```

Then capture both sides and compare:

```bash
scripts/run-calendar-e2e.sh bash -c '
  set -euo pipefail
  parity_root="$(mktemp -d)"
  baseline="../schedule-parity-baseline"
  test ! -e "$baseline"
  mkdir -p "$parity_root/baseline" "$parity_root/current"

  cleanup_parity() {
    command_status=$?
    trap - EXIT INT TERM
    set +e
    cleanup_status=0
    rm -f "$baseline/e2e/tmp-schedule-parity.spec.ts" || cleanup_status=$?
    if [ -d "$baseline" ]; then
      git worktree remove "$baseline" || cleanup_status=$?
    fi
    rm -f e2e/tmp-schedule-parity.spec.ts || cleanup_status=$?
    rm -r "$parity_root" || cleanup_status=$?
    if [ "$command_status" -ne 0 ]; then exit "$command_status"; fi
    exit "$cleanup_status"
  }
  trap cleanup_parity EXIT
  trap 'exit 130' INT
  trap 'exit 143' TERM

  git worktree add "$baseline" origin/main
  cp e2e/tmp-schedule-parity.spec.ts "$baseline/e2e/"
  (
    cd "$baseline"
    npm ci
    CI=1 PARITY_OUT_DIR="$parity_root/baseline" \
      npx playwright test e2e/tmp-schedule-parity.spec.ts \
      --project="Mobile Chrome" --workers=1
  )
  CI=1 PARITY_OUT_DIR="$parity_root/current" \
    npx playwright test e2e/tmp-schedule-parity.spec.ts \
    --project="Mobile Chrome" --workers=1

  shasum -a 256 \
    "$parity_root/baseline/empty.png" "$parity_root/current/empty.png" \
    "$parity_root/baseline/populated.png" "$parity_root/current/populated.png"
  compare_status=0
  cmp -s "$parity_root/baseline/empty.png" \
    "$parity_root/current/empty.png" || compare_status=1
  cmp -s "$parity_root/baseline/populated.png" \
    "$parity_root/current/populated.png" || compare_status=1
  test "$compare_status" -eq 0
'
```

Expected: each baseline/current pair has an identical hash and both `cmp`
commands exit 0. `CI=1` disables `reuseExistingServer`, and one explicit browser
plus one worker prevents parallel projects from racing on the same files. The
populated capture covers the compact date header, event card, avatar, location,
border and FAB clearance; the empty capture covers its inline SVG/copy.

If the hashes differ, diff the images before concluding it is a regression: font loading and any animation still in flight are the usual causes.

- [ ] **Step 4: Confirm automatic cleanup**

The nested trap removes the throwaway spec, baseline worktree and temporary
images on success or failure, while the outer runner independently tears down
Compose. Confirm `git worktree list` has no `schedule-parity-baseline` entry and
`git status --short` does not list the throwaway spec.

- [ ] **Step 5: Commit**

```bash
npm run build
git add e2e/large-screen-calendar-schedule.spec.ts
git commit -m "test(calendar): add large-screen schedule E2E coverage"
```

---

## Task 17: Schedule screenshot gate and final verification

- [ ] **Step 1: Extend and run the deterministic visual harness**

In `e2e/large-screen-calendar-visual.spec.ts`, rename `monthViewports` to
`calendarViewports` and use that shared constant in the Month test and this
Schedule test. This populated fixture is the viewport, gap, long-copy, past-day
and sticky-gutter proof; every date is tied to the fixed clock:

```ts
test("Schedule viewport matrix, full-width rows, paging and sticky gutter", async ({
  page,
  request,
}) => {
  await page.clock.setFixedTime(new Date(2026, 7, 15, 12, 0, 0));
  await page.emulateMedia({ reducedMotion: "reduce" });
  await page.setViewportSize({ width: 1440, height: 900 });
  await page.goto("/");
  await clearStorage(page);

  const reg = await registerFamily(request, {
    familyName: "Schedule Visual Matrix",
    members: [
      { name: "Alice", color: "coral" },
      { name: "Bob", color: "teal" },
      { name: "Charlie", color: "purple" },
    ],
  });
  const fixtures = [
    {
      title: "A deliberately very long Schedule title that must truncate inside the event row",
      date: "2026-08-15",
      location: "A deliberately long family destination that must not cap the Schedule surface",
      member: 0,
      allDay: false,
    },
    { title: "Today extra 1", date: "2026-08-15", member: 1, allDay: false },
    { title: "Today extra 2", date: "2026-08-15", member: 2, allDay: false },
    { title: "Today extra 3", date: "2026-08-15", member: 0, allDay: false },
    { title: "Today extra 4", date: "2026-08-15", member: 1, allDay: true },
    { title: "Event day 13", date: "2026-08-28", member: 2, allDay: false },
    { title: "Event day 20", date: "2026-09-04", member: 0, allDay: false },
    { title: "Past boundary", date: "2026-08-08", member: 1, allDay: false },
  ] as const;
  for (const fixture of fixtures) {
    await createCalendarEvent(request, reg.token, {
      title: fixture.title,
      date: fixture.date,
      startTime: fixture.allDay ? "12:00 AM" : "9:00 AM",
      endTime: fixture.allDay ? "11:59 PM" : "10:00 AM",
      memberId: reg.family.members[fixture.member].id,
      isAllDay: fixture.allDay,
      location: "location" in fixture ? fixture.location : undefined,
    });
  }
  // Keep Today at exactly five events while making the 1440x900 Schedule
  // surface unambiguously scrollable for the sticky-gutter browser proof.
  for (let index = 1; index <= 14; index++) {
    await createCalendarEvent(request, reg.token, {
      title: `Day 13 extra ${index}`,
      date: "2026-08-28",
      startTime: "10:00 AM",
      endTime: "11:00 AM",
      memberId: reg.family.members[index % 3].id,
      isAllDay: false,
    });
  }

  await seedBrowserAuth(page, reg);
  await page.reload();
  await waitForHydration(page);
  await waitForCalendarReady(page);
  await switchCalendarView(page, "schedule");
  await expect(
    page.getByRole("region", { name: /Today.*5 events/i }),
  ).toBeVisible();
  await expect(
    page.getByRole("group", { name: /nothing scheduled/i }),
  ).toHaveCount(1);

  for (const viewport of calendarViewports) {
    await page.setViewportSize(viewport);
    await settleAndCapture(
      page,
      `schedule/${viewport.width}x${viewport.height}/populated.png`,
    );
  }

  await page.setViewportSize({ width: 1920, height: 1080 });
  const wideRegion = page.getByRole("region", { name: /Today.*5 events/i });
  const wideBox = await wideRegion.boundingBox();
  expect(wideBox).not.toBeNull();
  expect((wideBox as { width: number }).width).toBeGreaterThan(1700);
  const longRow = page.getByRole("button", {
    name: /A deliberately very long Schedule title/i,
  });
  expect(
    await renderedTextContrast(longRow.getByRole("heading"), "button"),
  ).toBeGreaterThanOrEqual(4.5);
  expect(
    await renderedTextContrast(
      longRow.getByTestId("schedule-event-metadata"),
      "button",
    ),
  ).toBeGreaterThanOrEqual(4.5);

  await page.setViewportSize({ width: 2560, height: 1440 });
  const widestBox = await wideRegion.boundingBox();
  expect(widestBox).not.toBeNull();
  expect((widestBox as { width: number }).width).toBeGreaterThan(2300);

  await page.setViewportSize({ width: 1440, height: 900 });
  const scroller = page.getByTestId("schedule-scroll-surface");
  const todayRegion = page.getByRole("region", { name: /Today.*5 events/i });
  const gutter = todayRegion.getByTestId("schedule-date-gutter");
  const regionBefore = await todayRegion.boundingBox();
  expect(
    await scroller.evaluate(
      (element) => element.scrollHeight > element.clientHeight,
    ),
  ).toBe(true);
  await scroller.evaluate((element) => {
    element.scrollTop = 120;
  });
  const [scrollBox, regionAfter, gutterAfter] = await Promise.all([
    scroller.boundingBox(),
    todayRegion.boundingBox(),
    gutter.boundingBox(),
  ]);
  expect(scrollBox).not.toBeNull();
  expect(regionBefore).not.toBeNull();
  expect(regionAfter).not.toBeNull();
  expect(gutterAfter).not.toBeNull();
  expect((regionAfter as { y: number }).y).toBeLessThan(
    (regionBefore as { y: number }).y - 50,
  );
  expect((gutterAfter as { y: number }).y).toBeGreaterThanOrEqual(
    (scrollBox as { y: number }).y,
  );
  await settleAndCapture(page, "schedule/1440x900/sticky-gutter.png");

  await page.getByRole("button", { name: "Previous" }).click();
  await scroller.evaluate((element) => {
    element.scrollTop = 0;
  });
  await expect(page.getByText("Past boundary", { exact: true })).toBeVisible();
  const pastRow = page.getByRole("button", { name: /Past boundary/i });
  await expect(pastRow).toBeInViewport();
  expect(await renderedOpacity(pastRow)).toBeCloseTo(1, 5);
  expect(
    await renderedTextContrast(pastRow.getByRole("heading"), "button"),
  ).toBeGreaterThanOrEqual(4.5);
  await settleAndCapture(page, "schedule/1440x900/past-after-previous.png");
});
```

Add named Schedule tests in the same file for the remaining reachable states.
Every test seeds a fresh family/query state, fixes time, reduces motion, asserts
the expected UI before capture, and uses `settleAndCapture`:

| Test / output | Deterministic fixture and pre-capture assertion |
| --- | --- |
| `schedule genuinely empty` at 1280/1440 | three-member family, zero events; assert `No upcoming events` |
| `schedule filters hide window` at 1280/1440 | Alice and Bob exist; every in-window event belongs to Alice; toggle only Alice off so Bob remains selected; assert `No events match your filters` |
| `schedule all-day filter hides window` at 1280/1440 | seed only all-day events, turn the `All Day` pill off; assert `No events match your filters` |
| `schedule loading` at 1280/1440 | hold `**/api/calendar/events?**`; assert the Schedule skeleton, capture, then release in `finally` |
| `schedule error` at 1280/1440 | fulfill that route with HTTP 500; assert the retry action |
| `schedule offline-cached` at 1280/1440 | first assert a populated Schedule, call `context.setOffline(true)`, assert the global offline banner and the cached event remain visible, capture, and restore online in `finally` |

Do not create app-level zero-member or zero-selection screenshots. Released
registration rejects an empty member list, the authenticated shell routes a
real zero-member family to onboarding, and shipped `FamilyFilterPills`
immediately reselects everyone when selection reaches zero. The Task 15
component tests are the honest evidence for those defensive branches.

Run the complete visual file again so Month and Schedule evidence comes from
the final Schedule commit:

```bash
mkdir -p .calendar-evidence/current
CALENDAR_VISUAL_OUT_DIR="$PWD/.calendar-evidence/current" \
  scripts/run-calendar-e2e.sh npx playwright test \
  e2e/large-screen-calendar-visual.spec.ts --project=chromium --workers=1
```

Expected deterministic output is under
`.calendar-evidence/current/schedule/<width>x<height>/<scenario>.png`. Attach
the files to the PR before local cleanup. Confirm the day/event-row surfaces
keep using the available width at 1920 and 2560 while only the internal event
text measure is constrained.

If screenshot review changes a tracked file, verify and commit the correction
before final verification:

```bash
npm run build
git add src e2e
git commit -m "fix(calendar): address schedule screenshot gate"
```

- [ ] **Step 2: Rerun the preservation gate after Schedule**

Run the complete Task 12 Step 2 command verbatim against the final Schedule
commit. Require all twelve baseline/current files, print both SHA-256 hashes
for each pair, and require every `cmp` to pass. This final run supersedes the
Month phase run and proves Month/Schedule and Week/Day at 375, the protected
768 boundary, and 769 stayed pixel-identical to `origin/main`.

- [ ] **Step 3: Full verification**

```bash
scripts/run-calendar-e2e.sh bash -c '
  set -euo pipefail
  npm run lint
  npm test -- --run
  npm run build
  npm run test:e2e
'
```

Expected: all four verification commands exit 0, and Compose removes the E2E
backend even when verification fails. A failed teardown also fails this step,
without masking an earlier verification failure. The runner resolves the
released backend tag in this shell, so no Task 11 export is assumed. Record the
actual test counts from the output; do not infer them from this plan.

- [ ] **Step 4: Final self-check against the acceptance criteria**

Walk the spec Section 9 acceptance criteria one by one and map each to a test name or a screenshot filename. Any criterion without evidence is not done.

- [ ] **Step 5: Push and open the PR**

```bash
git push -u origin feat/large-screen-month-schedule
gh pr create --draft --title "feat(calendar): large-screen month and schedule" --body "$(cat <<'BODY'
## Summary
- Month at lg+ fills the viewport with slot capacity derived from measured row height, one full-cell activation target, a complete event popover, and multi-day runs welded by corner geometry.
- Schedule at lg+ moves to a date gutter with full-width rows, explicit gap rows, and de-emphasised past days.
- Mobile, the 769-1023px range, Week and Day are unchanged.

## Links
- Story: https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/large-screen-ux/large-screen-calendar-month-schedule.md
- Spec: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/specs/2026-07-18-large-screen-calendar-month-schedule-design.md
- Plan: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-07-18-large-screen-calendar-month-schedule.md

## Verification
- npm run lint / npm test -- --run / npm run build / npm run test:e2e
- Screenshot matrix per spec Section 9
- Twelve compact/boundary/tablet Month/Schedule and Week/Day screenshot pairs plus the dedicated empty/populated mobile Schedule pairs matched byte-for-byte against origin/main; SHA-256 pairs attached

## Contract evidence
| Contract | Evidence |
| --- | --- |
| 1024px gating and compact/tablet preservation | breakpoint component tests; twelve origin/main screenshot hash pairs at 375/768/769 |
| measured capacity and four-row month | month-capacity/month-matrix unit tests; February 2026 screenshot |
| reserved slots, overflow count and welded geometry | month-slots/chip tests; browser rectangle assertions |
| grid keyboard/ARIA and focus transfer | monthly-calendar tests; Month Playwright tests |
| adjacent-month range and cache honesty | calendar-module/month state tests |
| full-width Schedule, gutter, gaps, member and border | Schedule unit/Playwright tests; 1920/2560 screenshots |
| preserved 14-day Schedule window and 7-day overlap | boundary-seeded Previous/Next Playwright test |
| compact Schedule byte parity | empty and populated SHA-256 pairs |
| view states and retry | Month/Schedule state tests and state screenshots |

Closes #293
BODY
)"
```

Open the PR as a draft. Before requesting review, edit the verification section
to include the actual Vitest and Playwright counts read in Step 3 and link every
screenshot attachment. No execution-contract row may remain supported only by
an assertion in prose.

After every evidence file is attached and linked from the PR, remove only the
ignored local evidence directory and confirm the worktree is otherwise clean:

```bash
rm -r .calendar-evidence
git status --short
```

---

## Execution Contract

Copy this into the frontend implementation Issue.

**Non-negotiable:**

1. Everything is gated at 1024px via `useIsLargeScreen()`. Mobile (<= 768px), the 769-1023px range, Week and Day must be visually and behaviourally unchanged.
2. `ScheduleCalendar` is shared with mobile and is the mobile smart default. Its compact path must stay byte-for-byte as it ships today, proven by screenshot hash against `origin/main`.
3. Capacity is counted in **slots**, not events, because reserved blank slots occupy rows. `+N more` counts events only. The tested reserved-lane edge may show a blank plus `+N` with no event chip; the full gridcell/popover path remains complete.
4. `MONTH_CHIP_HEIGHT = 28` is a visual floor, not a tuning dial. Chips and `+N` are presentational, never small controls; the 96px-or-taller day gridcell is their one >=44px target.
5. Layout arithmetic lives in pure exported helpers with direct unit tests. The repo's Vitest setup mocks `ResizeObserver` as a no-op and `matchMedia` as `matches: false` for every query, so measured-geometry assertions must be Playwright, not Vitest.
6. The Month day cell is a `role="gridcell"` div, never a `<button>`. The grid owns only the weekday `row` and observed weeks `rowgroup`; every gridcell is owned by a week row.
7. The grid is a single tab stop. Dense chips and `+N` are `aria-hidden` visual content. `Enter` and `Space` activate the cell; `Space` prevents scrolling.
8. Observe the weeks rowgroup through a callback ref. Capacity CSS and arithmetic share the exact border, padding, header, 8px row/column gaps and 28px-slot constants; the corresponding weld bleed is 9px per active side and cells do not clip it. Browser rectangles prove inner segments touch and row edges do not overflow.
9. Escape dismissal returns focus to the day cell; an outside pointer dismissal leaves focus on the newly clicked target. Event selection closes without restoring cell focus so `EventDetailModal` can take it; closing that modal returns to the originating cell. Day navigation does not focus an unmounted grid.
10. Colour is never the only channel: Schedule shows the member name, every Month chip visibly prefixes the member's first name, and the Month cell exposes its dot summary with `aria-describedby`; popover rows carry the complete event/member details. No information is conveyed by opacity alone. Missing-member chips use `bg-muted text-foreground`, never white on the light muted token. Adjacent-month emphasis changes only cell chrome/date styling, never event-content opacity. WCAG AA is verified in the app's currently supported light theme; adding dark-theme infrastructure is out of scope.
11. Month's widened query range is `lg+`-only. Query identity comes from the calculated date parameters, not the breakpoint boolean. Offline cold cache is based on absent query data, never `events.length === 0`, and no cold-cache copy may render until persisted-query restoration completes.
12. Schedule day groups and rows have no fixed outer width cap through 2560px; only event text gets a readable internal measure. Schedule event titles are 20px and secondary metadata remains at least 14px.
13. Schedule empty states distinguish zero family members, active filters hiding every event in rendered offsets 0 through 13, and a genuinely empty window. A zero-selection branch exists defensively, but the shipped cross-view auto-reset makes persistent zero selection unreachable in this story. The query-only fifteenth boundary day does not affect the reason.
14. New animations opt out under reduced-motion preferences; the screenshot and parity gates run with reduced motion.
15. Pair every `setViewportWidth` with `resetViewportWidth`; reset both `matchMedia` and `window.innerWidth`.
16. Use the released bare-semver backend image, `docker compose up -d --wait`, and guaranteed teardown. `npm run build` passes after the final edit and before every commit.
17. Ship in two phases on one isolated-worktree branch, Month first; Schedule begins only after the Month screenshot gate passes.

**Known limitations to preserve, not fix:** Schedule's 14-day window with a
7-day paging step and constant `"Upcoming"` label; multi-day slot alignment at
a week boundary; the reserved-lane overflow-only visual edge; compact
Schedule's existing grey border; adjacent-month cells below 1024px; the
cross-view empty-selection auto-reset; and app-wide dark-theme infrastructure.
All are recorded in spec Section 11.
