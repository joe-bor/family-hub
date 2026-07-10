# Family Hub — Roadmap

Last updated: 2026-07-06

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
- [Polished PWA installability + honest offline UX](backlog/mobile-ux/pwa-installability.md) · FE #216/#217/#218 · `0.3.15` (deployed to prod; on-device smoke pending)
- Option C — read-only offline data persistence · FE #222 (TanStack Query read cache persisted to IndexedDB; cached calendar/chores/lists/meals/recipes/family viewable offline, read-only; per-account clearing on logout/401; no Workbox `/api` cache). Offline *writes* remain deferred (PRD §7.5.3).
- [Native-feel interaction polish (transitions + press feedback)](backlog/mobile-ux/native-feel-interaction-polish.md) · FE #229, PR #230 · `0.3.17`
- [Native hardware back-button handling](backlog/mobile-ux/native-back-button.md) · FE #231, PR #234 · `0.3.18`
- [Optional haptic feedback (Vibration API)](backlog/mobile-ux/optional-haptics.md) · FE #235, PR #236 · `0.3.18` (bottom-sheet-snap + destructive-confirm pulses deferred post-v1)
- [Home organizer summary (state line + "since you last opened" feed)](backlog/mobile-ux/home-organizer-summary.md) · FE #239, PR #252 · `0.3.19`
- Shared floating "+" add button standardized across module create flows (FAB stays viewport-anchored through module-switch fades) · FE #256, #258 · `0.3.19`
- Quick-capture friction fixes — Lists multi-add keep-open sheet, 5-minute calendar time picker with duration preservation, and event Location input · FE PR #276 · `0.3.21`

### Module foundations

- [Chores recurring routines](backlog/module-foundations/chores-recurring-routines.md) · FE #173, PR #175, BE #47, PR #48; current shipped chores contract
- [Chores core loop](backlog/module-foundations/chores-core-loop.md) · FE #158, PR #159, BE #41, historical shipped one-off release
- [Lists simple shared checklists](backlog/module-foundations/lists-simple-shared-checklists.md) · FE #163, PR #164, BE #44, PR #45
- [Lists family-managed categories](backlog/module-foundations/lists-family-managed-categories.md) · BE #60, PR #63, `v1.7.0`; FE #261, PR #268, `0.3.20`
- [Focused meal planning sessions](backlog/module-foundations/focused-meal-planning-sessions.md) · BE #64, PR #65, `v1.8.0`; FE #262, PR #273, `0.3.21`
- [Meals recipe ingredients to grocery list](backlog/module-foundations/meals-recipe-ingredients-to-grocery-list.md) · BE #68, PR #69, `v1.9.0`; FE #277, PR #279, `0.3.22`

## Active epics

### Module foundations

Current story:
- [Lists / Chores / Meals / Photos module foundations](backlog/module-foundations/module-surface-foundations.md) — Recipes/Meals implementation merged and released in FE `0.3.11`; Photos deferred

Most recent shipped story:
- [Meals recipe ingredients to grocery list](backlog/module-foundations/meals-recipe-ingredients-to-grocery-list.md) — reviewed recipe ingredient rows from the visible Meals week append to a chosen grocery list through the released generic bulk Lists endpoint

Latest release:
- FE release `family-hub-v0.3.22` published on 2026-07-06, including the Meals ingredients-to-grocery-list flow over the released BE `v1.9.0` bulk list-item append endpoint.

Next recommended product focus:
- Run a phone-first production dogfood pass across the newly shipped Meals planning, ingredients-to-grocery-list flow, Lists multi-add, calendar time/location capture, and existing create/edit/complete flows. After that, pick the next Mobile UX story: Notifications for reminder value, or calendar gestures (drag-to-create / pinch-to-zoom).

Exit criterion for this phase:
- Make the current organizer surfaces reliable and polished enough for daily phone use by Joe and Partner, while preserving the larger-screen/tablet product direction for later hardware.

Recommended order:
1. `Recipes` — reusable household recipe library and `Add to Meals` handoff; shipped in FE `0.3.11`
2. `Meals` — simple week-ahead planning; shipped in FE `0.3.11`
3. Mobile-first polish of current organizer workflows — shipped through FE `0.3.14`
4. Remaining Mobile UX stories — native-feel polish (transitions/press feedback, back-button, haptics) shipped in `0.3.17`–`0.3.18`, Home organizer summary shipped in `0.3.19`, and quick-capture friction fixes shipped in `0.3.21`; remaining picks are Notifications (reminder value) or calendar gestures (drag-to-create / pinch-to-zoom)
5. `Photos` — deferred; no longer the next product slice

## Planned epics

### Large-screen / tablet UX

