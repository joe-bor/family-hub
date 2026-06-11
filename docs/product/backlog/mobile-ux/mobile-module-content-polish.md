---
id: mobile-module-content-polish
title: Mobile module content polish
epic: mobile-ux
status: planned
priority: P2
created: 2026-06-11
updated: 2026-06-11
issues: []
prs: []
---

## Context

The mobile-ux shell stories (bottom nav, home redesign, sidebar/settings/onboarding, expandable bottom sheet) fixed navigation- and shell-level ergonomics. During the 0.3.12 device smoke on a **Galaxy S10 / Chrome (2026-06-11)**, the shell changes all passed, but several **per-module content/layout** rough edges surfaced. These are **pre-existing** — none were introduced by the shell stories — and they're about the *content area* of individual modules, not the chrome around them.

This story is a holding pen / discovery capture. Each item below is an independent, small-to-medium polish. They should be prioritized, specced, and split into FE issues individually (or in small batches) when picked up — this file is not yet a spec.

Scope is **mobile content density and layout**, FE-only, no backend. The app is still intended for a larger tablet/home-hub screen long-term, so changes should be mobile-first but not regress desktop.

## Findings (from 0.3.12 device testing)

- [ ] **F1 — Shared nav header is inconsistent across modules.** Every module renders the same `AppHeader` except Calendar, which suppresses it on mobile in favor of its own `MobileToolbar` (`App.tsx` gates `!(isMobile && activeModule === "calendar")`). Decide the intended pattern: either give each module an appropriate header treatment, or make the shared header carry module-aware context (title/actions) so Calendar isn't a one-off. Consistency decision first, then implement.

- [ ] **F2 — Some dialogs exceed the viewport on small screens.** Settings dialogs (and possibly others) can still be cut off / not fully fit on a Galaxy S10, even with the `max-h-[90dvh] overflow-y-auto` scroll bound added in #194. The durable fix is the **settings-dialogs → bottom-sheet conversion** already anticipated as a future story in the [expandable-bottom-sheet](expandable-bottom-sheet.md) spec — convert centered `DialogContent` settings/auth modals to the vaul `MobileSheet` so they're viewport-bounded and keyboard-safe on mobile. (Auth onboarding/login full screens were addressed in #194; re-confirm none regressed.)

- [ ] **F3 — Lists: category/completed visibility controls eat too much vertical space.** The show/hide-categories and show/hide-completed controls dominate the viewport and push the actual list items (the important content) below the fold. Make these controls more subtle/compact — e.g. collapse into an overflow/filter affordance or a single compact toggle row — so the item list gets the space.

- [ ] **F4 — Chores: redundant period label.** The day/week/month segmented picker at the top is immediately followed by a parent card whose title repeats the same period ("Today" / "This week" / "This month"). Remove the redundancy — drop the card's period title, or merge the picker into the card header so the period is stated once.

- [ ] **F5 — Recipes: recipe card images are oversized.** Each recipe card's image takes more than half the screen height on a Galaxy S10, so scrolling through several recipes is slow. Rework into a denser layout — e.g. a horizontal/rectangular card with a small thumbnail (and/or a compact list view) so more recipes are visible per screen.

## Notes

- Likely **more inconsistencies exist** beyond these five; treat this as an initial pass, not an exhaustive audit. Append new findings here as they're discovered, or split out a dedicated audit story if the list grows large.
- When an item is picked up: write a short spec/plan in `docs/superpowers/` if non-trivial, open an FE issue with `Story:`/`Spec:`/`Plan:` links, and check the box here with the PR reference.

## Out of scope

- Shell-level navigation (done in prior mobile-ux stories).
- Backend changes, desktop/tablet redesign, new module features.
