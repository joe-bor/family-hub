# Persistent Bottom Navigation

## Problem

Family Hub still behaves like a prototype on mobile at the shell level. The current mobile Home surface is a 2-column launcher grid, and there is no persistent module-switching surface on phones. To change modules, a user must go back through Home and tap a launcher tile. That friction is acceptable for a prototype, but it blocks the next stage of the product:

- the mobile Home redesign wants to stop being a launcher and become a real "what matters now?" surface
- the app needs a stable mobile navigation model before individual modules can receive more opinionated mobile polish

The missing prerequisite is a persistent mobile bottom nav.

## Current FE shell

The current frontend shell has a few important constraints:

- `frontend/src/App.tsx` has no router. Module switching is state-only through `useAppStore`, where `activeModule: ModuleType | null`.
- `null` already means "mobile Home" in the current shell.
- `frontend/src/components/shared/navigation-tabs.tsx` is desktop-only (`hidden sm:flex`).
- `frontend/src/components/shared/app-header.tsx` renders on every authenticated surface except mobile Calendar.
- mobile Calendar uses its own `MobileToolbar`, which currently includes both `Home` and `Menu`.
- `frontend/src/components/home/home-dashboard.tsx` is still the old 2-column launcher grid.

That means the bottom-nav story should solve exactly one problem: introduce first-class mobile module switching without forcing a routing rewrite or a desktop refactor.

## Goals

- Make primary surfaces reachable in one tap on mobile.
- Make `Home` a first-class mobile destination, not a hidden backdoor.
- Keep the current state-based shell architecture.
- Avoid a temporary regression in sidebar access while the old Home launcher still exists.
- Leave desktop behavior unchanged.

## Non-goals

- Introducing React Router or URL-based active state
- Reworking the desktop side rail
- Shipping the new Home dashboard in this story
- Adding tab badges, long-press actions, hide-on-scroll, or keyboard-aware nav behavior
- Solving tablet/touchscreen shell behavior

## Decision Summary

### D1. State-based navigation only

`MobileBottomNav` will switch surfaces by calling `useAppStore.setActiveModule()`. There is no router and no route transition work in this story.

### D2. Home is a real tab

The mobile bottom nav has 6 tabs, in this order:

1. `Home`
2. `Calendar`
3. `Lists`
4. `Chores`
5. `Meals`
6. `Photos`

`Home` maps to `setActiveModule(null)`.

### D3. Mobile-only nav; desktop untouched

The existing desktop `NavigationTabs` remains the desktop navigation model. This story adds a separate mobile-only `MobileBottomNav`.

### D4. Keep the current Home header for now

This story must not create a temporary "Home has no Menu button" regression.

So the mobile chrome rule for this story is:

- `Home` (`activeModule === null`): keep `AppHeader`
- `Calendar`: keep `MobileToolbar`, no `AppHeader`
- `Lists`, `Chores`, `Meals`, `Photos`: keep `AppHeader`

The later Home-dashboard redesign may intentionally suppress `AppHeader` on Home and replace it with a dedicated `DashboardHeader`, but that change does not belong in this story.

### D5. Remove redundant Home buttons once nav exists

Once the bottom nav ships, users should not see multiple "go home" controls on mobile.

This story removes:

- the mobile Home button from `AppHeader`
- the Home button from `MobileToolbar`

The bottom nav becomes the primary Home affordance.

### D6. The nav is a shell bottom rail, not an overlay

The bottom nav should render as a bottom rail inside the root app shell, not as a fixed overlay pinned on top of content.

That gives two advantages:

- module scroll containers naturally size above the nav instead of being hidden underneath it
- the shell stays simpler because the nav uses layout, not patchwork bottom padding across multiple modules

## Visual and Layout Contract

### Placement

`MobileBottomNav` renders as the final child of the root authenticated shell in `App.tsx`, after the main content area and before `SidebarMenu`.

Conceptually:

```tsx
<div className="h-screen flex flex-col bg-background">
  {!(isMobile && activeModule === "calendar") && <AppHeader />}

  <div className="flex-1 flex min-h-0 overflow-hidden">
    <NavigationTabs />
    <main className="flex-1 min-h-0 flex flex-col overflow-hidden">
      {renderModule(activeModule)}
    </main>
  </div>

  {isMobile && isAuthenticated && setupComplete && <MobileBottomNav />}
  <SidebarMenu />
</div>
```

