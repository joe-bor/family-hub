# Large-Screen Calendar Month and Schedule Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** At 1024px and above, make Calendar Month fill its viewport with density derived from measured row height and a working overflow path, and give Schedule a date-gutter composition that uses the width it has — without changing mobile, the 769-1023px range, Week, or Day.

**Architecture:** All layout arithmetic lives in pure, separately-tested helpers (`month-capacity.ts`, `month-slots.ts`, `schedule-rows.ts`) because the repo's Vitest environment cannot exercise measured geometry. Components consume those helpers and are responsible only for rendering and interaction. Every behavioural change is gated on `useIsLargeScreen()`; below 1024px both views take their existing code paths untouched.

**Tech Stack:** React 19, TypeScript, Vite, Tailwind CSS v4, Radix UI (`@radix-ui/react-popover`), TanStack Query, Zustand, Vitest + Testing Library, Playwright.

**Spec:** `docs/superpowers/specs/2026-07-18-large-screen-calendar-month-schedule-design.md`
**Story:** `docs/product/backlog/large-screen-ux/large-screen-calendar-month-schedule.md`

---

## Critical environment facts

Read these before Task 1. They change how tests must be written.

1. **`matchMedia` is mocked to `matches: false` for every query** in `src/test/setup.ts`. `useIsLargeScreen()` and `useIsMobile()` are therefore both `false` in every unit test unless the test overrides it. Task 1 promotes an existing working override into a shared helper.
2. **`ResizeObserver` is a no-op mock** and jsdom has no layout engine. Any assertion that depends on a measured height is unreachable in Vitest. Capacity and slot planning are proven on pure helpers; observer-to-render wiring is proven in Playwright only.
3. **Type errors are caught only by `npm run build`.** `npm run lint` and `npx tsc --noEmit` both miss them (the root tsconfig has `files: []`, and `tsconfig.app` excludes tests). Run `npm run build` before every commit.
4. **`resetAllStores()` in `src/test/setup.ts` resets stores by name.** No new store is added by this plan, so nothing needs adding there.
5. **All work happens in the frontend repo** at `/Users/joe.bor/code/family-hub/frontend`, on one branch, in **two phases** — Month, then Schedule — each ending at its own screenshot gate. Commit frequently within a phase; the phase boundary, not the commit count, is what provides the risk isolation. Schedule work does not begin until the Month gate (Task 12) passes.

---

## File Structure

**Create:**

| Path | Responsibility |
| --- | --- |
| `src/components/calendar/utils/month-matrix.ts` | `buildMonthMatrix`, `selectMonthDayDots` (moved from `day-rail.ts`) |
| `src/components/calendar/utils/month-matrix.test.ts` | Tests moved from `day-rail.test.ts` |
| `src/components/calendar/utils/month-capacity.ts` | Layout constants, `monthRowHeight`, `monthSlotCapacity` |
| `src/components/calendar/utils/month-capacity.test.ts` | Capacity formula + invariants |
| `src/components/calendar/utils/month-slots.ts` | `isMultiDay`, `multiDayEdge`, `orderRowMultiDay`, `planCellSlots` |
| `src/components/calendar/utils/month-slots.test.ts` | Ordering, reservation, overflow counting |
| `src/components/calendar/utils/schedule-rows.ts` | `buildScheduleRows` — day rows and gap runs |
| `src/components/calendar/utils/schedule-rows.test.ts` | Gap detection, boundaries, filtering |
| `src/components/calendar/components/month-event-chip.tsx` | One chip: edge geometry, markers, accessible name |
| `src/components/calendar/components/month-day-cell.tsx` | One cell: numeral, dot summary, slots, `+N more` |
| `src/components/calendar/components/month-overflow-popover.tsx` | Per-day full event list |
| `src/components/calendar/components/month-day-cell.test.tsx` | Cell rendering + a11y names |
| `src/components/calendar/components/month-overflow-popover.test.tsx` | Popover contents + focus return |
| `e2e/large-screen-calendar-month.spec.ts` | Month E2E against the real backend |
| `e2e/large-screen-calendar-schedule.spec.ts` | Schedule E2E against the real backend |

**Modify:**

| Path | Change |
| --- | --- |
| `src/test/test-utils.tsx` | Add `setViewportWidth` helper |
| `src/components/calendar/utils/day-rail.ts` | Remove the two moved functions; keep rail constants |
| `src/components/calendar/utils/day-rail.test.ts` | Remove the two moved test cases |
| `src/components/calendar/components/day-mini-month-rail.tsx` | Update import path |
| `src/components/calendar/components/index.ts` | Export new components |
| `src/components/calendar/views/monthly-calendar.tsx` | Full lg+ rewrite; tablet path preserved |
| `src/components/calendar/views/monthly-calendar.test.tsx` | New file (none exists today) |
| `src/components/calendar/views/schedule-calendar.tsx` | lg+ gutter composition; border fix; states |
| `src/components/calendar/views/schedule-calendar.test.tsx` | Migrate banner assertions to lg+/mobile blocks |
| `src/components/calendar/calendar-module.tsx` | lg+-gated Month query range |

---

## Task 0: Branch setup

- [ ] **Step 1: Cut the branch from fresh origin/main**

```bash
cd /Users/joe.bor/code/family-hub/frontend
git fetch origin
git checkout -b feat/large-screen-month-schedule origin/main
git log --oneline -1
```

Expected: HEAD matches `origin/main`'s newest commit. Do not skip `git fetch` — a stale local `main` has previously caused a session to mis-read merge state on this epic.

- [ ] **Step 2: Confirm a clean baseline**

```bash
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

Every describe block in this plan that calls `setViewportWidth` must pair it with `afterEach(resetViewportWidth)`. `calendar-module.test.tsx` already does this by hand at `:768` and `:843`; the new helper generalises that pattern.

- [ ] **Step 2: Delegate the existing local helper**

In `src/components/calendar/views/schedule-calendar.test.tsx`, delete the local `setMobile` function body and replace it with a delegation, importing `setViewportWidth` from `@/test/test-utils`:

```tsx
function setMobile(isMobile: boolean) {
  setViewportWidth(isMobile ? 390 : 1024);
}
```

- [ ] **Step 3: Verify existing tests still pass**

```bash
npm test -- --run src/components/calendar/views/schedule-calendar.test.tsx
```

Expected: 7 passed. Behaviour is identical — this is a pure extraction.

- [ ] **Step 4: Commit**

```bash
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

Move the two cases at `src/components/calendar/utils/day-rail.test.ts:44` (`"builds a 6x7 (or 5x7) month matrix covering the current month"`) and `:55` (the `selectMonthDayDots` case) into a new `src/components/calendar/utils/month-matrix.test.ts`, importing from `./month-matrix`. Leave the `railThresholdPx` cases in `day-rail.test.ts`.

Rename the moved matrix test to reflect reality, since a four-row month exists:

```ts
it("builds a 4x7, 5x7 or 6x7 month matrix covering the current month", () => {
```

- [ ] **Step 5: Add `selectMonthDayMembers`**

