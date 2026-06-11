# Module-Aware Mobile Header — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** One consistent mobile header across all modules — title left (module name / family name / calendar context label), contextual actions + Menu right — replacing the Calendar one-off and the duplicate per-module titles.

**Architecture:** `AppHeader` becomes module-aware on mobile: a small internal title map keyed by `activeModule`, the calendar context label via a shared `getContextLabel` util, and a Today action rendered only for Calendar. `MobileToolbar` shrinks to a single controls row (view pills + member dots) and keeps the member-filter init effect. The `App.tsx` calendar gate is deleted. Chores/Recipes/Lists hide their static mobile titles. Desktop untouched.

**Tech Stack:** React 19, TypeScript, Zustand, Tailwind CSS v4, date-fns, Vitest, Testing Library, Playwright.

**Spec:** `docs/superpowers/specs/2026-06-11-mobile-module-aware-header.md`
**Story:** `docs/product/backlog/mobile-ux/mobile-module-content-polish.md` (finding F1)

---

This is an **FE-only** plan. Execute it inside the `frontend/` repo on a fresh feature branch such as `feat/mobile-module-aware-header`. All paths below are relative to `frontend/`. Use **regular merge commits** (release-please) and conventional commit messages.

## File Structure

| File | Action | Purpose |
|------|--------|---------|
| `src/components/calendar/utils/context-label.ts` | Create | `getContextLabel` extracted from `mobile-toolbar.tsx` |
| `src/components/shared/app-header.tsx` | Modify | Mobile: module title map + Today (calendar) + right-side Menu |
| `src/components/shared/app-header.test.tsx` | Modify/extend | Lock title-per-module + Menu + Today behavior |
| `src/components/calendar/components/mobile-toolbar.tsx` | Modify | Drop Row 1; keep controls row + init effect |
| `src/components/calendar/calendar-module.tsx` | Modify | Drop `onOpenSidebar` prop pass-through (`:460`) if unused |
| `src/App.tsx` | Modify | Unconditional `<AppHeader />` (remove `:170` gate) |
| `src/components/chores-view.tsx` | Modify | Hide `<h1>` on mobile |
| `src/components/recipes-view.tsx` | Modify | Hide `<h1>` + subtitle on mobile |
| `src/components/lists-view.tsx` | Modify | Hide "My Lists" on mobile (both branches) |

---

### Task 1: Extract `getContextLabel` into a shared calendar util

- [ ] **Step 1: Create `src/components/calendar/utils/context-label.ts`** — move `getContextLabel` (and its date-fns imports) verbatim from `mobile-toolbar.tsx:21-40`. Type the first parameter with the calendar store's view type (find the exported `CalendarView`-style type in `@/stores` or `@/lib/types` and use it instead of `string`). Export it; re-export from the calendar barrel if `@/components/calendar` exposes utils (check `src/components/calendar/index.ts`).

- [ ] **Step 2: Update `mobile-toolbar.tsx`** to import it from the new location; delete the local copy.

- [ ] **Step 3: Verify**

  ```bash
  cd frontend
  npm run test -- --run src/components/calendar
  npm run lint
  ```

  Expected: PASS (pure move).

- [ ] **Step 4: Commit**

  ```bash
  git add src/components/calendar/utils/context-label.ts src/components/calendar/components/mobile-toolbar.tsx src/components/calendar/index.ts
  git commit -m "refactor(calendar): extract context label util"
  ```

---

### Task 2: Module-aware `AppHeader` (mobile)

- [ ] **Step 1: Write the failing tests** — extend `src/components/shared/app-header.test.tsx` (it exists; it mocks `useIsMobile` → true and locks "exactly one button: Menu"). New/updated cases:

  ```tsx
  it("shows the family name on Home", () => {
    useAppStore.setState({ activeModule: null });
    render(<AppHeader />);
    expect(screen.getByText("Test Family")).toBeInTheDocument();
  });

  it("shows the module name when a module is active", () => {
    useAppStore.setState({ activeModule: "chores" });
    render(<AppHeader />);
    expect(screen.getByText("Chores")).toBeInTheDocument();
    expect(screen.queryByText("Test Family")).toBeNull();
  });

  it("shows the calendar context label and a working Today button on Calendar", async () => {
    useAppStore.setState({ activeModule: "calendar" });
    seedCalendarStore({ calendarView: "monthly", currentDate: new Date(2026, 5, 11) });
    const { user } = renderWithUser(<AppHeader />);
    expect(screen.getByText("June 2026")).toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: /today/i }));
    // goToToday resets currentDate — assert via store
  });

  it("does not render Today outside Calendar", () => {
    useAppStore.setState({ activeModule: "lists" });
    render(<AppHeader />);
    expect(screen.queryByRole("button", { name: /today/i })).toBeNull();
  });
  ```

  Adjust seeding helpers to the real `seedCalendarStore` signature in `@/test/test-utils`, and update the existing "exactly one button" test: on Calendar there are two (Today + Menu); elsewhere still one (Menu).

- [ ] **Step 2: Run and confirm it fails**

  ```bash
  cd frontend
  npm run test -- --run src/components/shared/app-header.test.tsx
  ```

  Expected: FAIL — header always shows family name, no Today.

