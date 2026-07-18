---
id: large-screen-recipes
title: Large-screen Recipes card grid
epic: large-screen-ux
status: done
priority: P3
created: 2026-07-06
updated: 2026-07-17
issues:
  - https://github.com/joe-bor/FamilyHub/issues/290
prs:
  - https://github.com/joe-bor/FamilyHub/pull/291
plan: ../../../superpowers/plans/2026-07-15-large-screen-recipes.md
spec: ../../../superpowers/specs/2026-07-06-large-screen-recipes-design.md
---

## Context

The Recipes index renders one enormous mobile-width card at a time on large
screens, with half the viewport as blank margin. This story turns the index
into a browsable responsive card grid.

Design: [Large-screen Recipes](../../../superpowers/specs/2026-07-06-large-screen-recipes-design.md).
Plan: [Large-screen Recipes implementation plan](../../../superpowers/plans/2026-07-15-large-screen-recipes.md).
Depends on: [Large-screen foundations](large-screen-foundations.md).

## Scope

- 2-4 column card grid (by width) inside a ~1200px max-width container.
- Card images capped at a fixed aspect ratio.
- Title, bounded-width search, favorites/tag filters, and Add recipe in the
  foundations single toolbar row at `lg+`.

## Out of Scope

- Recipe detail composition; add/edit flows.
- New features (collections, sorting, import).
- Interactive favoriting from an index card; the heart stays display-only.
- Mobile behavior; backend changes.

## Acceptance Criteria

- [x] 2-4 column grid at 1024px / 1280px / 1440px+ inside the container.
- [x] No card fills the viewport; images render at the capped aspect ratio.
- [x] Search and favorites filtering work unchanged; mobile unchanged.
- [x] At `lg+`, title, search, filters, and Add recipe fit in one toolbar row
      with 44px interactive targets.
- [x] Screenshot review per spec matrix, iterated before done.

Delivered on [FE PR #291](https://github.com/joe-bor/FamilyHub/pull/291)
(merged 2026-07-16), closing
[FE issue #290](https://github.com/joe-bor/FamilyHub/issues/290). The PR's
real-backend Playwright suite verified the 1024px / 1280px / 1440px grid and
toolbar geometry, centered states, preserved detail width, filtering, and the
375px mobile regression baseline.
