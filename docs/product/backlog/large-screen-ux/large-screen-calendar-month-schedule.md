---
id: large-screen-calendar-month-schedule
title: Large-screen Calendar Month and Schedule polish
epic: large-screen-ux
status: planned
priority: P2
created: 2026-07-18
updated: 2026-07-18
issues: []
prs: []
spec: ../../../superpowers/specs/2026-07-18-large-screen-calendar-month-schedule-design.md
---

## Context

The large-screen epic shipped in FE `0.3.23`. Week and Day received substantive
large-screen redesigns in FE PR #285; Month and Schedule inherited only the
foundations chrome, by explicit decision — the shipped Calendar spec Section 5
records deeper Month and Schedule work as "a follow-up story."

This is that follow-up. It is a new story, not a reopening of the shipped epic,
which stays closed.

Two concrete defects motivate it. Month's grid is `shrink-0` inside a scroll
container, so at 1440x900 it renders roughly 600px tall with a hardcoded 3
events per cell and leaves the rest of the viewport empty. Schedule renders
`mx-auto max-w-3xl`, a 768px centred column leaving roughly 600px of dead
margin — a live violation of the shipped foundations spec Section 3.3.

Design: [Large-screen Calendar Month and Schedule](../../../superpowers/specs/2026-07-18-large-screen-calendar-month-schedule-design.md).
Builds on: [Large-screen foundations](large-screen-foundations.md) and
[Large-screen Calendar](large-screen-calendar.md), both shipped.

## Scope

- Month at `lg+`: grid fills available height; events-per-cell derived from
  measured row height; interactive `+N more` opening a per-day overflow
  popover; multi-day runs welded by corner geometry with the title on every
  day; keyboard grid navigation; weekend, today and selected-day emphasis.
- Month data at `lg+` only: widen the query range to the grid range so
  adjacent-month cells can show events. Gated at 1024px because the range is
  computed module-wide and `MobileMonthlyView` also renders adjacent-month
  days, so an ungated change would alter mobile.
- Schedule at `lg+`: date gutter plus full-width event rows carrying the
  member's name; event-free stretches rendered as explicit gap rows;
  de-emphasised past day groups.
- Both views: loading skeletons, empty and error states.
- Shared: `buildMonthMatrix()` reused rather than duplicated.
- Two defects on surfaces being rebuilt anyway are fixed as part of the work:
  Schedule's coloured left border, which `twMerge` currently collapses away, and
  Month's per-day member dots, which convey member identity by colour plus a
  `title` attribute only.

## Out of Scope

- Week and Day composition (shipped in FE PR #285).
- Mobile Calendar, and the 769-1023px range, for every view.
- Persistent mini-month/filter sidebar, permanent inline detail pane, Meals or
  Chores peek panels.
- Shell or navigation-rail redesign, global search, notifications, cross-module
  quick-add.
- Calendar gestures; event create/detail/edit/recurrence dialogs.
- Generalising the Day view mini-month rail to Month or Schedule.
- Backend changes.

## Dependencies

- [Large-screen foundations](large-screen-foundations.md) — shipped, FE PR #282.
- [Large-screen Calendar](large-screen-calendar.md) — shipped, FE PR #285.
- No backend dependency. The Month range change reuses the existing
  `GET /calendar-events` contract with different date parameters.

## Delivery Risk

`ScheduleCalendar` is a single component shared by mobile and desktop, and is
the mobile smart default for first-time users. Every Schedule change is gated
on `useIsLargeScreen()`, and the Schedule commit is not complete until mobile
rendering is proven unchanged by screenshot hash comparison against
`origin/main` — the method used in FE PR #285.

`MonthlyCalendar` is desktop and tablet only (mobile uses `MobileMonthlyView`),
so Month carries materially less risk and ships first.

## Acceptance Criteria

- [ ] Month at 1440x900 fills the available height with no dead space below the
      final week row; at 1024x768 a six-week month does the same, respecting
      the minimum row-height floor.
- [ ] Events per cell is derived from measured row height, not hardcoded, and a
      taller row yields strictly more events. Absolute counts are confirmed at
      the screenshot gate rather than fixed here.
- [ ] `+N more` is a button opening a popover listing all that day's events;
      selecting one opens the existing event detail modal.
- [ ] Multi-day runs render with rounded outer corners only at true start and
      end, square inner corners throughout, and the full title on every day,
      including across a week boundary.
- [ ] At `lg+`, adjacent-month cells display events when events exist on those
      dates; below 1024px the query range and those cells are unchanged.
- [ ] Month grid is one tab stop with arrow-key day navigation and the
      accessible names specified in the spec.
- [ ] Schedule at 1440x900 spans the available width with no centred narrow
      column; the date gutter carries label, date and event count; rows carry
      the member's name.
- [ ] Event-free stretches render as one explicit gap row.
- [ ] Mobile Schedule rendering is byte-identical to `origin/main`.
- [ ] Mobile Calendar, the 769-1023px range, and Week and Day views are all
      unchanged.
- [ ] Screenshot review per the spec matrix, iterated before done.

## Notes

Deliberately preserved, and recorded as known quirks in the spec rather than
silently changed: Schedule's 14-day window with a 7-day paging step (a 50% page
overlap) and its constant `"Upcoming"` toolbar label, which is shared with the
mobile toolbar.

Two constraints worth carrying into the implementation Issue, both surfaced by
spec review:

- **The 44px rule is narrowed for in-grid Month chips**, to a 28px floor. At
  44px the derived slot capacity would be 2 at 1440x900 and 1 at 1024x768,
  fewer events than ship today. 28px matches the shipped Week view's
  `min-h-[28px]` event buttons and the foundations spec, which scopes the 44px
  rule to chrome. All chrome, controls and Schedule rows stay at 44px.
- **The repo's Vitest setup cannot exercise the lg+ measured-height path.**
  `src/test/setup.ts` mocks `ResizeObserver` as a no-op and `matchMedia` as
  `matches: false` for every query, and jsdom has no layout engine. Capacity
  and slot planning must therefore be proven through pure exported helpers, and
  the observer-to-render wiring through Playwright.
