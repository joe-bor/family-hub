# Large-Screen Home Hub Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a first-class large-screen Home surface that uses an ambient, now-first landscape layout while preserving the existing mobile Home unchanged.

**Architecture:** Keep the current mobile Home as a separate component, then make `HomeDashboard` a responsive router: mobile renders the existing mobile dashboard, non-mobile renders a new large-screen Home. The large-screen Home reuses existing calendar derivation where appropriate, adds pure summary/agenda selectors for the large layout, routes deeper work through app-store intents into Calendar, Chores, Meals, and Lists, and adds an app-shell idle-return hook for large screens.

**Tech Stack:** React 19, TypeScript, Vite, TanStack Query, Zustand, Tailwind v4, Vitest, Playwright, Biome. No new dependencies.

**Spec:** `docs/superpowers/specs/2026-07-05-large-screen-home-design.md`

**Story:** `docs/product/backlog/large-screen-ux/large-screen-home.md`

> **Revised 2026-07-07:** rebased on the merged large-screen shell foundations
> (FE PR #282). Foundations already removed the fake weather chip entirely,
> slimmed the desktop header to one compact row, and added an
> `AppHeader (desktop)` describe (with a mutable `isMobileMock`) to
> `app-header.test.tsx`, including a no-`72°` regression test. Task 1's former
> weather-hiding steps were deleted, and `app-header.tsx` /
> `app-header.desktop.test.tsx` were removed from the file structure. The
> desktop `activeModule === null → calendar` redirect in `App.tsx` still
> exists and remains this plan's job to remove.

**Repo:** All implementation work is in `frontend/` (separate git repo). This plan lives in the root docs repo for cross-repo planning.

---

## Pre-flight

- [ ] Read `docs/superpowers/specs/2026-07-05-large-screen-home-design.md` end-to-end. The non-negotiables are: A1 now-first layout, mobile unchanged, no inline module workspaces on large Home, Chores/Meals/Lists-only state strip, no fake weather on Home, idle return to Home, and screenshot critique before completion.
- [ ] Read `frontend/AGENTS.md`. Follow its date/time rules: use `src/lib/time-utils.ts`, never raw date-string parsing or `toISOString()` for local dates.
- [ ] Confirm `frontend/` is clean and up to date: `cd frontend && git status --short && git fetch origin && git status -sb`.
- [ ] Create the FE branch: `cd frontend && git checkout -b feat/large-screen-home`.
- [ ] Do not add backend work. Stop only if the existing Chores, Meals, Lists, or Calendar query contracts are proven insufficient.
- [ ] Do not redesign Calendar, Meals, Chores, Lists, Recipes, Photos, or the whole desktop shell. The only shell changes are: large screens can stay on Home, desktop navigation can reach Home, and idle can return to Home. (Fake weather is already gone globally via the foundations story, PR #282.)

## Verified Codebase Facts

- `frontend/src/App.tsx` currently redirects non-mobile `activeModule === null` to Calendar. This must be removed.
- `frontend/src/stores/app-store.ts` already uses `activeModule: null` as Home and has list/calendar date intents.
- `frontend/src/components/home/home-dashboard.tsx` currently owns mobile Home and inline Calendar add/detail/edit modals. Large-screen Home must not own those module workflows.
- Existing Home derivation pieces live under `frontend/src/components/home/hooks/` and `frontend/src/components/home/lib/`: `use-dashboard-events.ts`, `use-hero-state.ts`, `hero-state.ts`, `event-time.ts`, `relative-time.ts`, and current mobile components.
- `frontend/src/components/shared/navigation-tabs.tsx` has no Home entry and includes desktop-only module navigation.
- `frontend/src/components/shared/app-header.tsx` no longer renders fake weather anywhere — the foundations story (PR #282) removed it and slimmed the desktop header to one compact row. `app-header.test.tsx` already has an `AppHeader (desktop)` describe with a no-`72°` regression test and a mutable `isMobileMock`. This plan does not touch the header.
- `frontend/src/components/meals-view.tsx` can open a specific slot by setting `visibleWeekStartDate` and then `selectedSlot` or `editingSlotId`; add a small app-store intent rather than changing Meals routing globally.
- `frontend/src/components/calendar/calendar-module.tsx` can already consume a date focus intent. Add a separate event focus intent so large Home event taps open Calendar with the tapped event obvious.
- MSW fixture helpers already exist for Chores, Meals, and Lists in `frontend/src/test/mocks/server.ts`.

## File Structure

Create:

```text
frontend/src/components/home/mobile-home-dashboard.tsx
frontend/src/components/home/large-home-dashboard.tsx
frontend/src/components/home/components/large-now-hero.tsx
frontend/src/components/home/components/large-now-hero.test.tsx
frontend/src/components/home/components/large-state-strip.tsx
frontend/src/components/home/components/large-state-strip.test.tsx
frontend/src/components/home/components/large-today-rail.tsx
frontend/src/components/home/components/large-today-rail.test.tsx
frontend/src/components/home/hooks/use-large-home-summaries.ts
frontend/src/components/home/hooks/use-large-home-summaries.test.tsx
frontend/src/components/home/lib/large-home-selectors.ts
frontend/src/components/home/lib/large-home-selectors.test.ts
frontend/src/hooks/use-large-screen-home-idle-return.ts
frontend/src/hooks/use-large-screen-home-idle-return.test.tsx
frontend/src/App.large-home.test.tsx
frontend/e2e/large-screen-home.spec.ts
```

Modify:

```text
frontend/src/App.tsx
frontend/src/components/home/home-dashboard.tsx
frontend/src/components/home/home-dashboard.test.tsx
frontend/src/components/home/index.ts
frontend/src/components/calendar/calendar-module.tsx
frontend/src/components/calendar/calendar-module.test.tsx
frontend/src/components/meals-view.tsx
frontend/src/components/meals-view.test.tsx
frontend/src/components/shared/navigation-tabs.tsx
frontend/src/components/shared/navigation-tabs.test.tsx
frontend/src/stores/app-store.ts
frontend/src/stores/app-store.test.ts
frontend/src/test/setup.ts
frontend/e2e/helpers/api-helpers.ts
```

Each task below ends with a commit. Use conventional commits: `feat(home):`, `test(home):`, `refactor(home):`, or `docs(home):`.

---

## Task 1: Let Large Screens Reach Home

**Files:**
- Modify: `frontend/src/App.tsx`
- Modify: `frontend/src/components/shared/navigation-tabs.tsx`
- Modify: `frontend/src/components/shared/navigation-tabs.test.tsx`
- Create: `frontend/src/App.large-home.test.tsx`

- [ ] **Step 1: Write desktop navigation tests**

Append these tests to `src/components/shared/navigation-tabs.test.tsx`:

```tsx
it("renders Home as a first-class desktop destination", () => {
  useAppStore.setState({ activeModule: "calendar", isSidebarOpen: false });
  render(<NavigationTabs />);

  expect(screen.getByRole("button", { name: /^home$/i })).toBeInTheDocument();
});

it("marks Home active and switches to activeModule null", async () => {
  useAppStore.setState({ activeModule: "calendar", isSidebarOpen: false });
  const { user } = renderWithUser(<NavigationTabs />);

  await user.click(screen.getByRole("button", { name: /^home$/i }));

  expect(useAppStore.getState().activeModule).toBeNull();
  expect(screen.getByRole("button", { name: /^home$/i })).toHaveAttribute(
    "aria-current",
    "page",
  );
});
```

Add the missing import:

```ts
import { render, renderWithUser, screen } from "@/test/test-utils";
```

- [ ] **Step 2: Run the navigation tests and verify failure**

Run:

```bash
cd frontend
npm test -- --run src/components/shared/navigation-tabs.test.tsx
```

Expected: FAIL because the desktop rail has no Home button and no active state.

- [ ] **Step 3: Implement the desktop Home tab**

Update `src/components/shared/navigation-tabs.tsx`:

```tsx
import {
  BookOpenText,
  Calendar,
  CheckSquare,
  Home,
  ImageIcon,
  ListTodo,
  UtensilsCrossed,
} from "lucide-react";
import { cn } from "@/lib/utils";
import { type ModuleType, useAppStore } from "@/stores";

// Keep TabType export for the shared-components barrel.
export type TabType = ModuleType;

type DesktopNavItem = {
  id: ModuleType | null;
  label: string;
  icon: typeof Home;
};

const tabs: DesktopNavItem[] = [
  { id: null, label: "Home", icon: Home },
  { id: "calendar", label: "Calendar", icon: Calendar },
  { id: "lists", label: "Lists", icon: ListTodo },
  { id: "chores", label: "Chores", icon: CheckSquare },
  { id: "meals", label: "Meals", icon: UtensilsCrossed },
  { id: "recipes", label: "Recipes", icon: BookOpenText },
  { id: "photos", label: "Photos", icon: ImageIcon },
];

export function NavigationTabs() {
  const activeModule = useAppStore((state) => state.activeModule);
  const setActiveModule = useAppStore((state) => state.setActiveModule);

  return (
    <nav
      aria-label="Module navigation"
      className="flex w-20 shrink-0 flex-col items-center gap-2 border-r border-border bg-card py-6"
    >
      {tabs.map((tab) => {
        const Icon = tab.icon;
        const isActive = activeModule === tab.id;

        return (
          <button
            key={tab.label}
            type="button"
            onClick={() => setActiveModule(tab.id)}
            aria-current={isActive ? "page" : undefined}
            className={cn(
              "flex w-16 flex-col items-center gap-1 rounded-xl px-2 py-3 transition-colors",
              isActive
                ? "bg-primary text-primary-foreground"
                : "text-muted-foreground hover:bg-muted hover:text-foreground",
            )}
          >
            <Icon className="h-5 w-5" />
            <span className="text-[10px] font-medium">{tab.label}</span>
          </button>
        );
      })}
    </nav>
  );
}
```

- [ ] **Step 4: Write App shell tests for no desktop redirect**

Create `src/App.large-home.test.tsx`:

```tsx
import { beforeEach, describe, expect, it, vi } from "vitest";
import { useAppStore } from "@/stores";
import { render, screen, seedAuthStore, seedFamilyStore } from "@/test/test-utils";
import FamilyHub from "./App";

const viewport = vi.hoisted(() => ({ isMobile: false }));

vi.mock("@/hooks", async (importOriginal) => {
  const actual = await importOriginal<typeof import("@/hooks")>();
  return {
    ...actual,
    useIsMobile: () => viewport.isMobile,
    useAndroidBackButton: () => undefined,
    useGoogleAuthReturn: () => undefined,
  };
});

vi.mock("@/api", async (importOriginal) => {
  const actual = await importOriginal<typeof import("@/api")>();
  return {
    ...actual,
    useSetupComplete: () => true,
  };
});

vi.mock("@/components/home", () => ({
  HomeDashboard: () => <div data-testid="home-dashboard">Large Home</div>,
}));

vi.mock("@/components/calendar", () => ({
  CalendarModule: () => <div data-testid="calendar-module">Calendar</div>,
}));

vi.mock("@/components/shared", async (importOriginal) => {
  const actual = await importOriginal<typeof import("@/components/shared")>();
  return {
    ...actual,
    OfflineBanner: () => null,
    SidebarMenu: () => null,
  };
});

describe("FamilyHub large-screen Home shell", () => {
  beforeEach(() => {
    viewport.isMobile = false;
    seedAuthStore({ isAuthenticated: true });
    seedFamilyStore({
      name: "Test Family",
      members: [{ id: "m1", name: "Alice", color: "coral" }],
    });
    useAppStore.setState({ activeModule: null, isSidebarOpen: false });
  });

  it("keeps activeModule null on non-mobile launch and renders Home", async () => {
    render(<FamilyHub />);

    expect(await screen.findByTestId("home-dashboard")).toBeInTheDocument();
    expect(screen.queryByTestId("calendar-module")).not.toBeInTheDocument();
    expect(useAppStore.getState().activeModule).toBeNull();
  });

  it("still renders Calendar when Calendar is manually active", async () => {
    useAppStore.setState({ activeModule: "calendar" });

    render(<FamilyHub />);

    expect(await screen.findByTestId("calendar-module")).toBeInTheDocument();
    expect(screen.queryByTestId("home-dashboard")).not.toBeInTheDocument();
  });
});
```

- [ ] **Step 5: Run the App shell tests and verify failure**

Run:

```bash
npm test -- --run src/App.large-home.test.tsx
```

Expected: FAIL because `App.tsx` redirects `activeModule === null` to Calendar on desktop.

- [ ] **Step 6: Remove the desktop redirect**

Delete this effect from `src/App.tsx`:

```tsx
// On desktop, redirect null (home) to calendar since home is mobile-only
useEffect(() => {
  if (!isMobile && activeModule === null) {
    setActiveModule("calendar");
  }
}, [isMobile, activeModule, setActiveModule]);
```

Then remove the now-unused `useEffect` import and `setActiveModule` binding if they are not used elsewhere in `App.tsx`:

```tsx
import { lazy, Suspense, useState } from "react";
```

(Former Steps 7-9 — hiding fake weather on Home — were removed in the
2026-07-07 revision: the foundations story (PR #282) already deleted the
weather chip globally and `app-header.test.tsx` carries the no-`72°`
regression test. The header needs no changes in this plan.)

- [ ] **Step 7: Run Task 1 tests**

Run:

```bash
npm test -- --run src/components/shared/navigation-tabs.test.tsx src/App.large-home.test.tsx
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add src/App.tsx src/App.large-home.test.tsx src/components/shared/navigation-tabs.tsx src/components/shared/navigation-tabs.test.tsx
git commit -m "feat(home): make home reachable on large screens"
```

---

## Task 2: Add Cross-Module Routing Intents

**Files:**
- Modify: `frontend/src/stores/app-store.ts`
- Modify: `frontend/src/stores/app-store.test.ts`
- Modify: `frontend/src/test/setup.ts`
- Modify: `frontend/src/components/calendar/calendar-module.tsx`
- Modify: `frontend/src/components/calendar/calendar-module.test.tsx`
- Modify: `frontend/src/components/meals-view.tsx`
- Modify: `frontend/src/components/meals-view.test.tsx`

- [ ] **Step 1: Write app-store intent tests**

Add to `src/stores/app-store.test.ts` inside `describe("navigation intents", ...)`:

```ts
it("openCalendarEvent stores event focus data and switches to calendar", () => {
  useAppStore.getState().openCalendarEvent({
    date: "2026-07-15",
    eventKey: "event-123",
  });

  expect(useAppStore.getState().activeModule).toBe("calendar");
  expect(useAppStore.getState().calendarEventIntent).toEqual({
    date: "2026-07-15",
    eventKey: "event-123",
  });

  useAppStore.getState().clearCalendarEventIntent();
  expect(useAppStore.getState().calendarEventIntent).toBeNull();
});

it("focusMealSlot stores a meal slot intent and switches to meals", () => {
  useAppStore.getState().focusMealSlot({
    weekStartDate: "2026-07-12",
    dayIndex: 2,
    mealType: "dinner",
  });

  expect(useAppStore.getState().activeModule).toBe("meals");
  expect(useAppStore.getState().consumeMealSlotIntent()).toEqual({
    weekStartDate: "2026-07-12",
    dayIndex: 2,
    mealType: "dinner",
  });
  expect(useAppStore.getState().consumeMealSlotIntent()).toBeNull();
});
```

- [ ] **Step 2: Run app-store tests and verify failure**

Run:

```bash
cd frontend
npm test -- --run src/stores/app-store.test.ts
```

Expected: FAIL because the new actions and state fields do not exist.

- [ ] **Step 3: Implement app-store intents**

In `src/stores/app-store.ts`, add:

```ts
export type CalendarEventIntent = {
  date: string;
  eventKey: string;
};

export type MealSlotIntent = {
  weekStartDate: string;
  dayIndex: number;
  mealType: MealType;
};
```

Extend `AppState`:

```ts
calendarEventIntent: CalendarEventIntent | null;
mealSlotIntent: MealSlotIntent | null;
openCalendarEvent: (intent: CalendarEventIntent) => void;
clearCalendarEventIntent: () => void;
focusMealSlot: (intent: MealSlotIntent) => void;
consumeMealSlotIntent: () => MealSlotIntent | null;
```

Add initial values:

```ts
calendarEventIntent: null,
mealSlotIntent: null,
```

Add actions:

```ts
openCalendarEvent: (intent) =>
  set({
    calendarEventIntent: intent,
    calendarFocusDate: null,
    activeModule: "calendar",
  }),
clearCalendarEventIntent: () => set({ calendarEventIntent: null }),
focusMealSlot: (intent) =>
  set({ mealSlotIntent: intent, activeModule: "meals" }),
consumeMealSlotIntent: () => {
  const v = get().mealSlotIntent;
  set({ mealSlotIntent: null });
  return v;
},
```

Update `src/test/setup.ts` app-store reset:

```ts
useAppStore.setState({
  activeModule: "calendar",
  isSidebarOpen: false,
  mealPlacementDraft: null,
  recipeCreationDraft: null,
  listDetailIntent: null,
  calendarFocusDate: null,
  calendarEventIntent: null,
  mealSlotIntent: null,
});
```

- [ ] **Step 4: Write Calendar event-intent integration test**

Add this test to `src/components/calendar/calendar-module.test.tsx`:

```tsx
import { useAppStore } from "@/stores";
```

```tsx
it("opens the requested event detail when a calendar event intent is present", async () => {
  const target = createTestEventResponse({
    id: "large-home-target",
    title: "Swim lesson",
    date: "2026-04-25",
    startTime: "9:00 AM",
    endTime: "10:00 AM",
    memberId: testMembers[0].id,
  });
  seedMockEvents([target]);
  seedCalendarStore({
    currentDate: new Date(2026, 3, 25),
    calendarView: "daily",
    filter: {
      selectedMembers: [testMembers[1].id],
      showAllDayEvents: true,
    },
  });
  useAppStore.getState().openCalendarEvent({
    date: "2026-04-25",
    eventKey: "large-home-target",
  });

  render(<CalendarModule />);

  expect(await screen.findByRole("dialog")).toBeInTheDocument();
  expect(screen.getByText("Swim lesson")).toBeInTheDocument();
  expect(useAppStore.getState().calendarEventIntent).toBeNull();
});
```

- [ ] **Step 5: Run the Calendar test and verify failure**

Run:

```bash
npm test -- --run src/components/calendar/calendar-module.test.tsx -t "opens the requested event detail"
```

Expected: FAIL because `CalendarModule` does not consume `calendarEventIntent`.

- [ ] **Step 6: Implement Calendar event-intent consumption**

In `src/components/calendar/calendar-module.tsx`, import `getEventKey`:

```ts
import {
  format24hTo12h,
  formatLocalDate,
  getEventKey,
  parseLocalDate,
} from "@/lib/time-utils";
```

Read the intent near the existing `consumeCalendarFocusDate` code:

```tsx
const calendarEventIntent = useAppStore((s) => s.calendarEventIntent);
const clearCalendarEventIntent = useAppStore(
  (s) => s.clearCalendarEventIntent,
);
```

Add the date-setting effect after the existing `consumeCalendarFocusDate` effect:

```tsx
useEffect(() => {
  if (!calendarEventIntent) return;
  setDate(parseLocalDate(calendarEventIntent.date));
}, [calendarEventIntent, setDate]);
```

Update the event derivation so the intent can search unfiltered API data while the rendered calendar still uses the user's filters:

```tsx
const rawEvents = eventsResponse?.data ?? [];

const events = useMemo(() => {
  return rawEvents.filter((event) => {
    const memberMatches = filter.selectedMembers.includes(event.memberId);
    const allDayMatches = filter.showAllDayEvents || !event.isAllDay;
    return memberMatches && allDayMatches;
  });
}, [filter, rawEvents]);
```

Add the event-opening effect after `deleteEvent` and `deleteInstance` are declared. It must search `rawEvents`, not the filtered `events`, so a persisted member/all-day filter cannot hide the Home-tapped event:

```tsx
const resetDeleteEvent = deleteEvent.reset;
const resetDeleteInstance = deleteInstance.reset;

useEffect(() => {
  if (!calendarEventIntent || isLoading) return;

  const match = rawEvents.find(
    (event) => getEventKey(event) === calendarEventIntent.eventKey,
  );
  if (!match) return;

  resetDeleteEvent();
  resetDeleteInstance();
  openDetailModal(match);
  clearCalendarEventIntent();
}, [
  calendarEventIntent,
  clearCalendarEventIntent,
  isLoading,
  openDetailModal,
  rawEvents,
  resetDeleteEvent,
  resetDeleteInstance,
]);
```

- [ ] **Step 7: Write Meals slot-intent tests**

Add to `src/components/meals-view.test.tsx`:

```tsx
it("opens an empty dinner slot from a large Home meal intent", async () => {
  seedMockMealsBoard(createEmptyMealsBoard());
  useAppStore.getState().focusMealSlot({
    weekStartDate: testWeekStartDate,
    dayIndex: 0,
    mealType: "dinner",
  });

  renderWithUser(<MealsView />);

  expect(
    await screen.findByRole("dialog", { name: /plan dinner/i }),
  ).toBeInTheDocument();
  expect(useAppStore.getState().mealSlotIntent).toBeNull();
});

it("opens an occupied dinner slot from a large Home meal intent", async () => {
  seedMockMealsBoard(withOccupiedDinnerSlot(createEmptyMealsBoard(), 0, "Tacos"));
  useAppStore.getState().focusMealSlot({
    weekStartDate: testWeekStartDate,
    dayIndex: 0,
    mealType: "dinner",
  });

  renderWithUser(<MealsView />);

  expect(
    await screen.findByRole("dialog", { name: /dinner plan/i }),
  ).toBeInTheDocument();
  expect(screen.getByText("Tacos")).toBeInTheDocument();
  expect(useAppStore.getState().mealSlotIntent).toBeNull();
});
```

- [ ] **Step 8: Run Meals tests and verify failure**

Run:

```bash
npm test -- --run src/components/meals-view.test.tsx -t "large Home meal intent"
```

Expected: FAIL because `MealsView` does not consume `mealSlotIntent`.

- [ ] **Step 9: Implement Meals slot-intent consumption**

In `src/components/meals-view.tsx`, read the pending intent:

```tsx
const pendingMealSlotIntent = useAppStore((state) => state.mealSlotIntent);
const consumeMealSlotIntent = useAppStore(
  (state) => state.consumeMealSlotIntent,
);
```

Add these effects after the existing meal-placement-draft effects:

```tsx
useEffect(() => {
  if (!pendingMealSlotIntent) return;
  setVisibleWeekStartDate(pendingMealSlotIntent.weekStartDate);
}, [pendingMealSlotIntent]);

useEffect(() => {
  if (!pendingMealSlotIntent || !persistedBoard) return;

  const day = persistedBoard.days[pendingMealSlotIntent.dayIndex];
  const slot = day?.slots.find(
    (candidate) => candidate.mealType === pendingMealSlotIntent.mealType,
  );
  if (!slot) return;

  if (slot.primary || slot.extras.length > 0) {
    setEditingSlotId({
      weekStartDate: slot.weekStartDate,
      dayIndex: slot.dayIndex,
      mealType: slot.mealType,
    });
  } else {
    setSelectedSlot(slot);
  }

  consumeMealSlotIntent();
}, [consumeMealSlotIntent, pendingMealSlotIntent, persistedBoard]);
```

- [ ] **Step 10: Run Task 2 tests**

Run:

```bash
npm test -- --run src/stores/app-store.test.ts src/components/calendar/calendar-module.test.tsx src/components/meals-view.test.tsx
```

Expected: PASS.

- [ ] **Step 11: Commit**

```bash
git add src/stores/app-store.ts src/stores/app-store.test.ts src/test/setup.ts src/components/calendar/calendar-module.tsx src/components/calendar/calendar-module.test.tsx src/components/meals-view.tsx src/components/meals-view.test.tsx
git commit -m "feat(home): add large home navigation intents"
```

---

## Task 3: Add Large-Home Pure Selectors

**Files:**
- Create: `frontend/src/components/home/lib/large-home-selectors.ts`
- Test: `frontend/src/components/home/lib/large-home-selectors.test.ts`

- [ ] **Step 1: Write failing selector tests**

Create `src/components/home/lib/large-home-selectors.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type {
  CalendarEvent,
  ChoresBoard,
  ListSummary,
  MealBoard,
} from "@/lib/types";
import {
  deriveChoresSummary,
  deriveListsSummary,
  deriveMealsSummary,
  getTodayDinnerTarget,
  selectRestOfDayItems,
  selectTomorrowPeek,
} from "./large-home-selectors";

const now = new Date(2026, 6, 5, 9, 0, 0);

const event = (overrides: Partial<CalendarEvent>): CalendarEvent => ({
  id: "e1",
  title: "Swim lesson",
  date: new Date(2026, 6, 5),
  startTime: "9:00 AM",
  endTime: "10:00 AM",
  memberId: "m1",
  isAllDay: false,
  source: "NATIVE",
  ...overrides,
});

const choresBoard = (remaining: number, total = Math.max(remaining, 1)): ChoresBoard => ({
  timezone: "America/Los_Angeles",
  today: {
    scope: "TODAY",
    periodStartDate: "2026-07-05",
    periodEndDate: "2026-07-05",
    summary: { total, completed: total - remaining, remaining },
    assignees: [],
  },
  thisWeek: {
    scope: "THIS_WEEK",
    periodStartDate: "2026-07-05",
    periodEndDate: "2026-07-11",
    summary: { total: 0, completed: 0, remaining: 0 },
    assignees: [],
  },
  thisMonth: {
    scope: "THIS_MONTH",
    periodStartDate: "2026-07-01",
    periodEndDate: "2026-07-31",
    summary: { total: 0, completed: 0, remaining: 0 },
    assignees: [],
  },
});

const mealsBoard = (dinnerTitle: string | null): MealBoard => ({
  weekStartDate: "2026-07-05",
  days: Array.from({ length: 7 }, (_, dayIndex) => ({
    date: `2026-07-${String(5 + dayIndex).padStart(2, "0")}`,
    dayIndex,
    slots: (["breakfast", "lunch", "dinner"] as const).map((mealType) => ({
      id: mealType === "dinner" && dinnerTitle ? "slot-dinner" : null,
      weekStartDate: "2026-07-05",
      dayIndex,
      mealType,
      primary:
        mealType === "dinner" && dinnerTitle
          ? {
              id: "entry-dinner",
              role: "primary",
              sourceType: "quick",
              recipeId: null,
              title: dinnerTitle,
              imageUrl: null,
              note: null,
            }
          : null,
      extras: [],
      note: null,
    })),
  })),
});

describe("large home selectors", () => {
  it("selects 3-5 rest-of-day items after excluding the hero event", () => {
    const hero = event({ id: "hero", startTime: "9:00 AM", endTime: "10:00 AM" });
    const later = [
      event({ id: "a", title: "Dentist", startTime: "11:00 AM", endTime: "12:00 PM" }),
      event({ id: "b", title: "Practice", startTime: "1:00 PM", endTime: "2:00 PM" }),
      event({ id: "c", title: "Pickup", startTime: "3:00 PM", endTime: "4:00 PM" }),
      event({ id: "d", title: "Dinner", startTime: "6:00 PM", endTime: "7:00 PM" }),
      event({ id: "e", title: "Bedtime", startTime: "8:00 PM", endTime: "8:30 PM" }),
      event({ id: "f", title: "Overflow", startTime: "9:00 PM", endTime: "9:30 PM" }),
    ];

    expect(selectRestOfDayItems([hero, ...later], hero, now)).toHaveLength(5);
    expect(selectRestOfDayItems([hero, ...later], hero, now).map((e) => e.title)).toEqual([
      "Dentist",
      "Practice",
      "Pickup",
      "Dinner",
      "Bedtime",
    ]);
  });

  it("selects a small tomorrow/near-future peek", () => {
    const tomorrow = new Date(2026, 6, 6);
    const dayAfter = new Date(2026, 6, 7);
    const items = selectTomorrowPeek(
      [
        event({ id: "t1", title: "Camp", date: tomorrow, startTime: "8:00 AM" }),
        event({ id: "t2", title: "Lunch", date: tomorrow, startTime: "12:00 PM" }),
        event({ id: "t3", title: "Dentist", date: tomorrow, startTime: "4:00 PM" }),
        event({ id: "d1", title: "Later", date: dayAfter, startTime: "9:00 AM" }),
      ],
      now,
    );

    expect(items.map((e) => e.title)).toEqual(["Camp", "Lunch", "Dentist"]);
  });

  it("derives chores remaining, done, empty, and unavailable states", () => {
    expect(deriveChoresSummary({ board: choresBoard(3), isLoading: false, isError: false })).toMatchObject({
      kind: "remaining",
      label: "3 chores left",
    });
    expect(deriveChoresSummary({ board: choresBoard(0, 4), isLoading: false, isError: false })).toMatchObject({
      kind: "done",
      label: "Chores done",
    });
    expect(deriveChoresSummary({ board: choresBoard(0, 0), isLoading: false, isError: false })).toMatchObject({
      kind: "empty",
      label: "No chores configured",
    });
    expect(deriveChoresSummary({ board: null, isLoading: false, isError: false })).toMatchObject({
      kind: "unavailable",
      label: "Chores unavailable",
    });
  });

  it("derives dinner planned and dinner missing states with a focus target", () => {
    expect(
      deriveMealsSummary({
        board: mealsBoard("Tacos"),
        today: new Date(2026, 6, 5),
        isLoading: false,
        isError: false,
      }),
    ).toMatchObject({
      kind: "planned",
      label: "Tacos tonight",
      target: { weekStartDate: "2026-07-05", dayIndex: 0, mealType: "dinner" },
    });

    expect(
      deriveMealsSummary({
        board: mealsBoard(null),
        today: new Date(2026, 6, 5),
        isLoading: false,
        isError: false,
      }),
    ).toMatchObject({
      kind: "missing",
      label: "Dinner not planned",
      target: { weekStartDate: "2026-07-05", dayIndex: 0, mealType: "dinner" },
    });
  });

  it("derives the best lists signal from active grocery items", () => {
    const lists: ListSummary[] = [
      { id: "g1", name: "Groceries", kind: "grocery", totalItems: 7, completedItems: 2 },
      { id: "todo", name: "Errands", kind: "to-do", totalItems: 5, completedItems: 1 },
    ];

    expect(deriveListsSummary({ lists, isLoading: false, isError: false })).toMatchObject({
      kind: "active",
      label: "5 grocery items",
      listId: "g1",
    });
  });

  it("finds today's dinner slot target", () => {
    expect(getTodayDinnerTarget(mealsBoard(null), new Date(2026, 6, 5))).toEqual({
      weekStartDate: "2026-07-05",
      dayIndex: 0,
      mealType: "dinner",
    });
  });
});
```

- [ ] **Step 2: Run selector tests and verify failure**

Run:

```bash
cd frontend
npm test -- --run src/components/home/lib/large-home-selectors.test.ts
```

Expected: FAIL because `large-home-selectors.ts` does not exist.

- [ ] **Step 3: Implement selectors**

Create `src/components/home/lib/large-home-selectors.ts`:

```ts
import { addDays, isSameDay, startOfDay } from "date-fns";
import {
  formatLocalDate,
  getEventKey,
  getWeekStartSunday,
} from "@/lib/time-utils";
import type {
  CalendarEvent,
  ChoresBoard,
  ListSummary,
  MealBoard,
  MealType,
} from "@/lib/types";
import { getEventDateTime } from "./event-time";

export type SummaryStatus =
  | "loading"
  | "unavailable"
  | "empty"
  | "done"
  | "remaining"
  | "planned"
  | "missing"
  | "active"
  | "quiet";

export type HomeSummaryTarget =
  | { module: "chores" }
  | { module: "lists"; listId?: string }
  | { module: "meals"; weekStartDate: string; dayIndex: number; mealType: MealType };

export interface HomeStateSummary {
  module: "chores" | "meals" | "lists";
  kind: SummaryStatus;
  label: string;
  target: HomeSummaryTarget;
}

export function selectRestOfDayItems(
  todayEvents: CalendarEvent[],
  heroEvent: CalendarEvent | null,
  now: Date,
  limit = 5,
): CalendarEvent[] {
  const heroKey = heroEvent ? getEventKey(heroEvent) : null;

  return todayEvents
    .filter((event) => getEventKey(event) !== heroKey)
    .filter((event) => {
      if (event.isAllDay) return true;
      return getEventDateTime(event, "end") > now;
    })
    .sort((left, right) => {
      if (left.isAllDay && !right.isAllDay) return -1;
      if (!left.isAllDay && right.isAllDay) return 1;
      return (
        getEventDateTime(left, "start").getTime() -
        getEventDateTime(right, "start").getTime()
      );
    })
    .slice(0, limit);
}

export function selectTomorrowPeek(
  comingUpEvents: CalendarEvent[],
  currentDate: Date,
  limit = 3,
): CalendarEvent[] {
  const tomorrow = startOfDay(addDays(currentDate, 1));

  const tomorrowItems = comingUpEvents
    .filter((event) => isSameDay(event.date, tomorrow))
    .sort(
      (left, right) =>
        getEventDateTime(left, "start").getTime() -
        getEventDateTime(right, "start").getTime(),
    )
    .slice(0, limit);

  if (tomorrowItems.length > 0) return tomorrowItems;

  return [...comingUpEvents]
    .sort(
      (left, right) =>
        getEventDateTime(left, "start").getTime() -
        getEventDateTime(right, "start").getTime(),
    )
    .slice(0, limit);
}

export function deriveChoresSummary({
  board,
  isLoading,
  isError,
}: {
  board: ChoresBoard | null | undefined;
  isLoading: boolean;
  isError: boolean;
}): HomeStateSummary {
  if (isLoading) {
    return { module: "chores", kind: "loading", label: "Loading chores", target: { module: "chores" } };
  }
  if (isError || !board) {
    return { module: "chores", kind: "unavailable", label: "Chores unavailable", target: { module: "chores" } };
  }

  const { total, remaining } = board.today.summary;
  if (total === 0) {
    return { module: "chores", kind: "empty", label: "No chores configured", target: { module: "chores" } };
  }
  if (remaining === 0) {
    return { module: "chores", kind: "done", label: "Chores done", target: { module: "chores" } };
  }

  return {
    module: "chores",
    kind: "remaining",
    label: `${remaining} chore${remaining === 1 ? "" : "s"} left`,
    target: { module: "chores" },
  };
}

export function getTodayDinnerTarget(board: MealBoard, today: Date) {
  const todayKey = formatLocalDate(today);
  const day = board.days.find((candidate) => candidate.date === todayKey);
  const slot = day?.slots.find((candidate) => candidate.mealType === "dinner");
  if (!day || !slot) return null;

  return {
    weekStartDate: board.weekStartDate,
    dayIndex: day.dayIndex,
    mealType: "dinner" as const,
  };
}

export function deriveMealsSummary({
  board,
  today,
  isLoading,
  isError,
}: {
  board: MealBoard | null | undefined;
  today: Date;
  isLoading: boolean;
  isError: boolean;
}): HomeStateSummary {
  const fallbackTarget = {
    module: "meals" as const,
    weekStartDate: formatLocalDate(getWeekStartSunday(today)),
    dayIndex: today.getDay(),
    mealType: "dinner" as const,
  };

  if (isLoading) {
    return { module: "meals", kind: "loading", label: "Loading meals", target: fallbackTarget };
  }
  if (isError || !board) {
    return { module: "meals", kind: "unavailable", label: "Meals unavailable", target: fallbackTarget };
  }

  const target = getTodayDinnerTarget(board, today) ?? fallbackTarget;
  const todayKey = formatLocalDate(today);
  const day = board.days.find((candidate) => candidate.date === todayKey);
  const dinner = day?.slots.find((slot) => slot.mealType === "dinner");
  const title = dinner?.primary?.title ?? dinner?.extras[0]?.title ?? null;

  if (title) {
    return {
      module: "meals",
      kind: "planned",
      label: `${title} tonight`,
      target: { module: "meals", ...target },
    };
  }

  return {
    module: "meals",
    kind: "missing",
    label: "Dinner not planned",
    target: { module: "meals", ...target },
  };
}

export function deriveListsSummary({
  lists,
  isLoading,
  isError,
}: {
  lists: ListSummary[] | null | undefined;
  isLoading: boolean;
  isError: boolean;
}): HomeStateSummary {
  if (isLoading) {
    return { module: "lists", kind: "loading", label: "Loading lists", target: { module: "lists" } };
  }
  if (isError || !lists) {
    return { module: "lists", kind: "unavailable", label: "Lists unavailable", target: { module: "lists" } };
  }

  const grocery = lists
    .filter((list) => list.kind === "grocery")
    .map((list) => ({
      ...list,
      activeItems: Math.max(0, list.totalItems - list.completedItems),
    }))
    .sort((left, right) => right.activeItems - left.activeItems)[0];

  if (grocery && grocery.activeItems > 0) {
    return {
      module: "lists",
      kind: "active",
      label: `${grocery.activeItems} grocery item${grocery.activeItems === 1 ? "" : "s"}`,
      target: { module: "lists", listId: grocery.id },
    };
  }

  return { module: "lists", kind: "quiet", label: "Lists quiet", target: { module: "lists" } };
}
```

- [ ] **Step 4: Run selector tests**

Run:

```bash
npm test -- --run src/components/home/lib/large-home-selectors.test.ts
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/components/home/lib/large-home-selectors.ts src/components/home/lib/large-home-selectors.test.ts
git commit -m "feat(home): add large home selectors"
```

---

## Task 4: Add Large-Screen Home UI

**Files:**
- Create: `frontend/src/components/home/mobile-home-dashboard.tsx`
- Modify: `frontend/src/components/home/home-dashboard.tsx`
- Create: `frontend/src/components/home/large-home-dashboard.tsx`
- Create: `frontend/src/components/home/components/large-now-hero.tsx`
- Test: `frontend/src/components/home/components/large-now-hero.test.tsx`
- Create: `frontend/src/components/home/components/large-state-strip.tsx`
- Test: `frontend/src/components/home/components/large-state-strip.test.tsx`
- Create: `frontend/src/components/home/components/large-today-rail.tsx`
- Test: `frontend/src/components/home/components/large-today-rail.test.tsx`
- Create: `frontend/src/components/home/hooks/use-large-home-summaries.ts`
- Test: `frontend/src/components/home/hooks/use-large-home-summaries.test.tsx`
- Modify: `frontend/src/components/home/home-dashboard.test.tsx`
- Modify: `frontend/src/components/home/index.ts`

- [ ] **Step 1: Preserve the current mobile Home component**

Move the existing implementation:

```bash
cd frontend
git mv src/components/home/home-dashboard.tsx src/components/home/mobile-home-dashboard.tsx
```

In `src/components/home/mobile-home-dashboard.tsx`, rename the export:

```tsx
export function MobileHomeDashboard({ nowOverride }: { nowOverride?: Date } = {}) {
```

Do not change the body in this step.

- [ ] **Step 2: Create the responsive HomeDashboard router**

Create a new `src/components/home/home-dashboard.tsx`:

```tsx
import { useIsMobile } from "@/hooks";
import { LargeHomeDashboard } from "./large-home-dashboard";
import { MobileHomeDashboard } from "./mobile-home-dashboard";

export function HomeDashboard({ nowOverride }: { nowOverride?: Date } = {}) {
  const isMobile = useIsMobile();
  return isMobile ? (
    <MobileHomeDashboard nowOverride={nowOverride} />
  ) : (
    <LargeHomeDashboard nowOverride={nowOverride} />
  );
}
```

- [ ] **Step 3: Write large component tests**

Create `src/components/home/components/large-now-hero.test.tsx`:

```tsx
import { describe, expect, it, vi } from "vitest";
import type { CalendarEvent, FamilyMember } from "@/lib/types";
import { render, renderWithUser, screen } from "@/test/test-utils";
import { LargeNowHero } from "./large-now-hero";

const member: FamilyMember = { id: "m1", name: "Alice", color: "coral" };
const event: CalendarEvent = {
  id: "e1",
  title: "Swim lesson with an intentionally long title that wraps cleanly",
  date: new Date(2026, 6, 5),
  startTime: "9:00 AM",
  endTime: "10:00 AM",
  memberId: "m1",
  isAllDay: false,
  source: "NATIVE",
  location: "Community pool",
};

describe("LargeNowHero", () => {
  it("renders the now message as the dominant labelled region", () => {
    render(
      <LargeNowHero
        state={{ kind: "UP_NEXT", event }}
        member={member}
        now={new Date(2026, 6, 5, 8, 30)}
        onOpenEvent={vi.fn()}
      />,
    );

    expect(screen.getByRole("button", { name: /up next: swim lesson/i })).toBeInTheDocument();
    expect(screen.getByText(/Community pool/)).toBeInTheDocument();
  });

  it("routes event taps through the callback", async () => {
    const onOpenEvent = vi.fn();
    const { user } = renderWithUser(
      <LargeNowHero
        state={{ kind: "UP_NEXT", event }}
        member={member}
        now={new Date(2026, 6, 5, 8, 30)}
        onOpenEvent={onOpenEvent}
      />,
    );

    await user.click(screen.getByRole("button", { name: /up next: swim lesson/i }));
    expect(onOpenEvent).toHaveBeenCalledWith(event);
  });
});
```

Create `src/components/home/components/large-state-strip.test.tsx`:

```tsx
import { describe, expect, it, vi } from "vitest";
import type { HomeStateSummary } from "../lib/large-home-selectors";
import { render, renderWithUser, screen } from "@/test/test-utils";
import { LargeStateStrip } from "./large-state-strip";

const summaries: HomeStateSummary[] = [
  { module: "chores", kind: "remaining", label: "3 chores left", target: { module: "chores" } },
  {
    module: "meals",
    kind: "missing",
    label: "Dinner not planned",
    target: { module: "meals", weekStartDate: "2026-07-05", dayIndex: 0, mealType: "dinner" },
  },
  { module: "lists", kind: "active", label: "5 grocery items", target: { module: "lists", listId: "g1" } },
];

describe("LargeStateStrip", () => {
  it("renders the provided Chores, Meals, and Lists summaries", () => {
    render(<LargeStateStrip summaries={summaries} onSelect={vi.fn()} />);

    expect(screen.getByRole("button", { name: /open chores. 3 chores left/i })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /open meals. dinner not planned/i })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /open lists. 5 grocery items/i })).toBeInTheDocument();
    expect(screen.queryByText(/recipes/i)).not.toBeInTheDocument();
    expect(screen.queryByText(/photos/i)).not.toBeInTheDocument();
    expect(screen.queryByText(/weather/i)).not.toBeInTheDocument();
  });

  it("selects the tapped summary target", async () => {
    const onSelect = vi.fn();
    const { user } = renderWithUser(<LargeStateStrip summaries={summaries} onSelect={onSelect} />);

    await user.click(screen.getByRole("button", { name: /open meals/i }));
    expect(onSelect).toHaveBeenCalledWith(summaries[1].target);
  });
});
```

Create `src/components/home/components/large-today-rail.test.tsx`:

```tsx
import { describe, expect, it, vi } from "vitest";
import type { CalendarEvent, FamilyMember } from "@/lib/types";
import { render, renderWithUser, screen } from "@/test/test-utils";
import { LargeTodayRail } from "./large-today-rail";

const members: FamilyMember[] = [{ id: "m1", name: "Alice", color: "coral" }];
const event = (id: string, title: string, date = new Date(2026, 6, 5)): CalendarEvent => ({
  id,
  title,
  date,
  startTime: "11:00 AM",
  endTime: "12:00 PM",
  memberId: "m1",
  isAllDay: false,
  source: "NATIVE",
});

describe("LargeTodayRail", () => {
  it("renders rest-of-day and tomorrow peek without calendar controls", () => {
    render(
      <LargeTodayRail
        currentDate={new Date(2026, 6, 5)}
        todayItems={[event("a", "Dentist"), event("b", "Practice")]}
        tomorrowItems={[event("c", "Camp", new Date(2026, 6, 6))]}
        members={members}
        onSelect={vi.fn()}
      />,
    );

    expect(screen.getByText("Today")).toBeInTheDocument();
    expect(screen.getByText("Tomorrow")).toBeInTheDocument();
    expect(screen.getByText("Dentist")).toBeInTheDocument();
    expect(screen.queryByText(/week/i)).not.toBeInTheDocument();
  });

  it("routes tapped events through the callback", async () => {
    const onSelect = vi.fn();
    const dentist = event("a", "Dentist");
    const { user } = renderWithUser(
      <LargeTodayRail
        currentDate={new Date(2026, 6, 5)}
        todayItems={[dentist]}
        tomorrowItems={[]}
        members={members}
        onSelect={onSelect}
      />,
    );

    await user.click(screen.getByRole("button", { name: /dentist/i }));
    expect(onSelect).toHaveBeenCalledWith(dentist);
  });
});
```

- [ ] **Step 4: Run component tests and verify failure**

Run:

```bash
npm test -- --run src/components/home/components/large-now-hero.test.tsx src/components/home/components/large-state-strip.test.tsx src/components/home/components/large-today-rail.test.tsx
```

Expected: FAIL because the components do not exist.

- [ ] **Step 5: Implement `LargeNowHero`**

Create `src/components/home/components/large-now-hero.tsx`:

```tsx
import { Sparkles } from "lucide-react";
import type { CalendarEvent, FamilyMember } from "@/lib/types";
import { colorMap } from "@/lib/types";
import { cn } from "@/lib/utils";
import { getEventDateTime } from "../lib/event-time";
import type { HeroState } from "../lib/hero-state";
import { formatRelativeStart, formatRemainingEnd } from "../lib/relative-time";

function isEventState(state: HeroState): state is Extract<HeroState, { event: CalendarEvent }> {
  return "event" in state;
}

function titleFor(state: HeroState) {
  switch (state.kind) {
    case "RIGHT_NOW":
    case "UP_NEXT":
    case "ALL_DAY_ONLY":
      return state.event.title;
    case "REST_OF_DAY_CLEAR":
      return "Rest of day clear";
    case "ALL_CLEAR_TODAY":
      return "All clear today";
  }
}

function metaFor(state: HeroState, now: Date) {
  switch (state.kind) {
    case "RIGHT_NOW":
      return `Now - ${formatRemainingEnd(getEventDateTime(state.event, "end"), now)}`;
    case "UP_NEXT":
      return `Up next - ${formatRelativeStart(getEventDateTime(state.event, "start"), now)}`;
    case "ALL_DAY_ONLY":
      return "Today";
    case "REST_OF_DAY_CLEAR":
      return "Nothing else scheduled";
    case "ALL_CLEAR_TODAY":
      return "Nothing scheduled";
  }
}

function ariaFor(state: HeroState, now: Date) {
  const prefix =
    state.kind === "RIGHT_NOW"
      ? "Right now"
      : state.kind === "UP_NEXT"
        ? "Up next"
        : state.kind === "ALL_DAY_ONLY"
          ? "Today"
          : "Home status";
  return `${prefix}: ${titleFor(state)}. ${metaFor(state, now)}`;
}

export function LargeNowHero({
  state,
  member,
  now,
  onOpenEvent,
}: {
  state: HeroState;
  member?: FamilyMember;
  now: Date;
  onOpenEvent: (event: CalendarEvent) => void;
}) {
  const colors = member ? colorMap[member.color] : null;
  const event = isEventState(state) ? state.event : null;
  const content = (
    <div className="relative min-h-[22rem] overflow-hidden rounded-lg border border-border/70 bg-card px-8 py-8 shadow-sm lg:min-h-[28rem] lg:px-10 lg:py-10 2xl:min-h-[34rem] 2xl:px-14 2xl:py-14">
      {event && colors && (
        <span
          data-testid="large-hero-accent"
          className="absolute inset-y-8 left-0 w-1.5 rounded-r-full"
          style={{ backgroundColor: colors.hex }}
        />
      )}
      <div className="flex h-full flex-col justify-between gap-8 pl-2">
        <div className="space-y-5">
          <p className="text-lg font-semibold text-foreground/65 lg:text-xl">
            {metaFor(state, now)}
          </p>
          <h2 className="max-w-[12ch] text-6xl font-semibold leading-[1.03] text-foreground lg:text-7xl 2xl:text-8xl">
            {titleFor(state)}
          </h2>
          {event?.location && (
            <p className="max-w-xl text-2xl leading-8 text-foreground/65">
              {event.location}
            </p>
          )}
        </div>
        <div className="flex items-center gap-3 text-base text-muted-foreground">
          {member && colors ? (
            <>
              <span
                aria-hidden="true"
                className={cn("h-3 w-3 rounded-full", colors.bg)}
              />
              <span>{member.name}</span>
            </>
          ) : (
            <>
              <Sparkles className="h-5 w-5 text-foreground/35 motion-safe:animate-home-breathing-fade motion-reduce:animate-none" />
              <span>Family schedule</span>
            </>
          )}
        </div>
      </div>
    </div>
  );

  return (
    <section aria-label={ariaFor(state, now)}>
      {event ? (
        <button
          type="button"
          onClick={() => onOpenEvent(event)}
          aria-label={ariaFor(state, now)}
          className="block w-full text-left focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2"
        >
          {content}
        </button>
      ) : (
        content
      )}
    </section>
  );
}
```

- [ ] **Step 6: Implement `LargeStateStrip`**

Create `src/components/home/components/large-state-strip.tsx`:

```tsx
import { CheckSquare, ListTodo, UtensilsCrossed } from "lucide-react";
import type { HomeStateSummary, HomeSummaryTarget } from "../lib/large-home-selectors";

const iconByModule = {
  chores: CheckSquare,
  meals: UtensilsCrossed,
  lists: ListTodo,
};

const labelByModule = {
  chores: "Chores",
  meals: "Meals",
  lists: "Lists",
};

export function LargeStateStrip({
  summaries,
  onSelect,
}: {
  summaries: HomeStateSummary[];
  onSelect: (target: HomeSummaryTarget) => void;
}) {
  return (
    <section aria-label="Household status" className="grid grid-cols-3 gap-3">
      {summaries.map((summary) => {
        const Icon = iconByModule[summary.module];
        const moduleLabel = labelByModule[summary.module];
        return (
          <button
            key={summary.module}
            type="button"
            onClick={() => onSelect(summary.target)}
            aria-label={`Open ${moduleLabel}. ${summary.label}`}
            className="min-h-24 rounded-lg border border-border/70 bg-card px-4 py-4 text-left shadow-sm transition-colors hover:bg-muted/40 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2"
          >
            <div className="mb-3 flex items-center gap-2 text-sm font-semibold text-muted-foreground">
              <Icon className="h-4 w-4" />
              <span>{moduleLabel}</span>
            </div>
            <p className="text-xl font-semibold leading-7 text-foreground">
              {summary.label}
            </p>
          </button>
        );
      })}
    </section>
  );
}
```

- [ ] **Step 7: Implement `LargeTodayRail`**

Create `src/components/home/components/large-today-rail.tsx`:

```tsx
import { format } from "date-fns";
import type { CalendarEvent, FamilyMember } from "@/lib/types";
import { colorMap, getFamilyMember } from "@/lib/types";
import { cn } from "@/lib/utils";
import { formatEventTimeForDisplay } from "../lib/event-time";

function EventRow({
  event,
  members,
  onSelect,
}: {
  event: CalendarEvent;
  members: FamilyMember[];
  onSelect: (event: CalendarEvent) => void;
}) {
  const member = getFamilyMember(members, event.memberId);
  const colors = member ? colorMap[member.color] : colorMap.coral;
  const time = event.isAllDay ? "All day" : formatEventTimeForDisplay(event.startTime);

  return (
    <button
      type="button"
      onClick={() => onSelect(event)}
      className="flex min-h-14 w-full items-center gap-3 rounded-lg px-2 py-2 text-left transition-colors hover:bg-muted/45 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2"
    >
      <span aria-hidden="true" className={cn("h-2.5 w-2.5 shrink-0 rounded-full", colors.bg)} />
      <span className="w-20 shrink-0 text-sm font-medium text-foreground/60">{time}</span>
      <span className="min-w-0 flex-1 truncate text-lg font-semibold leading-7 text-foreground">
        {event.title}
      </span>
      {member && <span className="shrink-0 text-sm text-muted-foreground">{member.name}</span>}
    </button>
  );
}

export function LargeTodayRail({
  currentDate,
  todayItems,
  tomorrowItems,
  members,
  onSelect,
}: {
  currentDate: Date;
  todayItems: CalendarEvent[];
  tomorrowItems: CalendarEvent[];
  members: FamilyMember[];
  onSelect: (event: CalendarEvent) => void;
}) {
  return (
    <aside className="flex min-h-0 flex-col gap-6 rounded-lg border border-border/70 bg-card px-5 py-5 shadow-sm">
      <section>
        <div className="mb-4 flex items-baseline justify-between gap-3">
          <h3 className="text-2xl font-semibold leading-8 text-foreground">Today</h3>
          <p className="text-sm font-medium text-muted-foreground">
            {format(currentDate, "EEE, MMM d")}
          </p>
        </div>
        {todayItems.length > 0 ? (
          <div className="space-y-1">
            {todayItems.map((event) => (
              <EventRow key={event.id ?? `${event.title}-${event.startTime}`} event={event} members={members} onSelect={onSelect} />
            ))}
          </div>
        ) : (
          <p className="rounded-lg border border-dashed border-border px-4 py-6 text-sm text-muted-foreground">
            Rest of day clear
          </p>
        )}
      </section>

      {tomorrowItems.length > 0 && (
        <section className="border-t border-border/70 pt-5">
          <h3 className="mb-3 text-base font-semibold leading-6 text-foreground/70">
            Tomorrow
          </h3>
          <div className="space-y-1">
            {tomorrowItems.map((event) => (
              <EventRow key={event.id ?? `${event.title}-${event.startTime}`} event={event} members={members} onSelect={onSelect} />
            ))}
          </div>
        </section>
      )}
    </aside>
  );
}
```

- [ ] **Step 8: Implement large-home summaries hook**

Create `src/components/home/hooks/use-large-home-summaries.ts`:

```ts
import { useChoresBoard, useLists, useMealsBoard } from "@/api";
import { formatLocalDate, getWeekStartSunday } from "@/lib/time-utils";
import {
  deriveChoresSummary,
  deriveListsSummary,
  deriveMealsSummary,
} from "../lib/large-home-selectors";

export function useLargeHomeSummaries({ now = new Date() }: { now?: Date } = {}) {
  const weekStart = formatLocalDate(getWeekStartSunday(now));
  const chores = useChoresBoard();
  const meals = useMealsBoard(weekStart);
  const lists = useLists();

  return [
    deriveChoresSummary({
      board: chores.data?.data ?? null,
      isLoading: chores.isLoading,
      isError: chores.isError,
    }),
    deriveMealsSummary({
      board: meals.data?.data ?? null,
      today: now,
      isLoading: meals.isLoading,
      isError: meals.isError,
    }),
    deriveListsSummary({
      lists: lists.data?.data ?? null,
      isLoading: lists.isLoading,
      isError: lists.isError,
    }),
  ];
}
```

- [ ] **Step 9: Write the summaries hook test**

Create `src/components/home/hooks/use-large-home-summaries.test.tsx`:

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";
import { describe, expect, it } from "vitest";
import { renderHook, waitFor } from "@/test/test-utils";
import { setupMswServer } from "@/test/mocks/server";
import { useLargeHomeSummaries } from "./use-large-home-summaries";

function createWrapper() {
  const client = new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: Infinity },
      mutations: { retry: false },
    },
  });

  return function Wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  };
}

