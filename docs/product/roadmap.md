# Family Hub — Roadmap

Last updated: 2026-07-21 (Large-screen / tablet UX shipped in FE `0.3.23`;
FE #293 Month/Schedule follow-on still open and not started; two dialog
viewport fixes merged to FE `main` and awaiting release `0.3.24`)

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

### Large-screen / tablet UX

Released as FE `0.3.23` and deployed to production on 2026-07-17. All seven
planned surfaces are complete, through FE PR #291, with no open large-screen
implementation Issues remaining.

A follow-up story has since been shaped for the two Calendar views this epic
deliberately left at chrome-only: see [Large-screen Calendar Month and
Schedule](backlog/large-screen-ux/large-screen-calendar-month-schedule.md)
under Planned epics. It does not reopen this epic.

Delivered implementation:
- [Large-screen shell foundations](backlog/large-screen-ux/large-screen-foundations.md) — slim header (drop fake weather), one merged toolbar row, chrome budget; FE PR #282 merged 2026-07-06. [spec](../superpowers/specs/2026-07-06-large-screen-foundations-design.md) · [plan](../superpowers/plans/2026-07-06-large-screen-foundations.md)
- [Large-screen Home hub](backlog/large-screen-ux/large-screen-home.md) — ambient A1 now-first Home (hero + Chores/Meals/Lists state strip + Today rail), cross-module routing intents, 10-min idle return; FE PR #284 merged 2026-07-07 (closes #278). [spec](../superpowers/specs/2026-07-05-large-screen-home-design.md) · [plan](../superpowers/plans/2026-07-05-large-screen-home.md)
- [Large-screen Calendar](backlog/large-screen-ux/large-screen-calendar.md) — Week zoom-out (one-line headers, denser hours) and Day member lanes with an optional mini-month rail; FE PR #285 merged 2026-07-10. [spec](../superpowers/specs/2026-07-06-large-screen-calendar-design.md) · [plan](../superpowers/plans/2026-07-06-large-screen-calendar.md)
- [Large-screen Lists](backlog/large-screen-ux/large-screen-lists.md) — two-pane master/detail (≥1024px rail + detail) so the kitchen tablet can keep a list open; mobile/tablet drill-in unchanged. FE PR #286 merged 2026-07-11. [spec](../superpowers/specs/2026-07-06-large-screen-lists-design.md) · [plan](../superpowers/plans/2026-07-06-large-screen-lists.md)
- [Large-screen Meals](backlog/large-screen-ux/large-screen-meals.md) — full-height week board at lg+ (all 7 day columns, equal rows, today highlighted, one-line Calendar-consistent headers) with the "Fill empty slots"/"Add ingredients" actions merged into the WeekHeader toolbar; the v1 full-width board and the v2 full-height revision shipped together. FE PR #287 merged 2026-07-13. [spec](../superpowers/specs/2026-07-06-large-screen-meals-design.md) · [v2 spec](../superpowers/specs/2026-07-12-large-screen-meals-v2-design.md) · [plan](../superpowers/plans/2026-07-12-large-screen-meals-v2.md)
- [Large-screen Chores](backlog/large-screen-ux/large-screen-chores.md) — full-width, full-height board: Today weighted as the primary column, This Week/This Month supporting, each column scrolling its own routines; ≥44px checkoff targets. Mobile/tablet unchanged. FE PR #289 merged 2026-07-15. [spec](../superpowers/specs/2026-07-06-large-screen-chores-design.md) · [plan](../superpowers/plans/2026-07-13-large-screen-chores.md)
- [Large-screen Recipes](backlog/large-screen-ux/large-screen-recipes.md) — responsive 2/3/4-column index grid at 1024px / 1280px / 1440px, a single large-screen toolbar row, and preserved detail/mobile composition. FE issue #290 and PR #291 closed 2026-07-16. [spec](../superpowers/specs/2026-07-06-large-screen-recipes-design.md) · [plan](../superpowers/plans/2026-07-15-large-screen-recipes.md)

### Cross-cutting UI reliability

Unplanned defect work found by dogfooding the shipped large-screen surfaces.
Merged to FE `main` after the `0.3.23` tag, so it is **not yet released or
deployed** — it ships with release-please PR #297 (`0.3.24`).

- Event form time labels no longer clip — the time row sizes its tracks from
  available space (`auto-fit`/`minmax`) instead of the `sm:` viewport
  breakpoint, which never matched the 768px sheet/dialog swap it was deciding
  for. FE PR #296 merged 2026-07-20.