Month needs per-day member **names** as well as colours, for the visually-hidden dot summary in Task 7. `selectMonthDayDots` returns colours only, so add the richer function and reduce the existing one to a wrapper over it — this is also what justifies the move, since otherwise Month would consume neither function.

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
    expect(monthRowHeight(744, 4)).toBe(186);
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
 * exception: MONTH_CHIP_HEIGHT is a floor, not a dial. See spec Section 7.
 */

/** Vertical space reserved for the date numeral row. */
export const MONTH_NUMERAL_BLOCK = 20;
/** Combined top and bottom cell padding. */
export const MONTH_CELL_PADDING_Y = 10;
/**
 * Chip height floor. Matches the shipped Week view's `min-h-[28px]` event
 * buttons. The 44px touch-target rule is scoped to chrome and controls; in-grid
 * chips are the documented exception because 44px chips would yield fewer
 * events per cell than ships today. Do not reduce this below 28.
 */
export const MONTH_CHIP_HEIGHT = 28;
/** Vertical gap between chips. */
export const MONTH_CHIP_GAP = 2;
/** Row height floor; guarantees monthSlotCapacity() >= 2. */
export const MONTH_MIN_ROW_HEIGHT = 96;
/** Vertical gap between week rows. */
export const MONTH_ROW_GAP = 4;

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
  const usable = rowHeight - MONTH_NUMERAL_BLOCK - MONTH_CELL_PADDING_Y;
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

Expected: PASS, 8 tests.

- [ ] **Step 5: Commit**

```bash
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
 * The result is truncated to `capacity`, reserving the last slot for the
 * overflow control when anything is hidden.
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

Expected: PASS, 17 tests. Build exits 0.

- [ ] **Step 5: Commit**

```bash
git add src/components/calendar/utils/month-slots.ts src/components/calendar/utils/month-slots.test.ts
git commit -m "feat(calendar): add month slot planning with multi-day reservation"
```

---

## Task 5: Month event chip

**Files:**
- Create: `src/components/calendar/components/month-event-chip.tsx`
- Modify: `src/components/calendar/components/index.ts`

- [ ] **Step 1: Implement the chip**

Create `src/components/calendar/components/month-event-chip.tsx`:

```tsx
import { Repeat } from "lucide-react";
import { useFamilyMembers } from "@/api";
import { type CalendarEvent, colorMap, getFamilyMember } from "@/lib/types";
import { cn } from "@/lib/utils";
import { MONTH_CHIP_HEIGHT } from "../utils/month-capacity";
import type { MonthChipEdge } from "../utils/month-slots";
import { GoogleBadge, isGoogleEvent } from "./calendar-event";

/**
 * Negative horizontal margin so adjacent chips physically touch across the
 * cell padding and grid gap. Without this the weld shows a seam at every cell
 * boundary. Must equal the cell's horizontal padding plus half the grid gap.
 */
const BLEED_PX = 4;

interface MonthEventChipProps {
  event: CalendarEvent;
  edge: MonthChipEdge;
  /** Position within the run, 1-based. Only set for multi-day events. */
  dayOfSpan?: { day: number; total: number };
  onClick: () => void;
}

export function MonthEventChip({
  event,
  edge,
  dayOfSpan,
  onClick,
}: MonthEventChipProps) {
  const familyMembers = useFamilyMembers();
  const member = getFamilyMember(familyMembers, event.memberId);
  const colors = member ? colorMap[member.color] : undefined;

  const timeLabel = event.isAllDay
    ? "all day"
    : `${event.startTime} to ${event.endTime}`;
  const memberLabel = member ? `, ${member.name}` : "";
  const spanLabel = dayOfSpan
    ? `, day ${dayOfSpan.day} of ${dayOfSpan.total}`
    : "";
  const recurringLabel = event.isRecurring ? ", repeats" : "";

  return (
    <button
      type="button"
      // Not a tab stop: the grid is a single tab stop and keyboard users reach
      // events through the day's overflow popover. See spec Section 4.5.
      tabIndex={-1}
      onClick={(e) => {
        e.stopPropagation();
        onClick();
      }}
      aria-label={`${event.title}, ${timeLabel}${memberLabel}${spanLabel}${recurringLabel}`}
      className={cn(
        "flex w-full items-center gap-1 px-1.5 text-left text-xs font-medium text-white",
        colors?.bg ?? "bg-muted",
        edge === "solo" && "rounded",
        edge === "start" && "rounded-l",
        edge === "end" && "rounded-r",
      )}
      style={{
        height: MONTH_CHIP_HEIGHT,
        marginLeft: edge === "start" || edge === "solo" ? 0 : -BLEED_PX,
        marginRight: edge === "end" || edge === "solo" ? 0 : -BLEED_PX,
      }}
    >
      {event.isAllDay && (
        <span
          aria-hidden="true"
          className="size-1.5 shrink-0 rounded-full bg-white/80"
        />
      )}
      {isGoogleEvent(event) && (
        <span className="shrink-0" aria-hidden="true">
          <GoogleBadge size={8} />
        </span>
      )}
      <span className="truncate">{event.title}</span>
      {event.isRecurring && (
        <Repeat aria-hidden="true" className="ml-auto size-3 shrink-0" />
      )}
    </button>
  );
}
```

Note: the all-day dot replaces the `"● "` string prefix that is concatenated into the title today. Do not reintroduce it.

- [ ] **Step 2: Export it**

Add to `src/components/calendar/components/index.ts`, keeping alphabetical order:

```ts
export { MonthEventChip } from "./month-event-chip";
```

- [ ] **Step 3: Typecheck**

```bash
npm run build
```

Expected: exits 0.

- [ ] **Step 4: Commit**

```bash
git add src/components/calendar/components/month-event-chip.tsx src/components/calendar/components/index.ts
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
import { describe, expect, it, vi } from "vitest";
import { createTestEvent, testMembers } from "@/test/fixtures";
import { render, screen, seedFamilyStore } from "@/test/test-utils";
import { MonthOverflowPopover } from "./month-overflow-popover";

const date = new Date(2026, 2, 8);

