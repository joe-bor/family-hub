---
id: large-screen-lists
title: Large-screen Lists two-pane master/detail
epic: large-screen-ux
status: done
priority: P2
created: 2026-07-06
updated: 2026-07-11
issues: []
prs:
  - https://github.com/joe-bor/FamilyHub/pull/286
spec: ../../../superpowers/specs/2026-07-06-large-screen-lists-design.md
---

## Context

Lists is the least adapted module on large screens: a narrow centered index
column, and opening a list replaces the whole screen phone-style. Lists is
also the module most likely to stay open on a shared kitchen device. This
story shows both panes at once on large screens.

Design: [Large-screen Lists](../../../superpowers/specs/2026-07-06-large-screen-lists-design.md).
Depends on: [Large-screen foundations](large-screen-foundations.md).

## Scope

- Two-pane layout at 1024px+: lists rail (~340px) plus detail pane filling
  the remaining width.
- First list auto-selected; create/delete selection fallbacks.
- Cross-module list handoff selects in the pane instead of pushing a screen.
- Back-handler semantics adjusted (no detail screen to back out of on `lg+`).

## Out of Scope

- New list features (sharing, reordering).
- Add-item and category-management flow changes beyond where they render.
- Mobile behavior; backend changes.

## Acceptance Criteria

- [ ] Rail + detail render simultaneously at 1024px+; selecting a list swaps
      only the right pane.
- [ ] Auto-select and create/delete fallbacks behave as specified.
- [ ] Existing item flows work unchanged inside the pane.
- [ ] Below 1024px and on mobile, drill-in flow unchanged.
- [ ] Screenshot review per spec matrix, iterated before done.
