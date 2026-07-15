---
id: large-screen-recipes
title: Large-screen Recipes card grid
epic: large-screen-ux
status: planned
priority: P3
created: 2026-07-06
updated: 2026-07-15
issues:
  - https://github.com/joe-bor/FamilyHub/issues/290
prs: []
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
- Search and favorites filters in the foundations toolbar row.

## Out of Scope

- Recipe detail composition; add/edit flows.
- New features (collections, sorting, import).
- Mobile behavior; backend changes.

## Acceptance Criteria

- [ ] 2-4 column grid at 1024px / 1280px / 1440px+ inside the container.
- [ ] No card fills the viewport; images render at the capped aspect ratio.
- [ ] Search and favorites filtering work unchanged; mobile unchanged.
- [ ] Screenshot review per spec matrix, iterated before done.
