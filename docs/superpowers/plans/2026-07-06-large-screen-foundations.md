# Large-Screen Foundations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Slim the desktop app header (removing the fake weather chip) and merge Calendar's two chrome rows into one toolbar, so large screens spend at most ~160px on chrome above module content.

**Architecture:** Two focused changes inside the existing FE component tree. (1) `AppHeader`'s desktop branch collapses from a two-line block plus weather into one compact row. (2) The per-view `CalendarNavigation` row moves out of `DailyCalendar` / `WeeklyCalendar` / `MonthlyCalendar` and into `CalendarModule`'s existing desktop toolbar row, using the shared `getContextLabel()` for the range label. Mobile branches are untouched. The other modules (Lists, Chores, Meals, Recipes) already render a single heading row, so they satisfy the one-toolbar-row rule without changes; their content-width work belongs to their own stories.

**Tech Stack:** React 19, Tailwind CSS v4, Zustand, Vitest + Testing Library. Repo: `frontend/` (FE repo), branch `feat/large-screen-module-polish`.

**Spec:** `docs/superpowers/specs/2026-07-06-large-screen-foundations-design.md`
**Story:** `docs/product/backlog/large-screen-ux/large-screen-foundations.md`

**Scope note:** The spec's content-width table (§3.3) is normative guidance enacted by the per-module stories; this plan implements the shared chrome (§3.1, §3.2) and breakpoint semantics (§3.4, no code needed — `useIsMobile()` at 768px already exists and nothing new activates below 1024px except the chrome itself, which applies to all non-mobile widths as today).

---

### Task 0: Branch setup

**Files:** none

- [ ] **Step 1: Create the FE branch**

```bash
cd /Users/joe.bor/code/family-hub/frontend
git checkout main && git pull && git checkout -b feat/large-screen-module-polish
```

Expected: new branch from up-to-date `main`.

---

### Task 1: Slim desktop app header (remove fake weather)

**Files:**
- Modify: `src/components/shared/app-header.tsx` (desktop branch, lines ~101-157)
- Test: `src/components/shared/app-header.test.tsx`

The test file currently mocks `useIsMobile` to a constant `true`. Convert the mock to a mutable `vi.fn()` so a desktop describe block can flip it. **Gotcha (project memory):** `setup.ts` uses `clearAllMocks` in `afterEach`, and factory-level `vi.fn()` return values are NOT reset by it — every describe block must set the value in its own `beforeEach`.

- [ ] **Step 1: Convert the useIsMobile mock and add failing desktop tests**

In `src/components/shared/app-header.test.tsx`, replace the existing mock:

```tsx
vi.mock("@/hooks", async (importOriginal) => {
  const actual = await importOriginal<typeof import("@/hooks")>();
  return { ...actual, useIsMobile: () => true };
});
```

with:

```tsx
const isMobileMock = vi.fn(() => true);
vi.mock("@/hooks", async (importOriginal) => {
  const actual = await importOriginal<typeof import("@/hooks")>();
  return { ...actual, useIsMobile: () => isMobileMock() };
});
```

Add `isMobileMock.mockReturnValue(true);` as the first line of the existing `AppHeader (mobile)` describe's `beforeEach`.

Append a new describe block at the end of the file:

```tsx
describe("AppHeader (desktop)", () => {
  beforeEach(() => {
    isMobileMock.mockReturnValue(false);
    seedFamilyStore({
      name: "Test Family",
      members: [{ id: "m1", name: "Alice", color: "coral" }],
    });
  });

  it("renders the family name and member dots", () => {
    render(<AppHeader />);
    expect(screen.getByText("Test Family")).toBeInTheDocument();
    expect(screen.getByTitle("Alice")).toBeInTheDocument();
  });

  it("does not render the fake weather chip", () => {
    render(<AppHeader />);
    expect(screen.queryByText(/72°/)).not.toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run the tests to verify the weather test fails**

```bash
npm test -- --run src/components/shared/app-header.test.tsx
```

Expected: mobile tests PASS, `does not render the fake weather chip` FAILS (the chip is currently rendered).

- [ ] **Step 3: Rewrite the desktop header branch**

In `src/components/shared/app-header.tsx`, change the lucide import from:

```tsx
import { Cloud, Menu, Sun } from "lucide-react";
```

to:

```tsx
import { Menu } from "lucide-react";
```

Replace the entire desktop return block (everything after the `if (isMobile) { ... }` block, currently commented `// Desktop: unchanged — Menu left + family name + date/time, weather, dots.`) with:

```tsx
  // Desktop: one compact row — Menu, family name, date/time inline, member
  // dots right. No weather chip until a real weather feature exists.
  return (
    <header
      className={cn(
        "shrink-0 border-b border-border bg-card/95 backdrop-blur supports-[backdrop-filter]:bg-card/85",
        "flex min-h-14 items-center justify-between gap-4",
        "px-6 py-2",
      )}
    >
      <div className="flex min-w-0 items-center gap-3">
        <Button
          variant="ghost"
          size="icon"
          aria-label="Menu"
          className="h-11 w-11 text-muted-foreground hover:text-foreground"
          onClick={openSidebar}
        >
          <Menu className="h-5 w-5" />
        </Button>
        <h1 className="truncate text-lg leading-7 font-semibold text-foreground">
          {familyName || "Family Hub"}
        </h1>
        <div className="flex items-center gap-2 text-sm text-muted-foreground">
          <span>{formatDate(currentDate)}</span>
          <span>•</span>
          <span>{formatTime(new Date())}</span>
        </div>
      </div>

      {/* Family member indicators - used for calendar filtering */}
      {familyMembers.length > 0 && (
        <div className="flex shrink-0 items-center gap-1.5">
          {familyMembers.slice(0, 6).map((member) => (
            <div
              key={member.id}
              className={`w-3 h-3 rounded-full ${colorMap[member.color].bg}`}
              title={member.name}
            />
          ))}
        </div>
      )}
    </header>
  );
```

`min-h-14` (56px) + `py-2` keeps the row under the 64px budget; the Menu
button grows to `h-11 w-11` (44px) per the spec's touch-target AC.

- [ ] **Step 4: Run the tests to verify they pass**

```bash
npm test -- --run src/components/shared/app-header.test.tsx
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/components/shared/app-header.tsx src/components/shared/app-header.test.tsx
git commit -m "feat(shell): slim desktop app header to one row, drop fake weather"
```

---

### Task 2: Merge calendar date navigation into the desktop toolbar

**Files:**
- Modify: `src/components/calendar/calendar-module.tsx` (toolbar at ~lines 488-496; `navigationProps` usage at ~lines 394-399, 466-483)
- Modify: `src/components/calendar/components/calendar-navigation.tsx` (drop its own vertical padding)
- Modify: `src/components/calendar/views/daily-calendar.tsx` (remove nav header + nav props)
- Modify: `src/components/calendar/views/weekly-calendar.tsx` (remove nav header + nav props)
- Modify: `src/components/calendar/views/monthly-calendar.tsx` (remove nav header + nav props, keep `onDateSelect`)
- Test: `src/components/calendar/calendar-module.test.tsx`

