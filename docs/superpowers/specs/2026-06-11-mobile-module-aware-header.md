# Module-Aware Mobile Header

## Problem

Finding **F1** from the 0.3.12 Galaxy S10 device pass: the mobile header is inconsistent across modules, and where it is consistent it's redundant.

- Calendar suppresses the shared `AppHeader` on mobile and renders its own `MobileToolbar` (context label, Today, Menu, view pills, member filters) — a one-off gated in `App.tsx`.
- Every other module renders `AppHeader`, which on mobile is just a hamburger + the family name — then renders its **own** in-content title ("Chores", "Recipes", "My Lists") directly below. Two header rows, one of which (the family name) carries no navigation value inside a module.
- The Menu button sits on the **left** in `AppHeader` but on the **right** in Calendar's toolbar.

Decision (made with Joe, 2026-06-11): **generalize Calendar's pattern** — the shared header becomes module-aware. One header row per screen, title reflects where you are, Menu lives in one consistent place, and the `App.tsx` one-off disappears.

## Current FE state (all paths relative to `frontend/`)

- **Gate:** `src/App.tsx:170` — `{!(isMobile && activeModule === "calendar") && <AppHeader />}`.
- **AppHeader:** `src/components/shared/app-header.tsx` — mobile renders Menu (left, `:48-55`) + family name (`:58-60`); date/time (`:62-68`), weather (`:74-82`), and member dots (`:85-95`) are desktop-only.
- **Calendar toolbar:** `src/components/calendar/components/mobile-toolbar.tsx` — Row 1 (`:72-99`): context label (via `getContextLabel`, `:21-40`) left; Today button + Menu button right. Row 2 (`:101-145`): view pills + member filter dots. Also hosts the member-filter initialization effect (`:54-66`).
- **Module in-content titles (mobile-redundant once the header carries the module name):**
  - Chores `<h1>` — `src/components/chores-view.tsx:117-119` (shares a row with the mobile scope switcher and the add button).
  - Recipes `<h1>` + subtitle — `src/components/recipes-view.tsx:150-153` (shares a row with "Add recipe").
  - Lists "My Lists" `<h2>` — `src/components/lists-view.tsx:47-49` and `:89-91` (two render branches).
  - Meals renders a contextual `WeekHeader` (`src/components/meals-view.tsx:119`), not a static title — it stays.
- **Stores:** `AppHeader` already reads `useCalendarStore` (currentDate) and `useAppStore` (openSidebar). `activeModule` lives in `useAppStore`.

## Goals

- Exactly one header row on every mobile screen, with a consistent layout: **title left, contextual actions + Menu right**.
- The title tells the user where they are: family name on Home, module name elsewhere, calendar context label (e.g. "June 2026") on Calendar.
- Calendar keeps its full capability (Today, view pills, member filters) with no extra vertical cost versus today.
- The `App.tsx` calendar gate is removed; no module suppresses the shared header.
- Desktop header is unchanged.

## Non-goals

- No desktop/tablet header redesign (family name + date/weather/dots layout stays).
- No route-based navigation, no per-module action registration framework — the header's module awareness is a simple internal map, not a plugin system.
- No moving module action buttons (Chores `+`, Recipes "Add recipe") into the header.
- Meals `WeekHeader` is untouched.

## Decision Summary

### D1. Mobile header layout: title left, actions + Menu right

On mobile, `AppHeader` adopts Calendar's row-1 composition: title (left), optional module actions, Menu button (right, 44px target). This moves the hamburger from left to right on non-calendar modules — accepted, consistency wins and it matches the already-shipped Calendar muscle memory. Desktop layout is untouched (Menu left, family name + date/time, weather + dots right).

### D2. Title comes from a module title map inside `AppHeader`

`AppHeader` derives the mobile title from `activeModule`:

| `activeModule` | Title |
|---|---|
| `null` (Home) | family name |
| `calendar` | context label (`getContextLabel(calendarView, currentDate)`) |
| `lists` | "Lists" |
| `chores` | "Chores" |
| `meals` | "Meals" |
| `recipes` | "Recipes" |
| `photos` (unreachable on mobile) | "Photos" |

`getContextLabel` moves from `mobile-toolbar.tsx` to a shared calendar util so both the header and any future consumer use one implementation. `AppHeader` already subscribes to the calendar store; it additionally reads `calendarView`. Reading calendar state for the calendar title is acceptable coupling — the header is shell chrome and already store-aware.

### D3. Calendar actions: Today renders in the header; pills + filters stay in the calendar

When `activeModule === "calendar"` (mobile), the header renders the Today button (same enabled/dimmed behavior driven by `useIsViewingToday`) between the title and Menu. `MobileToolbar` loses Row 1 entirely and keeps Row 2 (view pills + member filter dots) plus the member-filter initialization effect — it becomes a single-row controls bar owned by the calendar module. Net vertical cost on Calendar: unchanged (two rows today → header + controls row).

### D4. Modules drop their redundant mobile titles

- Chores: hide the `<h1>` on mobile (`hidden md:block` or `isMobile` gate — match file idiom; the row keeps the scope switcher + add button).
- Recipes: hide the `<h1>` + subtitle on mobile; keep the "Add recipe" button row.
- Lists: hide "My Lists" in both render branches on mobile.
- Desktop keeps all in-content titles (desktop header still shows the family name, so module titles remain meaningful there).

### D5. The `App.tsx` gate goes away

`App.tsx:170` becomes an unconditional `<AppHeader />`. No module-specific knowledge remains in the shell markup.

## Visual / Layout Contract

- **Mobile header (all modules):** `min-h-16 px-4 py-3`, single row: title `text-[22px] leading-7 font-semibold` (truncates, `min-w-0`) left; right cluster `gap-2`: \[Calendar only: Today button, current styling from `mobile-toolbar.tsx:78-89`\] + Menu icon button (`h-11 w-11`, `aria-label="Menu"`).
- **Calendar below the header:** one controls row (view pills left, member dots right), `px-4 pb-3` — visually identical to today's Row 2.
- **Home:** header shows the family name — unchanged appearance from today.
- **Desktop (≥768px):** pixel-identical to current header; module pages keep their in-content titles.

## Interaction Model

- Menu (any module) → opens sidebar. Today (calendar) → `goToToday()`, dimmed when already viewing today. No other header interactions.
- Switching modules updates the title in place; no animation requirements.

## Out of Scope

- Backend, routing, desktop header, Meals `WeekHeader`, module action buttons, notifications/weather widgets.
- F2/F3/F4/F5 changes (separate stories).

## Acceptance Criteria

- [ ] On mobile, every module (Home, Calendar, Lists, Chores, Meals, Recipes) shows exactly one header row, with the Menu button on the right; the sidebar opens from it on every module.
- [ ] Header title: family name on Home; "Lists"/"Chores"/"Meals"/"Recipes" on those modules; live context label on Calendar (changes with view/date navigation).
- [ ] Calendar: Today button works from the header (dimmed when on today); view pills and member filter dots still work in the calendar controls row; member filter initialization still runs.
- [ ] No module renders a duplicate static title under the mobile header (Chores h1, Recipes h1+subtitle, Lists "My Lists" hidden on mobile; Meals `WeekHeader` retained).
- [ ] `App.tsx` renders `<AppHeader />` unconditionally — no calendar-specific gate.
- [ ] Calendar's total chrome height (header + controls row) is not taller than today's two-row toolbar.
- [ ] Desktop header and desktop module titles are unchanged.
- [ ] All existing unit + E2E suites pass (updated where they assert the old header/toolbar structure).
