---
id: large-screen-meals
title: Large-screen Meals full-width week board
epic: large-screen-ux
status: in-progress
priority: P2
created: 2026-07-06
updated: 2026-07-13
issues: []
prs:
  - https://github.com/joe-bor/FamilyHub/pull/287
spec: ../../../superpowers/specs/2026-07-06-large-screen-meals-design.md
---

## Context

The Meals week grid horizontally scrolls at laptop width (Friday and Saturday
cut off) while a third of the screen sits empty beside it. The shape is right;
the layout math is not. This story makes the whole week visible at once with
today's column highlighted.

Design: [Large-screen Meals](../../../superpowers/specs/2026-07-06-large-screen-meals-design.md),
revised by [Large-screen Meals v2 — Full-Height Board](../../../superpowers/specs/2026-07-12-large-screen-meals-v2-design.md)
(shipped together on PR #287).
Depends on: [Large-screen foundations](large-screen-foundations.md).

## Scope

- All 7 day columns fit without horizontal scrolling at 1024px+; grid centers
  under a generous cap on very wide screens.
- Today's column highlighted, consistent with Calendar's today treatment.
- One-line weekday headers consistent with Calendar.
- Foundations chrome (single toolbar row).

## Out of Scope

- Meal planning workflow changes (composer, sessions, ingredients-to-grocery).
- New meal features; mobile behavior; backend changes.

## Acceptance Criteria

- [ ] No horizontal scrolling of the week grid at 1024px, 1280px, 1440px.
- [ ] Today's column visibly highlighted.
- [ ] All existing meal flows work unchanged; mobile unchanged.
- [ ] Screenshot review per spec matrix, iterated before done.
