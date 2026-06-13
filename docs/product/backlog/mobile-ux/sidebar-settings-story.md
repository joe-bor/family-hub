---
id: mobile-sidebar-settings-structure
title: Sidebar structure + family preferences surface
epic: mobile-ux
status: done
priority: P2
created: 2026-06-12
updated: 2026-06-13
issues:
  - BE #57 (family timezone expose/edit; closed by BE PR #58)
  - FE #208 (sidebar Preferences entry + sheet; closed by FE PR #210)
prs:
  - BE #58
  - FE #210
---

## Context

The core modules (Calendar, Lists, Chores, Recipes, Meals) are in good shape, and the mobile shell stories (bottom nav FE #148, module-aware header FE #201, sidebar Radix rebuild FE #194, settings bottom sheets FE #202) fixed the ergonomics of the chrome. What's missing is the connective tissue: a settled answer for **what the sidebar is**, **where settings live**, and **which preferences exist** — so Joe and Partner can use the product daily on their phones without dead ends.

This story is the exploration/spec for that. It is deliberately opinionated: each area below records the options considered, the trade-offs, and a single recommendation.

### Pre-implementation discovery summary (verified 2026-06-12)

This section records the state that shaped the story before implementation. See **Shipped outcome** for the current product state.

**Navigation.** Mobile module navigation is owned entirely by the bottom nav (`Home, Calendar, Lists, Chores` + More sheet with `Meals, Recipes`; Photos removed). The module-aware header (FE #201, merged) renders one row per screen with the Menu button on the right. The sidebar (`frontend/src/components/shared/sidebar-menu.tsx`, rebuilt on the Radix `SideSheet` in FE #194) contains: family-name header, family-member rows (→ `MemberProfileModal`), **Family Settings** (→ `FamilySettingsModal`), **Sign Out** (confirmed), and the app version footer. The sidebar carries **zero navigation** — it is already a de-facto settings/account surface.

**Settings surfaces.** `FamilySettingsModal` (family name edit, member CRUD max 7, Reset Family danger zone) and `MemberProfileModal` (avatar ≤500KB, name, color, email, Google Calendar connect) open as full-height bottom sheets on mobile via `ResponsiveFormDialog` (FE #202, merged); `MemberFormModal` opens half-height. Sheet-over-sheet stacking is shipped and device-verified.

**Family-wide state that exists today:**

| Setting | Where it lives | Editable? |
|---|---|---|
| Family name | BE `Family.name`, Family Settings | Yes |
| Members (name, color, avatar, email) | BE `FamilyMember`, Family Settings / Member Profile | Yes |
| Google Calendar connection | per-member, Member Profile | Yes |
| **Timezone** | BE `Family.timezone` (IANA, captured silently from the browser at registration, default `America/Los_Angeles`) | **No** — `PUT /api/family` accepts only `name`/`username`; `FamilyResponse` doesn't even return it |
| Lists "show completed by default" | BE `ListPreferences` (`GET/PATCH /api/lists/preferences`) | Yes — in-context via the Lists options sheet (FE #200) |
| Calendar view | FE `calendar-store` Zustand persist, **per device**; mobile smart-defaults to `schedule` | Implicitly — last-used view sticks |
| Week start day | **Hardcoded Sunday, cross-stack** (see D3a) | No |
| Username / password | BE `Family` | Username at API level only; password not at all (no endpoint) |

**Notable non-facts** (assumptions worth correcting): there is **no family color** — colors are per-member only. Dark mode is **not** latent: a `next-themes` wrapper exists but is unused, and `src/index.css` has no `.dark` variable block, so dark mode is real styling work, not a toggle flip. The desktop header shows a **hardcoded fake weather reading (72°)**.

**Architectural constraint that shapes everything below:** auth is family-level. There are no per-member sessions, so the app cannot know which member is holding a phone. "Per-member preference" cannot exist server-side today; "per-device preference" (localStorage) is the honest substitute and, for a 2-adult household with one phone each, is per-member in practice.

### Shipped outcome (verified from merged PRs/releases 2026-06-13)

Completed across BE [PR #58](https://github.com/joe-bor/family-hub-api/pull/58) and FE [PR #210](https://github.com/joe-bor/FamilyHub/pull/210).

- BE release [`v1.6.0`](https://github.com/joe-bor/family-hub-api/releases/tag/v1.6.0) exposes `timezone` on `GET /api/family` and accepts optional `timezone` on `PUT /api/family`, validated through `FamilyTimezoneResolver`.
- FE release [`family-hub-v0.3.14`](https://github.com/joe-bor/FamilyHub/releases/tag/family-hub-v0.3.14) adds the sidebar **Preferences** row, the `ResponsiveFormDialog`-based Preferences sheet, editable family timezone, and exactly two disabled stubs: Notifications and Appearance.
- Release sequencing was honored: BE `v1.6.0` was published before FE PR #210 started consuming the family timezone contract.

---

## Decisions

### D1. Sidebar: keep it, as the single non-module surface (not navigation)

**Recommendation:** The sidebar survives long-term on mobile as the home for *everything that isn't module content*: member profiles, Family Settings, a new **Preferences** entry (D2), Sign Out, and the version footer. It is explicitly **not** navigation — the bottom nav owns that — and nothing in this story adds module links to it.

Structure after this story (top to bottom):

1. Header: family name + "Menu" subtitle (unchanged)
2. **Family Members** — member rows → Member Profile sheet (unchanged)
3. **Family Settings** → Family Settings sheet (unchanged)
4. **Preferences** → new Preferences sheet (D2/D3)
5. **Sign Out** → confirm dialog (unchanged)
6. Footer: version (unchanged)

**Considered:**

- **(a) Keep + extend the side sheet (recommended).** Pros: just rebuilt on Radix with focus trap/swipe/Esc (FE #194) and device-verified; members-at-top gives it a personal, family feel; one consistent entry (Menu, top right) on every screen per FE #201. Cons: a left drawer is a slightly dated mobile idiom.
- **(b) Replace with a Settings screen behind a gear icon.** Pros: textbook iOS/Android mental model. Cons: re-adds a second header affordance that FE #194 deliberately removed (the dead gear); requires route-like full-screen surface the app doesn't have (no router; everything is modal/sheet over `activeModule`); throws away a just-shipped, verified primitive for zero user-visible gain in a 2-adult household.
- **(c) Fold settings into the bottom-nav More sheet.** Pros: one fewer surface. Cons: More is module overflow — mixing "Recipes" with "Sign Out" muddies both; More becomes load-bearing for account actions, which breaks when future modules push the overflow longer.

(a) wins on cost and consistency. Revisit only when the tablet/touchscreen shell gets its own design pass.

### D2. Settings surface: a new Preferences sheet on the established adaptive pattern

**Recommendation:** Add a **Preferences** entry to the sidebar that opens a `ResponsiveFormDialog` (full-height bottom sheet on mobile, centered dialog on desktop — exactly the FE #202 pattern). The mental model the family learns: **Family Settings = who we are** (name, members, danger zone) · **Preferences = how the app behaves** (timezone, future notifications/appearance). Member Profile remains member *data*, opened from the member rows.

**Considered:**

- **(a) New Preferences sheet (recommended).** Pros: clean two-bucket model; durable home for the settings the roadmap is already promising (notifications config will need a place to live); reuses the shipped adapter, so it costs one sidebar row plus one sheet. Cons: launches thin — one real setting plus two stubs (D5). Accepted: the point of this story is to create the *place*; the roadmap fills it.
- **(b) Extend Family Settings with a preferences section.** Pros: zero new surfaces. Cons: that modal is already three sections deep ending in a danger zone; appending toggles puts "week behavior" next to "delete everything"; it becomes a junk drawer the next three stories make worse.
- **(c) Inline toggles directly in the sidebar.** Pros: fewest taps. Cons: the sheet is `min(20rem, 85vw)` wide — form controls in a nav drawer are cramped; every future setting bloats the menu everyone passes through to sign out.

### D3. Family-wide preferences — what's real, what's not

#### D3a. Week start day: keep Sunday; do not build a toggle

**Recommendation:** No week-start preference in this story, and no stub for it. Keep Sunday. Fix the PRD instead (it still says week view shows "Monday–Sunday", which the shipped product contradicts), and put the *actual* question to the family — see Open Questions.

The real considerations, honestly:

- **It is not a display toggle — it's a cross-stack contract.** BE `MealService` rejects any `weekStartDate` that isn't a Sunday (`MealService.java:353`); meal slots are *keyed* by Sunday dates in the DB. BE `ChoreService` anchors weekly chore periods at `previousOrSame(SUNDAY)` (`ChoreService.java:69,340`) — moving the anchor changes *which week a completion belongs to*, retroactively, mid-week. FE hardcodes Sunday in ~8 places (`weekStartsOn: 0` in `calendar-module.tsx`, `mobile-weekly-view.tsx`, `mobile-monthly-view.tsx`, `context-label.ts`, plus raw `getDay()` math in `weekly-calendar.tsx`, `monthly-calendar.tsx`, `time-utils.ts:120`, and `useIsViewingToday` in `calendar-store.ts`).
- **Locale:** the family is US-based (`America/Los_Angeles`); Sunday start is the US convention and what Google Calendar shows them by default. Google sync is unaffected either way — week start is presentation, events carry absolute dates.
- **Habit:** the family has been using Sunday weeks daily since 0.3.x. Switching has a real re-learning cost; switching *only the calendar* while Meals/Chores stay Sunday-anchored would be worse than either consistent option.
- **Single-family reality:** this is one household, not a SaaS user base. If the family wants Monday, the right move is a **one-time cross-stack re-anchor story** (with a meals-slot key migration and a chores period cutover plan) — not a per-family preference whose entire migration machinery would exist to serve one toggle flip ever.

**Considered:** (a) full BE-backed preference — rejected, cost wildly disproportionate (meals data migration + chores semantics + 8 FE sites) for a setting nobody has asked to change; (b) calendar-only preference — rejected, creates Mon-start calendar over Sun-anchored Meals/Chores, an inconsistency worse than the status quo; (c) greyed-out stub — rejected, stubs signal roadmap intent (D5) and we have none here; (d) **keep Sunday, settle it as a product decision, document the escape hatch (recommended)**.

#### D3b. Calendar default view on mobile: no new setting

**Recommendation:** None. The mechanism already exists and is better than a setting: `calendar-store` persists the last-used view per device, and first-time mobile users smart-default to `schedule`. Whatever view Partner prefers, she gets by using it once. An explicit "default view" control would be a second mechanism fighting the first ("I set Month as default but it keeps opening in Week because I last used Week" — or the setting silently loses). Considered and rejected; documented here so it isn't re-litigated.

#### D3c. Family name / family color

Family name **already has the right home** — Family Settings — and stays there; it is identity, not behavior. **Family color does not exist** and should not be invented: member colors carry all the meaning in this product (event ownership, filters, chips), and a family-level color has no consumer. If a future theming story wants an accent color, that's an Appearance concern (D5), not a family-identity field.

#### D3d. Timezone: the one real family-wide setting to add now

**Recommendation:** Surface **family timezone** in the Preferences sheet, editable. This is the strongest candidate found in discovery: it already exists in BE, it was captured **silently** from whichever browser registered the family, it is never shown in the UI (it rides along unused in `ChoreBoardResponse`), it cannot be fixed if wrong — and it quietly controls what "today" means for Chores period boundaries (`ChoreService` builds boards from `LocalDate.now` in the family zone). A family that registers while traveling, or moves, has a permanently subtly-wrong chores board with no recourse.

Mechanics: BE adds `timezone` to `FamilyResponse` and accepts an optional `timezone` in `PUT /api/family`, validated/normalized through the existing `FamilyTimezoneResolver` (the validation logic already exists for registration). FE renders it as a curated select (US zones + the current value + a "use this device's timezone" detect action), not a 400-entry IANA dropdown.

**Considered:** (a) read-only display — rejected, showing a wrong value you can't fix is worse than not showing it; (b) editable (recommended); (c) skip entirely — rejected, this is exactly the class of invisible-but-consequential setting a settings surface exists for.

#### D3e. Other family-wide toggles considered, not added

- **Lists "show completed by default"** — already family-wide in BE and editable in-context via the Lists options sheet (FE #200). Duplicating it in Preferences would create two sources of truth for one toggle. Leave in-context; in-context is the right home for module-behavior toggles generally.
- **Home dashboard content** (which modules feed the summary) — premature; the Home-as-organizer-summary evolution is still unshaped on the roadmap. Preferences is its natural future home; noted, not built.

### D4. Per-member preferences: none in v1, by architecture

**Recommendation:** No per-member settings concept in this story or v1. With family-level auth only, a server-side per-member preference has no way to bind to the person holding the device — it would be a setting *about* a member that any device can flip, which is just a family-wide setting with extra steps. The honest split for v1:

- **Family-wide, server-backed:** timezone, future notifications defaults, (existing) list preferences.
- **Per-device, localStorage:** calendar view, member filters — already shipped and correctly scoped, since each adult has their own phone.
- **Member data (not preferences):** name, color, avatar, email, Google connection — stays in Member Profile, unchanged. Everything editable about a member is already editable there; nothing moves into Preferences.

The future unlock is a lightweight **device–member association** ("this phone is Joe's") — see New Ideas. It enables member-focused defaults without building real multi-user auth. Out of scope here.

### D5. Stubbed future features: exactly two — Notifications and Appearance

**Recommendation:** The Preferences sheet shows two disabled rows, each with a "Coming soon" pill, real icon, honest one-line description, `aria-disabled`, and no click handler:

1. **Notifications** — "Event reminders on your phone." Justified: it's already a planned backlog story ([notifications](notifications.md)), the family will want it, and the stub both sets expectation and reserves its place in the surface.
2. **Appearance (Light/Dark)** — "Light theme for now; dark mode is planned." Justified: the most-requested personal app setting; evening phone use in a household makes it genuinely desirable; `next-themes` is already a dependency. The stub costs one row; the real story is a styling pass (no `.dark` token set exists yet).

**Considered and rejected as stubs** — each fails the test "plausibly coming, self-explanatory, and its absence is otherwise confusing":

- **Language/locale** — English-only family, zero i18n on the roadmap; pure noise.
- **Recurring reminders** — subsumed by Notifications; a second reminder-ish stub reads as clutter.
- **Shopping integrations** — speculative, nothing on the roadmap; would be a promise nobody has decided to keep.
- **Export data** — PRD explicitly lists data export under "Explicitly NOT building."
- **Family invite link** — PRD explicitly excludes social/sharing; the auth model is one shared family credential, so an "invite" has nothing to attach to.
- **Week start** — see D3a; no intent, so no stub.

The cap matters: with one real setting shipping (timezone), two stubs is the ceiling before the surface reads as vaporware. When Notifications ships, its stub becomes the entry point; net stub count goes down over time, not up.

### D6. Onboarding tie-in: everything onboarding sets must be revisitable — timezone is the gap this story closes

Onboarding collects: family name → revisitable (Family Settings ✓); members + colors → revisitable (Family Settings / Member Profile ✓); **timezone (silent capture) → not revisitable today, fixed by D3d ✓**; username + password → **not revisitable** (no password-change endpoint; username is API-editable but unexposed). 

Credentials remain the one acknowledged gap. Recommendation: defer — it's security-sensitive BE work (current-password verification, hashing flow), and a 2-adult household sharing one credential has low rotation urgency. Recorded as a backlog seed, not silently dropped.

---

## Recommended story scope

**In:**

1. Sidebar restructure per D1 (add Preferences row; light section grouping; no other behavior changes).
2. New **Preferences sheet** on `ResponsiveFormDialog` (full-height mobile sheet / centered desktop dialog) containing: editable family timezone (D3d) + Notifications and Appearance stub rows (D5).
3. BE: expose `timezone` in `FamilyResponse`; accept optional `timezone` in `PUT /api/family` with `FamilyTimezoneResolver` validation.
4. PRD correction: week view is Sunday-start (D3a); record the Monday question in Open Questions.

**Out (explicitly):** week-start toggle or re-anchor (D3a); default-view setting (D3b); family color (D3c); any per-member preferences (D4); implementing notifications or dark mode (stubs only); password/username change; PWA install row; Home-dashboard content settings; moving Sign Out or member rows; desktop/tablet shell redesign; any change to Lists preferences.

## Acceptance criteria

- [x] The sidebar shows, in order: header, Family Members, Family Settings, Preferences, Sign Out, version footer. All existing entries behave as before.
- [x] Tapping Preferences opens a full-height bottom sheet on mobile (≤768px) and a centered dialog on desktop, via the existing `ResponsiveFormDialog` pattern; dismissal (Cancel/scrim/Esc/flick-down on mobile) and focus return work as in the FE #202 surfaces.
- [x] Preferences shows the family's current timezone (fetched from the API, not assumed); the family can change it from a curated list or via a "use this device's timezone" action; the change persists across reload and is reflected in `GET /api/family`.
- [x] An invalid timezone is rejected by the BE with a validation error and surfaced non-destructively in the FE form.
- [x] `GET /api/family` returns `timezone`; `PUT /api/family` with `timezone` omitted leaves it unchanged (existing partial-update semantics preserved for `name`/`username`).
- [x] Notifications and Appearance render as visibly disabled rows with a "Coming soon" indicator; they are not focusable as actions (or are `aria-disabled` with no handler); screen readers announce the disabled state.
- [x] No regression to Family Settings, Member Profile, Member Form, or Sign Out flows (existing unit + E2E suites green; new E2E covers open-Preferences → edit-timezone → persist).
- [x] Desktop sidebar and dialogs visually unchanged except for the new Preferences row.
- [x] BE change ships in a released BE version before the FE consuming it merges to `main` (shipping semantics).
- [x] PRD §7.1.1 ("Show 7 days (Monday-Sunday)") corrected to reflect the shipped Sunday-start week, per D3a. — Done 2026-06-12 alongside the week-start decision.

## Implementation / verification

- BE [PR #58](https://github.com/joe-bor/family-hub-api/pull/58) closed [BE #57](https://github.com/joe-bor/family-hub-api/issues/57), merged 2026-06-12, and shipped in [`v1.6.0`](https://github.com/joe-bor/family-hub-api/releases/tag/v1.6.0). PR verification: `./mvnw test` — 407 tests, 0 failures.
- FE [PR #210](https://github.com/joe-bor/FamilyHub/pull/210) closed [FE #208](https://github.com/joe-bor/FamilyHub/issues/208), merged 2026-06-13, and shipped in [`family-hub-v0.3.14`](https://github.com/joe-bor/FamilyHub/releases/tag/family-hub-v0.3.14). PR verification: lint clean, unit suite 911 passed / 82 files, build green, focused Preferences E2E 8/8, and sidebar/settings/onboarding regression E2E 13 passed / 15 skipped.

## Non-goals

- Building notification delivery, dark-mode styling, or any stubbed capability.
- Week-start preference or Monday re-anchor (separate decision, see Open Questions).
- Per-member or per-device settings UI beyond what already exists.
- New family-identity fields (family color, avatar for the family, etc.).
- Router/full-screen settings architecture; the modal/sheet model stays.
- Touching the bottom nav, More sheet, or module-aware header.

---

## FE implementation issue body (repo: joe-bor/FamilyHub) — opened as [FE #208](https://github.com/joe-bor/FamilyHub/issues/208), closed by [FE PR #210](https://github.com/joe-bor/FamilyHub/pull/210)

> **Title:** Sidebar Preferences entry + family preferences sheet (timezone + roadmap stubs)
>
> Story: `joe-bor/family-hub` → `docs/product/backlog/mobile-ux/sidebar-settings-story.md`
> Spec: same file, §Decisions D1–D5 (this story file is the spec)
> Plan: to be authored at pickup per the operating workflow (`docs/superpowers/plans/`)
>
> **Execution contract (non-negotiable):**
> 1. Add a **Preferences** row to `SidebarMenu` between Family Settings and Sign Out; ≥44px target; no other sidebar behavior changes.
> 2. New `PreferencesSheet` rendered through the existing `ResponsiveFormDialog` (`initialHeight="full"`); desktop renders the centered dialog. Do not fork a new adaptive pattern.
> 3. Timezone control: shows the value from `GET /api/family` (never a hardcoded default); curated zone list + "use this device's timezone" (`Intl.DateTimeFormat().resolvedOptions().timeZone`); saves via the family update mutation; optimistic-update + rollback consistent with existing family mutations; BE validation errors surfaced inline.
> 4. Exactly two stub rows — **Notifications**, **Appearance** — visibly disabled, "Coming soon" indicator, `aria-disabled`, no click handlers. No additional stubs.
> 5. Depends on a **released** BE version exposing `timezone` in `FamilyResponse` and accepting it in `PUT /api/family` (BE issue below). Gate merge on that release per shipping semantics.
> 6. Tests: unit for sidebar row + sheet rendering both breakpoints; E2E: open Preferences → change timezone → reload → persisted. Existing settings/sidebar E2E stays green.

## BE implementation issue body (repo: joe-bor/family-hub-api) — opened as [BE #57](https://github.com/joe-bor/family-hub-api/issues/57), closed by [BE PR #58](https://github.com/joe-bor/family-hub-api/pull/58)

> **Title:** Expose and allow updating family timezone
>
> Story: `joe-bor/family-hub` → `docs/product/backlog/mobile-ux/sidebar-settings-story.md`
> Spec: same file, §D3d
> Plan: to be authored at pickup per the operating workflow
>
> **Execution contract (non-negotiable):**
> 1. `FamilyResponse` gains `timezone` (the stored IANA string, resolved through `FamilyTimezoneResolver.resolveStoredTimezoneOrDefault` so legacy rows still return a valid zone).
> 2. `FamilyRequest` gains optional `timezone`; `PUT /api/family` updates it only when present (omitted ⇒ unchanged — match existing partial semantics for `name`/`username`).
> 3. Validation/normalization reuses `FamilyTimezoneResolver` (no second validation path); invalid zone ⇒ 400 `VALIDATION_ERROR` with the resolver's message.
> 4. No change to chores/meals logic — `ChoreService` already resolves the zone per request; document in the PR that editing timezone shifts the family's "today" for chore boards from the next request onward (accepted, forward-looking).
> 5. Tests (TestContainers): GET returns timezone; PUT round-trips a valid zone; PUT rejects an invalid zone; PUT without timezone leaves it untouched.
> 6. Update `docs/calendar-events-api-reference.md`-adjacent family contract docs if any document the family payload; cut a BE release after merge so the FE can consume it.

---

## Risks

- **Timezone edit shifts chore-period boundaries.** Changing the zone changes what "today"/"this week" means for `ChoreService`, so a completion logged near midnight could attribute to a different period after an edit. Frequency: effectively once-ever per family. Mitigation: forward-looking semantics, called out in the BE contract; no migration of past data.
- **Thin launch.** One real setting + two stubs. Accepted deliberately (D2): the story's value is the settled structure; Notifications is the next occupant.
- **Stub perception.** Disabled rows must read as "planned," not "broken." The "Coming soon" pill + description copy is the mitigation; the two-stub cap keeps the ratio honest.
- **Release sequencing.** FE consuming an unreleased BE field would break the FE-CI-against-released-BE contract; the FE issue gates on the BE release explicitly.
- **PRD drift.** If the PRD's "Monday–Sunday" line isn't corrected alongside this story, the next person specs week-start work against the wrong source of truth.

## Open questions

1. ~~**Does anyone in the family actually want Monday-start weeks?**~~ **Decided 2026-06-12: keep Sunday** (Joe confirmed). The PRD §7.1.1 line has been corrected to Sunday–Saturday. If the family ever changes its mind, the path is a dedicated one-time cross-stack re-anchor story (FE ~8 call sites, BE Meals slot-key migration, Chores period cutover) — not a preference toggle.
2. **Timezone picker breadth:** curated list (recommended: US zones + current value + device-detect) vs full IANA search. Decide at FE plan time; the BE accepts any valid IANA zone either way.
3. **Credentials management** (password change, username surfacing): deferred backlog seed — when, if ever, does it matter for a shared-credential household? Likely trigger: a lost/leaked password event or a future member-identity story.

## New ideas surfaced (backlog seeds, not in scope)

- **PWA install row in the sidebar** — "Add Family Hub to your home screen." The manifest already ships (vite-plugin-pwa); on Android/Chrome (Partner's Galaxy S10) `beforeinstallprompt` makes this a one-tap install; hide the row when already installed/unsupported. Probably the single highest-leverage daily-use improvement available for one row of UI.
- **Device–member association** — a localStorage-only "This phone belongs to: \[member\]" selector in Preferences. Unlocks member-focused Home defaults, pre-filled event member, and per-person calendar filters without touching auth. Natural successor to D4.
- **Home dashboard content settings** — when Home evolves into the organizer summary the roadmap calls for, its configuration belongs in the Preferences sheet established here.
- **Remove the fake desktop weather** — the desktop header renders a hardcoded 72°. On a daily-use product a fabricated reading erodes trust; remove it (or gate it behind the future weather story). Candidate finding to append to [mobile-module-content-polish](mobile-module-content-polish.md) despite being desktop-side.
- **Credentials management story** — password change with current-password verification; surface username (read-only) somewhere discoverable so the family can recall their login.
