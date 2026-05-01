# Family Hub — Roadmap

Last updated: 2026-05-01

Use this document as a summary/index. Story status lives in `docs/product/backlog/<epic>/<story>.md`. GitHub Project **Family Hub** is the live task board for issue-level work.

## Shipped

### Foundation

- Family registration + JWT auth — FE + BE (retrofit)
- Docker + GHCR + prod deploy — [deployment-guide](../deployment-guide.md) · BE #8, #9
- E2E against real backend — FE #105
- BE versioned releases (release-please) — [spec](../superpowers/specs/2026-03-19-be-versioned-releases-design.md) · BE (multiple)

### Calendar core

- CRUD calendar events — [API reference](../calendar-events-api-reference.md) · FE #81–#93, BE #8–#9
- All-day event rendering — [all-day + multi-day design](../all-day-and-multi-day-events-design.md) · FE #110
- Multi-day events (endDate) — same · BE #18, #19, FE #111
- Recurring events (RRULE + expansion) — [recurring events design](../recurring-events-design.md) · BE #20, #21, FE #117

### Mobile shell + home

- [Persistent bottom navigation](backlog/mobile-ux/persistent-bottom-nav.md) · FE #148, PR #152
- [Home dashboard redesign](backlog/mobile-ux/home-dashboard-redesign.md) · FE #149, PR #154

## Active epics

### Mobile UX polish

Current story:
- [Visual identity refinement](backlog/mobile-ux/visual-identity-refinement.md)

Remaining stories in this epic:
- [Expandable bottom sheet pattern](backlog/mobile-ux/expandable-bottom-sheet.md)
- [Sidebar + settings + onboarding mobile pass](backlog/mobile-ux/sidebar-settings-onboarding-mobile.md)
- [Notifications (event reminders)](backlog/mobile-ux/notifications.md)
- [Drag-to-create event on time grid](backlog/mobile-ux/drag-to-create.md)
- [Pinch-to-zoom calendar views](backlog/mobile-ux/pinch-to-zoom.md)

## Planned epics

### Module foundations

- [Lists / Chores / Meals / Photos module foundations](backlog/module-foundations/module-surface-foundations.md)

### Google Calendar read-only sync

Completed in this epic:
- OAuth flow — [Google Cal integration design](../google-calendar-integration-design.md) · BE #22
- Calendar selection — same · BE #23
- Full sync — same · BE #24, #25

Remaining story:
- [Incremental sync + scheduler](backlog/google-calendar/incremental-sync.md) — postponed for now while FE focus stays on mobile UX + module foundations

### Google Calendar write-back

- [Write-back (create/edit/delete to Google)](backlog/google-calendar/write-back.md) — [Google Cal integration design](../google-calendar-integration-design.md)

## Deferred

### Lower-priority PRD items

Deferred post-MVP per PRD v2.0 realignment. Items can move back into planned work as dedicated backlog stories are created.