Shipped:
- [Large-screen shell foundations](backlog/large-screen-ux/large-screen-foundations.md) — slim header (drop fake weather), one merged toolbar row, chrome budget; FE PR #282 merged 2026-07-06. [spec](../superpowers/specs/2026-07-06-large-screen-foundations-design.md) · [plan](../superpowers/plans/2026-07-06-large-screen-foundations.md)
- [Large-screen Home hub](backlog/large-screen-ux/large-screen-home.md) — ambient A1 now-first Home (hero + Chores/Meals/Lists state strip + Today rail), cross-module routing intents, 10-min idle return; FE PR #284 merged 2026-07-07 (closes #278). [spec](../superpowers/specs/2026-07-05-large-screen-home-design.md) · [plan](../superpowers/plans/2026-07-05-large-screen-home.md)

In progress:
- [Large-screen Calendar](backlog/large-screen-ux/large-screen-calendar.md) — Week zoom-out (one-line headers, denser hours) and Day member lanes with an optional mini-month rail; FE PR #285 opened 2026-07-10. [spec](../superpowers/specs/2026-07-06-large-screen-calendar-design.md) · [plan](../superpowers/plans/2026-07-06-large-screen-calendar.md)

Next up (spec written, plan not yet written):
- [Large-screen Lists](backlog/large-screen-ux/large-screen-lists.md) — two-pane master/detail so the kitchen tablet can keep a list open. [spec](../superpowers/specs/2026-07-06-large-screen-lists-design.md)
- [Large-screen Meals](backlog/large-screen-ux/large-screen-meals.md) — full-width week board, all 7 days visible, today highlighted. [spec](../superpowers/specs/2026-07-06-large-screen-meals-design.md)
- [Large-screen Chores](backlog/large-screen-ux/large-screen-chores.md) — board polish: Today weighted primary, bigger checkoff targets, member identity forward. [spec](../superpowers/specs/2026-07-06-large-screen-chores-design.md)
- [Large-screen Recipes](backlog/large-screen-ux/large-screen-recipes.md) — responsive 2-4 column card grid with capped images. [spec](../superpowers/specs/2026-07-06-large-screen-recipes-design.md)

### Module foundations follow-on

`Photos` remains deferred; use the production dogfood pass to decide whether the next slice should be more module depth or mobile workflow polish.

Recent shipped follow-on:
- [Meals recipe ingredients to grocery list](backlog/module-foundations/meals-recipe-ingredients-to-grocery-list.md) shipped in BE `v1.9.0` and FE `0.3.22`: add reviewed recipe ingredients from the visible Meals week to a chosen grocery list, via a generic BE bulk list-item append endpoint. [spec](../superpowers/specs/2026-07-04-meals-recipe-ingredients-to-grocery-list.md) · [plan](../superpowers/plans/2026-07-04-meals-recipe-ingredients-to-grocery-list.md) · BE #68 → PR #69; FE #277 → PR #279.

### Mobile UX polish backlog

Next practical product focus after the mobile shell/preferences cleanup. The app is still intended for a tablet/bigger-screen home hub, but near-term daily use will be on phones until dedicated touchscreen hardware exists.

Next captured stories — calendar gestures + reminders:
- [Notifications (event reminders)](backlog/mobile-ux/notifications.md) — decide web push vs in-app reminders, then make event reminders configurable and testable.
- [Drag-to-create event on time grid](backlog/mobile-ux/drag-to-create.md) — let touch/mouse users create a calendar event by dragging an empty time range, without breaking event tap/edit.
- [Pinch-to-zoom calendar views](backlog/mobile-ux/pinch-to-zoom.md) — let users adjust calendar time granularity with pinch gestures and persist the chosen zoom level.

Shipped — home organizer (kept here for the future-sibling pointers):
- [Home organizer summary](backlog/mobile-ux/home-organizer-summary.md) shipped in `0.3.19` (FE #252) — see the "Mobile shell + home" shipped list above. The **backend activity log** and the **AI day-summary / overload-warnings** remain explicit future siblings that consume the same client-side `ActivityItem` seam.

Native-feel polish (transitions/press feedback, back-button, haptics) shipped in `0.3.17`–`0.3.18` — see the Shipped list above.

Quick-capture friction fixes shipped in `0.3.21` — Lists multi-add, calendar time-picker duration preservation, and event Location input are no longer open in the mobile polish queue.

Recommended additions to shape before implementation:
- Device-member association is the likely successor to Preferences if member-focused defaults become important; seed captured in [Sidebar structure + family preferences surface](backlog/mobile-ux/sidebar-settings-story.md#new-ideas-surfaced-backlog-seeds-not-in-scope).
- Decide whether the `Photos` tab should be hidden, demoted, or left as an explicit deferred surface while the product is polished for daily use.
- Run a phone-first production dogfood pass across create/edit/complete flows before adding new modules.
- ~~**Option C / offline reads**~~ — **delivered** (FE #222): TanStack Query read cache persisted to IndexedDB so already-fetched data shows offline, read-only. See the "Mobile shell + home" shipped list above. Remaining offline-**writes** work (outbox/queue/background sync, PRD §7.5.3) is still deferred and intentionally out of scope.

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