- [ ] **Step 3: Implement in `app-header.tsx`**
  - Read `activeModule` from `useAppStore`, `calendarView` from `useCalendarStore`, `goToToday` + `useIsViewingToday` (import the same hooks `mobile-toolbar.tsx` uses).
  - Title map:

    ```tsx
    const MODULE_TITLES: Record<Exclude<ModuleType, null>, string> = {
      calendar: "", // derived from getContextLabel
      lists: "Lists",
      chores: "Chores",
      meals: "Meals",
      recipes: "Recipes",
      photos: "Photos",
    };
    const mobileTitle =
      activeModule === null
        ? familyName || "Family Hub"
        : activeModule === "calendar"
          ? getContextLabel(calendarView, currentDate)
          : MODULE_TITLES[activeModule];
    ```

    (Match the real `ModuleType` union from `@/stores`.)
  - **Mobile layout** (when `isMobile`): single row — `<h1>` with `mobileTitle` (`text-[22px] leading-7 font-semibold`, `min-w-0 truncate`) left; right cluster: Today button when `activeModule === "calendar"` (copy markup/behavior from `mobile-toolbar.tsx:78-89`), then the Menu button (`h-11 w-11`, right side).
  - **Desktop layout:** untouched — keep the current JSX for `!isMobile` (Menu left + family name + date/time, weather, dots). Cleanest structure: early-return a dedicated mobile JSX block, leaving the existing desktop markup as-is below.

- [ ] **Step 4: Run tests**

  ```bash
  cd frontend
  npm run test -- --run src/components/shared
  ```

  Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add src/components/shared/app-header.tsx src/components/shared/app-header.test.tsx
  git commit -m "feat(mobile): module-aware app header with calendar context and Today"
  ```

---

### Task 3: Slim `MobileToolbar` + remove the `App.tsx` gate

- [ ] **Step 1: Update `mobile-toolbar.tsx`** — delete Row 1 (`:72-99`: context label, Today, Menu) and the now-unused `onOpenSidebar` prop + `Menu` import; keep Row 2 (view pills + member dots) and the member-filter init effect (`:54-66`). The remaining wrapper keeps `border-b border-border bg-background` with `px-4 py-3`-equivalent spacing so the controls row reads as part of the chrome.

- [ ] **Step 2: Update `calendar-module.tsx:460`** — drop the `onOpenSidebar` argument (and its `openSidebar` plumbing if now unused in that file).

- [ ] **Step 3: Update `App.tsx:170`** — replace the gated render with an unconditional `<AppHeader />`.

- [ ] **Step 4: Update affected tests** — `mobile-toolbar` tests asserting the context label/Today/Menu move to the `app-header` suite (done in Task 2); `App.shell.test.tsx` expectations about header presence on mobile calendar flip to "always present".

  ```bash
  cd frontend
  npm run test -- --run src/components/calendar src/App.shell.test.tsx src/components/shared
  ```

  Expected: PASS after updates.

- [ ] **Step 5: Visual smoke** — 390×844: Calendar shows header ("June 2026" + Today + Menu) over one controls row, total chrome no taller than before; switching D/W/M/S updates the header label; Menu opens the sidebar from Calendar and from Chores.

- [ ] **Step 6: Commit**

  ```bash
  git add src/components/calendar/components/mobile-toolbar.tsx src/components/calendar/calendar-module.tsx src/App.tsx
  git commit -m "feat(mobile): render shared header on calendar; slim mobile toolbar to controls row"
  ```

---

### Task 4: Drop redundant module titles on mobile

- [ ] **Step 1: Chores** — `chores-view.tsx:117-119`: hide the `<h1>` on mobile. The file already has `isMobile`; simplest is `{!isMobile && <h1 …>Chores</h1>}` (the surrounding row keeps the scope switcher + add button; verify spacing without the h1).

- [ ] **Step 2: Recipes** — `recipes-view.tsx:150-153`: wrap the `<h1>` + subtitle `<p>` in `hidden md:block` (CSS gate; file doesn't import `useIsMobile`). Keep the "Add recipe" button row aligned right on mobile.

- [ ] **Step 3: Lists** — `lists-view.tsx:47-49` and `:89-91`: same `hidden md:block` treatment on both "My Lists" headings.

- [ ] **Step 4: Verify + smoke**

  ```bash
  cd frontend
  npm run test -- --run src/components/lists src/components/chores src/components/recipes-view.test.tsx
  npm run lint
  ```

  390×844 smoke: each module shows its name once (in the header); desktop width shows in-content titles as before.

- [ ] **Step 5: Commit**

  ```bash
  git add src/components/chores-view.tsx src/components/recipes-view.tsx src/components/lists-view.tsx
  git commit -m "fix(mobile): remove module titles duplicated by the module-aware header"
  ```

---

### Task 5: E2E + verification + PR

- [ ] **Step 1: Sweep E2E specs** — mobile specs that opened the sidebar via the calendar toolbar Menu or asserted toolbar Row-1 text need updating to the header locations (`getByRole("button", { name: /^menu$/i })` still matches — position changed, name didn't). Run the mobile project and fix fallout.

- [ ] **Step 2: Full gate**

  ```bash
  cd frontend
  npm run lint
  npm run test -- --run
  npm run test:e2e
  ```

  Expected: all green.

- [ ] **Step 3: Open the PR**

  ```bash
  git push -u origin feat/mobile-module-aware-header
  gh pr create --fill
  ```

  PR body includes `Closes #<N>` and a checklist mapping each acceptance criterion to its commit.

---

## Self-Review

**Spec coverage:** D1+D2 → Task 2; D3 → Tasks 2 (Today in header) + 3 (toolbar slimming); D4 → Task 4; D5 → Task 3 Step 3. Acceptance criteria covered across Tasks 2–5.

**Open verification risks:**
- The "chrome not taller than today" criterion is visual — explicit smoke step in Task 3.
- `seedCalendarStore` signature and the `ModuleType` union must be confirmed against `@/test/test-utils` / `@/stores` before writing Task 2 tests.
- Moving Menu to the right on non-calendar modules may surprise an E2E spec that asserted button order — name-based locators should hold, but Task 5 Step 1 explicitly sweeps for it.