describe("useLargeHomeSummaries", () => {
  setupMswServer();

  it("returns Chores, Meals, and Lists summaries from existing queries", async () => {
    const { result } = renderHook(
      () => useLargeHomeSummaries({ now: new Date(2026, 4, 17, 12) }),
      { wrapper: createWrapper() },
    );

    await waitFor(() => {
      expect(result.current.map((summary) => summary.module)).toEqual([
        "chores",
        "meals",
        "lists",
      ]);
      expect(result.current.map((summary) => summary.kind)).toEqual([
        "empty",
        "missing",
        "quiet",
      ]);
    });
  });
});
```

- [ ] **Step 10: Run the summaries hook test**

Run:

```bash
npm test -- --run src/components/home/hooks/use-large-home-summaries.test.tsx
```

Expected: PASS.

- [ ] **Step 11: Implement `LargeHomeDashboard`**

Create `src/components/home/large-home-dashboard.tsx`:

```tsx
import { useMemo } from "react";
import { useFamilyMembers, useFamilyName } from "@/api";
import { useDashboardEvents } from "@/components/home/hooks/use-dashboard-events";
import { useDashboardNow } from "@/components/home/hooks/use-hero-state";
import { formatLocalDate, getEventKey } from "@/lib/time-utils";
import type { CalendarEvent } from "@/lib/types";
import { useAppStore } from "@/stores";
import { LargeNowHero } from "./components/large-now-hero";
import { LargeStateStrip } from "./components/large-state-strip";
import { LargeTodayRail } from "./components/large-today-rail";
import { useLargeHomeSummaries } from "./hooks/use-large-home-summaries";
import { deriveHeroState } from "./lib/hero-state";
import {
  type HomeSummaryTarget,
  selectRestOfDayItems,
  selectTomorrowPeek,
} from "./lib/large-home-selectors";