The critical additions are the `min-h-0` constraints and rendering the nav as a bottom rail sibling rather than a fixed overlay.

### Height and safe area

The nav should use:

- approximately `56px` for the visible tab row
- bottom padding that includes `env(safe-area-inset-bottom)` so the iOS home indicator does not collide with the nav

The safe-area inset belongs inside the nav itself, not as global body padding.

### Layering

- bottom nav: `z-30`
- sidebar backdrop: `z-40`
- calendar FAB: `z-40`
- modals/sheets: `z-50`

This means:

- the sidebar backdrop covers the nav when the drawer is open
- dialogs and sheets cover the nav
- the calendar FAB can still float above the nav

### Calendar FAB clearance

Because the nav becomes a persistent rail on mobile, the calendar FAB must sit above it. The current FAB offset is too low once the nav exists.

On mobile nav surfaces, the FAB should move to a bottom offset equivalent to:

```css
bottom: max(4.5rem, calc(env(safe-area-inset-bottom) + 4.5rem));
```

Desktop keeps the existing offset.

## Interaction Model

- tapping a tab calls `setActiveModule(...)`
- tapping `Home` calls `setActiveModule(null)`
- the active tab uses the same visual language as desktop `NavigationTabs`
- each tab shows icon + short label
- repeat-tapping the active tab is a no-op in MVP
- there is no swipe-to-change-module behavior in this story

## App-Shell Integration

### `App.tsx`

This story changes the shell in three ways:

1. import and render `MobileBottomNav`
2. add `min-h-0` to the content wrappers so internal scrolling still works with the bottom rail present
3. keep the existing `AppHeader` suppression rule for Calendar only

The important non-change is that this story does **not** suppress `AppHeader` on Home yet.

### `AppHeader`

`AppHeader` keeps:

- Menu button
- family name
- desktop-only date/time and member indicators

`AppHeader` loses:

- the mobile Home button

### `MobileToolbar`

`MobileToolbar` keeps:

- Today button
- Menu button
- view switcher
- member filter dots

`MobileToolbar` loses:

- the Home button

## Coordination With Home Dashboard Redesign

Bottom nav must ship before the Home-dashboard redesign, but that does not mean this story should take ownership of the future Home chrome.

The intended sequence is:

1. ship mobile bottom nav while the old launcher-grid Home still exists
2. keep `AppHeader` on Home during that interim state so Menu access remains intact
3. later, when the Home-dashboard redesign lands, that story may replace Home's `AppHeader` with a dedicated `DashboardHeader` in the same PR

This keeps the dependency real without introducing a temporary regression.

## Out of Scope

- desktop side-rail refactor
- route-based navigation
- tab badges or unread counts
- long-press or repeat-tap shortcuts
- hide-on-scroll nav behavior
- keyboard-aware `visualViewport` handling
- tablet/touchscreen shell treatment
- the Home-dashboard visual redesign itself

## Acceptance Criteria

- [ ] On authenticated mobile surfaces, a bottom nav is visible with 6 tabs in this order: Home, Calendar, Lists, Chores, Meals, Photos.
- [ ] Tapping a tab switches surfaces through the existing `activeModule` store without a full page reload.
- [ ] `Home` maps to `activeModule === null` and remains reachable in one tap from every mobile module.
- [ ] The active tab is visually distinct and uses the same active-state language as desktop `NavigationTabs`.
- [ ] `AppHeader` remains visible on mobile Home, Lists, Chores, Meals, and Photos.
- [ ] `AppHeader` remains suppressed on mobile Calendar, which still uses `MobileToolbar`.
- [ ] The mobile Home button is removed from `AppHeader`, and the Home button is removed from `MobileToolbar`.
- [ ] The nav is not rendered on desktop, login, or onboarding/setup screens.
- [ ] Scrollable module content is not obscured by the nav.
- [ ] On mobile Calendar, the Add Event FAB clears the nav and remains tappable.
- [ ] Sidebar backdrop and modal/sheet layers cover the nav correctly.
