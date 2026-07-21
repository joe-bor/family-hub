---
id: large-screen-calendar-month-schedule
title: Large-screen Calendar Month and Schedule polish
epic: large-screen-ux
status: planned
priority: P2
created: 2026-07-18
updated: 2026-07-21
issues:
  - https://github.com/joe-bor/FamilyHub/issues/293
prs: []
spec: ../../../superpowers/specs/2026-07-18-large-screen-calendar-month-schedule-design.md
plan: ../../../superpowers/plans/2026-07-18-large-screen-calendar-month-schedule.md
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
Plan: [Large-screen Calendar Month and Schedule implementation](../../../superpowers/plans/2026-07-18-large-screen-calendar-month-schedule.md).
Builds on: [Large-screen foundations](large-screen-foundations.md) and
[Large-screen Calendar](large-screen-calendar.md), both shipped.

## Scope

- Month at `lg+`: grid fills available height; visual slot capacity derived from
  measured row height; the full day cell opens a per-day event popover while
  28px chips and `+N more` remain dense visual summaries; multi-day runs welded
  by corner geometry with a one-line 14px title summary and visible member name
  on every day; keyboard grid navigation; weekend, today and selected-day
  emphasis.
- Month data at `lg+` only: widen the query range to the grid range so
  adjacent-month cells can show events. Gated at 1024px because the range is
  computed module-wide and `MobileMonthlyView` also renders adjacent-month
  days, so an ungated change would alter mobile.
- Schedule at `lg+`: date gutter plus full-width event rows carrying the
  member's name; event-free stretches rendered as explicit gap rows;
  de-emphasised past day groups.
- Both views at `lg+`: loading skeletons, empty and error states.
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
- App-wide dark-theme tokens, provider wiring and theme switching. The shipped
  app exposes only the light theme; this story verifies contrast there.
- Backend changes.

## Baseline drift

Re-verified 2026-07-21: FE #293 is still open with no linked PR and no branch.

The Issue contract pins frontend `origin/main` at `f9dc7e8`. Three commits have
landed since — FE #292 (`ws` bump), #296 (event form time-label clipping) and
#298 (`DialogContent` viewport bound) — moving `origin/main` to `e74c5c7`.
Refresh the pinned SHA in the Issue before starting, and branch from the fresh
baseline rather than the stale commit.

The twelve byte-identical preservation pairs are captured against whatever
`origin/main` is at execution time. All three superseding commits touch dialog
chrome or dependencies, not the Month, Schedule, Week or Day surfaces the pairs
photograph, so the preservation gate is expected to hold unchanged — but
re-capture the baselines from the fresh commit, do not reuse pre-drift images.

Note that #296 and #298 land inside this story's declared Out of Scope
("event create/detail/edit/recurrence dialogs"). They neither satisfy nor
reduce any acceptance criterion here.

## Dependencies

- [Large-screen foundations](large-screen-foundations.md) — shipped, FE PR #282.
- [Large-screen Calendar](large-screen-calendar.md) — shipped, FE PR #285.
- No backend dependency. The Month range change reuses the existing
  `GET /calendar/events` contract with different date parameters.

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
- [ ] Visual slot capacity is derived from measured row height, not hardcoded.
      Capacity never decreases as row height grows, and the measured 1440x900
      five-week row yields greater capacity than the 1024x768 six-week row.
      Absolute counts are confirmed at the screenshot gate rather than fixed
      here.
- [ ] Activating a day with events opens a popover listing all that day's
      events; `+N more` accurately reports hidden events, and selecting an
      event in the popover opens the existing event detail modal.
- [ ] Multi-day runs render with rounded outer corners only at true start and
      end, square inner corners throughout, and the complete title string in a
      one-line 14px summary on every day, including across a week boundary.
      The summary may ellipsize visually; the full title remains available in
      the day popover.
- [ ] At `lg+`, adjacent-month cells display events when events exist on those
      dates; below 1024px the query range and those cells are unchanged.
- [ ] Month grid is one tab stop with arrow-key day navigation and the
      accessible names specified in the spec.
- [ ] Schedule at 1440x900, 1920x1080 and 2560x1440 spans the available width
      with no centred narrow column; the date gutter carries label, date and
      event count; rows carry the member's name and constrain long text inside
      the row rather than capping the whole Schedule surface. Event titles are
      20px and secondary metadata is at least 14px.
- [ ] Event-free stretches render as one explicit gap row.
- [ ] Schedule's whole-view empty state tells apart a family with no members,
      filters hiding all events in the rendered window, and a genuinely empty
      14-day window. A defensive no-selection branch is present, while the
      shipped cross-view filter reset remains a documented follow-up. The
      zero-member branch is component-tested: the authenticated app correctly
      routes a real zero-member family to onboarding before Calendar mounts,
      so an app-level Calendar screenshot would be artificial.
- [ ] Mobile Schedule rendering is byte-identical to `origin/main`.
- [ ] Schedule still renders offsets 0 through 13 and Previous/Next still move
      exactly 7 days, preserving the existing 50% overlap.
- [ ] Month and large-screen Schedule render loading skeletons and recoverable
      error states; Month waits for persisted-query restoration before deciding
      that an offline range is uncached, and a cached empty response still
      renders the normal grid.
- [ ] Escape restores the originating Month cell, outside pointer dismissal
      keeps the newly clicked target, and closing Event Detail restores the
      event's originating cell. New motion respects reduced-motion preference.
- [ ] Text meets WCAG AA in the currently supported light theme, including
      missing-member, adjacent-month, and de-emphasised Schedule past-day event
      content; colour is never the only visible identity channel.
- [ ] Mobile Calendar, the 769-1023px range, and Week and Day views are all
      unchanged.
- [ ] Screenshot review per the spec matrix, iterated before done.

## Notes

Deliberately preserved, and recorded as known quirks in the spec rather than
silently changed: Schedule's 14-day window with a 7-day paging step (a 50% page
overlap) and its constant `"Upcoming"` toolbar label, which is shared with the
mobile toolbar.

Three constraints worth carrying into the implementation Issue, all confirmed
by the end-to-end contract review:

- **The 44px product rule remains intact.** In-grid Month chips and `+N more`
  are 28px visual summaries, not separate interactive controls. The day
  gridcell is the single large target and opens the full-day popover; popover
  actions and Schedule rows remain at least 44px, and adjacent day cells keep
  the PRD's 8px minimum spacing. This preserves useful Month density without
  weakening the PRD's touch-first contract for the four-year-old persona.
- **The Schedule surface has no fixed outer width cap.** The product targets a
  dedicated 20-inch-or-larger display and the PRD requires scaling through
  2560px. Long text is constrained inside event rows, so readability does not
  recreate the centred dead margins that motivated this story.
- **The repo's Vitest setup cannot exercise the lg+ measured-height path.**
  `src/test/setup.ts` mocks `ResizeObserver` as a no-op and `matchMedia` as
  `matches: false` for every query, and jsdom has no layout engine. Capacity
  and slot planning must therefore be proven through pure exported helpers, and
  the observer-to-render wiring through Playwright.