function LoadingLargeHome() {
  return (
    <div className="grid min-h-0 flex-1 grid-cols-[minmax(0,1.5fr)_minmax(22rem,0.9fr)] gap-6 p-6">
      <div className="animate-pulse rounded-lg bg-card" />
      <div className="animate-pulse rounded-lg bg-card" />
    </div>
  );
}

export function LargeHomeDashboard({ nowOverride }: { nowOverride?: Date } = {}) {
  const familyName = useFamilyName();
  const members = useFamilyMembers();
  const liveNow = useDashboardNow();
  const now = nowOverride ?? liveNow;
  const { today, comingUp, isLoading, isError, error } = useDashboardEvents({
    currentDate: now,
    memberFocusId: null,
  });
  const summaries = useLargeHomeSummaries({ now });
  const heroState = useMemo(() => deriveHeroState({ todayEvents: today, now }), [now, today]);
  const heroEvent = "event" in heroState ? heroState.event : null;
  const heroMember = heroEvent ? members.find((member) => member.id === heroEvent.memberId) : undefined;
  const todayItems = useMemo(
    () => selectRestOfDayItems(today, heroEvent, now),
    [heroEvent, now, today],
  );
  const tomorrowItems = useMemo(
    () => selectTomorrowPeek(comingUp, now),
    [comingUp, now],
  );

  const openEvent = (event: CalendarEvent) => {
    useAppStore.getState().openCalendarEvent({
      date: formatLocalDate(event.date),
      eventKey: getEventKey(event),
    });
  };

  const openSummary = (target: HomeSummaryTarget) => {
    const store = useAppStore.getState();
    if (target.module === "chores") store.setActiveModule("chores");
    if (target.module === "lists") {
      if (target.listId) store.openListDetail(target.listId);
      else store.setActiveModule("lists");
    }
    if (target.module === "meals") {
      store.focusMealSlot({
        weekStartDate: target.weekStartDate,
        dayIndex: target.dayIndex,
        mealType: target.mealType,
      });
    }
  };

  if (isLoading) return <LoadingLargeHome />;

  if (isError) {
    return (
      <div className="flex-1 p-8 text-sm text-destructive">
        Error loading events: {error?.message ?? "Unknown error"}
      </div>
    );
  }

  return (
    <div
      data-testid="large-home-dashboard"
      className="flex-1 overflow-y-auto bg-background"
    >
      <div className="mx-auto grid min-h-full max-w-[118rem] grid-cols-[minmax(0,1.42fr)_minmax(22rem,0.88fr)] gap-6 px-6 py-6 lg:px-8 lg:py-8 2xl:gap-8 2xl:px-12 2xl:py-10">
        <div className="flex min-w-0 flex-col gap-5">
          <div>
            <p className="text-sm font-semibold uppercase tracking-normal text-muted-foreground">
              {familyName || "Family Hub"}
            </p>
            <h1 className="mt-1 text-2xl font-semibold text-foreground">
              Home
            </h1>
          </div>
          <LargeNowHero
            state={heroState}
            member={heroMember}
            now={now}
            onOpenEvent={openEvent}
          />
          <LargeStateStrip summaries={summaries} onSelect={openSummary} />
        </div>
        <LargeTodayRail
          currentDate={now}
          todayItems={todayItems}
          tomorrowItems={tomorrowItems}
          members={members}
          onSelect={openEvent}
        />
      </div>
    </div>
  );
}
```

- [ ] **Step 12: Update HomeDashboard integration tests**

In `src/components/home/home-dashboard.test.tsx`, replace the old desktop assertion:

```tsx
it("does NOT mount the feed/state-line on desktop", () => {
  setViewportWidth(1024);
  render(<HomeDashboard nowOverride={new Date(2026, 5, 21, 12)} />);
  expect(
    screen.queryByText(/Since you last opened/i),
  ).not.toBeInTheDocument();
});
```

with:

```tsx
it("renders the large-screen home dashboard on desktop without mobile feed", async () => {
  setViewportWidth(1024);
  seedMockEvents([
    createTestEventResponse({
      id: "today",
      title: "Swim lesson",
      date: "2026-06-21",
      startTime: "1:00 PM",
      endTime: "2:00 PM",
      memberId: testMembers[0].id,
    }),
  ]);

  render(<HomeDashboard nowOverride={new Date(2026, 5, 21, 12)} />);

  expect(await screen.findByTestId("large-home-dashboard")).toBeInTheDocument();
  expect(screen.getByText("Swim lesson")).toBeInTheDocument();
  expect(screen.queryByText(/Since you last opened/i)).not.toBeInTheDocument();
  expect(screen.queryByRole("button", { name: /add event/i })).not.toBeInTheDocument();
});
```

Add a routing test:

```tsx
it("routes large-screen event taps to Calendar instead of opening inline detail", async () => {
  setViewportWidth(1024);
  seedMockEvents([
    createTestEventResponse({
      id: "swim",
      title: "Swim lesson",
      date: "2026-06-21",
      startTime: "1:00 PM",
      endTime: "2:00 PM",
      memberId: testMembers[0].id,
    }),
  ]);

  const { user } = renderWithUser(
    <HomeDashboard nowOverride={new Date(2026, 5, 21, 12)} />,
  );

  await user.click(await screen.findByRole("button", { name: /up next: swim lesson/i }));

  expect(useAppStore.getState().activeModule).toBe("calendar");
  expect(useAppStore.getState().calendarEventIntent).toEqual({
    date: "2026-06-21",
    eventKey: "swim",
  });
  expect(screen.queryByRole("dialog")).not.toBeInTheDocument();
});
```

Add summary routing coverage. Add imports:

```tsx
import type { ListDetail } from "@/lib/types";
import { seedMockLists } from "@/test/mocks/server";
```

Add a minimal grocery fixture near the top of the file:

```tsx
const groceryList: ListDetail = {
  id: "grocery-1",
  name: "Groceries",
  kind: "grocery",
  categoryDisplayMode: "grouped",
  showCompletedOverride: null,
  categories: [],
  items: [
    {
      id: "item-1",
      text: "Milk",
      completed: false,
      completedAt: null,
      categoryId: null,
      createdAt: "2026-06-21T09:00:00",
      updatedAt: "2026-06-21T09:00:00",
    },
  ],
  createdAt: "2026-06-21T09:00:00",
  updatedAt: "2026-06-21T09:00:00",
};
```

Add tests:

```tsx
it("routes large-screen state strip taps to Chores, Meals, and Lists", async () => {
  setViewportWidth(1024);
  seedMockLists([groceryList]);
  const { user } = renderWithUser(
    <HomeDashboard nowOverride={new Date(2026, 5, 21, 12)} />,
  );

  await user.click(await screen.findByRole("button", { name: /open chores/i }));
  expect(useAppStore.getState().activeModule).toBe("chores");

  useAppStore.getState().setActiveModule(null);
  await user.click(await screen.findByRole("button", { name: /open meals/i }));
  expect(useAppStore.getState().activeModule).toBe("meals");
  expect(useAppStore.getState().mealSlotIntent).toMatchObject({
    mealType: "dinner",
  });

  useAppStore.getState().setActiveModule(null);
  await user.click(await screen.findByRole("button", { name: /open lists/i }));
  expect(useAppStore.getState().activeModule).toBe("lists");
  expect(useAppStore.getState().listDetailIntent).toBe("grocery-1");
});
```

Add import:

```tsx
import { useAppStore } from "@/stores";
```

- [ ] **Step 13: Run Task 4 tests**

Run:

```bash
npm test -- --run src/components/home/components/large-now-hero.test.tsx src/components/home/components/large-state-strip.test.tsx src/components/home/components/large-today-rail.test.tsx src/components/home/hooks/use-large-home-summaries.test.tsx src/components/home/lib/large-home-selectors.test.ts src/components/home/home-dashboard.test.tsx
```

Expected: PASS.

- [ ] **Step 14: Commit**

```bash
git add src/components/home
git commit -m "feat(home): add large screen dashboard layout"
```

---

## Task 5: Add Large-Screen Idle Return

**Files:**
- Create: `frontend/src/hooks/use-large-screen-home-idle-return.ts`
- Test: `frontend/src/hooks/use-large-screen-home-idle-return.test.tsx`
- Modify: `frontend/src/stores/app-store.ts`
- Modify: `frontend/src/stores/app-store.test.ts`
- Modify: `frontend/src/test/setup.ts`
- Modify: `frontend/src/test/test-utils.tsx`
- Modify: `frontend/src/components/meals-view.tsx`
- Modify: `frontend/src/components/meals-view.test.tsx`
- Modify: `frontend/src/App.tsx`

- [ ] **Step 1: Write idle-return hook tests**

Create `src/hooks/use-large-screen-home-idle-return.test.tsx`:

```tsx
import { renderHook } from "@testing-library/react";
import { afterEach, describe, expect, it, vi } from "vitest";
import { useLargeScreenHomeIdleReturn } from "./use-large-screen-home-idle-return";

