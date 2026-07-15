---
id: large-screen-chores
title: Large-screen Chores board polish
epic: large-screen-ux
status: done
priority: P3
created: 2026-07-06
updated: 2026-07-15
issues: []
prs:
  - https://github.com/joe-bor/FamilyHub/pull/289
spec: ../../../superpowers/specs/2026-07-06-large-screen-chores-design.md
---

## Context

Chores already renders three scope columns (Today / This Week / This Month) on
desktop — the right structure, unpolished. This story is a refinement pass:
weight Today as primary, grow checkoff targets, and bring member identity in
line with the Calendar Day lane treatment.

Design: [Large-screen Chores](../../../superpowers/specs/2026-07-06-large-screen-chores-design.md).
Depends on: [Large-screen foundations](large-screen-foundations.md).

## Scope

- Widen the board container with comfortable column widths; Today visually
  primary.
- Larger checkoff targets on large screens.
- Assignee group headers: avatar + name + member color, consistent with
  Calendar Day lanes.
- Foundations chrome (single toolbar row).

## Out of Scope

- Member-lane board mode (possible follow-up).
- New chore features; mobile behavior; backend changes.

## Acceptance Criteria

- [ ] Widened board with Today visually primary at 1024px and 1440px.
- [ ] Checkoff targets at least 44px on large screens.
- [ ] All existing chore flows behave unchanged; mobile unchanged.
- [ ] Screenshot review per spec matrix, iterated before done.
