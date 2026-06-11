# Family Hub — Roadmap

Last updated: 2026-06-11

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
- [Visual identity refinement](backlog/mobile-ux/visual-identity-refinement.md) · FE #155, PR #156
- [Sidebar + settings + onboarding mobile pass](backlog/mobile-ux/sidebar-settings-onboarding-mobile.md) · FE #193, PR #194 · `0.3.12` (verified Galaxy S10/Chrome)
- [Expandable bottom sheet pattern](backlog/mobile-ux/expandable-bottom-sheet.md) · FE PR #198 · `0.3.12` (verified Galaxy S10/Chrome)

### Module foundations

- [Chores recurring routines](backlog/module-foundations/chores-recurring-routines.md) · FE #173, PR #175, BE #47, PR #48; current shipped chores contract
- [Chores core loop](backlog/module-foundations/chores-core-loop.md) · FE #158, PR #159, BE #41, historical shipped one-off release
- [Lists simple shared checklists](backlog/module-foundations/lists-simple-shared-checklists.md) · FE #163, PR #164, BE #44, PR #45

## Active epics

### Module foundations

Current story:
- [Lists / Chores / Meals / Photos module foundations](backlog/module-foundations/module-surface-foundations.md) — Recipes/Meals implementation merged; FE release/deploy pending; Photos deferred

Most recent shipped story:
- [Chores recurring routines](backlog/module-foundations/chores-recurring-routines.md) — replaces the initial one-off chores release with recurring day / week / month routines

Active execution:
- `Recipes` frontend foundation — FE #183 closed by PR #185; consolidated fixes in PR #191
- `Meals` frontend foundation — FE #184 closed by PR #187; consolidated fixes in PR #191
- FE release PR #186 for `family-hub` `v0.3.11` is pending; production deploy should follow the released FE commit

Next recommended story after FE release/deploy:
- Mobile-first product polish for the current family organizer surfaces: `Home`, `Calendar`, `Chores`, `Lists`, `Recipes`, and `Meals`

Exit criterion for this phase:
- Make the current organizer surfaces reliable and polished enough for daily phone use by Joe and Partner, while preserving the larger-screen/tablet product direction for later hardware.

Recommended order:
1. `Recipes` — reusable household recipe library and `Add to Meals` handoff; implementation merged, FE release/deploy pending
2. `Meals` — simple week-ahead planning; implementation merged, FE release/deploy pending
3. Mobile-first polish of current organizer workflows
4. `Photos` — deferred; no longer the next product slice

## Planned epics

### Mobile UX polish backlog

Next practical product focus after Recipes/Meals release and deploy. The app is still intended for a tablet/bigger-screen home hub, but near-term daily use will be on phones until dedicated touchscreen hardware exists.

Already captured polish stories:
- [Mobile module content polish](backlog/mobile-ux/mobile-module-content-polish.md) — per-module layout/density fixes found in 0.3.12 device testing (list filters, chores redundant labels, recipe card density, shared nav-header inconsistency, settings-dialog cutoff). Pre-existing; not introduced by the shell stories.
- [Notifications (event reminders)](backlog/mobile-ux/notifications.md)
- [Drag-to-create event on time grid](backlog/mobile-ux/drag-to-create.md)
- [Pinch-to-zoom calendar views](backlog/mobile-ux/pinch-to-zoom.md)

Recommended additions to shape before implementation:
- Home dashboard should evolve from calendar-only into a true organizer summary now that `Chores`, `Lists`, `Recipes`, and `Meals` have real data.
- Decide whether the `Photos` tab should be hidden, demoted, or left as an explicit deferred surface while the product is polished for daily use.
- Run a phone-first production dogfood pass across create/edit/complete flows before adding new modules.

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

### Module surfaces

- `Photos` — deferred while the family uses and polishes the current organizer surfaces.

### Lower-priority PRD items

Deferred post-MVP per PRD v2.0 realignment. Items can move back into planned work as dedicated backlog stories are created.