describe("useLargeScreenHomeIdleReturn", () => {
  afterEach(() => {
    vi.useRealTimers();
    document.body.innerHTML = "";
  });

  it("returns to Home after idle on large screens", () => {
    vi.useFakeTimers();
    const setActiveModule = vi.fn();

    renderHook(() =>
      useLargeScreenHomeIdleReturn({
        enabled: true,
        activeModule: "calendar",
        setActiveModule,
        idleMs: 1000,
      }),
    );

    vi.advanceTimersByTime(1000);
    expect(setActiveModule).toHaveBeenCalledWith(null);
  });

  it("does not return when already on Home", () => {
    vi.useFakeTimers();
    const setActiveModule = vi.fn();

    renderHook(() =>
      useLargeScreenHomeIdleReturn({
        enabled: true,
        activeModule: null,
        setActiveModule,
        idleMs: 1000,
      }),
    );

    vi.advanceTimersByTime(1000);
    expect(setActiveModule).not.toHaveBeenCalled();
  });

  it("resets the timer on user activity", () => {
    vi.useFakeTimers();
    const setActiveModule = vi.fn();

    renderHook(() =>
      useLargeScreenHomeIdleReturn({
        enabled: true,
        activeModule: "lists",
        setActiveModule,
        idleMs: 1000,
      }),
    );

    vi.advanceTimersByTime(800);
    window.dispatchEvent(new KeyboardEvent("keydown", { key: "Tab" }));
    vi.advanceTimersByTime(800);
    expect(setActiveModule).not.toHaveBeenCalled();
    vi.advanceTimersByTime(200);
    expect(setActiveModule).toHaveBeenCalledWith(null);
  });

  it("blocks idle return while a dialog is open", () => {
    vi.useFakeTimers();
    const setActiveModule = vi.fn();
    const dialog = document.createElement("div");
    dialog.setAttribute("role", "dialog");
    document.body.append(dialog);

    renderHook(() =>
      useLargeScreenHomeIdleReturn({
        enabled: true,
        activeModule: "meals",
        setActiveModule,
        idleMs: 1000,
      }),
    );

    vi.advanceTimersByTime(1000);
    expect(setActiveModule).not.toHaveBeenCalled();
  });

  it("blocks idle return while an explicit module flow blocker is active", () => {
    vi.useFakeTimers();
    const setActiveModule = vi.fn();

    renderHook(() =>
      useLargeScreenHomeIdleReturn({
        enabled: true,
        activeModule: "meals",
        setActiveModule,
        idleMs: 1000,
        isBlocked: () => true,
      }),
    );

    vi.advanceTimersByTime(1000);
    expect(setActiveModule).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run hook tests and verify failure**

Run:

```bash
cd frontend
npm test -- --run src/hooks/use-large-screen-home-idle-return.test.tsx
```

Expected: FAIL because the hook does not exist.

- [ ] **Step 3: Add explicit idle blockers to the app store**

Add this state and action to `src/stores/app-store.ts`:

```ts
idleReturnBlockers: Record<string, true>;
setIdleReturnBlocked: (key: string, blocked: boolean) => void;
```

Add the initial value:

```ts
idleReturnBlockers: {},
```

Add the action:

```ts
setIdleReturnBlocked: (key, blocked) =>
  set((state) => {
    const next = { ...state.idleReturnBlockers };
    if (blocked) next[key] = true;
    else delete next[key];
    return { idleReturnBlockers: next };
  }),
```

Add this test to `src/stores/app-store.test.ts`:

```ts
it("tracks explicit idle-return blockers by key", () => {
  useAppStore.getState().setIdleReturnBlocked("meals-planning", true);
  expect(useAppStore.getState().idleReturnBlockers).toEqual({
    "meals-planning": true,
  });

  useAppStore.getState().setIdleReturnBlocked("meals-planning", false);
  expect(useAppStore.getState().idleReturnBlockers).toEqual({});
});
```

Update `src/test/setup.ts` app-store reset:

```ts
useAppStore.setState({
  activeModule: "calendar",
  isSidebarOpen: false,
  mealPlacementDraft: null,
  recipeCreationDraft: null,
  listDetailIntent: null,
  calendarFocusDate: null,
  calendarEventIntent: null,
  mealSlotIntent: null,
  idleReturnBlockers: {},
});
```

Update `src/test/test-utils.tsx` `resetAppStore()` so direct utility resets also clear the blocker:

```ts
export function resetAppStore(): void {
  useAppStore.setState({
    activeModule: "calendar",
    isSidebarOpen: false,
    mealPlacementDraft: null,
    recipeCreationDraft: null,
    listDetailIntent: null,
    calendarFocusDate: null,
    calendarEventIntent: null,
    mealSlotIntent: null,
    idleReturnBlockers: {},
  });
}
```

- [ ] **Step 4: Implement the idle-return hook**

Create `src/hooks/use-large-screen-home-idle-return.ts`:

```ts
import { useEffect, useRef } from "react";
import type { ModuleType } from "@/stores";

export const LARGE_SCREEN_HOME_IDLE_MS = 10 * 60 * 1000;

const ACTIVITY_EVENTS = ["pointerdown", "keydown", "touchstart", "wheel"] as const;

function hasBlockingSurface(): boolean {
  if (typeof document === "undefined") return false;
  return Boolean(
    document.querySelector(
      '[role="dialog"], [data-vaul-drawer][data-state="open"]',
    ),
  );
}

export function useLargeScreenHomeIdleReturn({
  enabled,
  activeModule,
  setActiveModule,
  isBlocked,
  idleMs = LARGE_SCREEN_HOME_IDLE_MS,
}: {
  enabled: boolean;
  activeModule: ModuleType | null;
  setActiveModule: (module: ModuleType | null) => void;
  isBlocked?: () => boolean;
  idleMs?: number;
}) {
  const activeModuleRef = useRef(activeModule);
  const setActiveModuleRef = useRef(setActiveModule);
  const isBlockedRef = useRef(isBlocked);

  useEffect(() => {
    activeModuleRef.current = activeModule;
  }, [activeModule]);

  useEffect(() => {
    setActiveModuleRef.current = setActiveModule;
  }, [setActiveModule]);

  useEffect(() => {
    isBlockedRef.current = isBlocked;
  }, [isBlocked]);

  useEffect(() => {
    if (!enabled || typeof window === "undefined") return;

    let timer: ReturnType<typeof window.setTimeout> | null = null;

    const schedule = () => {
      if (timer) window.clearTimeout(timer);
      timer = window.setTimeout(() => {
        if (activeModuleRef.current === null) return;
        if (isBlockedRef.current?.() || hasBlockingSurface()) {
          schedule();
          return;
        }
        setActiveModuleRef.current(null);
      }, idleMs);
    };

    schedule();
    for (const eventName of ACTIVITY_EVENTS) {
      window.addEventListener(eventName, schedule, { passive: true });
    }

    return () => {
      if (timer) window.clearTimeout(timer);
      for (const eventName of ACTIVITY_EVENTS) {
        window.removeEventListener(eventName, schedule);
      }
    };
  }, [enabled, idleMs]);
}
```

- [ ] **Step 5: Wire Meals active flows into idle blockers**

In `src/components/meals-view.tsx`, read the blocker action:

```tsx
const setIdleReturnBlocked = useAppStore(
  (state) => state.setIdleReturnBlocked,
);
```

Add this derived flag after `planningActive` is declared:

```tsx
const hasActiveMealFlow =
  selectedSlot !== null ||
  editingSlotId !== null ||
  placementDraft !== null ||
  scopeOpen ||
  planningActive;
```

Add this effect near the other MealsView effects:

```tsx
useEffect(() => {
  setIdleReturnBlocked("meals-active-flow", hasActiveMealFlow);
  return () => setIdleReturnBlocked("meals-active-flow", false);
}, [hasActiveMealFlow, setIdleReturnBlocked]);
```

Add this test to `src/components/meals-view.test.tsx`:

```tsx
it("blocks large-screen idle return while a meal flow is active", async () => {
  seedMockMealsBoard(createEmptyMealsBoard());
  const { user } = renderWithUser(<MealsView />);

  const dinnerButtons = await screen.findAllByRole("button", {
    name: /add dinner/i,
  });
  await user.click(dinnerButtons[0]);

  expect(useAppStore.getState().idleReturnBlockers).toEqual({
    "meals-active-flow": true,
  });
});
```

The DOM fallback in the hook still blocks Calendar, Chores, Lists, and sheet/dialog flows that render as dialogs. The explicit Meals blocker covers inline planning/session state that may not be represented by a dialog.

- [ ] **Step 6: Wire idle return into App**

In `src/App.tsx`, import:

```tsx
import { useLargeScreenHomeIdleReturn } from "@/hooks/use-large-screen-home-idle-return";
```

Reintroduce `setActiveModule` if removed in Task 1:

```tsx
const setActiveModule = useAppStore((state) => state.setActiveModule);
const idleReturnBlocked = useAppStore(
  (state) => Object.keys(state.idleReturnBlockers).length > 0,
);
```

Call the hook after `isMobile` is known:

```tsx
useLargeScreenHomeIdleReturn({
  enabled: isAuthenticated && setupComplete && !isMobile,
  activeModule,
  setActiveModule,
  isBlocked: () => idleReturnBlocked,
});
```

- [ ] **Step 7: Run idle and App tests**

Run:

```bash
npm test -- --run src/hooks/use-large-screen-home-idle-return.test.tsx src/stores/app-store.test.ts src/components/meals-view.test.tsx src/App.large-home.test.tsx
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add src/hooks/use-large-screen-home-idle-return.ts src/hooks/use-large-screen-home-idle-return.test.tsx src/stores/app-store.ts src/stores/app-store.test.ts src/test/setup.ts src/test/test-utils.tsx src/components/meals-view.tsx src/components/meals-view.test.tsx src/App.tsx
git commit -m "feat(home): return large screens to home after idle"
```

---

## Task 6: Add E2E Coverage And Screenshot Evidence

**Files:**
- Modify: `frontend/e2e/helpers/api-helpers.ts`
- Create: `frontend/e2e/large-screen-home.spec.ts`

- [ ] **Step 1: Add E2E helper API methods**

Append to `e2e/helpers/api-helpers.ts`:

```ts
import type {
  ChoreTemplate,
  CreateChoreTemplateRequest,
  UpdateCurrentPeriodCompletionRequest,
  CreateListItemRequest,
  CreateListRequest,
  ListDetail,
  MealSlot,
  UpsertMealSlotRequest,
} from "../../src/lib/types";
```

If this creates duplicate imports, merge the imported types into the existing import block.

Add helpers:

```ts
export async function createChoreTemplate(
  request: APIRequestContext,
  token: string,
  chore: CreateChoreTemplateRequest,
): Promise<ChoreTemplate> {
  const response = await request.post(`${API_BASE}/chores/templates`, {
    headers: { Authorization: `Bearer ${token}` },
    data: chore,
  });
  if (!response.ok()) {
    throw new Error(`Create chore failed (${response.status()}): ${await response.text()}`);
  }
  const json = await response.json();
  return json.data;
}

export async function completeCurrentChore(
  request: APIRequestContext,
  token: string,
  templateId: string,
  completion: UpdateCurrentPeriodCompletionRequest,
): Promise<void> {
  const response = await request.put(
    `${API_BASE}/chores/templates/${templateId}/current-period-completion`,
    {
      headers: { Authorization: `Bearer ${token}` },
      data: completion,
    },
  );
  if (!response.ok()) {
    throw new Error(`Complete chore failed (${response.status()}): ${await response.text()}`);
  }
}

export async function upsertMealSlot(
  request: APIRequestContext,
  token: string,
  slot: UpsertMealSlotRequest,
): Promise<MealSlot> {
  const response = await request.put(`${API_BASE}/meals/slots`, {
    headers: { Authorization: `Bearer ${token}` },
    data: slot,
  });
  if (!response.ok()) {
    throw new Error(`Upsert meal failed (${response.status()}): ${await response.text()}`);
  }
  const json = await response.json();
  return json.data;
}

export async function createList(
  request: APIRequestContext,
  token: string,
  list: CreateListRequest,
): Promise<ListDetail> {
  const response = await request.post(`${API_BASE}/lists`, {
    headers: { Authorization: `Bearer ${token}` },
    data: list,
  });
  if (!response.ok()) {
    throw new Error(`Create list failed (${response.status()}): ${await response.text()}`);
  }
  const json = await response.json();
  return json.data;
}

export async function createListItem(
  request: APIRequestContext,
  token: string,
  listId: string,
  item: CreateListItemRequest,
): Promise<void> {
  const response = await request.post(`${API_BASE}/lists/${listId}/items`, {
    headers: { Authorization: `Bearer ${token}` },
    data: item,
  });
  if (!response.ok()) {
    throw new Error(`Create list item failed (${response.status()}): ${await response.text()}`);
  }
}
```

- [ ] **Step 2: Write large-screen Home E2E spec**

Create `e2e/large-screen-home.spec.ts`:

```ts
import { expect, test } from "@playwright/test";
import {
  createCalendarEvent,
  completeCurrentChore,
  createChoreTemplate,
  createList,
  createListItem,
  registerFamily,
  seedBrowserAuth,
  upsertMealSlot,
} from "./helpers/api-helpers";
import {
  clearStorage,
  safeClick,
  waitForHydration,
  waitForOfflineCachePersisted,
} from "./helpers/test-helpers";
import {
  format24hTo12h,
  formatLocalDate,
  getWeekStartSunday,
} from "../../src/lib/time-utils";

const FIXED_NOW = new Date(2026, 6, 5, 8, 0, 0);

function to24h(date: Date) {
  return `${String(date.getHours()).padStart(2, "0")}:${String(date.getMinutes()).padStart(2, "0")}`;
}

function eventTime(minutesFromNow: number, durationMinutes = 60) {
  const start = new Date(FIXED_NOW);
  start.setSeconds(0, 0);
  start.setMinutes(start.getMinutes() + minutesFromNow);
  const end = new Date(start.getTime() + durationMinutes * 60_000);

  return {
    date: formatLocalDate(start),
    startTime: format24hTo12h(to24h(start)),
    endTime: format24hTo12h(to24h(end)),
  };
}

function localDate(daysFromToday = 0) {
  const date = new Date(FIXED_NOW);
  date.setHours(0, 0, 0, 0);
  date.setDate(date.getDate() + daysFromToday);
  return formatLocalDate(date);
}

function currentWeekStart() {
  return formatLocalDate(getWeekStartSunday(FIXED_NOW));
}

test.describe("Large-screen Home", () => {
  test.beforeEach(async ({ page, isMobile }) => {
    test.skip(isMobile, "Large-screen only");
    await page.setViewportSize({ width: 1440, height: 900 });
    await page.clock.setFixedTime(FIXED_NOW);
    await page.goto("/");
    await clearStorage(page);
  });

  test("fresh launch lands on Home and routes event taps to Calendar", async ({ page, request }, testInfo) => {
    const registration = await registerFamily(request, {
      familyName: "Large Home Family",
      members: [
        { name: "Alice", color: "coral" },
        { name: "Bob", color: "teal" },
      ],
    });
    const [alice] = registration.family.members;
    const swimTime = eventTime(75);

    await createCalendarEvent(request, registration.token, {
      title: "Swim lesson",
      startTime: swimTime.startTime,
      endTime: swimTime.endTime,
      date: swimTime.date,
      memberId: alice.id,
      isAllDay: false,
    });

    await seedBrowserAuth(page, registration);
    await page.reload();
    await waitForHydration(page);

    const home = page.getByTestId("large-home-dashboard");
    await expect(home).toBeVisible();
    await expect(page.getByRole("button", { name: /^home$/i })).toHaveAttribute(
      "aria-current",
      "page",
    );
    await expect(page.getByText("72°")).not.toBeVisible();
    await expect(home.getByText("Swim lesson")).toBeVisible();
    await expect(home.getByText(/recipes/i)).not.toBeVisible();
    await expect(home.getByText(/photos/i)).not.toBeVisible();
    await testInfo.attach("large-home-sparse-day", {
      body: await page.screenshot({ fullPage: true }),
      contentType: "image/png",
    });

    await safeClick(page.getByRole("button", { name: /swim lesson/i }).first());

    await expect(page.getByRole("button", { name: /^calendar$/i })).toHaveAttribute(
      "aria-current",
      "page",
    );
    await expect(page.getByRole("dialog")).toContainText("Swim lesson");
  });

  test("captures required visual states for review", async ({ page, request }, testInfo) => {
    const registration = await registerFamily(request, {
      familyName: "Visual Matrix Family",
      members: [
        { name: "Alice", color: "coral" },
        { name: "Bob", color: "teal" },
      ],
    });
    const [alice, bob] = registration.family.members;
    const today = localDate(0);
    const tomorrow = localDate(1);
    const weekStartDate = currentWeekStart();
    const longTitleTime = eventTime(30);
    const swimTime = eventTime(150);

    await createCalendarEvent(request, registration.token, {
      title: "Very long orchestra rehearsal title that should wrap without colliding",
      startTime: longTitleTime.startTime,
      endTime: longTitleTime.endTime,
      date: longTitleTime.date,
      memberId: alice.id,
      isAllDay: false,
    });
    await createCalendarEvent(request, registration.token, {
      title: "Swim lesson",
      startTime: swimTime.startTime,
      endTime: swimTime.endTime,
      date: swimTime.date,
      memberId: bob.id,
      isAllDay: false,
    });
    await createCalendarEvent(request, registration.token, {
      title: "Camp dropoff",
      startTime: "8:00 AM",
      endTime: "8:30 AM",
      date: tomorrow,
      memberId: alice.id,
      isAllDay: false,
    });
    await createChoreTemplate(request, registration.token, {
      title: "Unload dishwasher",
      assignedToMemberId: alice.id,
      cadence: "DAILY",
      activeFrom: today,
    });
    await upsertMealSlot(request, registration.token, {
      weekStartDate,
      dayIndex: FIXED_NOW.getDay(),
      mealType: "dinner",
      primary: {
        sourceType: "quick",
        recipeId: null,
        title: "Tacos",
        imageUrl: null,
        note: null,
      },
      extras: [],
      note: null,
      collisionMode: null,
    });
    const groceries = await createList(request, registration.token, {
      name: "Groceries",
      kind: "grocery",
    });
    await createListItem(request, registration.token, groceries.id, { text: "Milk" });
    await createListItem(request, registration.token, groceries.id, { text: "Apples" });

    await seedBrowserAuth(page, registration);

    for (const viewport of [
      { name: "horizontal-tablet", width: 1024, height: 768 },
      { name: "desktop", width: 1440, height: 900 },
      { name: "large-display", width: 1920, height: 1080 },
    ]) {
      await page.setViewportSize({ width: viewport.width, height: viewport.height });
      await page.reload();
      await waitForHydration(page);
      await expect(page.getByTestId("large-home-dashboard")).toBeVisible();
      await testInfo.attach(`large-home-${viewport.name}`, {
        body: await page.screenshot({ fullPage: true }),
        contentType: "image/png",
      });
    }
  });

  test("captures quiet and offline-cached visual states", async ({ page, request, context }, testInfo) => {
    const registration = await registerFamily(request, {
      familyName: "Quiet Home Family",
      members: [{ name: "Alice", color: "coral" }],
    });
    const [alice] = registration.family.members;
    const today = localDate(0);
    const chore = await createChoreTemplate(request, registration.token, {
      title: "Wipe counters",
      assignedToMemberId: alice.id,
      cadence: "DAILY",
      activeFrom: today,
    });
    await completeCurrentChore(request, registration.token, chore.id, {
      scope: "TODAY",
      periodStartDate: today,
    });

    await seedBrowserAuth(page, registration);
    await page.reload();
    await waitForHydration(page);

    const home = page.getByTestId("large-home-dashboard");
    await expect(home).toBeVisible();
    await expect(home.getByText(/all clear|nothing on the calendar/i)).toBeVisible();
    await expect(home.getByText(/dinner not planned/i)).toBeVisible();
    await expect(home.getByText(/chores done/i)).toBeVisible();
    await expect(home.getByText(/lists quiet/i)).toBeVisible();
    await testInfo.attach("large-home-all-clear-missing-dinner-chores-done-lists-quiet", {
      body: await page.screenshot({ fullPage: true }),
      contentType: "image/png",
    });

    await waitForOfflineCachePersisted(page);
    await context.setOffline(true);
    await page.evaluate(() => window.dispatchEvent(new Event("offline")));
    await expect(home).toBeVisible();
    await testInfo.attach("large-home-offline-cached-data", {
      body: await page.screenshot({ fullPage: true }),
      contentType: "image/png",
    });
    await context.setOffline(false);
  });
});

test.describe("Mobile Home visual baseline", () => {
  test.beforeEach(async ({ page, isMobile }) => {
    test.skip(!isMobile, "Mobile baseline only");
    await page.clock.setFixedTime(FIXED_NOW);
    await page.goto("/");
    await clearStorage(page);
  });

  test("captures existing mobile Home baseline for comparison", async ({ page, request }, testInfo) => {
    const registration = await registerFamily(request, {
      familyName: "Mobile Baseline Family",
      members: [{ name: "Alice", color: "coral" }],
    });
    const [alice] = registration.family.members;
    const next = eventTime(60);
    await createCalendarEvent(request, registration.token, {
      title: "Piano lesson",
      startTime: next.startTime,
      endTime: next.endTime,
      date: next.date,
      memberId: alice.id,
      isAllDay: false,
    });

    await seedBrowserAuth(page, registration);
    await page.reload();
    await waitForHydration(page);

    await expect(page.getByTestId("dashboard-header")).toBeVisible();
    await testInfo.attach("mobile-home-baseline", {
      body: await page.screenshot({ fullPage: true }),
      contentType: "image/png",
    });
  });
});
```

- [ ] **Step 3: Run the E2E spec**

Run:

```bash
cd frontend
npx playwright test e2e/large-screen-home.spec.ts --project=chromium
```

Expected: PASS. The HTML report should include attached screenshots for sparse day, busy day/long title, all-clear, missing dinner, chores done, lists quiet, offline cached data, horizontal tablet, desktop, and large display.

- [ ] **Step 4: Capture existing mobile Home baseline**

Run:

```bash
npx playwright test e2e/large-screen-home.spec.ts --project="Mobile Chrome" -g "captures existing mobile Home baseline"
npx playwright test e2e/mobile-home-dashboard.spec.ts --project="Mobile Chrome"
```

Expected: PASS. Use the `mobile-home-baseline` screenshot attachment from `large-screen-home.spec.ts` as the visual comparison baseline, and use `mobile-home-dashboard.spec.ts` as behavioral regression evidence.

- [ ] **Step 5: Perform screenshot critique**

Open the Playwright HTML report:

```bash
npx playwright show-report
```

Record a short work-log note in the PR body using this exact checklist:

```md
### Large Home Screenshot Critique

- Mobile baseline unchanged: yes/no
- Horizontal tablet screenshot reviewed: yes/no
- Desktop screenshot reviewed: yes/no
- Large-display screenshot reviewed: yes/no
- Sparse day screenshot reviewed: yes/no
- Busy day screenshot reviewed: yes/no
- All-clear day screenshot reviewed: yes/no
- Missing dinner screenshot reviewed: yes/no
- Chores done screenshot reviewed: yes/no
- Lists quiet screenshot reviewed: yes/no
- Offline cached data screenshot reviewed: yes/no
- Now Hero is unmistakable focal point: yes/no
- Next important thing understandable within two seconds: yes/no
- Layout reads intentionally sparse, not accidentally empty: yes/no
- Chores, Meals, and Lists are present but subordinate: yes/no
- No weather, Recipes tile, Photos placeholder, or generic feed on Home: yes/no
- Long titles wrap or truncate without overlap: yes/no
- Issues found and fixed before PR: list the fix commits
```

If any item is `no`, fix the UI before opening the PR.

- [ ] **Step 6: Commit**

```bash
git add e2e/helpers/api-helpers.ts e2e/large-screen-home.spec.ts
git commit -m "test(home): cover large screen home experience"
```

---

## Task 7: Final Verification And Handoff

**Files:**
- Modify if needed: `docs/product/backlog/large-screen-ux/large-screen-home.md`

- [ ] **Step 1: Run targeted unit/integration tests**

Run:

```bash
cd frontend
npm test -- --run \
  src/App.large-home.test.tsx \
  src/components/shared/navigation-tabs.test.tsx \
  src/components/shared/app-header.test.tsx \
  src/stores/app-store.test.ts \
  src/components/calendar/calendar-module.test.tsx \
  src/components/meals-view.test.tsx \
  src/hooks/use-large-screen-home-idle-return.test.tsx \
  src/components/home/hooks/use-large-home-summaries.test.tsx \
  src/components/home/lib/large-home-selectors.test.ts \
  src/components/home/home-dashboard.test.tsx \
  src/components/home/components/large-now-hero.test.tsx \
  src/components/home/components/large-state-strip.test.tsx \
  src/components/home/components/large-today-rail.test.tsx
```

Expected: PASS.

- [ ] **Step 2: Run full frontend quality gates**

Run:

```bash
npm run lint
npm test -- --run
npm run build
```

Expected: all PASS.

- [ ] **Step 3: Run E2E coverage**

Run:

```bash
npx playwright test e2e/large-screen-home.spec.ts --project=chromium
npx playwright test e2e/large-screen-home.spec.ts --project="Mobile Chrome" -g "captures existing mobile Home baseline"
npx playwright test e2e/mobile-home-dashboard.spec.ts --project="Mobile Chrome"
```

Expected: PASS.

- [ ] **Step 4: Report root-doc status changes to the coordinator**

The FE implementation agent must not commit root docs from inside the `frontend/` repo. When the FE issue or PR exists, report the URLs in the work log so the root coordinator can update `docs/product/backlog/large-screen-ux/large-screen-home.md` from the root repo.

The root coordinator may update only these frontmatter fields:

```yaml
status: in-progress
updated: 2026-07-05
issues:
  - <frontend issue URL>
prs:
  - <frontend PR URL>
```

Do not edit `docs/product/roadmap.md` for task-level progress.

- [ ] **Step 5: Final implementation checklist for the PR**

Add this checklist to the FE PR body:

```md
### Non-Negotiable Contract Check

- [ ] Large screens can land on Home instead of redirecting to Calendar.
- [ ] Fresh launch on a large screen lands on Home.
- [ ] Long idle returns to Home unless a dialog/sheet/create/edit flow is active.
- [ ] Now Hero handles current event, next event, all-day-only, rest-of-day clear, and all-clear states.
- [ ] Today rail shows only rest-of-day context and is not a mini calendar.
- [ ] Tomorrow peek is limited and omitted when empty.
- [ ] State strip contains Chores, Meals, and Lists only.
- [ ] Event taps open full Calendar with the tapped event/date obvious.
- [ ] Chores, Meals, and Lists taps open owning modules with available context.
- [ ] Large Home does not render inline module workspaces.
- [ ] Mobile Home remains visually and behaviorally unchanged.
- [ ] No fake weather, Recipes tile, Photos placeholder, or generic feed appears on Home.
- [ ] Required screenshots were produced, critiqued, and iterated.
```

- [ ] **Step 6: Root coordinator docs commit if needed**

Run this only from `/Users/joe.bor/code/family-hub`, not from `frontend/`:

```bash
git add docs/product/backlog/large-screen-ux/large-screen-home.md
git commit -m "docs(home): update large screen home story links"
```

Skip this step when the story file did not change.

---

## Implementation Issue Contract

When opening the FE implementation issue, include this execution contract at the top:

```md
Story: ../docs/product/backlog/large-screen-ux/large-screen-home.md
Spec: ../docs/superpowers/specs/2026-07-05-large-screen-home-design.md
Plan: ../docs/superpowers/plans/2026-07-05-large-screen-home.md

Execution contract:
- Implement the A1 now-first large-screen Home layout only.
- Do not change mobile Home behavior or visuals except for mechanical extraction.
- Do not add backend work unless a frontend contract gap is proven and documented first.
- Home summarizes and routes; it must not host inline Calendar/Meals/Chores/Lists workspaces on large screens.
- The state strip is Chores, Meals, and Lists only.
- No fake weather, Recipes tile, Photos placeholder, generic activity feed, or child/table mode.
- Fresh large-screen launch lands on Home; long idle returns to Home unless an active create/edit/dialog flow is open.
- Event taps route to Calendar with the tapped event/date obvious.
- Visual QA screenshots are required before PR readiness.
```
