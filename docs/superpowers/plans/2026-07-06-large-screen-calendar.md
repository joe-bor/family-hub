# Large-Screen Calendar Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the desktop Calendar Week view read as the shape of the week (one-line day headers, denser hour rows, auto-scroll to the useful window) and the desktop Day view read as the family's day side-by-side (one lane per member with an aligned all-day band, plus an optional mini-month navigator rail), while leaving the mobile Calendar and the 768–1023px tablet range untouched.

**Architecture:** Three focused efforts inside the existing FE calendar tree, none of which touch the mobile view components (`views/mobile/*`) or the Home→Calendar routing effects (`calendarFocusDate` and `calendarEventIntent`):

1. **Shared hour-grid geometry.** Extract the `ROW_HEIGHT = 80` / `h-20` / `TIME_SLOTS` duplication (currently copy-pasted across `weekly-calendar.tsx` and `daily-calendar.tsx`) into one `utils/hour-grid.ts` module with pure geometry helpers and two density constants. A new `useIsLargeScreen()` hook (min-width 1024px, built on the existing `useMediaQuery`) lets the Week and Day-lanes views pick the dense row height at `lg+` only; the shared `CurrentTimeIndicator` / auto-scroll hooks already accept an explicit `rowHeight`, and mobile keeps passing its own `60`, so mobile is insulated.
2. **Week zoom-out.** Rewrite the `WeeklyCalendar` day-header "tower" into one line, drive its cell height and event positioning from the shared geometry, and replace its always-scroll-to-now behavior with a now-or-first-event target.
3. **Day member lanes + rail.** Extract the current single-canvas overlap layout into a shared `views/day-lane-layout.ts` (so the tablet Day view keeps identical behavior), then add a new `lg+`-only `DayLanesCalendar` that renders one lane per member with an aligned all-day band, plus an optional `~300px` mini-month rail. The rail's visibility is a *pure* function of member count and viewport width (`railThresholdPx(memberCount)` + `useMediaQuery`), gated by a persisted calendar-store toggle. `CalendarModule` switches Day → `DayLanesCalendar` at `lg+` and renders a toolbar rail toggle when the rail can fit.

**Tech Stack:** React 19, TypeScript, Vite, TanStack Query, Zustand (persisted calendar store), Tailwind CSS v4, Vitest + Testing Library, Playwright, Biome. No new dependencies.

