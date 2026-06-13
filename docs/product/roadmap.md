# Family Hub — Roadmap

Last updated: 2026-06-13

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
- [Mobile module content polish](backlog/mobile-ux/mobile-module-content-polish.md) · FE #199–#202, PR #203/#205/#206/#207 · `0.3.13`
- [Sidebar structure + family preferences surface](backlog/mobile-ux/sidebar-settings-story.md) · BE #57, PR #58, `v1.6.0`; FE #208, PR #210, `0.3.14`

### Module foundations

- [Chores recurring routines](backlog/module-foundations/chores-recurring-routines.md) · FE #173, PR #175, BE #47, PR #48; current shipped chores contract
- [Chores core loop](backlog/module-foundations/chores-core-loop.md) · FE #158, PR #159, BE #41, historical shipped one-off release
- [Lists simple shared checklists](backlog/module-foundations/lists-simple-shared-checklists.md) · FE #163, PR #164, BE #44, PR #45

## Active epics

### Module foundations

Current story:
- [Lists / Chores / Meals / Photos module foundations](backlog/module-foundations/module-surface-foundations.md) — Recipes/Meals implementation merged and released in FE `0.3.11`; Photos deferred

Most recent shipped story:
- [Chores recurring routines](backlog/module-foundations/chores-recurring-routines.md) — replaces the initial one-off chores release with recurring day / week / month routines

Active execution:
- `Recipes` frontend foundation — FE #183 closed by PR #185; consolidated fixes in PR #191
- `Meals` frontend foundation — FE #184 closed by PR #187; consolidated fixes in PR #191
- FE release PR #186 for `family-hub` `v0.3.11` merged 2026-06-09

Next recommended product focus:
- Continue from the Mobile UX queue below. The module surfaces are now functional enough that the next work should either add reminders or shape the Home organizer summary, rather than start a new module.

Exit criterion for this phase:
- Make the current organizer surfaces reliable and polished enough for daily phone use by Joe and Partner, while preserving the larger-screen/tablet product direction for later hardware.

Recommended order:
1. `Recipes` — reusable household recipe library and `Add to Meals` handoff; shipped in FE `0.3.11`
2. `Meals` — simple week-ahead planning; shipped in FE `0.3.11`
3. Mobile-first polish of current organizer workflows — shipped through FE `0.3.14`
4. Remaining Mobile UX stories — Notifications first if reminder value is the priority; calendar gestures if creation/view ergonomics are the priority
5. `Photos` — deferred; no longer the next product slice

## Planned epics

### Mobile UX polish backlog

Next practical product focus after the mobile shell/preferences cleanup. The app is still intended for a tablet/bigger-screen home hub, but near-term daily use will be on phones until dedicated touchscreen hardware exists.

Next captured stories:
- [Polished PWA installability + honest offline UX](backlog/mobile-ux/pwa-installability.md) — **merged 2026-06-13, release `0.3.15` pending**: controlled update prompt, sidebar install row, honest offline banner, and PWA config cleanups (UI + config only; offline *data* is the separate Option C). Supersedes the loose sidebar install-row seed. FE PRs merged (rebase, order c→b→a): (c) #215→[#216](https://github.com/joe-bor/FamilyHub/pull/216) · (b) #214→[#217](https://github.com/joe-bor/FamilyHub/pull/217) · (a) #213→[#218](https://github.com/joe-bor/FamilyHub/pull/218); release-please [#219](https://github.com/joe-bor/FamilyHub/pull/219).
- [Notifications (event reminders)](backlog/mobile-ux/notifications.md) — decide web push vs in-app reminders, then make event reminders configurable and testable.
- [Drag-to-create event on time grid](backlog/mobile-ux/drag-to-create.md) — let touch/mouse users create a calendar event by dragging an empty time range, without breaking event tap/edit.
- [Pinch-to-zoom calendar views](backlog/mobile-ux/pinch-to-zoom.md) — let users adjust calendar time granularity with pinch gestures and persist the chosen zoom level.

Recommended additions to shape before implementation:
- Home dashboard should evolve from calendar-only into a true organizer summary now that `Chores`, `Lists`, `Recipes`, and `Meals` have real data. Path: create a new story under `docs/product/backlog/mobile-ux/`.
- Device-member association is the likely successor to Preferences if member-focused defaults become important; seed captured in [Sidebar structure + family preferences surface](backlog/mobile-ux/sidebar-settings-story.md#new-ideas-surfaced-backlog-seeds-not-in-scope).
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