Today the desktop chrome is two stacked rows: toolbar (`CalendarViewSwitcher` + `FamilyFilterPills`) and, inside each view, a `CalendarNavigation` row. After this task the toolbar is one row — switcher left, navigation center, filter pills right — and works for all four views including Schedule (the store's `goToPrevious`/`goToNext` already handle the `schedule` case). Mobile (`MobileToolbar`) is untouched.

- [ ] **Step 1: Add a failing test for schedule-view navigation in the toolbar**

`src/components/calendar/calendar-module.test.tsx` has a `describe("Date Navigation")` block (~line 569) whose tests use `seedMockEvents`, `seedCalendarStore`, and `renderWithUser(<CalendarModule />)`. Add inside that block, using the same harness:

```tsx
it("renders date navigation in the toolbar for the schedule view", async () => {
  seedMockEvents([]);
  seedCalendarStore({ calendarView: "schedule" });

  renderWithUser(<CalendarModule />);

  await waitFor(() => {
    expect(screen.queryByText("Loading events...")).not.toBeInTheDocument();
  });

  expect(screen.getByRole("button", { name: /previous/i })).toBeInTheDocument();
  expect(screen.getByRole("button", { name: /next/i })).toBeInTheDocument();
  expect(screen.getByText("Upcoming")).toBeInTheDocument();
});
```

- [ ] **Step 2: Run it to verify it fails**

```bash
npm test -- --run src/components/calendar/calendar-module.test.tsx
```

Expected: the new test FAILS (schedule view currently renders no Previous/Next buttons).

- [ ] **Step 3: Update the existing nav-label assertions to the toolbar's label format**

The three existing "Date Navigation" tests assert the daily view's long label
(`weekday: "long", month: "long", day: "numeric", year: "numeric"`, e.g.
"Sunday, July 5, 2026"). After the merge, the toolbar renders
`getContextLabel("daily", date)` = `"EEE, MMM d"` (e.g. "Sun, Jul 5"). In all
three tests, change the `expectedLabel` options to the short form:

```tsx
const expectedLabel = yesterday.toLocaleDateString("en-US", {
  weekday: "short",
  month: "short",
  day: "numeric",
});
```

(`en-US` short form renders "Sun, Jul 5", matching date-fns `"EEE, MMM d"`.
Apply the same change for the `tomorrow` and today variants.)

- [ ] **Step 4: Move CalendarNavigation into the toolbar**

In `src/components/calendar/calendar-module.tsx`:

Add imports:

```tsx
import { CalendarNavigation } from "@/components/calendar/components/calendar-navigation";
import { getContextLabel } from "@/components/calendar/utils/context-label";
```

(If `CalendarNavigation` is exported from the `@/components/calendar` barrel, extend the existing barrel import instead — check `src/components/calendar/index.ts` and follow the file's existing import style.)

Replace the desktop toolbar block:

```tsx
        <div className="flex flex-col sm:flex-row items-start sm:items-center justify-between gap-3 px-4 py-3 bg-card border-b border-border">
          <CalendarViewSwitcher />
          <FamilyFilterPills />
        </div>
```

with:

```tsx
        <div className="flex flex-wrap items-center justify-between gap-x-4 gap-y-2 px-4 py-2 bg-card border-b border-border">
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

(`flex-wrap` lets the pills drop to a second line in the 769-1023px range; at `lg+` all three fit one row, which is what the AC measures.)

Then delete the now-redundant `navigationProps` spread from the desktop view rendering: `DailyCalendar`, `WeeklyCalendar`, and `MonthlyCalendar` no longer receive `{...navigationProps}` (keep `onDateSelect={selectDateAndSwitchToDaily}` on `MonthlyCalendar`). Delete the `navigationProps` object itself and use `goToPrevious` / `goToNext` / `goToToday` / `isViewingToday` directly in the toolbar.

- [ ] **Step 5: Remove the per-view navigation headers**

In each of `daily-calendar.tsx`, `weekly-calendar.tsx`, `monthly-calendar.tsx`:

1. Delete the `CalendarNavigation` import and the `{/* Navigation header */}` JSX block that renders it.
2. Remove `onPrevious`, `onNext`, `onToday`, `isViewingToday` from the component's props interface and destructuring.
3. Delete any label-formatting helper that is now unused (e.g. `formatWeekLabel` in `weekly-calendar.tsx`) — confirm with a grep before deleting:

```bash
grep -rn "formatWeekLabel\|CalendarNavigation" src/components/calendar/views/
```

In `calendar-navigation.tsx`, change the wrapper class from

```tsx
    <div className="flex items-center justify-center gap-2 py-3">
```

to

```tsx
    <div className="flex items-center justify-center gap-2">
```

(the toolbar now owns vertical padding; button/touch sizes inside are unchanged).

- [ ] **Step 6: Grow toolbar touch targets to 44px**

The spec's AC requires all chrome touch targets at 44px minimum; several
toolbar controls are currently 40px. All of these components are desktop-only
(mobile uses `MobileToolbar`), so the changes are safe:

- `calendar-navigation.tsx`: Previous/Next buttons `h-10 w-10` → `h-11 w-11`;
  Today button `min-h-10` → `min-h-11`.
- `calendar-view-switcher.tsx`: buttons `min-h-10 min-w-10` → `min-h-11 min-w-11`.
- `family-filter-pills.tsx`: add `min-h-11` to each pill button's class list
  (all three variants: the "All" pill, the all-day pill, and the member
  pills), keeping the existing rounded-full pill styling.

- [ ] **Step 7: Run the full unit suite**

```bash
npm test -- --run
```

Expected: PASS, including the pre-existing Previous/Next/Today tests (the buttons moved to the toolbar but keep their accessible names, and Step 3 updated their label assertions) and the new schedule-view test.

- [ ] **Step 8: Commit**

```bash
git add src/components/calendar/
git commit -m "feat(calendar): merge date navigation into single desktop toolbar row"
```

---

### Task 3: Quality gates

**Files:** none (verification only)

- [ ] **Step 1: Type-check via build**

```bash
npm run build
```

**Gotcha (project memory):** type errors are caught ONLY by `npm run build` (`tsc -b`); the lint+test gate misses them. Expected: build succeeds. Removed-but-still-passed props (e.g. a stray `{...navigationProps}`) would surface here.

- [ ] **Step 2: Lint**

```bash
npm run lint
```

Expected: clean (Biome flags the unused imports/helpers if Step 4 of Task 2 missed any).

- [ ] **Step 3: Commit any fixes**

Only if Steps 1-2 required changes:

```bash
git add -A src/ && git commit -m "fix(calendar): clean up after toolbar merge"
```

---

### Task 4: Visual verification (spec polish gate)

**Files:** none (screenshots recorded for the PR)

The spec requires screenshot review, not just green tests. The app needs the real backend locally. **Gotcha (project memory):** the compose `latest` default is stale — pin `BE_IMAGE_TAG` to the latest published BE release.

- [ ] **Step 1: Start backend + dev server**

```bash
BE_IMAGE_TAG=$(gh release list --repo joe-bor/family-hub-api --limit 1 --json tagName -q '.[0].tagName') docker compose up -d
npm run dev
```

(Use the repo's documented compose file/flow if it differs; the point is a released BE tag, not `latest`.)

- [ ] **Step 2: Measure and screenshot at 1440x900**

With the browser/preview tooling at 1440x900, log in to the seeded test family and verify on the Calendar Week view:

- Header row height ≤ 64px (inspect the `<header>` bounding box).
- Header + calendar toolbar combined ≤ ~160px above the time grid.
- Toolbar is one row: switcher, ‹ Today · range label ›, member pills.
- No weather chip anywhere.
- Repeat the toolbar/header check on Day, Month, Schedule (Schedule now has Previous/Today/Next), and glance at Lists/Chores/Meals/Recipes to confirm the slim header applies and nothing overlaps.

Capture screenshots: week view 1440x900, schedule view 1440x900.

- [ ] **Step 3: Mobile regression screenshot at 375x812**

Verify the mobile header (module-aware single row) and `MobileToolbar` are pixel-identical in behavior: title, Today button on Calendar, Menu button, bottom nav. Capture one screenshot.

- [ ] **Step 4: Iterate if the review finds issues**

If spacing/hierarchy looks off (cramped label, wrapping pills at 1440px, misaligned dots), fix, re-run `npm test -- --run && npm run lint`, re-screenshot, and commit as `fix(shell)`/`fix(calendar)` before proceeding.

---

### Task 5: Ship

**Files:**
- Modify (root repo): `docs/product/backlog/large-screen-ux/large-screen-foundations.md` (frontmatter only)

- [ ] **Step 1: Push and open the PR**

```bash
git push -u origin feat/large-screen-module-polish
gh pr create --title "feat: large-screen shell foundations (slim header, single calendar toolbar)" --body "$(cat <<'EOF'
Story: https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/large-screen-ux/large-screen-foundations.md
Spec: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/specs/2026-07-06-large-screen-foundations-design.md
Plan: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-07-06-large-screen-foundations.md

- Desktop app header: one compact row, fake weather chip removed
- Calendar desktop chrome: view switcher + date navigation + member filters in one toolbar row; per-view navigation headers removed; Schedule view gains navigation
- Mobile shell and mobile calendar toolbar unchanged

Visual QA screenshots attached (1440x900 week/schedule, 375x812 mobile baseline).
EOF
)"
```

Attach the Task 4 screenshots to the PR description or a comment.

**Note:** this branch stays open — subsequent large-screen module stories (calendar, lists, meals, chores, recipes) land on the same `feat/large-screen-module-polish` branch per the delivery decision in the foundations spec. Mark the PR as draft until the epic's planned stories for this branch are in, or ship it per-story if CI/review prefers smaller merges — decide with the reviewer at PR time.

- [ ] **Step 2: Update story status (root repo)**

In `docs/product/backlog/large-screen-ux/large-screen-foundations.md`, set `status: in-progress`, update `updated:`, and record the PR URL under `prs:`. Commit in the root repo:

```bash
cd /Users/joe.bor/code/family-hub
git add docs/product/backlog/large-screen-ux/large-screen-foundations.md
git commit -m "docs(large-screen-ux): mark foundations story in progress"
```
