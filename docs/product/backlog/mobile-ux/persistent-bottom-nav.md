---
id: mobile-persistent-bottom-nav
title: Persistent bottom navigation
epic: mobile-ux
status: planned
priority: P2
created: 2026-04-23
updated: 2026-04-23
issues: []
prs: []
---

## Context

Currently users must return to the Home dashboard to switch modules. A persistent bottom nav (Calendar, Chores, Meals, Lists, Photos) would feel more app-like and reduce friction. (From `docs/mobile-ux-polish-backlog.md` item 2.)

## Acceptance Criteria

- [ ] Bottom nav visible on all module screens at mobile breakpoint
- [ ] Active module indicated
- [ ] Tapping a nav item navigates without full-page reload (SPA routing)
- [ ] Does not compete with module-level headers (layout decision documented)

## Notes

Coordinate with home dashboard redesign; they share mobile shell.