- Every dialog is viewport-bounded by default — `max-h-[90dvh] overflow-y-auto`
  moved from six per-call-site opt-ins onto `DialogContent` itself. Ten
  previously unbounded dialogs were one long field away from an unreachable
  submit button, because a `fixed` centred element does not extend the document
  and so cannot be scrolled to. FE PR #298 merged 2026-07-21.

## Active epics

### Module foundations

Current story:
- [Lists / Chores / Meals / Photos module foundations](backlog/module-foundations/module-surface-foundations.md) — Recipes/Meals implementation merged and released in FE `0.3.11`; Photos deferred

Most recent shipped story:
- [Meals recipe ingredients to grocery list](backlog/module-foundations/meals-recipe-ingredients-to-grocery-list.md) — reviewed recipe ingredient rows from the visible Meals week append to a chosen grocery list through the released generic bulk Lists endpoint

Latest release:
- FE `family-hub-v0.3.23` published 2026-07-17 and deployed to production — the complete large-screen / tablet epic across all seven surfaces.
- FE `0.3.24` is **cut but unreleased**: release-please PR #297 is open with the two dialog viewport fixes (#296, #298). Merging it publishes the release; production still runs `0.3.23`.
- BE `v1.9.0` published 2026-07-05 remains the current released backend contract, with no unreleased commits on BE `main`.

Next recommended product focus (**decision open as of 2026-07-21**):

The dogfood pass is underway and already paying out — FE #296 and #298 were
both found by using the shipped surfaces rather than by planned work. Keep it
running as background signal, not as a blocking phase.

With the large-screen epic closed and no BE work in flight, exactly one shaped
story is ready to execute (FE #293) and one BE Issue is open (#67, login
throttle). The next slice is a genuine fork with no default:

1. **Finish large-screen** — execute FE #293. Closes the last two Calendar
   views still at chrome-only and clears the live foundations §3.3 violation.
2. **Return to mobile** — Notifications, or calendar gestures (drag-to-create /
   pinch-to-zoom). All three are captured but none are shaped to spec/plan.
3. **Security hardening** — BE #67 login throttle, the only open P1 in either
   repo. The accessibility pass surfaced in the July security review is still
   unshaped and has no Issue.
4. **Reduce carrying cost** — 12 open FE dependency PRs, several with major
   version bumps (`vite` 7→8, `lucide-react` 0.x→1.x, `jsdom` 25→29).

Nothing here blocks anything else, so sequence by what the household actually
feels first.

Exit criterion for this phase:
- Make the current organizer surfaces reliable and polished enough for daily phone use by Joe and Partner, while preserving the larger-screen/tablet product direction for later hardware.

Recommended order:
1. `Recipes` — reusable household recipe library and `Add to Meals` handoff; shipped in FE `0.3.11`
2. `Meals` — simple week-ahead planning; shipped in FE `0.3.11`
3. Mobile-first polish of current organizer workflows — shipped through FE `0.3.14`
4. Remaining Mobile UX stories — native-feel polish (transitions/press feedback, back-button, haptics) shipped in `0.3.17`–`0.3.18`, Home organizer summary shipped in `0.3.19`, and quick-capture friction fixes shipped in `0.3.21`; remaining picks are Notifications (reminder value) or calendar gestures (drag-to-create / pinch-to-zoom)
5. `Photos` — deferred; no longer the next product slice

## Planned epics

### Large-screen / tablet UX follow-on

- [Large-screen Calendar Month and Schedule](backlog/large-screen-ux/large-screen-calendar-month-schedule.md) —
  Month fills the viewport with density derived from measured row height, an
  all-events day popover opened through the full day-cell target, presentational
  `+N more`, and multi-day runs welded by corner geometry; Schedule moves to a
  date gutter with full-width rows and explicit gap rows, resolving the
  foundations §3.3 narrow-centred-column violation it still carries. Shaped
  2026-07-18; FE #293 open with no linked PR and no branch — re-verified
  2026-07-21, still not started.

  The Issue contract's pinned frontend baseline has since drifted; see the
  story's Baseline drift note before starting.
  [spec](../superpowers/specs/2026-07-18-large-screen-calendar-month-schedule-design.md) ·
  [plan](../superpowers/plans/2026-07-18-large-screen-calendar-month-schedule.md)

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
