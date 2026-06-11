---
id: mobile-module-content-polish
title: Mobile module content polish
epic: mobile-ux
status: in-progress
priority: P2
created: 2026-06-11
updated: 2026-06-11
issues:
  - FE #199 (F4+F5 density quick wins)
  - FE #200 (F3 lists options sheet)
  - FE #201 (F1 module-aware header)
  - FE #202 (F2 settings sheets)
prs: []
---

## Context

The mobile-ux shell stories (bottom nav, home redesign, sidebar/settings/onboarding, expandable bottom sheet) fixed navigation- and shell-level ergonomics. During the 0.3.12 device smoke on a **Galaxy S10 / Chrome (2026-06-11)**, the shell changes all passed, but several **per-module content/layout** rough edges surfaced. These are **pre-existing** — none were introduced by the shell stories — and they're about the *content area* of individual modules, not the chrome around them.

This story is a holding pen / discovery capture. Each item below is an independent, small-to-medium polish. All five findings were specced and split into FE issues on 2026-06-11 (design forks for F1/F2/F5 decided with Joe; see the per-finding spec links below). Suggested execution order: F4+F5 quick wins → F3 → F1 → F2.

Scope is **mobile content density and layout**, FE-only, no backend. The app is still intended for a larger tablet/home-hub screen long-term, so changes should be mobile-first but not regress desktop.

## Findings (from 0.3.12 device testing)

- [ ] **F1 — Shared nav header is inconsistent across modules.** Every module renders the same `AppHeader` except Calendar, which suppresses it on mobile in favor of its own `MobileToolbar` (`App.tsx` gates `!(isMobile && activeModule === "calendar")`). **Decided 2026-06-11: module-aware shared header** — generalize Calendar's pattern (title left = module name / family name / calendar context label; actions + Menu right), remove the gate, drop duplicate in-content module titles on mobile.
  - Spec: `docs/superpowers/specs/2026-06-11-mobile-module-aware-header.md` · Plan: `docs/superpowers/plans/2026-06-11-mobile-module-aware-header.md` · Issue: [FE #201](https://github.com/joe-bor/FamilyHub/issues/201)

- [ ] **F2 — Some dialogs exceed the viewport on small screens.** Settings dialogs (and possibly others) can still be cut off / not fully fit on a Galaxy S10, even with the `max-h-[90dvh] overflow-y-auto` scroll bound added in #194. **Decided 2026-06-11: convert to `MobileSheet`** — `FamilySettingsModal`/`MemberProfileModal` (full) + `MemberFormModal` (half) via a new `ResponsiveFormDialog` adapter; desktop keeps centered dialogs; confirms stay centered. (Auth onboarding/login full screens were addressed in #194; re-confirm none regressed.)
  - Spec: `docs/superpowers/specs/2026-06-11-mobile-settings-sheets.md` · Plan: `docs/superpowers/plans/2026-06-11-mobile-settings-sheets.md` · Issue: [FE #202](https://github.com/joe-bor/FamilyHub/issues/202)

- [ ] **F3 — Lists: category/completed visibility controls eat too much vertical space.** The show/hide-categories and show/hide-completed controls dominate the viewport and push the actual list items (the important content) below the fold. Specced: extract a `ListOptionsControls` component; on mobile it lives in a half-height `MobileSheet` behind a `SlidersHorizontal` icon trigger; desktop stays inline.
  - Spec: `docs/superpowers/specs/2026-06-11-mobile-lists-options-density.md` · Plan: `docs/superpowers/plans/2026-06-11-mobile-lists-options-density.md` · Issue: [FE #200](https://github.com/joe-bor/FamilyHub/issues/200)

- [ ] **F4 — Chores: redundant period label.** The day/week/month segmented picker at the top is immediately followed by a parent card whose title repeats the same period ("Today" / "This week" / "This month"). Specced: hide the card heading on mobile (`showHeading` prop, `aria-label` keeps the accessible name); desktop 3-column board keeps headings. Batched with F5.
  - Spec: `docs/superpowers/specs/2026-06-11-mobile-density-quick-wins.md` · Plan: `docs/superpowers/plans/2026-06-11-mobile-density-quick-wins.md` · Issue: [FE #199](https://github.com/joe-bor/FamilyHub/issues/199)

- [ ] **F5 — Recipes: recipe card images are oversized.** Each recipe card's image takes more than half the screen height on a Galaxy S10, so scrolling through several recipes is slow. **Decided 2026-06-11: horizontal thumbnail card** — mobile card becomes a row with a ~96px square thumbnail, title, one row of tags, inline favorite heart (CSS-first, desktop vertical card unchanged). Batched with F4.
  - Spec: `docs/superpowers/specs/2026-06-11-mobile-density-quick-wins.md` · Plan: `docs/superpowers/plans/2026-06-11-mobile-density-quick-wins.md` · Issue: [FE #199](https://github.com/joe-bor/FamilyHub/issues/199)

## Notes

- Likely **more inconsistencies exist** beyond these five; treat this as an initial pass, not an exhaustive audit. Append new findings here as they're discovered, or split out a dedicated audit story if the list grows large.
- When an item is picked up: write a short spec/plan in `docs/superpowers/` if non-trivial, open an FE issue with `Story:`/`Spec:`/`Plan:` links, and check the box here with the PR reference.

## Out of scope

- Shell-level navigation (done in prior mobile-ux stories).
- Backend changes, desktop/tablet redesign, new module features.
