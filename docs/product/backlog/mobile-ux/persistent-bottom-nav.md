---
id: mobile-persistent-bottom-nav
title: Persistent bottom navigation
epic: mobile-ux
status: done
priority: P2
created: 2026-04-23
updated: 2026-05-17
issues:
  - FE #148
prs:
  - FE PR #152
---

## Context

This shipped story replaced the old "return to Home to switch modules" mobile pattern with a persistent bottom nav that makes Home and the five module surfaces reachable in one tap without introducing route-based navigation or changing the desktop shell.

## Acceptance Criteria

- [x] On authenticated mobile surfaces, a bottom nav is visible with 6 tabs in this order: Home, Calendar, Lists, Chores, Meals, Photos.
- [x] Tapping a tab switches surfaces through the existing `activeModule` store without a full page reload.
- [x] `Home` maps to `activeModule === null` and remains reachable in one tap from every mobile module.
- [x] The active tab is visually distinct and matches the existing desktop `NavigationTabs` active-state language.
- [x] `AppHeader` remains visible on mobile Home, Lists, Chores, Meals, and Photos.
- [x] `AppHeader` remains suppressed on mobile Calendar, which keeps `MobileToolbar`.
- [x] The mobile Home button is removed from `AppHeader`, and the Home button is removed from `MobileToolbar`.
- [x] The nav is not rendered on desktop, login, or onboarding/setup screens.
- [x] Scrollable content is not obscured by the nav, and the calendar FAB clears it on mobile.

## Notes

- This story is the mobile-shell prerequisite for the Home dashboard redesign.
- It ships before the Home dashboard redesign, but it must not create a temporary Home/Menu regression.
- Home keeps its current `AppHeader` in this story; any future `DashboardHeader` replacement belongs to the later Home-dashboard work.