function setup(eventCount: number, open = true) {
  seedFamilyStore({ name: "Test Family", members: testMembers });
  const events = Array.from({ length: eventCount }, (_, i) =>
    createTestEvent({
      id: `e${i}`,
      title: `Event ${i}`,
      date,
      memberId: testMembers[0].id,
    }),
  );
  const onEventClick = vi.fn();
  const onOpenDay = vi.fn();
  const onOpenChange = vi.fn();
  const onCloseFocus = vi.fn();

  render(
    <MonthOverflowPopover
      date={date}
      events={events}
      open={open}
      onOpenChange={onOpenChange}
      onEventClick={onEventClick}
      onOpenDay={onOpenDay}
      onCloseFocus={onCloseFocus}
    >
      <div data-testid="anchor">anchor</div>
    </MonthOverflowPopover>,
  );
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
    expect(
      screen.getByRole("dialog", { name: /events for march 8, 2026/i }),
    ).toBeInTheDocument();
  });

  it("offers an open-in-day-view action", async () => {
    const user = userEvent.setup();
    const { onOpenDay } = setup(6);

    await user.click(screen.getByRole("button", { name: /open in day view/i }));

    expect(onOpenDay).toHaveBeenCalledWith(date);
  });

  it("opens the detail modal for a selected event", async () => {
    const user = userEvent.setup();
    const { onEventClick } = setup(6);

    await user.click(screen.getByRole("button", { name: /Event 3/ }));

    expect(onEventClick).toHaveBeenCalledTimes(1);
    expect(onEventClick.mock.calls[0][0].title).toBe("Event 3");
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
import { format } from "date-fns";
import type { ReactNode } from "react";
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
  /** Focus target on close; both open paths return focus to the day cell. */
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

  return (
    // Anchor-based, deliberately trigger-less. The popover must be openable on
    // ANY day that has events — keyboard users reach every event through it —
    // while the visible "+N more" button only exists on overflowing days. Using
    // PopoverTrigger would tie the two together and make Enter dead on a day
    // with one event. `open` is always a boolean, never undefined, so the
    // Radix instance is unambiguously controlled.
    <Popover open={open} onOpenChange={onOpenChange}>
      <PopoverAnchor asChild>{children}</PopoverAnchor>
      <PopoverContent
        align="start"
        aria-label={`Events for ${format(date, "MMMM d, yyyy")}`}
        // Radix returns focus to its Trigger by default, but the popover has
        // two triggers (this button for pointer users, the cell for keyboard
        // users) and one focus-return target. See spec Section 4.4.
        onCloseAutoFocus={(event) => {
          if (!onCloseFocus) return;
          event.preventDefault();
          onCloseFocus();
        }}
      >
        <p className="mb-2 text-sm font-semibold">
          {format(date, "EEEE, MMMM d")}
        </p>
        <ul className="flex flex-col gap-1">
          {events.map((event) => {
            const member = getFamilyMember(familyMembers, event.memberId);
            const colors = member ? colorMap[member.color] : undefined;
            return (
              <li key={getEventKey(event)}>
                <button
                  type="button"
                  onClick={() => onEventClick(event)}
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
                    <span className="block text-xs text-muted-foreground">
                      {event.isAllDay
                        ? "All day"
                        : `${event.startTime} – ${event.endTime}`}
                      {member ? ` · ${member.name}` : ""}
                    </span>
                  </span>
                </button>
              </li>
            );
          })}
        </ul>
        <button
          type="button"
          onClick={() => onOpenDay(date)}
          className="mt-2 min-h-11 w-full rounded-lg border border-border text-sm font-medium hover:bg-accent"
        >
          Open in Day view
        </button>
      </PopoverContent>
    </Popover>
  );
}
```

The `+N more` button is **not** part of this component — it lives in `MonthDayCell` (Task 7) as a plain button that calls `onOpenChange(true)`. That separation is what lets a non-overflowing day still open the popover from the keyboard.

- [ ] **Step 4: Run to verify it passes**

```bash
npm test -- --run src/components/calendar/components/month-overflow-popover.test.tsx
npm run build
```

Expected: PASS, 5 tests. Build exits 0.

- [ ] **Step 5: Export and commit**

Add `export { MonthOverflowPopover } from "./month-overflow-popover";` to `src/components/calendar/components/index.ts`.

```bash
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
import { describe, expect, it, vi } from "vitest";
import { createTestEvent, testMembers } from "@/test/fixtures";
import { render, screen, seedFamilyStore } from "@/test/test-utils";
import { MonthDayCell } from "./month-day-cell";

const date = new Date(2026, 2, 8);

function renderCell(overrides: Partial<Parameters<typeof MonthDayCell>[0]> = {}) {
  seedFamilyStore({ name: "Test Family", members: testMembers });
  const props = {
    date,
    plan: { slots: [], overflowCount: 0 },
    allEvents: [],
    memberColors: [],
    memberNames: [],
    isToday: false,
    isFocused: false,
    isOutsideMonth: false,
    isWeekend: false,
    rowHeight: 124,
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
  it("is a gridcell, not a button, so chips can nest inside it", () => {
    renderCell();
    const cell = screen.getByRole("gridcell");
    expect(cell).toBeInTheDocument();
    expect(cell.tagName).not.toBe("BUTTON");
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
    expect(screen.getByRole("gridcell")).toHaveAccessibleName(/outside/i);
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
  });

  it("uses singular phrasing for one member", () => {
    renderCell({ memberColors: ["coral"], memberNames: ["John"] });
    expect(screen.getByText("John has events")).toBeInTheDocument();
  });

  it("renders a blank placeholder for a reserved slot", () => {
    renderCell({
      plan: { slots: [{ kind: "blank" }], overflowCount: 0 },
    });
    expect(screen.getByTestId("month-slot-blank")).toBeInTheDocument();
  });

  it("opens the popover from the +N more button", async () => {
    const user = userEvent.setup();
    const props = renderCell({
      plan: { slots: [], overflowCount: 3 },
      allEvents: [createTestEvent({ id: "a", title: "Soccer", date })],
    });

    await user.click(screen.getByRole("button", { name: /show all/i }));

    expect(props.onPopoverOpenChange).toHaveBeenCalledWith(true);
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

Add `import userEvent from "@testing-library/user-event";` to this test file.

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
import { getEventKey } from "@/lib/time-utils";
import {
  MONTH_CHIP_GAP,
  MONTH_CHIP_HEIGHT,
} from "../utils/month-capacity";
import { isMultiDay, type MonthCellPlan } from "../utils/month-slots";
import { MonthEventChip } from "./month-event-chip";
import { MonthOverflowPopover } from "./month-overflow-popover";

interface MonthDayCellProps {
  date: Date;
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
  onSelectDay: (date: Date) => void;
  onEventClick: (event: CalendarEvent) => void;
  onFocusDay: (date: Date) => void;
  onKeyDown: (event: React.KeyboardEvent<HTMLDivElement>, date: Date) => void;
  popoverOpen?: boolean;
  onPopoverOpenChange?: (open: boolean) => void;
}

function memberSummary(names: string[]): string {
  if (names.length === 1) return `${names[0]} has events`;
  const joined = `${names.slice(0, -1).join(", ")} and ${names[names.length - 1]}`;
  return `${joined} have events`;
}

/**
 * Position within a multi-day run, for the chip's accessible name. Returns
 * undefined on the run's first day, where the plain title is unambiguous —
 * announcing "day 1 of 5" there is noise (spec Section 4.3).
 */
function dayOfSpan(event: CalendarEvent, date: Date) {
  if (!isMultiDay(event)) return undefined;
  const start = event.date;
  const end = event.endDate as Date;
  const dayMs = 24 * 60 * 60 * 1000;
  const total = Math.round((end.getTime() - start.getTime()) / dayMs) + 1;
  const day = Math.round((date.getTime() - start.getTime()) / dayMs) + 1;
  if (day <= 1) return undefined;
  return { day, total };
}

export function MonthDayCell({
  date,
  plan,
  allEvents,
  memberColors,
  memberNames,
  isToday,
  isFocused,
  isOutsideMonth,
  isWeekend,
  rowHeight,
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
    ? `, outside ${format(date, "MMMM")}`
    : "";

  return (
    <div
      ref={cellRef}
      // Deliberately a gridcell rather than a button: event chips inside are
      // buttons, and a button may not contain a button. See spec Section 4.5.
      role="gridcell"
      tabIndex={isFocused ? 0 : -1}
      aria-current={isToday ? "date" : undefined}
      aria-label={`${format(date, "MMMM d, yyyy")}, ${countLabel}${outsideLabel}`}
      data-date={format(date, "yyyy-MM-dd")}
      onClick={() => onSelectDay(date)}
      onFocus={() => onFocusDay(date)}
      onKeyDown={(event) => onKeyDown(event, date)}
      className={cn(
        "flex cursor-pointer flex-col overflow-hidden rounded-lg border px-1 py-1 transition-colors",
        "focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary focus-visible:ring-offset-1",
        isWeekend ? "bg-muted/40" : "bg-card",
        isOutsideMonth && "opacity-50",
        isToday
          ? "border-primary bg-primary/10"
          : "border-border/50 hover:bg-accent/50",
        // Today's own ring, and the selected ring, must both be able to show
        // on the same cell — spec 4.7 requires them separable when they
        // coincide. Today uses an inset ring, selected an offset outline.
        isToday && "ring-1 ring-primary ring-inset",
        isFocused && "outline outline-2 outline-offset-[-3px] outline-ring",
      )}
      style={{ height: rowHeight, gap: MONTH_CHIP_GAP }}
    >
      <div className="flex shrink-0 items-center justify-between">
        <span
          className={cn(
            "text-sm font-semibold tabular-nums",
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
            <span className="sr-only">{memberSummary(memberNames)}</span>
          </span>
        )}
      </div>

      <div className="flex min-h-0 flex-col" style={{ gap: MONTH_CHIP_GAP }}>
        {plan.slots.map((slot, index) =>
          slot.kind === "blank" || !slot.event ? (
            <div
              // biome-ignore lint/suspicious/noArrayIndexKey: blanks are positional
              key={`blank-${index}`}
              data-testid="month-slot-blank"
              aria-hidden="true"
              style={{ height: MONTH_CHIP_HEIGHT }}
            />
          ) : (
            <MonthEventChip
              key={getEventKey(slot.event)}
              event={slot.event}
              edge={slot.edge ?? "solo"}
              dayOfSpan={dayOfSpan(slot.event, date)}
              onClick={() => onEventClick(slot.event as CalendarEvent)}
            />
          ),
        )}

        {plan.overflowCount > 0 && (
          <button
            type="button"
            // Not a tab stop: otherwise every overflowing day adds one and the
            // grid stops being a single tab stop (spec Section 4.5).
            tabIndex={-1}
            onClick={(event) => {
              event.stopPropagation();
              onPopoverOpenChange?.(true);
            }}
            aria-label={`Show all ${allEvents.length} events for ${format(date, "MMMM d")}`}
            className="w-full rounded px-1.5 text-left text-xs font-semibold text-primary hover:bg-accent"
            style={{ height: MONTH_CHIP_HEIGHT }}
          >
            +{plan.overflowCount} more
          </button>
        )}
      </div>
    </div>
  );
}
```

**Wrapping the cell in the popover.** Assemble the return value so that the popover anchors to the cell, and mount it only for days that actually have events — a month has up to 42 cells and most are empty:

```tsx
  // ...`cell` is the <div role="gridcell"> element above.
  if (allEvents.length === 0) return cell;

  return (
    <MonthOverflowPopover
      date={date}
      events={allEvents}
      open={popoverOpen ?? false}
      onOpenChange={(open) => onPopoverOpenChange?.(open)}
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

Expected: PASS, 9 tests. Build exits 0.

- [ ] **Step 5: Export and commit**

Add `export { MonthDayCell } from "./month-day-cell";` to the components barrel.

```bash
git add src/components/calendar/components
git commit -m "feat(calendar): add month day cell with gridcell semantics"
```

---

## Task 8: Month grid — sizing, ARIA, keyboard

Rewrite `MonthlyCalendar` so that at `lg+` it renders the ARIA grid with measured row heights, and below `lg` it renders exactly what it renders today.

**Files:**
- Modify: `src/components/calendar/views/monthly-calendar.tsx`

- [ ] **Step 1: Preserve the tablet path**

Extract the current JSX body (everything returned today) into a local component `MonthlyCalendarCompact` in the same file, taking the same props. Do not change its markup, classes, or the `eventsToShow` value of 3.

Replace `eventsToShow` with the literal `3` and **delete the `useIsMobile` import as well as the call** — `tsconfig.app.json` sets `"noUnusedLocals": true`, so leaving an unused import fails `npm run build`. The `isMobile` branch is dead code because this component never renders at or below 768px.

- [ ] **Step 2: Add the imports the large component needs**

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
  isEventOnDate,
} from "@/lib/time-utils";
import { cn } from "@/lib/utils";
import { MonthDayCell } from "../components/month-day-cell";
import {
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

- [ ] **Step 3: Add the large-screen grid**

```tsx
export function MonthlyCalendar(props: MonthlyCalendarProps) {
  const isLargeScreen = useIsLargeScreen();
  if (!isLargeScreen) return <MonthlyCalendarCompact {...props} />;
  return <MonthlyCalendarLarge {...props} />;
}
```

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
     className="flex min-h-0 flex-1 flex-col overflow-auto p-4">
  <div role="row" className="grid shrink-0 grid-cols-7"
       style={{ gap: MONTH_ROW_GAP }}>
    {WEEKDAY_LABELS.map((label) => (
      <div key={label} role="columnheader"
           className="py-1 text-center text-sm font-medium text-muted-foreground">
        {label}
      </div>
    ))}
  </div>

  <div ref={setWeeksEl} className="flex min-h-0 flex-1 flex-col"
       style={{ gap: MONTH_ROW_GAP }}>
    {weeks.map((week) => (
      <div key={formatLocalDate(week[0])} role="row"
           className="grid grid-cols-7" style={{ gap: MONTH_ROW_GAP }}>
        {week.map((day) => { /* cell derivation below */ })}
      </div>
    ))}
  </div>
</div>
```

Per week and day:

```tsx
const rowMultiDay = orderRowMultiDay(visibleEvents, week);
// ...then per day:
const dayEvents = visibleEvents
  .filter((event) => isEventOnDate(event, day))
  .sort(compareEventsAllDayFirst);
const singleDayEvents = dayEvents.filter((event) => !isMultiDay(event));
const plan = planCellSlots({ rowMultiDay, day, singleDayEvents, capacity });
const members = dayMembers.get(day.toDateString()) ?? [];

<MonthDayCell
  key={formatLocalDate(day)}
  date={day}
  plan={plan}
  allEvents={dayEvents}
  memberColors={members.map((m) => m.color)}
  memberNames={members.map((m) => m.name)}
  isToday={day.toDateString() === new Date().toDateString()}
  isFocused={formatLocalDate(day) === formatLocalDate(focusedDate)}
  isOutsideMonth={day.getMonth() !== currentDate.getMonth()}
  isWeekend={day.getDay() === 0 || day.getDay() === 6}
  rowHeight={rowHeight}
  onSelectDay={handleSelectDay}
  onEventClick={handleEventClick}
  onFocusDay={setFocusedDate}
  onKeyDown={handleKeyDown}
  popoverOpen={
    openPopoverDate !== null &&
    formatLocalDate(openPopoverDate) === formatLocalDate(day)
  }
  onPopoverOpenChange={(open) => setOpenPopoverDate(open ? day : null)}
/>
```

Define `const WEEKDAY_LABELS = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"] as const;` at module scope.

`onDateSelect` is optional on `MonthlyCalendarProps` but `onSelectDay` is required on `MonthDayCell`, so wrap rather than passing through directly:

```tsx
const handleSelectDay = (date: Date) => onDateSelect?.(date);
const handleEventClick = (event: CalendarEvent) => onEventClick?.(event);
```

- [ ] **Step 4: Add roving tabindex and keyboard navigation**

```tsx
const [focusedDate, setFocusedDate] = useState<Date>(currentDate);
const [openPopoverDate, setOpenPopoverDate] = useState<Date | null>(null);

// Keep the focused day inside the visible grid when the month changes.
useEffect(() => {
  setFocusedDate((previous) =>
    days.some((day) => day.toDateString() === previous.toDateString())
      ? previous
      : currentDate,
  );
}, [days, currentDate]);

const pendingFocus = useRef(false);

// Restore focus after a month change re-renders the grid, so crossing a grid
// edge does not dump focus back to the first cell.
useEffect(() => {
  if (!pendingFocus.current) return;
  pendingFocus.current = false;
  weeksEl
    ?.querySelector<HTMLElement>(`[data-date="${formatLocalDate(focusedDate)}"]`)
    ?.focus();
});

// Membership in the rendered matrix, not month equality. March 2026 renders
// Apr 1-4 as trailing cells; arrowing onto one of those must move focus within
// the existing grid, not re-page the whole month — otherwise the adjacent-month
// events fetched in Task 9 are unreachable by keyboard.
const matrixKeys = useMemo(
  () => new Set(days.map((day) => formatLocalDate(day))),
  [days],
);

const moveFocus = (next: Date) => {
  pendingFocus.current = true;
  setFocusedDate(next);
  if (!matrixKeys.has(formatLocalDate(next))) onMonthChange?.(next);
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
    const hasEvents = visibleEvents.some((e) => isEventOnDate(e, date));
    if (hasEvents) setOpenPopoverDate(date);
    else onDateSelect?.(date);
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
    moveFocus(subMonths(date, 1));
  } else if (event.key === "PageDown") {
    event.preventDefault();
    moveFocus(addMonths(date, 1));
  }
};
```

Add `onMonthChange?: (date: Date) => void` to `MonthlyCalendarProps`.

- [ ] **Step 5: Typecheck and run the full suite**

```bash
npm run build
npm test -- --run
```

Expected: build exits 0; no previously-passing test regresses.

- [ ] **Step 6: Commit**

```bash
git add src/components/calendar/views/monthly-calendar.tsx
git commit -m "feat(calendar): fill viewport in large-screen month grid"
```

---

## Task 9: Large-screen Month query range

**Files:**
- Modify: `src/components/calendar/calendar-module.tsx:173-203`

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
    />
  );
```

- [ ] **Step 3: Add a regression test for the gate**

Append to `src/components/calendar/calendar-module.test.tsx`, inside a new describe block:

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
});
```

Import `server` from the file's existing MSW setup — reuse whichever import `calendar-module.test.tsx` already uses at the top rather than adding a new one. `seedCalendarStore` comes from `@/test/test-utils`.

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

## Task 10: Month and Schedule view states

`renderCalendarView()` currently returns the shared centred "Loading events..." / error text **before** the view switch is reached, so a per-view state inside `MonthlyCalendar` would never render. The early return has to become conditional first.

**Files:**
- Modify: `src/components/calendar/calendar-module.tsx:465-484`
- Modify: `src/components/calendar/views/monthly-calendar.tsx`
- Create: `src/components/calendar/components/calendar-view-states.tsx`

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

```tsx
case "monthly":
  return (
    <MonthlyCalendar
      {...commonProps}
      onDateSelect={selectDateAndSwitchToDaily}
      onMonthChange={setDate}
      isLoading={isLoading}
      isError={isError}
      errorMessage={error?.message}
      onRetry={refetch}
    />
  );
```

`commonProps` is spread into all three `ScheduleCalendar` call sites (`calendar-module.tsx:524`, `:526`, `:552`), two of which are the mobile branch — routing state props through it would push new props onto the mobile component this story must leave untouched. Do not introduce an unused intermediate object either: `tsconfig.app.json` sets `"noUnusedLocals": true`, so a declared-but-unwired `viewStateProps` would fail `npm run build`.

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

In `monthly-calendar.tsx`, render in this priority order inside `MonthlyCalendarLarge`:

1. `isError` → `<CalendarErrorState message={errorMessage} onRetry={onRetry} />`
2. `isLoading` → the skeleton below
3. `!isLoading && events.length === 0 && !navigator.onLine` → `<CalendarOfflineState message="This month isn't cached yet." />` — the cold-cache case created by the breakpoint-gated query key (spec Section 6)
4. otherwise the grid

```tsx
function MonthGridSkeleton({
  weekCount,
  rowHeight,
}: {
  weekCount: number;
  rowHeight: number;
}) {
  return (
    <div
      role="status"
      aria-label="Loading month"
      className="flex min-h-0 flex-1 flex-col gap-1 p-4"
    >
      {Array.from({ length: weekCount }).map((_, week) => (
        // biome-ignore lint/suspicious/noArrayIndexKey: static skeleton
        <div key={week} className="grid grid-cols-7 gap-1" >
          {Array.from({ length: 7 }).map((__, day) => (
            <div
              // biome-ignore lint/suspicious/noArrayIndexKey: static skeleton
              key={day}
              className="animate-pulse rounded-lg border border-border/50 bg-muted/40"
              style={{ height: rowHeight }}
            />
          ))}
        </div>
      ))}
    </div>
  );
}
```

- [ ] **Step 4: Verify and commit**

```bash
npm test -- --run && npm run build
git add src/components/calendar
git commit -m "feat(calendar): give month and schedule their own view states"
```

---

## Task 11: Month E2E

**Files:**
- Create: `e2e/large-screen-calendar-month.spec.ts`

- [ ] **Step 1: Start the real backend**

FE E2E hits the real backend through the Vite proxy, not MSW. Start it with a **released** BE tag — the compose default `latest` is stale and 404s newer endpoints:

```bash
BE_IMAGE_TAG=$(gh release list --repo joe-bor/family-hub-api --limit 1 --json tagName -q '.[0].tagName') \
  docker compose -f docker-compose.e2e.yml up -d
```

Tear down with `docker compose -f docker-compose.e2e.yml down` when the gate is complete.

- [ ] **Step 2: Write the spec**

Use the repo's existing helpers rather than raw selectors. `switchCalendarView` handles the desktop/mobile switcher difference, which matters because the view switcher is **icon-only below xl (1280px)** — a `name: /month/i` selector would not match there.

Follow the repo's canonical bootstrap exactly — `registerFamily` takes an `APIRequestContext` and only hits the backend; the browser must then be seeded and reloaded. See `e2e/calendar-navigation.spec.ts:11-27` for the pattern this mirrors.

```ts
import { expect, test } from "@playwright/test";
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
  test.beforeEach(async ({ page, request }) => {
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
    // The last row's bottom edge should reach the grid's bottom edge.
    expect(
      (lastBox as { y: number; height: number }).y +
        (lastBox as { height: number }).height,
    ).toBeGreaterThan(
      (gridBox as { y: number; height: number }).y +
        (gridBox as { height: number }).height -
        24,
    );
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
    // Chips and overflow buttons must not add stops either.
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
      .getByRole("gridcell", { name: /, 1 event$/ })
      .first();
    await singleEventDay.focus();
    await page.keyboard.press("Enter");

    await expect(page.getByRole("dialog")).toBeVisible();
  });

  test("arrow keys move the focused day", async ({ page }) => {
    const cells = page.getByRole("gridcell");
    await cells.nth(10).focus();
    const before = await page.evaluate(() =>
      document.activeElement?.getAttribute("data-date"),
    );

    await page.keyboard.press("ArrowRight");
    const after = await page.evaluate(() =>
      document.activeElement?.getAttribute("data-date"),
    );

    expect(after).not.toBe(before);
  });

  test("Enter on a day with events opens the overflow popover", async ({
    page,
  }) => {
    // The six overflow events are seeded in beforeEach via the API.
    const busyDay = page.getByRole("gridcell", { name: /6 events/ }).first();
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

  test("weekday header is a row of columnheaders inside the grid", async ({
    page,
  }) => {
    const grid = page.getByRole("grid");
    await expect(grid.getByRole("columnheader")).toHaveCount(7);
  });
});
```

- [ ] **Step 3: Run**

```bash
npm run test:e2e -- e2e/large-screen-calendar-month.spec.ts
```

Expected: all pass. Fix real failures; do not weaken assertions to make them pass.

- [ ] **Step 4: Commit**

```bash
git add e2e/large-screen-calendar-month.spec.ts
git commit -m "test(calendar): add large-screen month E2E coverage"
```

---

## Task 12: Month phase gate

- [ ] **Step 1: Capture the matrix**

With the real backend running and a family of three members seeded, capture Month at: **375x812**, **768** (must show the mobile view), **769**, **1024x768**, **1280x800**, **1440x900**.

At 1280x800 and 1440x900 also capture: a busy month, a sparse month, **February 2026** (four rows), a month with a multi-day run crossing a week boundary, a day with overflow and its open popover, long event titles, a family with zero members, loading, empty, error, and offline.

- [ ] **Step 2: Review against the spec**

Confirm: no dead space below the last week row; chips at least 28px; runs welded with rounded corners only at true start and end; weekend, today and focused day all visually distinct; `+N more` legible.

If capacity at 1024x768 for a six-week month reads too tight, apply the spec Section 4.2 levers **in order**: reduce `MONTH_NUMERAL_BLOCK`, then `MONTH_CELL_PADDING_Y`, then raise `MONTH_MIN_ROW_HEIGHT` and accept scrolling. Do not reduce `MONTH_CHIP_HEIGHT` below 28.

- [ ] **Step 3: Confirm Month is complete**

```bash
npm run lint && npm test -- --run && npm run build
```

Expected: all exit 0. Month is done and gated; Schedule begins. Do not start Task 13 until this passes — the phase boundary is what keeps Schedule's mobile risk from contaminating a Month regression hunt.

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
import { buildScheduleRows } from "./schedule-rows";

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
import { compareEventsAllDayFirst, isEventOnDate } from "@/lib/time-utils";
import type { CalendarEvent, FilterState } from "@/lib/types";

export type ScheduleRow =
  | { kind: "day"; date: Date; events: CalendarEvent[] }
  | { kind: "gap"; start: Date; end: Date };

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
    const date = new Date(startDate);
    date.setDate(startDate.getDate() + offset);

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
      const previous = new Date(day.date);
      previous.setDate(day.date.getDate() - 1);
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

Expected: PASS, 7 tests.

- [ ] **Step 5: Commit**

```bash
git add src/components/calendar/utils/schedule-rows.ts src/components/calendar/utils/schedule-rows.test.ts
git commit -m "feat(calendar): add schedule row grouping with gap runs"
```

---

## Task 14: Schedule large-screen composition

**Files:**
- Modify: `src/components/calendar/views/schedule-calendar.tsx`

- [ ] **Step 1: Split the component by breakpoint**

Rename the current exported body to `ScheduleCalendarCompact`, unchanged. Add:

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

- [ ] **Step 3: Build the large composition**

`ScheduleCalendarLarge` renders `buildScheduleRows(...)` output inside `mx-auto w-full` with `maxWidth: SCHEDULE_MAX_WIDTH` (1400). Each day row is a `<section>` with an accessible name:

```tsx
<section
  key={formatLocalDate(row.date)}
  aria-label={`${relativeLabel(row.date)}, ${format(row.date, "EEEE MMMM d")}, ${row.events.length} events`}
  className="grid gap-4 border-t border-border py-3"
  style={{ gridTemplateColumns: `${SCHEDULE_GUTTER_WIDTH}px 1fr` }}
>
  <div className="sticky top-0 self-start">
    <p className={cn("text-xs font-bold uppercase tracking-wide",
      isToday(row.date) ? "text-primary" : "text-muted-foreground")}>
      {relativeLabel(row.date)}
    </p>
    <p className="text-sm font-semibold">{format(row.date, "EEE, MMM d")}</p>
    <p className="text-xs text-muted-foreground">
      {row.events.length} {row.events.length === 1 ? "event" : "events"}
    </p>
  </div>
  <div className="flex flex-col gap-2">{/* event rows */}</div>
</section>
```

Gap rows:

```tsx
<div
  key={`gap-${formatLocalDate(row.start)}`}
  className="grid gap-4 border-t border-border py-3"
  style={{ gridTemplateColumns: `${SCHEDULE_GUTTER_WIDTH}px 1fr` }}
>
  <span className="sr-only">
    {`${format(row.start, "EEEE MMMM d")} to ${format(row.end, "EEEE MMMM d")}, nothing scheduled`}
  </span>
  <p aria-hidden="true" className="text-xs font-medium text-muted-foreground">
    {gapLabel(row.start, row.end)}
  </p>
  <p aria-hidden="true" className="text-sm italic text-muted-foreground">
    Nothing scheduled
  </p>
</div>
```

Event rows keep the existing anatomy and add the member's name as visible text next to time and location, with `min-h-14`.

Define the constants at the top of the file:

```tsx
/** Gutter width at lg+. Tunable at the screenshot gate. */
const SCHEDULE_GUTTER_WIDTH = 168;
/** Cap so rows do not become unreadably wide on very large displays. */
const SCHEDULE_MAX_WIDTH = 1400;
```

- [ ] **Step 4: De-emphasise past days**

For `row.date < startOfToday()`, apply `text-muted-foreground` to the gutter label and `border-border/40` to the section border. **Do not** reduce event text contrast and do not use opacity — spec Sections 4.3 and 5.3.

- [ ] **Step 5: Verify and commit**

```bash
npm test -- --run && npm run build
git add src/components/calendar/views/schedule-calendar.tsx
git commit -m "feat(calendar): add large-screen schedule gutter composition"
```

---

## Task 15: Schedule states and test migration

**Files:**
- Modify: `src/components/calendar/views/schedule-calendar.tsx`
- Modify: `src/components/calendar/views/schedule-calendar.test.tsx`

- [ ] **Step 1: Fix the empty states**

Delete the locally duplicated inline `Calendar` SVG function at the bottom of `schedule-calendar.tsx` and use the shared components from Task 10 instead.

Distinguish the two empty cases, which are currently conflated — the copy "Events for the next 2 weeks will appear here" is wrong when the user has simply deselected every member:

```tsx
const allMembersFiltered = filter.selectedMembers.length === 0;

if (rows.length === 0) {
  return allMembersFiltered ? (
    <CalendarEmptyState
      title="No members selected"
      description="Choose a family member to see their events."
    />
  ) : (
    <CalendarEmptyState
      title="No upcoming events"
      description="Nothing scheduled in the next 2 weeks."
    />
  );
}
```

- [ ] **Step 2: Add the Schedule loading and error states — `lg+` only**

**`ScheduleCalendarCompact` takes no new props and renders no new states.** Below 1024px the shared centred text in `calendar-module.tsx` continues to handle loading and error exactly as it ships today. Adding a skeleton there would be a visible mobile change on the app's highest-traffic mobile surface, which the contract forbids.

`ScheduleCalendarLarge` takes `isLoading` / `isError` / `errorMessage` / `onRetry` and renders in priority order: error, then loading, then offline-with-no-data, then content.

Wire them at the **desktop** `ScheduleCalendar` call site only (`calendar-module.tsx:552`), leaving the two mobile call sites (`:524`, `:526`) spreading `commonProps` alone. Then extend the Task 10 gate to cover Schedule, in this same commit so lg+ Schedule never ships without states:

```tsx
const ownsItsStates =
  isLargeScreen &&
  (calendarView === "monthly" || calendarView === "schedule");
```

```tsx
function ScheduleSkeleton() {
  return (
    <div
      role="status"
      aria-label="Loading schedule"
      className="mx-auto flex w-full flex-col gap-4 p-4"
      style={{ maxWidth: SCHEDULE_MAX_WIDTH }}
    >
      {Array.from({ length: 4 }).map((_, group) => (
        // biome-ignore lint/suspicious/noArrayIndexKey: static skeleton
        <div
          key={group}
          className="grid gap-4"
          style={{ gridTemplateColumns: `${SCHEDULE_GUTTER_WIDTH}px 1fr` }}
        >
          <div className="h-10 animate-pulse rounded-lg bg-muted/60" />
          <div className="flex flex-col gap-2">
            <div className="h-14 animate-pulse rounded-xl bg-muted/40" />
            <div className="h-14 animate-pulse rounded-xl bg-muted/40" />
          </div>
        </div>
      ))}
    </div>
  );
}
```

Same for the empty states in Step 1: they apply **only in the large branch**. `ScheduleCalendarCompact` keeps its existing empty-state markup and copy, so its rendering is unchanged.

**Net effect: `ScheduleCalendarCompact` is untouched by this entire task.** Its only edit in the whole plan is the mechanical extraction in Task 14 Step 1. That is what makes the byte-identical hash comparison in Task 16 a meaningful check rather than a formality — if it fails, something genuinely leaked past the gate.

- [ ] **Step 3: Migrate the banner assertions**

**Pin every existing test to a viewport.** All 7 current tests run at whatever `matchMedia` happens to be set to. Once a `describe` in this file calls `setViewportWidth(1280)`, the leak described in Task 1 means unpinned tests may run against `ScheduleCalendarLarge` — and they would still pass, silently ceasing to guard the compact path they exist for.

Wrap **all seven** existing tests in `describe("compact", ...)` with:

```tsx
describe("compact", () => {
  beforeEach(() => setViewportWidth(390));
  afterEach(resetViewportWidth);
  // ...all 7 existing tests, assertions unchanged
});
```

Their assertions must not change — including the three banner strings (`"Today — Wed, Mar 18"`, `"Tomorrow — Thu, Mar 19"`, `"Friday, Mar 20"`) and `"reserves bottom clearance on mobile for the floating action button"`. They are the regression guard for the mobile path.

Then add a parallel large-screen block. It needs a shared render helper and pinned time, neither of which exists in the file today:

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
    expect(screen.getByText("Today")).toBeInTheDocument();
    expect(screen.getByText("Wed, Mar 18")).toBeInTheDocument();
    expect(screen.getByText("1 event")).toBeInTheDocument();
  });

  it("renders a gap row for an event-free stretch", () => {
    renderSchedule();
    expect(screen.getByText("Nothing scheduled")).toBeInTheDocument();
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
});
```

Add the imports this block needs and the file currently lacks: `createTestEvent` from `@/test/fixtures`, `colorMap` from `@/lib/types`, and `setViewportWidth` / `resetViewportWidth` from `@/test/test-utils`.

`testMembers[0]` is John with colour `coral`; `colorMap.coral.hex` is the value the border must resolve to.

- [ ] **Step 4: Verify and commit**

```bash
npm test -- --run src/components/calendar/views/schedule-calendar.test.tsx
npm run build
git add src/components/calendar/views/schedule-calendar.tsx src/components/calendar/views/schedule-calendar.test.tsx
git commit -m "feat(calendar): correct schedule empty states and migrate tests"
```

---

## Task 16: Schedule E2E and mobile parity proof

**Files:**
- Create: `e2e/large-screen-calendar-schedule.spec.ts`

- [ ] **Step 1: Write the spec**

The window must be **seeded with a gap** — one populated day, several empty, another populated. With no events at all, `buildScheduleRows` short-circuits to `[]` and the view renders the whole-view empty state, so every assertion below would fail. A gap row cannot exist unless at least one day is populated.

```ts
import { expect, test } from "@playwright/test";
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
  test.beforeEach(async ({ page, request }) => {
    await page.setViewportSize({ width: 1440, height: 900 });
    await page.goto("/");
    await clearStorage(page);

    const reg = await registerFamily(request, {
      familyName: "Test Family",
      members: [{ name: "Alice", color: "coral" }],
    });

    const today = new Date(`${getTodayDateString()}T00:00:00`);
    const plusDays = (n: number) => {
      const d = new Date(today);
      d.setDate(today.getDate() + n);
      return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}-${String(d.getDate()).padStart(2, "0")}`;
    };

    // Today and day +4 populated, leaving days +1..+3 as an explicit gap.
    for (const offset of [0, 4]) {
      await createCalendarEvent(request, reg.token, {
        title: `Event day ${offset}`,
        date: plusDays(offset),
        startTime: "9:00 AM",
        endTime: "10:00 AM",
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
    const section = page.getByRole("region").first();
    const box = await section.boundingBox();
    expect(box).not.toBeNull();
    // 1440 minus the 80px nav rail leaves ~1360; a max-w-3xl column is 768.
    expect((box as { width: number }).width).toBeGreaterThan(900);
  });

  test("the date gutter carries label, date and count", async ({ page }) => {
    await expect(page.getByText("Today")).toBeVisible();
    await expect(page.getByText(/\d+ events?/).first()).toBeVisible();
  });

  test("an event-free stretch renders one gap row", async ({ page }) => {
    await expect(page.getByText("Nothing scheduled").first()).toBeVisible();
  });
});
```

- [ ] **Step 2: Run**

```bash
npm run test:e2e -- e2e/large-screen-calendar-schedule.spec.ts
npm run test:e2e -- e2e/mobile-calendar.spec.ts
```

Expected: both pass. The mobile spec is the regression guard.

- [ ] **Step 3: Prove mobile parity by screenshot hash**

Add a throwaway Playwright spec that captures mobile Schedule deterministically, so both runs produce comparable bytes:

```ts
// e2e/tmp-schedule-parity.spec.ts — delete after the gate passes
import { test } from "@playwright/test";
import { registerFamily, seedBrowserAuth } from "./helpers/api-helpers";
import {
  clearStorage,
  switchCalendarView,
  waitForCalendarReady,
  waitForHydration,
} from "./helpers/test-helpers";

test("capture mobile schedule", async ({ page, request }) => {
  await page.setViewportSize({ width: 375, height: 812 });
  await page.goto("/");
  await clearStorage(page);

  // Fixed family name and colours so both captures render identically.
  const reg = await registerFamily(request, {
    familyName: "Parity Family",
    members: [{ name: "Alice", color: "coral" }],
  });
  await seedBrowserAuth(page, reg);

  await page.reload();
  await waitForHydration(page);
  await waitForCalendarReady(page);
  await switchCalendarView(page, "schedule");
  // Fonts must be settled before the shot; PR #285 found the hash only
  // stabilises once font loading matches between the two captures.
  await page.evaluate(() => document.fonts.ready);
  await page.screenshot({
    path: process.env.PARITY_OUT ?? "parity.png",
    fullPage: true,
  });
});
```

Then capture both sides and compare:

```bash
git worktree add ../fh-baseline origin/main
cd ../fh-baseline && npm ci
cp ../frontend/e2e/tmp-schedule-parity.spec.ts e2e/
PARITY_OUT=/tmp/baseline.png npx playwright test e2e/tmp-schedule-parity.spec.ts
cd ../frontend
PARITY_OUT=/tmp/current.png npx playwright test e2e/tmp-schedule-parity.spec.ts
shasum -a 256 /tmp/baseline.png /tmp/current.png
```

Expected: identical hashes. A worktree is used rather than `git stash` so the baseline build is complete and independent — checking out a single file against a modified tree would mix new imports with old markup and fail to compile.

If the hashes differ, diff the images before concluding it is a regression: font loading and any animation still in flight are the usual causes.

- [ ] **Step 4: Clean up**

```bash
rm e2e/tmp-schedule-parity.spec.ts
git worktree remove ../fh-baseline
```

- [ ] **Step 5: Commit**

```bash
git add e2e/large-screen-calendar-schedule.spec.ts
git commit -m "test(calendar): add large-screen schedule E2E coverage"
```

---

## Task 17: Schedule screenshot gate and final verification

- [ ] **Step 1: Capture the matrix**

Schedule at 375x812, 768, 769, 1024x768, 1280x800, 1440x900, covering: populated window, window with a long gap, whole-window empty, all-members-filtered empty, loading, error, offline, past days visible after paging back, and very long titles with long locations.

- [ ] **Step 2: Full verification**

```bash
npm run lint
npm test -- --run
npm run build
npm run test:e2e
```

Expected: all four exit 0. Record the actual test counts; do not claim success without reading the output.

- [ ] **Step 3: Final self-check against the acceptance criteria**

Walk the spec Section 9 acceptance criteria one by one and map each to a test name or a screenshot filename. Any criterion without evidence is not done.

- [ ] **Step 4: Push and open the PR**

```bash
git push -u origin feat/large-screen-month-schedule
gh pr create --title "feat(calendar): large-screen month and schedule" --body "$(cat <<'BODY'
## Summary
- Month at lg+ fills the viewport with slot capacity derived from measured row height, an interactive overflow popover, and multi-day runs welded by corner geometry.
- Schedule at lg+ moves to a date gutter with full-width rows, explicit gap rows, and de-emphasised past days.
- Mobile, the 769-1023px range, Week and Day are unchanged.

## Links
- Story: docs/product/backlog/large-screen-ux/large-screen-calendar-month-schedule.md
- Spec: docs/superpowers/specs/2026-07-18-large-screen-calendar-month-schedule-design.md
- Plan: docs/superpowers/plans/2026-07-18-large-screen-calendar-month-schedule.md

## Verification
- npm run lint / npm test -- --run / npm run build / npm run test:e2e
- Screenshot matrix per spec Section 9
- Mobile Schedule screenshot hash matched against origin/main
BODY
)"
```

---

## Execution Contract

Copy this into the frontend implementation Issue.

**Non-negotiable:**

1. Everything is gated at 1024px via `useIsLargeScreen()`. Mobile (<= 768px), the 769-1023px range, Week and Day must be visually and behaviourally unchanged.
2. `ScheduleCalendar` is shared with mobile and is the mobile smart default. Its compact path must stay byte-for-byte as it ships today, proven by screenshot hash against `origin/main`.
3. Capacity is counted in **slots**, not events, because reserved blank slots occupy rows. `+N more` counts events only.
4. `MONTH_CHIP_HEIGHT = 28` is a floor, not a tuning dial. The 44px rule applies to chrome and controls; in-grid chips are the documented exception.
5. Layout arithmetic lives in pure exported helpers with direct unit tests. The repo's Vitest setup mocks `ResizeObserver` as a no-op and `matchMedia` as `matches: false` for every query, so measured-geometry assertions must be Playwright, not Vitest.
6. The Month day cell is a `role="gridcell"` div, never a `<button>` — chips inside it are buttons and buttons cannot nest.
7. The grid is a single tab stop. Chips and `+N more` are `tabIndex={-1}`.
8. The weekday header is a `role="row"` of `role="columnheader"` inside the same `role="grid"`.
9. Colour is never the only channel: member names in accessible names and in Schedule's visible text, a visually-hidden member summary on the dot group, marker elements for all-day and recurring. No meaning carried by opacity.
10. Month's widened query range is `lg+`-only. An ungated change would alter mobile.
11. `npm run build` must pass before every commit — it is the only gate that catches type errors.

**Known limitations to preserve, not fix:** Schedule's 14-day window with a 7-day paging step, its constant `"Upcoming"` toolbar label, and multi-day slot alignment across a week boundary. All three are recorded in spec Section 11.
