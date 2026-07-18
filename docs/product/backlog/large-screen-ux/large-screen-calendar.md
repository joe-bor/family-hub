---
id: large-screen-calendar
title: Large-screen Calendar (Week zoom-out, Day member lanes)
epic: large-screen-ux
status: done
priority: P2
created: 2026-07-06
updated: 2026-07-10
issues: []
prs:
  - https://github.com/joe-bor/FamilyHub/pull/285
spec: ../../../superpowers/specs/2026-07-06-large-screen-calendar-design.md
---

## Context

On a 13-inch laptop, Calendar Week spends roughly half the viewport on chrome
and oversized day headers, showing only about four hours of the week. Day view
is a single wide canvas of ad-hoc overlap columns — the most screen space in
the app with the least structure. This story makes Week read as the shape of
the week and Day read as the family's day side-by-side.

Design: [Large-screen Calendar](../../../superpowers/specs/2026-07-06-large-screen-calendar-design.md).
Depends on: [Large-screen foundations](large-screen-foundations.md).

## Scope

- Week: one-line day headers, denser hour rows (12-14 visible hours),
  auto-scroll to now/first event, merged toolbar.
- Day on `lg+`: one lane per member (avatar + name + color headers) with an
  all-day band aligned to lanes.
- Day: optional mini-month rail (~300px) that auto-shows when lane math
  leaves spare width, with a persisted toolbar toggle to hide it.
- Month/Schedule inherit foundations chrome only.

## Out of Scope

- Month/Schedule composition redesign.
- Calendar gestures (drag-to-create, pinch-to-zoom).
- Event detail/edit dialogs; backend changes.
- Mobile Calendar.

## Acceptance Criteria

- [x] Week at 1440x900: 12+ visible hour rows, one-line day headers, single
      toolbar row.
- [x] Day on `lg+`: correct events per member lane; within-lane overlaps
      stack correctly; all-day band aligned to lanes.
- [x] Mini-month rail auto-show/hide mechanics and persisted toggle work as
      specified; tapping a day navigates Day view.
- [x] Mobile Calendar unchanged.
- [x] Screenshot review per spec matrix, iterated before done.