**Spec:** `docs/superpowers/specs/2026-07-06-large-screen-calendar-design.md`
**Story:** `docs/product/backlog/large-screen-ux/large-screen-calendar.md`
**Depends on (merged):** `docs/superpowers/plans/2026-07-06-large-screen-foundations.md` (PR #282), `docs/superpowers/plans/2026-07-05-large-screen-home.md` (PR #284)

**Repo:** All implementation work is in `frontend/` (separate git repo). This plan lives in the root docs repo for cross-repo planning.

---

## Pre-flight

- [ ] Read the spec `docs/superpowers/specs/2026-07-06-large-screen-calendar-design.md` end-to-end. The non-negotiables: Week one-line headers + `48–56px` rows at `lg+` (12–14 hours visible) + auto-scroll to now-or-first-event; Day `lg+` one lane **per member** (avatar + name + color header) with an all-day band aligned to lanes and **no "Everyone" lane** (`memberId` is required + singular); a Day mini-month rail (~300px) that auto-shows only when lane math leaves spare width, has a persisted toolbar toggle to hide it, is never on mobile, and is a navigator only (tap day → jump Day view); Month/Schedule inherit foundations chrome only; **mobile Calendar (all views) visually + behaviorally unchanged**; a screenshot-critique gate before done.
- [ ] Read `frontend/AGENTS.md` / `frontend/CLAUDE.md`. Follow the date/time rule: use `src/lib/time-utils.ts` helpers, never raw `new Date("yyyy-mm-dd")` or `toISOString()`. Use `cn()` from `@/lib/utils` for all className merging.
- [ ] Confirm `frontend/` is clean and on the feature branch: `cd frontend && git status -sb`. This plan assumes branch `feat/large-screen-calendar` (Task 0 creates/reuses it).
- [ ] Do **not** modify `src/components/calendar/views/mobile/*`, `mobile-toolbar.tsx`, `mobile-event-*`, or the shared `CurrentTimeIndicator` / `useAutoScrollToNow` **defaults** (mobile depends on them). Do **not** change the Home→Calendar routing effects in `calendar-module.tsx` — the `calendarFocusDate` → `setDate` effect (`:140-143`) and the `calendarEventIntent` ({date, eventKey}) effect (`:145-296`) that sets the date, finds the event in **unfiltered** `rawEvents` via `getEventKey`, then calls `openDetailModal` + `clearCalendarEventIntent`. Leave both intact; event taps in the new views must still call `onEventClick`.
- [ ] No backend changes. Shared/family-wide events do not exist; every event has exactly one `memberId`.

## Verified Codebase Facts

- **Row height is duplicated.** `weekly-calendar.tsx` and `daily-calendar.tsx` each declare `const ROW_HEIGHT = 80` (event px math) **and** hard-code `h-20` (80px) cell classes **and** an identical 18-entry `timeSlots` array (`"6 AM"`…`"11 PM"`). `CurrentTimeIndicator` / `useAutoScrollToNow` (`components/current-time-indicator.tsx`) default `rowHeight = 80`, `startHour = 6`.
- **Mobile is insulated.** `views/mobile/mobile-daily-view.tsx` uses its own `const ROW_HEIGHT = 60`, inline `style={{ height }}` cells, and passes `rowHeight={60}` explicitly to `CurrentTimeIndicator` / `useAutoScrollToNow`. It never relies on the `80` default. Weekly/Daily desktop views are the only callers relying on the default.
- **View switching** happens in `calendar-module.tsx:461-477` (desktop `switch (calendarView)`), separate from the mobile `switch` at `:417-458`. `isMobile = useIsMobile()` (max-width 768). Desktop chrome (merged toolbar) already lives in the module at `:485-497`.
- **The merged toolbar** (foundations PR #282) is `<CalendarViewSwitcher/> <CalendarNavigation .../> <FamilyFilterPills/>` in one `flex-wrap` row. The view switcher is icon-only below `xl` (1280px). Do not regress this.
- **Overlap layout** lives entirely in `daily-calendar.tsx`: `eventsOverlap()`, `calculateEventColumns()` (greedy column packing, capped at 3 visible columns), `getEventGridPosition()`. Weekly does simple stacking (no columns) via `getEventPosition()`.
- **Calendar store** (`stores/calendar-store.ts`) is `persist`-wrapped with `name: "family-hub-calendar"`; `partialize` currently persists `filter`, `calendarView`, `hasUserSetView`. Adding a persisted field means updating **three** reset/seed sites: `partialize`, `src/test/setup.ts` `resetAllStores()`, and `src/test/test-utils.tsx` `seedCalendarStore()` + `resetCalendarStore()` (CLAUDE.md: "store reset by name, not generic").
- **`useCalendarEvents(dateRange)`** returns `{ data: { data: CalendarEvent[] }, isLoading, isFetching, isError }`; `dateRange = { startDate, endDate }` as `"yyyy-MM-dd"`. The module derives `rawEvents = eventsResponse.data ?? []` (unfiltered — the `calendarEventIntent` lookup uses it so a persisted filter can't hide the tapped event) then `events = rawEvents.filter(...)` by `filter.selectedMembers` + `filter.showAllDayEvents`. The new Day-lanes view receives the filtered `events` (via `commonProps`-style props).
- **Test env** (`src/test/setup.ts`): `window.matchMedia` is globally mocked to `matches: false` for every query, `ResizeObserver` is a no-op mock, `Element.prototype.scrollTo` is a mock, `afterEach` runs `vi.clearAllMocks()` + `resetAllStores()`. Consequence: breakpoint-gated integration paths render the **non-large** branch unless a test overrides `matchMedia` (mirror `use-is-mobile.test.tsx`); test the large-only components directly with props instead of relying on the gate.
- **Types:** `FamilyMember { id, name, color }`, `colorMap[color] = { bg, text, light, hex }` (Tailwind class strings + raw `hex`), `getFamilyMember(members, id)`, `CalendarViewType = "daily" | "weekly" | "monthly" | "schedule"`, `MemberAvatar` (`size` `sm|md|lg`, `variant` `filled|ring`). Tailwind defaults: `lg`=1024, `xl`=1280, `2xl`=1536 (no custom breakpoints in `index.css`).

## File Structure

Create:

```text
frontend/src/hooks/use-is-large-screen.ts
frontend/src/hooks/use-is-large-screen.test.tsx
frontend/src/components/calendar/utils/hour-grid.ts
frontend/src/components/calendar/utils/hour-grid.test.ts
frontend/src/components/calendar/views/day-lane-layout.ts
frontend/src/components/calendar/views/day-lane-layout.test.ts
frontend/src/components/calendar/views/day-lanes-calendar.tsx
frontend/src/components/calendar/views/day-lanes-calendar.test.tsx
frontend/src/components/calendar/utils/day-rail.ts
frontend/src/components/calendar/utils/day-rail.test.ts
frontend/src/components/calendar/components/day-mini-month-rail.tsx
frontend/src/components/calendar/components/day-mini-month-rail.test.tsx
frontend/src/components/calendar/components/day-rail-toggle.tsx
frontend/src/components/calendar/components/day-rail-toggle.test.tsx
```

Modify:

```text
frontend/src/hooks/index.ts
frontend/src/components/calendar/components/current-time-indicator.tsx
frontend/src/components/calendar/views/weekly-calendar.tsx
frontend/src/components/calendar/views/daily-calendar.tsx
frontend/src/components/calendar/views/index.ts
frontend/src/components/calendar/components/index.ts
frontend/src/stores/calendar-store.ts
frontend/src/stores/index.ts
frontend/src/components/calendar/calendar-module.tsx
frontend/src/components/calendar/calendar-module.test.tsx
frontend/src/test/setup.ts
frontend/src/test/test-utils.tsx
```

Each task ends with a commit. Use conventional commits scoped `feat(calendar):`, `refactor(calendar):`, `test(calendar):`, `feat(shell):`.

---

## Task 0: Branch setup

**Files:** none

- [ ] **Step 1: Ensure the FE feature branch exists and is current**

```bash
cd /Users/joe.bor/code/family-hub/frontend
git fetch origin
git checkout feat/large-screen-calendar 2>/dev/null || git checkout -b feat/large-screen-calendar origin/main
git status -sb
```

Expected: on `feat/large-screen-calendar`, clean tree.

---

## Task 1: Shared hour-grid geometry + large-screen hook

Extract the duplicated row-height/time-slot/positioning logic into one tested module and add the density breakpoint hook. No visual change yet — this is a pure refactor that Weekly and Day-lanes build on.

**Files:**
- Create: `src/components/calendar/utils/hour-grid.ts`
- Test: `src/components/calendar/utils/hour-grid.test.ts`
- Create: `src/hooks/use-is-large-screen.ts`
- Test: `src/hooks/use-is-large-screen.test.tsx`
- Modify: `src/hooks/index.ts`

- [ ] **Step 1: Write the failing hour-grid geometry tests**

Create `src/components/calendar/utils/hour-grid.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { CALENDAR_START_HOUR } from "@/lib/time-utils";
import {
  DEFAULT_HOUR_ROW_HEIGHT,
  DENSE_HOUR_ROW_HEIGHT,
  earliestEventStartMinutes,
  getEventOffsets,
  hourRowHeightFor,
  minutesFromStartHour,
  pxFromOffsets,
  TIME_SLOTS,
} from "./hour-grid";

describe("hour-grid geometry", () => {
  it("exposes the 6 AM–11 PM slot labels", () => {
    expect(TIME_SLOTS).toHaveLength(18);
    expect(TIME_SLOTS[0]).toBe("6 AM");
    expect(TIME_SLOTS.at(-1)).toBe("11 PM");
  });

  it("picks the dense row height only on large screens", () => {
    expect(hourRowHeightFor(false)).toBe(DEFAULT_HOUR_ROW_HEIGHT);
    expect(hourRowHeightFor(true)).toBe(DENSE_HOUR_ROW_HEIGHT);
    expect(DENSE_HOUR_ROW_HEIGHT).toBeLessThan(DEFAULT_HOUR_ROW_HEIGHT);
  });

  it("computes event offsets in hours from the start hour", () => {
    // 9:00 AM–10:30 AM with a 6 AM start hour → starts 3h in, spans 1.5h
    expect(getEventOffsets("9:00 AM", "10:30 AM")).toEqual({
      startOffsetHours: 3,
      spanHours: 1.5,
    });
  });

  it("converts offsets to pixels with a per-row height and a minimum", () => {
    expect(pxFromOffsets({ startOffsetHours: 3, spanHours: 1 }, 52)).toEqual({
      top: 156,
      height: 52,
    });
    // A 10-minute event never collapses below the 30px minimum.
    expect(
      pxFromOffsets({ startOffsetHours: 0, spanHours: 10 / 60 }, 52).height,
    ).toBe(30);
  });

  it("converts a time string to minutes from the start hour", () => {
    expect(minutesFromStartHour("6:00 AM", CALENDAR_START_HOUR)).toBe(0);
    expect(minutesFromStartHour("9:30 AM", CALENDAR_START_HOUR)).toBe(210);
  });

  it("finds the earliest timed-event start, ignoring all-day, or null", () => {
    const base = { date: new Date(2026, 6, 6), memberId: "m1", isAllDay: false };
    expect(
      earliestEventStartMinutes(
        [
          { ...base, startTime: "2:00 PM", endTime: "3:00 PM" },
          { ...base, startTime: "8:15 AM", endTime: "9:00 AM" },
          { ...base, isAllDay: true, startTime: "12:00 AM", endTime: "12:00 AM" },
        ],
        CALENDAR_START_HOUR,
      ),
    ).toBe(135); // 8:15 AM = 2h15m after 6 AM
    expect(earliestEventStartMinutes([], CALENDAR_START_HOUR)).toBeNull();
  });
});
```

- [ ] **Step 2: Run it to verify failure**

```bash
npm test -- --run src/components/calendar/utils/hour-grid.test.ts
```

Expected: FAIL (`hour-grid.ts` does not exist).

- [ ] **Step 3: Implement the hour-grid module**

Create `src/components/calendar/utils/hour-grid.ts`:

```ts
import { CALENDAR_START_HOUR, parseTime } from "@/lib/time-utils";
import type { CalendarEvent } from "@/lib/types";

/** Desktop hour-row height at tablet widths (768–1023px), matching today's grid. */
export const DEFAULT_HOUR_ROW_HEIGHT = 80;
/**
 * Denser hour-row height applied on lg+ (1024px+). Spec §3 calls for 48–56px so
 * 12–14 hours are visible at 1440x900; tuned during the screenshot gate (Task 8).
 */
export const DENSE_HOUR_ROW_HEIGHT = 52;

/** Minimum rendered event height so a short event stays tappable. */
export const MIN_EVENT_PX = 30;

/** 6 AM–11 PM row labels shared by the Week and Day grids (18 rows). */
export const TIME_SLOTS = [
  "6 AM", "7 AM", "8 AM", "9 AM", "10 AM", "11 AM",
  "12 PM", "1 PM", "2 PM", "3 PM", "4 PM", "5 PM",
  "6 PM", "7 PM", "8 PM", "9 PM", "10 PM", "11 PM",
] as const;

/** Row height for the current breakpoint. */
export function hourRowHeightFor(isLargeScreen: boolean): number {
  return isLargeScreen ? DENSE_HOUR_ROW_HEIGHT : DEFAULT_HOUR_ROW_HEIGHT;
}

export interface EventOffsets {
  startOffsetHours: number;
  spanHours: number;
}

/** Offsets (in hours from CALENDAR_START_HOUR) for an event's start and span. */
export function getEventOffsets(startTime: string, endTime: string): EventOffsets {
  const start = parseTime(startTime);
  const end = parseTime(endTime);
  const startOffsetHours =
    start.hours - CALENDAR_START_HOUR + start.minutes / 60;
  const endOffsetHours = end.hours - CALENDAR_START_HOUR + end.minutes / 60;
  return { startOffsetHours, spanHours: endOffsetHours - startOffsetHours };
}

/** Absolute top/height in px for the given offsets at a row height. */
export function pxFromOffsets(
  { startOffsetHours, spanHours }: EventOffsets,
  rowHeight: number,
): { top: number; height: number } {
  return {
    top: startOffsetHours * rowHeight,
    height: Math.max(spanHours * rowHeight, MIN_EVENT_PX),
  };
}

/** Minutes from the start hour for a "9:30 AM"-style time string. */
export function minutesFromStartHour(timeStr: string, startHour: number): number {
  const { hours, minutes } = parseTime(timeStr);
  return (hours - startHour) * 60 + minutes;
}

/**
 * Earliest timed-event start (minutes from the start hour) across the given
 * events, ignoring all-day events. Returns null when there are no timed events.
 */
export function earliestEventStartMinutes(
  events: Pick<CalendarEvent, "startTime" | "isAllDay">[],
  startHour: number,
): number | null {
  const timed = events.filter((event) => !event.isAllDay);
  if (timed.length === 0) return null;
  return Math.min(
    ...timed.map((event) => minutesFromStartHour(event.startTime, startHour)),
  );
}
```

- [ ] **Step 4: Run the geometry tests**

```bash
npm test -- --run src/components/calendar/utils/hour-grid.test.ts
```

Expected: PASS.

- [ ] **Step 5: Write the failing `useIsLargeScreen` test**

Create `src/hooks/use-is-large-screen.test.tsx` (mirrors `use-is-mobile.test.tsx`'s matchMedia pattern):

```tsx
import { renderHook } from "@testing-library/react";
import { beforeEach, describe, expect, it, vi } from "vitest";
import { useIsLargeScreen } from "./use-is-large-screen";

function mockMatchMedia(matches: boolean) {
  vi.mocked(window.matchMedia).mockImplementation((query: string) => ({
    matches,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  }));
}

describe("useIsLargeScreen", () => {
  beforeEach(() => {
    // afterEach clears call data but not the impl; set it explicitly each test.
    mockMatchMedia(false);
  });

  it("is false below the lg breakpoint", () => {
    mockMatchMedia(false);
    const { result } = renderHook(() => useIsLargeScreen());
    expect(result.current).toBe(false);
  });

  it("is true at lg and above", () => {
    mockMatchMedia(true);
    const { result } = renderHook(() => useIsLargeScreen());
    expect(result.current).toBe(true);
  });
});
```

- [ ] **Step 6: Run it to verify failure**

```bash
npm test -- --run src/hooks/use-is-large-screen.test.tsx
```

Expected: FAIL (`use-is-large-screen.ts` does not exist).

- [ ] **Step 7: Implement `useIsLargeScreen` and export it**

Create `src/hooks/use-is-large-screen.ts`:

```ts
import { useMediaQuery } from "./use-media-query";

/** Tailwind `lg` breakpoint — the large-screen calendar boundary (spec §Scope). */
export const LARGE_SCREEN_BREAKPOINT = 1024;

/**
 * True when the viewport is at least the `lg` breakpoint (1024px). Drives the
 * denser Week rows and the Day member-lanes layout; below this the calendar
 * keeps its current desktop/tablet rendering.
 */
export function useIsLargeScreen(): boolean {
  return useMediaQuery(`(min-width: ${LARGE_SCREEN_BREAKPOINT}px)`);
}
```

Add to `src/hooks/index.ts` (keep alphabetical grouping near `useIsMobile`):

```ts
export { LARGE_SCREEN_BREAKPOINT, useIsLargeScreen } from "./use-is-large-screen";
```

- [ ] **Step 8: Run the hook test**

```bash
npm test -- --run src/hooks/use-is-large-screen.test.tsx
```

Expected: PASS.

- [ ] **Step 9: Commit**

```bash
git add src/components/calendar/utils/hour-grid.ts src/components/calendar/utils/hour-grid.test.ts src/hooks/use-is-large-screen.ts src/hooks/use-is-large-screen.test.tsx src/hooks/index.ts
git commit -m "feat(calendar): add shared hour-grid geometry and useIsLargeScreen"
```

---

## Task 2: Week/Day — auto-scroll-to-minutes hook

Add the `useAutoScrollToMinutes` hook that the Week rewrite (Task 3) consumes, without disturbing the existing `useAutoScrollToNow` (still used by the tablet Day view and mobile). It lands first — small, isolated, reviewed on its own — so every later task that imports it starts from a green bar.

**Files:**
- Modify: `src/components/calendar/components/current-time-indicator.tsx`
- Test: add a co-located test file `src/components/calendar/components/current-time-indicator.test.tsx`

- [ ] **Step 1: Write the failing hook test**

Create `src/components/calendar/components/current-time-indicator.test.tsx`:

```tsx
import { renderHook } from "@testing-library/react";
import { useRef } from "react";
import { describe, expect, it, vi } from "vitest";
import { useAutoScrollToMinutes } from "./current-time-indicator";

describe("useAutoScrollToMinutes", () => {
  it("scrolls to the target row minus a lead offset", () => {
    const el = document.createElement("div");
    const scrollTo = vi.spyOn(el, "scrollTo");

    renderHook(() => {
      const ref = useRef<HTMLDivElement>(el);
      useAutoScrollToMinutes(ref, 180, 52); // 3h after start, 52px rows
    });

    // (180/60)*52 - 200 = 156 - 200 = -44 → clamped to 0
    expect(scrollTo).toHaveBeenCalledWith({ top: 0, behavior: "smooth" });
  });

  it("scrolls to a positive offset for a later target", () => {
    const el = document.createElement("div");
    const scrollTo = vi.spyOn(el, "scrollTo");

    renderHook(() => {
      const ref = useRef<HTMLDivElement>(el);
      useAutoScrollToMinutes(ref, 600, 52); // 10h after start
    });

    // (600/60)*52 - 200 = 520 - 200 = 320
    expect(scrollTo).toHaveBeenCalledWith({ top: 320, behavior: "smooth" });
  });

  it("does nothing when the target is null", () => {
    const el = document.createElement("div");
    const scrollTo = vi.spyOn(el, "scrollTo");

    renderHook(() => {
      const ref = useRef<HTMLDivElement>(el);
      useAutoScrollToMinutes(ref, null, 52);
    });

    expect(scrollTo).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run it to verify failure**

```bash
npm test -- --run src/components/calendar/components/current-time-indicator.test.tsx
```

Expected: FAIL (`useAutoScrollToMinutes` is not exported).

- [ ] **Step 3: Add the hook**

In `src/components/calendar/components/current-time-indicator.tsx`, append after the existing `useAutoScrollToNow`:

```tsx
/**
 * Scroll the grid so a target time is comfortably in view. `targetMinutes` is
 * minutes from the grid's start hour; pass null to skip (e.g. no events and not
 * today). Mirrors useAutoScrollToNow's 200px lead but takes an explicit target
 * so callers can choose "now" or "first event" (spec §3, Week auto-scroll).
 */
export function useAutoScrollToMinutes(
  containerRef: React.RefObject<HTMLElement | null>,
  targetMinutes: number | null,
  rowHeight = 80,
) {
  useEffect(() => {
    if (!containerRef.current || targetMinutes == null) return;

    const scrollPosition = (targetMinutes / 60) * rowHeight - 200;
    containerRef.current.scrollTo({
      top: Math.max(0, scrollPosition),
      behavior: "smooth",
    });
  }, [containerRef, targetMinutes, rowHeight]);
}
```

(The file already imports `useEffect` and `type React`.)

- [ ] **Step 4: Run the hook test (and confirm the calendar suite stays green)**

```bash
npm test -- --run src/components/calendar/components/current-time-indicator.test.tsx src/components/calendar/calendar-module.test.tsx
```

Expected: PASS (the new hook test passes; the existing calendar suite is unaffected — the Week wiring comes in Task 3).

- [ ] **Step 5: Commit**

```bash
git add src/components/calendar/components/current-time-indicator.tsx src/components/calendar/components/current-time-indicator.test.tsx
git commit -m "feat(calendar): add useAutoScrollToMinutes for now-or-first-event scroll"
```

---

## Task 3: Week — one-line day headers + denser rows

Rewrite the Week day-header tower into one line and drive the grid from the shared geometry so rows are dense at `lg+` (and stay 80px at 768–1023px). Behavior (events, all-day row, filtering) is otherwise unchanged.

**Files:**
- Modify: `src/components/calendar/views/weekly-calendar.tsx`
- Modify: `src/components/calendar/calendar-module.test.tsx` (add a Week-header assertion)

- [ ] **Step 1: Add a failing Week one-line-header integration test**

In `src/components/calendar/calendar-module.test.tsx`, add a new `describe` block near the other view tests. It renders the desktop Week view (default non-mobile) and asserts the collapsed header no longer shows the "TODAY" pill tower while still showing the day number:

```tsx
describe("Week view desktop chrome", () => {
  it("renders one-line day headers without the TODAY pill tower", async () => {
    seedMockEvents([]);
    seedCalendarStore({
      currentDate: new Date(2026, 6, 8), // Wed Jul 8 2026
      calendarView: "weekly",
      filter: { selectedMembers: testMembers.map((m) => m.id), showAllDayEvents: true },
    });

    render(<CalendarModule />);

    await waitFor(() => {
      expect(screen.queryByText("Loading events...")).not.toBeInTheDocument();
    });

    // Collapsed header: no stacked "TODAY" pill anymore.
    expect(screen.queryByText("TODAY")).not.toBeInTheDocument();
    // Day numerals still render (7 days rendered as day-of-month numbers).
    expect(screen.getAllByText(/^\d{1,2}$/).length).toBeGreaterThanOrEqual(7);
  });
});
```

- [ ] **Step 2: Run it to verify failure**

```bash
npm test -- --run src/components/calendar/calendar-module.test.tsx -t "one-line day headers"
```

Expected: FAIL (`TODAY` pill is still rendered today).

- [ ] **Step 3: Rewrite `weekly-calendar.tsx` to the shared geometry + one-line header**

Replace the imports and the module constants at the top of `src/components/calendar/views/weekly-calendar.tsx`. Change:

```tsx
import { useMemo, useRef } from "react";
import { useFamilyMembers } from "@/api";
import {
  CALENDAR_START_HOUR,
  compareEventsByTime,
  getEventKey,
  isEventOnDate,
  parseTime,
} from "@/lib/time-utils";
import { type CalendarEvent, colorMap, getFamilyMember } from "@/lib/types";
import { cn } from "@/lib/utils";
import { CalendarEventCard } from "../components/calendar-event";
import type { FilterState } from "../components/calendar-filter";
import {
  CurrentTimeIndicator,
  useAutoScrollToNow,
} from "../components/current-time-indicator";

interface WeeklyCalendarProps {
  events: CalendarEvent[];
  currentDate: Date;
  onEventClick?: (event: CalendarEvent) => void;
  filter: FilterState;
}

const ROW_HEIGHT = 80; // px per hour
const START_HOUR = CALENDAR_START_HOUR;

function getEventPosition(startTime: string, endTime: string) {
  const start = parseTime(startTime);
  const end = parseTime(endTime);

  const startOffset = start.hours - START_HOUR + start.minutes / 60;
  const endOffset = end.hours - START_HOUR + end.minutes / 60;

  const top = startOffset * ROW_HEIGHT;
  const height = Math.max((endOffset - startOffset) * ROW_HEIGHT, 30); // Minimum 30px

  return { top, height };
}
```

to:

```tsx
import { useMemo, useRef } from "react";
import { useFamilyMembers } from "@/api";
import { useIsLargeScreen } from "@/hooks";
import {
  CALENDAR_START_HOUR,
  compareEventsByTime,
  getEventKey,
  isEventOnDate,
} from "@/lib/time-utils";
import { type CalendarEvent, colorMap, getFamilyMember } from "@/lib/types";
import { cn } from "@/lib/utils";
import { CalendarEventCard } from "../components/calendar-event";
import type { FilterState } from "../components/calendar-filter";
import {
  CurrentTimeIndicator,
  useAutoScrollToMinutes,
} from "../components/current-time-indicator";
import {
  earliestEventStartMinutes,
  getEventOffsets,
  hourRowHeightFor,
  pxFromOffsets,
  TIME_SLOTS,
} from "../utils/hour-grid";

interface WeeklyCalendarProps {
  events: CalendarEvent[];
  currentDate: Date;
  onEventClick?: (event: CalendarEvent) => void;
  filter: FilterState;
}

const START_HOUR = CALENDAR_START_HOUR;
```

Inside the `WeeklyCalendar` component body, replace the top of the function (the refs / auto-scroll / `today`) — change:

```tsx
  const familyMembers = useFamilyMembers();
  const scrollContainerRef = useRef<HTMLDivElement>(null);
  const today = new Date();

  useAutoScrollToNow(scrollContainerRef);
```

to:

```tsx
  const familyMembers = useFamilyMembers();
  const scrollContainerRef = useRef<HTMLDivElement>(null);
  const today = new Date();
  const rowHeight = hourRowHeightFor(useIsLargeScreen());
```

Delete the now-unused local helpers `formatDayName`, `formatDayNumber`, and the inline `timeSlots` array, and rename **both** `timeSlots.map(...)` call sites — the time column (`weekly-calendar.tsx:278`) and the day-column background rows (`:303`) — to `TIME_SLOTS.map(...)`. Keep `isToday`, `getEventsForDay`, `getAllDayEventsForDay`. Then compute the auto-scroll target and wire the hook **immediately after the `getAllDayEventsForDay` helper** — it must come after `isToday`, `getEventsForDay`, and the `timedByDay` memo it reads:

```tsx
  const isTodayInWeek = weekDays.some((date) => isToday(date));

  // Auto-scroll target (minutes from START_HOUR): now when today is in the
  // visible week, otherwise the earliest timed event across the week (spec §3).
  const autoScrollMinutes = useMemo(() => {
    if (isTodayInWeek) {
      const now = new Date();
      return Math.max(0, (now.getHours() - START_HOUR) * 60 + now.getMinutes());
    }
    const weekTimedEvents = weekDays.flatMap((date) => getEventsForDay(date));
    return earliestEventStartMinutes(weekTimedEvents, START_HOUR);
  }, [isTodayInWeek, weekDays, timedByDay]);

  useAutoScrollToMinutes(scrollContainerRef, autoScrollMinutes, rowHeight);
```

Replace the entire day-header block (the `{weekDays.map(...)}` inside `{/* Days header */}`) with a one-line header:

```tsx
        {weekDays.map((date, index) => {
          const dayIsToday = isToday(date);
          const busyMembers = familyMembers.filter(
            (member) =>
              getEventsForDay(date).some((e) => e.memberId === member.id) ||
              getAllDayEventsForDay(date).some((e) => e.memberId === member.id),
          );
          return (
            <div
              key={index}
              className={cn(
                "flex items-center justify-center gap-2 border-l border-border px-2 py-2",
                dayIsToday && "bg-primary/10",
              )}
            >
              <span
                className={cn(
                  "text-xs font-medium uppercase tracking-wide",
                  dayIsToday ? "text-primary" : "text-muted-foreground",
                )}
              >
                {date.toLocaleDateString("en-US", { weekday: "short" })}
              </span>
              <span
                className={cn(
                  "flex h-7 min-w-7 items-center justify-center rounded-full px-1 text-sm font-bold tabular-nums",
                  dayIsToday
                    ? "bg-primary text-primary-foreground"
                    : "text-foreground",
                )}
              >
                {date.getDate()}
              </span>
              <span className="flex items-center gap-0.5">
                {busyMembers.slice(0, 4).map((member) => (
                  <span
                    key={member.id}
                    className={cn(
                      "h-1.5 w-1.5 rounded-full",
                      colorMap[member.color]?.bg,
                    )}
                    title={member.name}
                  />
                ))}
              </span>
            </div>
          );
        })}
```

- [ ] **Step 4: Drive the grid cells and events from `rowHeight`**

In the same file, the time column, day-column background rows, and event blocks currently hard-code `h-20` and `getEventPosition`. Replace the time column cell:

```tsx
              <div
                key={index}
                className="h-20 flex items-start justify-end pr-2 pt-1 border-b border-border/50"
              >
```

with:

```tsx
              <div
                key={index}
                style={{ height: rowHeight }}
                className="flex items-start justify-end pr-2 pt-1 border-b border-border/50"
              >
```

Replace the day-column background row:

```tsx
                    <div
                      key={index}
                      className={cn(
                        "h-20 border-b border-border/50",
                        index % 2 === 0 && !isTodayColumn && "bg-muted/20",
                      )}
                    />
```

with:

```tsx
                    <div
                      key={index}
                      style={{ height: rowHeight }}
                      className={cn(
                        "border-b border-border/50",
                        index % 2 === 0 && !isTodayColumn && "bg-muted/20",
                      )}
                    />
```

Replace the event-positioning call:

```tsx
                      const { top, height } = getEventPosition(
                        event.startTime,
                        event.endTime,
                      );
```

with:

```tsx
                      const { top, height } = pxFromOffsets(
                        getEventOffsets(event.startTime, event.endTime),
                        rowHeight,
                      );
```

Pass `rowHeight` to the current-time indicator — change `{isTodayColumn && <CurrentTimeIndicator />}` to:

```tsx
                  {isTodayColumn && <CurrentTimeIndicator rowHeight={rowHeight} />}
```

- [ ] **Step 5: Run the Week tests + the full calendar suite**

```bash
npm test -- --run src/components/calendar/calendar-module.test.tsx
```

Expected: PASS, including the new one-line-header test and the pre-existing Week/navigation tests.

- [ ] **Step 6: Commit**

```bash
git add src/components/calendar/views/weekly-calendar.tsx src/components/calendar/calendar-module.test.tsx
git commit -m "feat(calendar): collapse week day headers to one line and densify lg+ rows"
```

---

## Task 4: Extract Day overlap layout (shared by tablet Day + lanes)

Move the overlap-column logic out of `daily-calendar.tsx` into a tested module so the new `DayLanesCalendar` stacks within-lane overlaps identically. The tablet `DailyCalendar` keeps behaving exactly as before (it just imports the helpers and the shared geometry).

**Files:**
- Create: `src/components/calendar/views/day-lane-layout.ts`
- Test: `src/components/calendar/views/day-lane-layout.test.ts`
- Modify: `src/components/calendar/views/daily-calendar.tsx`

- [ ] **Step 1: Write failing layout tests**

Create `src/components/calendar/views/day-lane-layout.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type { CalendarEvent } from "@/lib/types";
import { calculateEventColumns, eventsOverlap } from "./day-lane-layout";

const ev = (overrides: Partial<CalendarEvent>): CalendarEvent => ({
  id: overrides.id ?? "e",
  title: "Event",
  date: new Date(2026, 6, 6),
  startTime: "9:00 AM",
  endTime: "10:00 AM",
  memberId: "m1",
  isAllDay: false,
  source: "NATIVE",
  ...overrides,
});

describe("day-lane-layout", () => {
  it("detects overlapping and non-overlapping events", () => {
    const a = ev({ startTime: "9:00 AM", endTime: "10:00 AM" });
    const b = ev({ startTime: "9:30 AM", endTime: "10:30 AM" });
    const c = ev({ startTime: "10:00 AM", endTime: "11:00 AM" });
    expect(eventsOverlap(a, b)).toBe(true);
    expect(eventsOverlap(a, c)).toBe(false); // touching edges do not overlap
  });

  it("packs non-overlapping events into a single column", () => {
    const result = calculateEventColumns([
      ev({ id: "a", startTime: "9:00 AM", endTime: "10:00 AM" }),
      ev({ id: "b", startTime: "10:00 AM", endTime: "11:00 AM" }),
    ]);
    expect(result.map((e) => e.column)).toEqual([0, 0]);
    expect(result.every((e) => e.totalColumns === 1)).toBe(true);
  });

  it("assigns overlapping events to separate columns", () => {
    const result = calculateEventColumns([
      ev({ id: "a", startTime: "9:00 AM", endTime: "10:30 AM" }),
      ev({ id: "b", startTime: "9:30 AM", endTime: "10:00 AM" }),
    ]);
    expect(result.find((e) => e.id === "a")?.column).toBe(0);
    expect(result.find((e) => e.id === "b")?.column).toBe(1);
    expect(result.every((e) => e.totalColumns === 2)).toBe(true);
  });

  it("returns an empty array for no events", () => {
    expect(calculateEventColumns([])).toEqual([]);
  });
});
```

- [ ] **Step 2: Run it to verify failure**

```bash
npm test -- --run src/components/calendar/views/day-lane-layout.test.ts
```

Expected: FAIL (`day-lane-layout.ts` does not exist).

- [ ] **Step 3: Create the layout module (moved verbatim from `daily-calendar.tsx`)**

Create `src/components/calendar/views/day-lane-layout.ts`:

```ts
import { getTimeInMinutes } from "@/lib/time-utils";
import type { CalendarEvent } from "@/lib/types";

export interface EventWithLayout extends CalendarEvent {
  column: number;
  totalColumns: number;
}

export function eventsOverlap(a: CalendarEvent, b: CalendarEvent): boolean {
  const aStart = getTimeInMinutes(a.startTime);
  const aEnd = getTimeInMinutes(a.endTime);
  const bStart = getTimeInMinutes(b.startTime);
  const bEnd = getTimeInMinutes(b.endTime);
  return aStart < bEnd && bStart < aEnd;
}

/**
 * Greedy column packing for overlapping timed events. Each event gets the first
 * column where it does not overlap an existing event; totalColumns is the width
 * of its overlap cluster. Used by the tablet Day canvas and each member lane.
 */
export function calculateEventColumns(
  events: CalendarEvent[],
): EventWithLayout[] {
  if (events.length === 0) return [];

  const sorted = [...events].sort((a, b) => {
    const aStart = getTimeInMinutes(a.startTime);
    const bStart = getTimeInMinutes(b.startTime);
    if (aStart !== bStart) return aStart - bStart;
    const aDuration = getTimeInMinutes(a.endTime) - aStart;
    const bDuration = getTimeInMinutes(b.endTime) - bStart;
    return bDuration - aDuration; // Longer events first
  });

  const result: EventWithLayout[] = [];
  const columns: CalendarEvent[][] = [];

  for (const event of sorted) {
    let assignedColumn = -1;
    for (let col = 0; col < columns.length; col++) {
      const hasOverlap = columns[col].some((e) => eventsOverlap(e, event));
      if (!hasOverlap) {
        assignedColumn = col;
        break;
      }
    }
    if (assignedColumn === -1) {
      assignedColumn = columns.length;
      columns.push([]);
    }
    columns[assignedColumn].push(event);
    result.push({ ...event, column: assignedColumn, totalColumns: 0 });
  }

  for (const eventWithLayout of result) {
    const overlapping = result.filter((e) => eventsOverlap(e, eventWithLayout));
    const maxColumn = Math.max(...overlapping.map((e) => e.column));
    eventWithLayout.totalColumns = maxColumn + 1;
  }

  return result;
}
```

- [ ] **Step 4: Update `daily-calendar.tsx` to import the shared helpers**

In `src/components/calendar/views/daily-calendar.tsx`, delete the now-moved local `eventsOverlap`, `calculateEventColumns`, the `EventWithLayout` interface, and the local `timeSlots` array + `getEventGridPosition`/`ROW_HEIGHT` (replace with shared geometry). Change the imports block:

```tsx
import { useMemo, useRef } from "react";
import { useFamilyMembers } from "@/api";
import {
  CALENDAR_START_HOUR,
  compareEventsByTime,
  getEventKey,
  getTimeInMinutes,
  isEventOnDate,
  parseTime,
} from "@/lib/time-utils";
```

to:

```tsx
import { useMemo, useRef } from "react";
import { useFamilyMembers } from "@/api";
import {
  compareEventsByTime,
  getEventKey,
  isEventOnDate,
} from "@/lib/time-utils";
import {
  getEventOffsets,
  pxFromOffsets,
  TIME_SLOTS,
} from "../utils/hour-grid";
import { calculateEventColumns } from "./day-lane-layout";
```

Delete the local `const START_HOUR`, `const ROW_HEIGHT = 80`, `getEventGridPosition`, `eventsOverlap`, `interface EventWithLayout`, and `calculateEventColumns`. Replace the `getEventGridPosition(...)` call inside the render with:

```tsx
            const { top, height } = pxFromOffsets(
              getEventOffsets(event.startTime, event.endTime),
              80,
            );
```

(The tablet Day canvas keeps the 80px row height — only Week and the new lanes densify.) Replace the local `timeSlots` array usage with the imported `TIME_SLOTS` (rename the two `.map` references from `timeSlots` to `TIME_SLOTS`, and keep the `h-20` cell classes as-is for the tablet canvas).

- [ ] **Step 5: Run the layout test + full calendar suite (tablet Day unchanged)**

```bash
npm test -- --run src/components/calendar/views/day-lane-layout.test.ts src/components/calendar/calendar-module.test.tsx
```

Expected: PASS — `daily-calendar.tsx` renders identically; the extraction is behavior-preserving.

- [ ] **Step 6: Type-check + commit**

```bash
npm run build
git add src/components/calendar/views/day-lane-layout.ts src/components/calendar/views/day-lane-layout.test.ts src/components/calendar/views/daily-calendar.tsx
git commit -m "refactor(calendar): extract shared day overlap layout"
```

---

## Task 5: Day rail math (pure) + persisted store toggle

Add the pure "does the rail fit?" math and the month-dot selector, plus the persisted `dayRailHidden` calendar-store field and its reset/seed wiring. No UI yet.

**Files:**
- Create: `src/components/calendar/utils/day-rail.ts`
- Test: `src/components/calendar/utils/day-rail.test.ts`
- Modify: `src/stores/calendar-store.ts`
- Modify: `src/stores/index.ts`
- Modify: `src/test/setup.ts`
- Modify: `src/test/test-utils.tsx`

- [ ] **Step 1: Write failing day-rail math tests**

Create `src/components/calendar/utils/day-rail.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type { CalendarEvent, FamilyMember } from "@/lib/types";
import {
  MIN_LANE_WIDTH,
  RAIL_WIDTH,
  buildMonthMatrix,
  railThresholdPx,
  selectMonthDayDots,
} from "./day-rail";

const member = (id: string, color: FamilyMember["color"]): FamilyMember => ({
  id,
  name: id.toUpperCase(),
  color,
});

const ev = (date: Date, memberId: string): CalendarEvent => ({
  id: `${memberId}-${date.getDate()}`,
  title: "E",
  date,
  startTime: "9:00 AM",
  endTime: "10:00 AM",
  memberId,
  isAllDay: false,
  source: "NATIVE",
});

describe("day-rail math", () => {
  it("threshold grows with member count and accounts for the desktop nav", () => {
    expect(railThresholdPx(3)).toBeLessThan(railThresholdPx(6));
    // 3 comfortable lanes + axis + rail + the 80px desktop nav fit a ~13" laptop
    // (~1280) but need more than a bare 1024…
    expect(railThresholdPx(3)).toBeGreaterThan(1024);
    expect(railThresholdPx(3)).toBeLessThanOrEqual(1120);
    // …and 6 lanes must not fit until well past 1440 (rail yields width to lanes).
    expect(railThresholdPx(6)).toBeGreaterThan(1440);
  });

  it("uses the documented lane/rail widths", () => {
    expect(RAIL_WIDTH).toBe(300);
    expect(MIN_LANE_WIDTH).toBeGreaterThanOrEqual(160);
  });

  it("builds a 6x7 (or 5x7) month matrix covering the current month", () => {
    const matrix = buildMonthMatrix(new Date(2026, 6, 15)); // July 2026
    expect(matrix.length % 7).toBe(0);
    expect(matrix.some((d) => d.getMonth() === 6 && d.getDate() === 1)).toBe(true);
    expect(matrix.some((d) => d.getMonth() === 6 && d.getDate() === 31)).toBe(true);
  });

  it("maps each day to its unique member colors (deduped, filtered)", () => {
    const members = [member("m1", "coral"), member("m2", "teal")];
    const july6 = new Date(2026, 6, 6);
    const dots = selectMonthDayDots(
      [ev(july6, "m1"), ev(july6, "m1"), ev(july6, "m2")],
      members,
    );
    expect(dots.get(july6.toDateString())).toEqual(["coral", "teal"]);
  });
});
```

- [ ] **Step 2: Run it to verify failure**

```bash
npm test -- --run src/components/calendar/utils/day-rail.test.ts
```

Expected: FAIL (`day-rail.ts` does not exist).

- [ ] **Step 3: Implement the day-rail math**

Create `src/components/calendar/utils/day-rail.ts`:

```ts
import { isEventOnDate } from "@/lib/time-utils";
import type { CalendarEvent, FamilyColor, FamilyMember } from "@/lib/types";

/** Fixed rail width when shown (spec §4.2, ~300px). Tunable in the screenshot gate. */
export const RAIL_WIDTH = 300;
/** Minimum comfortable member-lane width before the rail yields its space back. */
export const MIN_LANE_WIDTH = 190;
/** Time-axis gutter width (matches the `w-16` axis = 64px). */
export const TIME_AXIS_WIDTH = 64;
/**
 * Persistent desktop module-nav rail (`NavigationTabs`, `w-20` = 80px, rendered
 * `{!isMobile && <NavigationTabs/>}` in `App.tsx`) sits left of the calendar, so
 * the calendar's content box is `viewport - 80`. The threshold must include it.
 */
export const DESKTOP_NAV_WIDTH = 80;
/** Slack for lane borders/padding so the threshold is not razor-thin. */
export const RAIL_LAYOUT_SLACK = 48;

/**
 * Minimum *viewport* width at which the mini-month rail can appear for a family
 * of `memberCount`: the desktop nav + time axis + every lane at its comfortable
 * minimum + the rail + slack. `useMediaQuery` compares this to the viewport, so
 * the nav term keeps the check honest about the calendar's real content width.
 * Above this the rail shows (unless hidden); below it the lanes need the width
 * more and the rail stays hidden (spec §4.2).
 */
export function railThresholdPx(memberCount: number): number {
  return (
    DESKTOP_NAV_WIDTH +
    TIME_AXIS_WIDTH +
    MIN_LANE_WIDTH * Math.max(memberCount, 1) +
    RAIL_WIDTH +
    RAIL_LAYOUT_SLACK
  );
}

/**
 * Days (Sunday-aligned weeks) spanning the month containing `currentDate`,
 * padded with leading/trailing days to whole weeks. Navigator grid for the rail.
 */
export function buildMonthMatrix(currentDate: Date): Date[] {
  const year = currentDate.getFullYear();
  const month = currentDate.getMonth();
  const firstDay = new Date(year, month, 1);
  const lastDay = new Date(year, month + 1, 0);

  const result: Date[] = [];
  const leading = firstDay.getDay();
  const prevMonthLastDate = new Date(year, month, 0).getDate();
  for (let i = leading - 1; i >= 0; i--) {
    result.push(new Date(year, month - 1, prevMonthLastDate - i));
  }
  for (let d = 1; d <= lastDay.getDate(); d++) {
    result.push(new Date(year, month, d));
  }
  let nextDay = 1;
  while (result.length % 7 !== 0) {
    result.push(new Date(year, month + 1, nextDay++));
  }
  return result;
}

/**
 * For each day that has at least one event, the deduped list of member colors
 * (in family order) to render as dots. Respects only events passed in, so the
 * caller controls filtering (the rail mirrors the active member/all-day filter).
 */
export function selectMonthDayDots(
  events: CalendarEvent[],
  members: FamilyMember[],
): Map<string, FamilyColor[]> {
  // Per-day set of member ids that own an event on that day.
  const dayMemberIds = new Map<string, Set<string>>();
  for (const event of events) {
    for (const day of uniqueEventDays(event)) {
      const key = day.toDateString();
      if (!dayMemberIds.has(key)) dayMemberIds.set(key, new Set());
      dayMemberIds.get(key)?.add(event.memberId);
    }
  }

  const result = new Map<string, FamilyColor[]>();
  for (const [key, ids] of dayMemberIds) {
    const colors = members.filter((m) => ids.has(m.id)).map((m) => m.color);
    if (colors.length > 0) result.set(key, colors);
  }
  return result;
}

/** The distinct calendar days an event covers (single- or multi-day). */
function uniqueEventDays(event: CalendarEvent): Date[] {
  const days: Date[] = [event.date];
  if (event.endDate) {
    const cursor = new Date(event.date);
    cursor.setDate(cursor.getDate() + 1);
    while (cursor <= event.endDate) {
      days.push(new Date(cursor));
      cursor.setDate(cursor.getDate() + 1);
    }
  }
  return days.filter((day) => isEventOnDate(event, day));
}
```

- [ ] **Step 4: Run the math tests + build**

```bash
npm test -- --run src/components/calendar/utils/day-rail.test.ts
npm run build
```

Expected: PASS + clean build.

- [ ] **Step 5: Add the persisted `dayRailHidden` store field**

In `src/stores/calendar-store.ts`:

Add to the `CalendarState` interface (near `filter`):

```ts
  /** User preference: hide the Day view mini-month rail even when it would fit. */
  dayRailHidden: boolean;
  toggleDayRail: () => void;
```

Add the initial value (near `isAddEventModalOpen: false`):

```ts
      dayRailHidden: false,
```

Add the action (near the filter actions):

```ts
      toggleDayRail: () =>
        set((state) => ({ dayRailHidden: !state.dayRailHidden })),
```

Extend `partialize` to persist it:

```ts
      partialize: (state) => ({
        filter: state.filter,
        calendarView: state.calendarView,
        hasUserSetView: state.hasUserSetView,
        dayRailHidden: state.dayRailHidden,
      }),
```

Add a small selector at the bottom of the file (near `useIsViewingToday`):

```ts
/** Selector: the Day mini-month rail hide preference + its toggle. */
export const useDayRailState = () =>
  useCalendarStore(
    useShallow((state) => ({
      dayRailHidden: state.dayRailHidden,
      toggleDayRail: state.toggleDayRail,
    })),
  );
```

The `@/stores` barrel is a selective re-export, so add `useDayRailState` to the `./calendar-store` export block in `src/stores/index.ts`:

```ts
export {
  useCalendarActions,
  useCalendarState,
  useCalendarStore,
  useDayRailState,
  useEditModalState,
  useEventDetailState,
  useFilterPillsState,
  useHasUserSetView,
  useIsViewingToday,
} from "./calendar-store";
```

- [ ] **Step 6: Wire the new field into test reset/seed (prevents state leakage)**

In `src/test/setup.ts`, add `dayRailHidden: false,` to the `useCalendarStore.setState({ ... })` block inside `resetAllStores()`.

In `src/test/test-utils.tsx`:
- Add `dayRailHidden?: boolean;` to the `seedCalendarStore` param type and a spread line `...(data.dayRailHidden !== undefined && { dayRailHidden: data.dayRailHidden }),`.
- Add `dayRailHidden: false,` to the `resetCalendarStore()` `setState({ ... })` block.

- [ ] **Step 7: Run the store tests + full suite**

```bash
npm test -- --run src/stores/calendar-store.test.ts src/components/calendar/calendar-module.test.tsx
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add src/components/calendar/utils/day-rail.ts src/components/calendar/utils/day-rail.test.ts src/stores/calendar-store.ts src/stores/index.ts src/test/setup.ts src/test/test-utils.tsx
git commit -m "feat(calendar): add day-rail fit math and persisted rail toggle state"
```

---

## Task 6: Day mini-month rail component + toolbar toggle

Build the pure navigator rail and the toolbar toggle button. Both are unit-tested with props; the month fetch lives in a thin container mounted only when the rail shows (Task 7).

**Files:**
- Create: `src/components/calendar/components/day-mini-month-rail.tsx`
- Test: `src/components/calendar/components/day-mini-month-rail.test.tsx`
- Create: `src/components/calendar/components/day-rail-toggle.tsx`
- Test: `src/components/calendar/components/day-rail-toggle.test.tsx`
- Modify: `src/components/calendar/components/index.ts`

- [ ] **Step 1: Write the failing rail test**

Create `src/components/calendar/components/day-mini-month-rail.test.tsx`:

```tsx
import { describe, expect, it, vi } from "vitest";
import type { CalendarEvent, FamilyMember } from "@/lib/types";
import { render, renderWithUser, screen } from "@/test/test-utils";
import { DayMiniMonthRail } from "./day-mini-month-rail";

const members: FamilyMember[] = [
  { id: "m1", name: "Alice", color: "coral" },
  { id: "m2", name: "Ben", color: "teal" },
];

const ev = (date: Date, memberId: string): CalendarEvent => ({
  id: `${memberId}-${date.getDate()}`,
  title: "E",
  date,
  startTime: "9:00 AM",
  endTime: "10:00 AM",
  memberId,
  isAllDay: false,
  source: "NATIVE",
});

describe("DayMiniMonthRail", () => {
  it("renders a labelled month navigator with the selected day pressed", () => {
    render(
      <DayMiniMonthRail
        currentDate={new Date(2026, 6, 15)}
        monthEvents={[ev(new Date(2026, 6, 6), "m1")]}
        members={members}
        onSelectDate={vi.fn()}
      />,
    );

    expect(screen.getByRole("grid", { name: /july 2026/i })).toBeInTheDocument();
    expect(
      screen.getByRole("gridcell", { name: /july 15/i, selected: true }),
    ).toBeInTheDocument();
  });

  it("routes a day tap through onSelectDate", async () => {
    const onSelectDate = vi.fn();
    const { user } = renderWithUser(
      <DayMiniMonthRail
        currentDate={new Date(2026, 6, 15)}
        monthEvents={[]}
        members={members}
        onSelectDate={onSelectDate}
      />,
    );

    await user.click(screen.getByRole("gridcell", { name: /july 20/i }));
    expect(onSelectDate).toHaveBeenCalledTimes(1);
    const selected = onSelectDate.mock.calls[0][0] as Date;
    expect(selected.getFullYear()).toBe(2026);
    expect(selected.getMonth()).toBe(6);
    expect(selected.getDate()).toBe(20);
  });
});
```

- [ ] **Step 2: Run it to verify failure**

```bash
npm test -- --run src/components/calendar/components/day-mini-month-rail.test.tsx
```

Expected: FAIL (component does not exist).

- [ ] **Step 3: Implement the pure rail**

Create `src/components/calendar/components/day-mini-month-rail.tsx`:

```tsx
import { format } from "date-fns";
import { DAY_INITIALS } from "@/lib/time-utils";
import { type CalendarEvent, colorMap, type FamilyMember } from "@/lib/types";
import { cn } from "@/lib/utils";
import { buildMonthMatrix, RAIL_WIDTH, selectMonthDayDots } from "../utils/day-rail";

interface DayMiniMonthRailProps {
  currentDate: Date;
  monthEvents: CalendarEvent[];
  members: FamilyMember[];
  onSelectDate: (date: Date) => void;
}

/**
 * Compact month navigator for the Day view (spec §4.2): member-colored dots on
 * days with events, today ringed, the viewed date highlighted. Navigator only —
 * a day tap stays in Day view and jumps to that date. Never rendered on mobile.
 */
export function DayMiniMonthRail({
  currentDate,
  monthEvents,
  members,
  onSelectDate,
}: DayMiniMonthRailProps) {
  const today = new Date();
  const matrix = buildMonthMatrix(currentDate);
  const dots = selectMonthDayDots(monthEvents, members);
  const monthLabel = format(currentDate, "MMMM yyyy");

  const isSameDay = (a: Date, b: Date) => a.toDateString() === b.toDateString();
  const isCurrentMonth = (d: Date) => d.getMonth() === currentDate.getMonth();

  return (
    <aside
      aria-label="Month navigator"
      className="shrink-0 border-l border-border bg-card px-3 py-4"
      style={{ width: RAIL_WIDTH }}
    >
      <p className="mb-3 px-1 text-sm font-semibold text-foreground">
        {monthLabel}
      </p>
      <div
        role="grid"
        aria-label={monthLabel}
        className="grid grid-cols-7 gap-y-1"
      >
        {DAY_INITIALS.map((initial, i) => (
          <div
            key={`h-${i}`}
            className="pb-1 text-center text-[10px] font-medium text-muted-foreground"
          >
            {initial}
          </div>
        ))}
        {matrix.map((date) => {
          const dayDots = dots.get(date.toDateString()) ?? [];
          const selected = isSameDay(date, currentDate);
          const dayIsToday = isSameDay(date, today);
          return (
            <button
              type="button"
              key={date.toDateString()}
              role="gridcell"
              aria-selected={selected}
              aria-label={format(date, "MMMM d")}
              onClick={() => onSelectDate(date)}
              className={cn(
                "relative mx-auto flex h-11 w-11 flex-col items-center justify-center rounded-full text-sm transition-colors",
                !isCurrentMonth(date) && "text-muted-foreground/40",
                dayIsToday && !selected && "ring-2 ring-primary ring-inset",
                selected
                  ? "bg-primary font-bold text-primary-foreground"
                  : "hover:bg-muted",
              )}
            >
              <span className="tabular-nums">{date.getDate()}</span>
              <span className="absolute bottom-1 flex gap-0.5">
                {dayDots.slice(0, 4).map((color, i) => (
                  <span
                    key={`${color}-${i}`}
                    className={cn(
                      "h-1 w-1 rounded-full",
                      selected ? "bg-primary-foreground/80" : colorMap[color]?.bg,
                    )}
                  />
                ))}
              </span>
            </button>
          );
        })}
      </div>
    </aside>
  );
}
```

- [ ] **Step 4: Run the rail test**

```bash
npm test -- --run src/components/calendar/components/day-mini-month-rail.test.tsx
```

Expected: PASS.

- [ ] **Step 5: Write the failing toggle test**

Create `src/components/calendar/components/day-rail-toggle.test.tsx`:

```tsx
import { describe, expect, it } from "vitest";
import { useCalendarStore } from "@/stores";
import { renderWithUser, screen } from "@/test/test-utils";
import { DayRailToggle } from "./day-rail-toggle";

describe("DayRailToggle", () => {
  it("reflects and flips the persisted rail preference", async () => {
    useCalendarStore.setState({ dayRailHidden: false });
    const { user } = renderWithUser(<DayRailToggle />);

    const button = screen.getByRole("button", { name: /hide month navigator/i });
    await user.click(button);

    expect(useCalendarStore.getState().dayRailHidden).toBe(true);
    expect(
      screen.getByRole("button", { name: /show month navigator/i }),
    ).toBeInTheDocument();
  });
});
```

- [ ] **Step 6: Run it to verify failure**

```bash
npm test -- --run src/components/calendar/components/day-rail-toggle.test.tsx
```

Expected: FAIL (component does not exist).

- [ ] **Step 7: Implement the toggle**

Create `src/components/calendar/components/day-rail-toggle.tsx`:

```tsx
import { PanelRightClose, PanelRightOpen } from "lucide-react";
import { cn } from "@/lib/utils";
import { useDayRailState } from "@/stores";

/**
 * Toolbar toggle for the Day mini-month rail. Rendered only when the rail can
 * fit (see CalendarModule); flips the persisted `dayRailHidden` preference.
 */
export function DayRailToggle() {
  const { dayRailHidden, toggleDayRail } = useDayRailState();
  const Icon = dayRailHidden ? PanelRightOpen : PanelRightClose;

  return (
    <button
      type="button"
      onClick={toggleDayRail}
      aria-pressed={!dayRailHidden}
      aria-label={dayRailHidden ? "Show month navigator" : "Hide month navigator"}
      className={cn(
        "flex min-h-11 min-w-11 items-center justify-center rounded-full border transition-colors",
        dayRailHidden
          ? "border-border bg-muted/50 text-muted-foreground hover:bg-muted"
          : "border-primary/30 bg-primary/10 text-primary",
      )}
    >
      <Icon className="h-5 w-5" />
    </button>
  );
}
```

Add both to `src/components/calendar/components/index.ts`:

```ts
export { DayMiniMonthRail } from "./day-mini-month-rail";
export { DayRailToggle } from "./day-rail-toggle";
```

- [ ] **Step 8: Run both component tests**

```bash
npm test -- --run src/components/calendar/components/day-mini-month-rail.test.tsx src/components/calendar/components/day-rail-toggle.test.tsx
```

Expected: PASS.

- [ ] **Step 9: Commit**

```bash
git add src/components/calendar/components/day-mini-month-rail.tsx src/components/calendar/components/day-mini-month-rail.test.tsx src/components/calendar/components/day-rail-toggle.tsx src/components/calendar/components/day-rail-toggle.test.tsx src/components/calendar/components/index.ts
git commit -m "feat(calendar): add day mini-month rail and toolbar toggle"
```

---

## Task 7: Day member-lanes view + module wiring

Add the `lg+`-only `DayLanesCalendar` (member lanes + aligned all-day band + optional rail), and switch `CalendarModule` to it for Day at `lg+`, rendering the toolbar toggle when the rail can fit.

**Files:**
- Create: `src/components/calendar/views/day-lanes-calendar.tsx`
- Test: `src/components/calendar/views/day-lanes-calendar.test.tsx`
- Modify: `src/components/calendar/views/index.ts`
- Modify: `src/components/calendar/calendar-module.tsx`
- Modify: `src/components/calendar/calendar-module.test.tsx`

- [ ] **Step 1: Write the failing lanes-view test**

Create `src/components/calendar/views/day-lanes-calendar.test.tsx`:

```tsx
import { beforeEach, describe, expect, it, vi } from "vitest";
import type { CalendarEvent, FamilyMember } from "@/lib/types";
import {
  render,
  renderWithUser,
  screen,
  seedFamilyStore,
  within,
} from "@/test/test-utils";
import { DayLanesCalendar } from "./day-lanes-calendar";

const members: FamilyMember[] = [
  { id: "m1", name: "Alice", color: "coral" },
  { id: "m2", name: "Ben", color: "teal" },
];

const base = {
  date: new Date(2026, 6, 6),
  source: "NATIVE" as const,
  isAllDay: false,
};

const events: CalendarEvent[] = [
  { ...base, id: "a", title: "Swim", startTime: "9:00 AM", endTime: "10:00 AM", memberId: "m1" },
  { ...base, id: "b", title: "Soccer", startTime: "11:00 AM", endTime: "12:00 PM", memberId: "m2" },
  { ...base, id: "c", title: "Camp week", startTime: "12:00 AM", endTime: "12:00 AM", memberId: "m1", isAllDay: true },
];

const noopFilter = { selectedMembers: ["m1", "m2"], showAllDayEvents: true };

describe("DayLanesCalendar", () => {
  // CalendarEventCard resolves member colors via useFamilyMembers (the useFamily
  // query), which seedFamilyStore hydrates.
  beforeEach(() => {
    seedFamilyStore({ name: "Test Family", members });
  });

  it("renders one labelled lane per member in family order", () => {
    render(
      <DayLanesCalendar
        events={events}
        currentDate={new Date(2026, 6, 6)}
        members={members}
        filter={noopFilter}
        showRail={false}
        onEventClick={vi.fn()}
        onSelectDate={vi.fn()}
      />,
    );

    const alice = screen.getByRole("group", { name: /alice's schedule/i });
    const ben = screen.getByRole("group", { name: /ben's schedule/i });
    expect(alice).toBeInTheDocument();
    expect(ben).toBeInTheDocument();
    // Each event sits in its owner's lane.
    expect(within(alice).getByText("Swim")).toBeInTheDocument();
    expect(within(ben).getByText("Soccer")).toBeInTheDocument();
  });

  it("renders all-day chips in the band aligned under the owning member", () => {
    render(
      <DayLanesCalendar
        events={events}
        currentDate={new Date(2026, 6, 6)}
        members={members}
        filter={noopFilter}
        showRail={false}
        onEventClick={vi.fn()}
        onSelectDate={vi.fn()}
      />,
    );

    const band = screen.getByRole("group", { name: /all-day events/i });
    expect(within(band).getByRole("button", { name: /camp week/i })).toBeInTheDocument();
  });

  it("routes lane event taps through onEventClick", async () => {
    const onEventClick = vi.fn();
    const { user } = renderWithUser(
      <DayLanesCalendar
        events={events}
        currentDate={new Date(2026, 6, 6)}
        members={members}
        filter={noopFilter}
        showRail={false}
        onEventClick={onEventClick}
        onSelectDate={vi.fn()}
      />,
    );

    await user.click(screen.getByText("Swim"));
    expect(onEventClick).toHaveBeenCalledWith(
      expect.objectContaining({ id: "a" }),
    );
  });

  it("does not render the rail when showRail is false", () => {
    render(
      <DayLanesCalendar
        events={events}
        currentDate={new Date(2026, 6, 6)}
        members={members}
        filter={noopFilter}
        showRail={false}
        onEventClick={vi.fn()}
        onSelectDate={vi.fn()}
      />,
    );
    expect(screen.queryByRole("grid", { name: /july 2026/i })).not.toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run it to verify failure**

```bash
npm test -- --run src/components/calendar/views/day-lanes-calendar.test.tsx
```

Expected: FAIL (component does not exist).

- [ ] **Step 3: Implement `DayLanesCalendar`**

Create `src/components/calendar/views/day-lanes-calendar.tsx`. It reuses the shared geometry (dense rows — it is `lg+`-only) and the extracted overlap layout **per member**; the rail is a lazily-fetched container mounted only when `showRail`.

```tsx
import { endOfMonth, format, startOfMonth } from "date-fns";
import { useMemo, useRef } from "react";
import { useCalendarEvents } from "@/api";
import {
  CALENDAR_START_HOUR,
  compareEventsByTime,
  getEventKey,
  isEventOnDate,
} from "@/lib/time-utils";
import { type CalendarEvent, colorMap, type FamilyMember } from "@/lib/types";
import { cn } from "@/lib/utils";
import { CalendarEventCard } from "../components/calendar-event";
import type { FilterState } from "../components/calendar-filter";
import {
  CurrentTimeIndicator,
  useAutoScrollToNow,
} from "../components/current-time-indicator";
import { DayMiniMonthRail } from "../components/day-mini-month-rail";
import { MemberAvatar } from "../components/member-avatar";
import {
  DENSE_HOUR_ROW_HEIGHT,
  getEventOffsets,
  pxFromOffsets,
  TIME_SLOTS,
} from "../utils/hour-grid";
import { calculateEventColumns } from "./day-lane-layout";

interface DayLanesCalendarProps {
  events: CalendarEvent[];
  currentDate: Date;
  members: FamilyMember[];
  filter: FilterState;
  showRail: boolean;
  onEventClick: (event: CalendarEvent) => void;
  onSelectDate: (date: Date) => void;
}

const ROW_HEIGHT = DENSE_HOUR_ROW_HEIGHT;

/** Lazily-fetched month data for the rail; only mounted when the rail shows. */
function DayRailPanel({
  currentDate,
  members,
  filter,
  onSelectDate,
}: {
  currentDate: Date;
  members: FamilyMember[];
  filter: FilterState;
  onSelectDate: (date: Date) => void;
}) {
  const monthRange = useMemo(
    () => ({
      startDate: format(startOfMonth(currentDate), "yyyy-MM-dd"),
      endDate: format(endOfMonth(currentDate), "yyyy-MM-dd"),
    }),
    [currentDate],
  );
  const { data } = useCalendarEvents(monthRange);
  const monthEvents = useMemo(() => {
    const raw = data?.data ?? [];
    // Rail mirrors the active filter so dots match what the lanes show.
    return raw.filter(
      (event) =>
        filter.selectedMembers.includes(event.memberId) &&
        (filter.showAllDayEvents || !event.isAllDay),
    );
  }, [data, filter]);

  return (
    <DayMiniMonthRail
      currentDate={currentDate}
      monthEvents={monthEvents}
      members={members}
      onSelectDate={onSelectDate}
    />
  );
}

export function DayLanesCalendar({
  events,
  currentDate,
  members,
  filter,
  showRail,
  onEventClick,
  onSelectDate,
}: DayLanesCalendarProps) {
  const today = new Date();
  const isCurrentDay = currentDate.toDateString() === today.toDateString();
  const scrollContainerRef = useRef<HTMLDivElement>(null);
  useAutoScrollToNow(
    isCurrentDay ? scrollContainerRef : { current: null },
    CALENDAR_START_HOUR,
    ROW_HEIGHT,
  );

  // Events on this day passing the active filter, grouped by member id.
  const timedByMember = useMemo(() => {
    const map = new Map<string, CalendarEvent[]>();
    for (const member of members) map.set(member.id, []);
    for (const event of events) {
      if (event.isAllDay) continue;
      if (!isEventOnDate(event, currentDate)) continue;
      if (!filter.selectedMembers.includes(event.memberId)) continue;
      map.get(event.memberId)?.push(event);
    }
    for (const list of map.values()) list.sort(compareEventsByTime);
    return map;
  }, [events, members, currentDate, filter.selectedMembers]);

  const allDayByMember = useMemo(() => {
    const map = new Map<string, CalendarEvent[]>();
    for (const member of members) map.set(member.id, []);
    if (!filter.showAllDayEvents) return map;
    for (const event of events) {
      if (!event.isAllDay) continue;
      if (!isEventOnDate(event, currentDate)) continue;
      if (!filter.selectedMembers.includes(event.memberId)) continue;
      map.get(event.memberId)?.push(event);
    }
    return map;
  }, [events, members, currentDate, filter.selectedMembers, filter.showAllDayEvents]);

  const hasAnyAllDay = useMemo(
    () => [...allDayByMember.values()].some((list) => list.length > 0),
    [allDayByMember],
  );

  // One CSS grid template shared by the header, all-day band, and body so lane
  // columns stay aligned in every scroll position. minmax(0, 1fr) lets lanes
  // grow to fill spare width and shrink to fit rather than overflow — so the
  // shrink-0 header/all-day grids never clip and the body scrolls only
  // vertically (mirrors WeeklyCalendar's header). The 4rem first track is the
  // time axis (= w-16 = 64px). The rail shows only when lanes stay comfortable
  // (railThresholdPx), so shrink-to-fit never makes visible lanes too thin at
  // the widths the rail is present; Task 8 tunes the many-members-narrow case.
  const gridTemplateColumns = `4rem repeat(${members.length}, minmax(0, 1fr))`;

  return (
    <div className="flex-1 flex overflow-hidden bg-background">
      {/* Lanes region (fills remaining width beside the rail) */}
      <div className="flex-1 flex flex-col overflow-hidden">
        {/* Lane headers */}
        <div
          className="grid border-b border-border bg-card shrink-0"
          style={{ gridTemplateColumns }}
        >
          <div className="border-r border-border" />
          {members.map((member) => (
            <div
              key={member.id}
              className="flex items-center gap-2 border-l border-border px-3 py-2"
            >
              <MemberAvatar name={member.name} color={member.color} size="md" />
              <span className="truncate text-sm font-semibold text-foreground">
                {member.name}
              </span>
              <span
                className={cn(
                  "ml-auto h-1.5 w-6 shrink-0 rounded-full",
                  colorMap[member.color]?.bg,
                )}
              />
            </div>
          ))}
        </div>

        {/* All-day band aligned under each lane */}
        {hasAnyAllDay && (
          <div
            role="group"
            aria-label="All-day events"
            className="grid border-b border-border bg-card shrink-0"
            style={{ gridTemplateColumns }}
          >
            <div className="border-r border-border flex items-center justify-end pr-2">
              <span className="text-xs font-medium text-muted-foreground">
                All Day
              </span>
            </div>
            {members.map((member) => {
              const colors = colorMap[member.color];
              return (
                <div
                  key={member.id}
                  className="flex flex-col gap-1 border-l border-border p-1 min-h-[36px]"
                >
                  {(allDayByMember.get(member.id) ?? []).map((event) => (
                    <button
                      type="button"
                      key={getEventKey(event)}
                      onClick={() => onEventClick(event)}
                      title={event.title}
                      aria-label={`${event.title} - All day event`}
                      className={cn(
                        "text-[10px] font-medium px-2 py-1 min-h-[28px] rounded-md text-white truncate w-full text-left transition-all hover:brightness-110",
                        colors?.bg || "bg-muted",
                      )}
                    >
                      {event.title}
                    </button>
                  ))}
                </div>
              );
            })}
          </div>
        )}

        {/* Time axis + lane bodies (single scroll container keeps them aligned) */}
        <div className="flex-1 overflow-auto" ref={scrollContainerRef}>
          <div className="grid min-h-full" style={{ gridTemplateColumns }}>
            {/* Time axis */}
            <div className="bg-card border-r border-border">
              {TIME_SLOTS.map((time, index) => (
                <div
                  key={index}
                  style={{ height: ROW_HEIGHT }}
                  className="flex items-start justify-end pr-2 pt-1 border-b border-border/50"
                >
                  <span className="text-xs text-foreground/70 font-semibold">
                    {time}
                  </span>
                </div>
              ))}
            </div>

            {/* Member lanes */}
            {members.map((member) => {
              const laneEvents = calculateEventColumns(
                timedByMember.get(member.id) ?? [],
              );
              return (
                <div
                  key={member.id}
                  role="group"
                  aria-label={`${member.name}'s schedule`}
                  className="relative border-l border-border"
                >
                  {TIME_SLOTS.map((_, index) => (
                    <div
                      key={index}
                      style={{ height: ROW_HEIGHT }}
                      className={cn(
                        "border-b border-border/50",
                        index % 2 === 0 && "bg-muted/20",
                      )}
                    />
                  ))}

                  {isCurrentDay && <CurrentTimeIndicator rowHeight={ROW_HEIGHT} />}

                  <div className="absolute inset-0 px-0.5">
                    {laneEvents.map((event) => {
                      const { top, height } = pxFromOffsets(
                        getEventOffsets(event.startTime, event.endTime),
                        ROW_HEIGHT,
                      );
                      const effectiveColumns = Math.min(event.totalColumns, 2);
                      const columnWidth = 100 / effectiveColumns;
                      const left = Math.min(event.column, 1) * columnWidth;
                      const variant =
                        event.totalColumns >= 2 ? "compact" : "default";
                      return (
                        <div
                          key={getEventKey(event)}
                          className="absolute overflow-hidden rounded-xl"
                          style={{
                            top: `${top}px`,
                            height: `${height}px`,
                            left: `calc(${left}% + 2px)`,
                            width: `calc(${columnWidth}% - 4px)`,
                          }}
                        >
                          <CalendarEventCard
                            event={event}
                            onClick={() => onEventClick(event)}
                            variant={variant}
                          />
                        </div>
                      );
                    })}
                  </div>
                </div>
              );
            })}
          </div>
        </div>
      </div>

      {/* Mini-month navigator rail */}
      {showRail && (
        <DayRailPanel
          currentDate={currentDate}
          members={members}
          filter={filter}
          onSelectDate={onSelectDate}
        />
      )}
    </div>
  );
}
```

- [ ] **Step 4: Run the lanes-view test**

```bash
npm test -- --run src/components/calendar/views/day-lanes-calendar.test.tsx
```

Expected: PASS.

- [ ] **Step 5: Export the view and wire the module switch**

Add to `src/components/calendar/views/index.ts`:

```ts
export { DayLanesCalendar } from "./day-lanes-calendar";
```

In `src/components/calendar/calendar-module.tsx`:

1. Extend the barrel import to include `DayLanesCalendar` and `DayRailToggle`, and add hook/util imports:

```tsx
import { useIsLargeScreen, useIsMobile, useMediaQuery } from "@/hooks";
import { railThresholdPx } from "@/components/calendar/utils/day-rail";
import { useDayRailState } from "@/stores";
```

(Add `DayLanesCalendar` and `DayRailToggle` to the existing `@/components/calendar` import list.)

2. Inside `CalendarModule`, after `const isViewingToday = useIsViewingToday();`, add:

```tsx
  const isLargeScreen = useIsLargeScreen();
  const { dayRailHidden } = useDayRailState();
  const memberCount = members.length;
  // Pure threshold → matchMedia keeps this reactive without a ResizeObserver.
  const canFitRail = useMediaQuery(
    `(min-width: ${railThresholdPx(memberCount)}px)`,
  );
  const isDayLanes = !isMobile && isLargeScreen && calendarView === "daily";
  const showDayRail = isDayLanes && canFitRail && !dayRailHidden && memberCount > 0;
  const showRailToggle = isDayLanes && canFitRail && memberCount > 0;
```

> Hooks rule: `useMediaQuery` is called unconditionally every render (member count only changes the query string), so this respects the rules of hooks.

3. In the desktop `switch (calendarView)` (the second switch, ~`:461`), replace the `daily` case:

```tsx
      case "daily":
        return <DailyCalendar {...commonProps} />;
```

with:

```tsx
      case "daily":
        return isLargeScreen ? (
          <DayLanesCalendar
            events={events}
            currentDate={currentDate}
            members={members}
            filter={filter}
            showRail={showDayRail}
            onEventClick={handleEventClick}
            onSelectDate={selectDateAndSwitchToDaily}
          />
        ) : (
          <DailyCalendar {...commonProps} />
        );
```

4. Render the toggle in the merged toolbar. Change the toolbar block:

```tsx
        <div className="flex flex-wrap items-center justify-between gap-x-2 gap-y-2 px-3 py-2 bg-card border-b border-border">
          <CalendarViewSwitcher />
          <CalendarNavigation
            label={getContextLabel(calendarView, currentDate)}
            onPrevious={goToPrevious}
            onNext={goToNext}
            onToday={goToToday}
            isViewingToday={isViewingToday}
          />
          <FamilyFilterPills />
        </div>
```

to add the toggle beside the filter pills:

```tsx
        <div className="flex flex-wrap items-center justify-between gap-x-2 gap-y-2 px-3 py-2 bg-card border-b border-border">
          <CalendarViewSwitcher />
          <CalendarNavigation
            label={getContextLabel(calendarView, currentDate)}
            onPrevious={goToPrevious}
            onNext={goToNext}
            onToday={goToToday}
            isViewingToday={isViewingToday}
          />
          <div className="flex items-center gap-2">
            <FamilyFilterPills />
            {showRailToggle && <DayRailToggle />}
          </div>
        </div>
```

- [ ] **Step 6: Add a module-level Day-lanes integration test (matchMedia override)**

In `src/components/calendar/calendar-module.test.tsx`, add a describe block that forces the large-screen + rail-fit media queries true. Because `@/hooks` is mocked in this file (only `useIsMobile` overridden, `actual` for the rest), and `useIsLargeScreen`/`useMediaQuery` call the global `window.matchMedia`, override `matchMedia` to match min-width queries:

```tsx
describe("Day view large-screen lanes", () => {
  // matchMedia is globally mocked to matches:false. Override so min-width queries
  // (the lg breakpoint + the rail threshold) match, then restore to the default
  // in afterEach so the override cannot leak to later tests in this file
  // (setup.ts's clearAllMocks clears call data, not the implementation).
  function setMatchMedia(matches: (query: string) => boolean) {
    vi.mocked(window.matchMedia).mockImplementation((query: string) => ({
      matches: matches(query),
      media: query,
      onchange: null,
      addListener: vi.fn(),
      removeListener: vi.fn(),
      addEventListener: vi.fn(),
      removeEventListener: vi.fn(),
      dispatchEvent: vi.fn(),
    }));
  }

  beforeEach(() => setMatchMedia((query) => /min-width/.test(query)));
  afterEach(() => setMatchMedia(() => false));

  it("renders member lanes and the rail toggle on the daily view at lg+", async () => {
    seedMockEvents([]);
    seedCalendarStore({
      currentDate: new Date(2026, 6, 6),
      calendarView: "daily",
      filter: { selectedMembers: testMembers.map((m) => m.id), showAllDayEvents: true },
    });

    render(<CalendarModule />);

    await waitFor(() => {
      expect(screen.queryByText("Loading events...")).not.toBeInTheDocument();
    });

    // One lane group per member.
    expect(
      screen.getByRole("group", { name: new RegExp(`${testMembers[0].name}'s schedule`, "i") }),
    ).toBeInTheDocument();
    // Rail toggle present (rail fits at forced-wide width).
    expect(
      screen.getByRole("button", { name: /month navigator/i }),
    ).toBeInTheDocument();
  });
});
```

- [ ] **Step 7: Run the full calendar suite + build**

```bash
npm test -- --run src/components/calendar/
npm run build
```

Expected: PASS + clean build. The Day-lanes switch does not touch the `calendarEventIntent` effect (`calendar-module.tsx:145-296`), so the Home→Calendar event-tap path is unaffected — confirm the existing `calendar-module.test.tsx` test "opens the requested event detail when a calendar event intent is present" (`:308`) still passes.

- [ ] **Step 8: Commit**

```bash
git add src/components/calendar/views/day-lanes-calendar.tsx src/components/calendar/views/day-lanes-calendar.test.tsx src/components/calendar/views/index.ts src/components/calendar/calendar-module.tsx src/components/calendar/calendar-module.test.tsx
git commit -m "feat(calendar): render per-member day lanes with optional rail on lg+"
```

---

## Task 8: Quality gates + screenshot critique

**Files:** none (verification + iteration)

- [ ] **Step 1: Full gate**

```bash
npm run build && npm run lint && npm test -- --run
```

Expected: all green. Fix any Biome unused-import flags from the refactors (Task 3/4 removed helpers) before continuing.

- [ ] **Step 2: Start the real backend + dev server**

The screenshot gate must run against the real backend (not MSW). The compose file hard-fails without `BE_IMAGE_TAG`; resolve the latest published BE release the same way CI does (GHCR tag has **no** `v` prefix; env vars do **not** persist across shell calls):

```bash
cd /Users/joe.bor/code/family-hub/frontend
BE_IMAGE_TAG=$(GITHUB_TOKEN=$(gh auth token) bash .github/scripts/resolve-backend-version.sh) \
  docker compose -f docker-compose.e2e.yml up -d --wait
npm run dev
```

Then register/seed a test family, add events so the matrix scenarios exist, and open the Calendar module. Note (verified on `main`): desktop lands on **Home**, which prefetches chores/meals/lists — navigate to Calendar explicitly; this calendar work must not disturb that landing or the Home→Calendar `calendarEventIntent` tap-through (a Home event tap should still open the event's detail in Calendar).

- [ ] **Step 3: Capture the spec's screenshot matrix**

At **1024px, 1440px, and a large-display width** (e.g. 1920px), capture and critique:

| Scenario | View | What to verify |
|---|---|---|
| Busy week | Week | ≥12 hour rows at 1440x900; one-line headers; single toolbar row (no wrap at 1024+); auto-scroll landed on now (today in week) |
| Sparse week (no today) | Week | Auto-scroll landed on the first event; empty days read as clear |
| 3-member day **with rail** | Day | Three lanes + ~300px rail; today ringed, viewed day highlighted, member-colored dots; all-day band aligned to lanes |
| 6-member day **without rail** | Day | Six lanes shrink to fit the width (no horizontal scroll), **no** rail, toggle hidden; confirm lanes stay readable — if too thin at 1024, tune here (e.g. add a body min-width + shared horizontal scroll) |
| Long event titles | Week + Day | Titles truncate gracefully; no overflow into neighbors; text stays ≥12px |

Tune `DENSE_HOUR_ROW_HEIGHT` (48–56), `MIN_LANE_WIDTH`, and `RAIL_WIDTH` if the review finds cramped rows or a rail that appears/disappears at the wrong width. Re-run `npm test -- --run && npm run lint` after any change.

- [ ] **Step 4: Rail toggle + persistence behavior**

At a width where the rail fits (3-member @ 1440), click the toolbar toggle: rail hides, icon flips, and the preference survives a reload (persisted). Widen/narrow across the threshold: rail appears/disappears without reflowing the lane design (spec §4.2 — it only releases width back to lanes).

- [ ] **Step 5: Mobile + tablet regression**

At 375px (mobile) confirm Calendar is pixel-identical to `main` (all views). At ~900px (tablet, 768–1023) confirm Week keeps 80px rows and Day still renders the single-canvas `DailyCalendar` (no lanes, no rail). Capture one mobile Week screenshot for the PR.

- [ ] **Step 6: Tear down**

```bash
docker compose -f docker-compose.e2e.yml down
```

- [ ] **Step 7: Commit any tuning fixes**

```bash
git add -A src/
git commit -m "fix(calendar): screenshot-gate tuning for large-screen week and day"
```

---

## Task 9: Ship

**Files:**
- Modify (root repo): `docs/product/backlog/large-screen-ux/large-screen-calendar.md` (frontmatter)
- Modify (root repo): `docs/product/roadmap.md` (index entry if present)

- [ ] **Step 1: Push and open the PR**

```bash
cd /Users/joe.bor/code/family-hub/frontend
git push -u origin feat/large-screen-calendar
gh pr create --title "feat: large-screen calendar (week zoom-out, day member lanes)" --body "$(cat <<'EOF'
Story: https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/large-screen-ux/large-screen-calendar.md
Spec: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/specs/2026-07-06-large-screen-calendar-design.md
Plan: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-07-06-large-screen-calendar.md

## Week (lg+)
- One-line day headers (no TODAY-pill tower); denser hour rows (~52px) so 12–14 hours are visible at 1440x900; 80px preserved at 768–1023px.
- Auto-scroll to now when today is in the visible week, else the week's first event.

## Day (lg+)
- One vertical lane per member (avatar + name + color header); all-day band aligned under each lane; within-lane overlaps stack as before. No "Everyone" lane (memberId is required + singular).
- Optional ~300px mini-month rail: auto-shows only when member/width math leaves room, hides via a persisted toolbar toggle, navigator-only (tap a day → jump Day view). Never on mobile.

## Unchanged
- Mobile Calendar (all views) and the 768–1023px tablet Day/Week rendering.
- Month/Schedule inherit foundations chrome only.
- Home→Calendar event-tap routing (app-store `calendarEventIntent` → open detail) intact.

## Verification
- Unit: shared geometry, day-lane layout, rail math, rail + lanes components, and module integration (lanes + toggle at forced-large matchMedia).
- Screenshot matrix at 1024/1440/large: busy week, sparse week, 3-member day with rail, 6-member day without rail, long titles.
EOF
)"
```

Attach the Task 8 screenshots to the PR.

- [ ] **Step 2: Update story + roadmap (root repo)**

In `docs/product/backlog/large-screen-ux/large-screen-calendar.md`, set `status: in-progress` (or `done` after merge per the team's convention), update `updated:` to today, and record the PR URL under `prs:` and the issue (if one exists) under `issues:`. If `docs/product/roadmap.md` carries a per-story status for this epic, update only its index line. Commit in the root repo:

```bash
cd /Users/joe.bor/code/family-hub
git add docs/product/backlog/large-screen-ux/large-screen-calendar.md docs/product/roadmap.md
git commit -m "docs(large-screen-ux): mark calendar story in progress"
```

---

## Self-Review (completed during planning)

- **Spec coverage:** Week one-line headers (T3) · denser rows 48–56 lg+ (T1+T3) · auto-scroll now/first (T1 helper + T2 hook + T3 wiring) · Day lanes per member with avatar/name/color (T7) · events in correct lane + within-lane overlap + all-day band aligned (T4+T7) · mini-month rail auto-show/persisted-toggle/tap-navigate (T5+T6+T7) · Month/Schedule inherit chrome only (untouched) · mobile unchanged (no mobile files touched; density insulated by explicit mobile `rowHeight`) · screenshot matrix (T8). All ACs map to a task.
- **No "Everyone" lane:** lanes iterate `members` and partition by required `memberId`; no shared-event path (spec §4.1).
- **Home path intact:** no change to the `calendarEventIntent`/`calendarFocusDate` effects (`calendar-module.tsx:140-296`); the intent lookup keeps using unfiltered `rawEvents`, and event taps in the new views still call `onEventClick` → `openDetailModal`.
- **Type consistency:** `getEventOffsets`/`pxFromOffsets`/`calculateEventColumns`/`railThresholdPx`/`selectMonthDayDots`/`useDayRailState`/`useIsLargeScreen`/`useAutoScrollToMinutes` names are used identically across defining and consuming tasks.
- **Store discipline:** `dayRailHidden` added to `partialize` + `setup.ts` reset + `test-utils` seed/reset (T5 Step 5–6), per CLAUDE.md.
