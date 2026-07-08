# Large-Screen Calendar - Design Spec

**Date:** 2026-07-06
**Status:** Draft for review
**Story:** `docs/product/backlog/large-screen-ux/large-screen-calendar.md`
**Depends on:** `2026-07-06-large-screen-foundations-design.md`
**Scope:** Week and Day views on large screens (1024px+). Month and Schedule
views only inherit the foundations chrome. Mobile Calendar unchanged.

## 1. Context

Calendar is the flagship module and the worst offender on large screens.

**Week view** at 1440x900: the app header, view-switcher row, date-nav row,
~170px-tall day headers (day name, oversized date numeral, member dots, TODAY
pill), and the all-day row consume roughly 55% of the viewport. Hour rows are a
fixed 80px (`h-20`), so about four hours of the week are visible. The view
cannot answer "what does this week look like?" without scrolling.

**Day view** is one wide canvas where overlapping events split into ad-hoc
side-by-side columns. On a laptop it renders a few giant pastel rectangles with
enormous empty space: the most screen real estate in the app carrying the least
information.

## 2. Design Goal

Week view should read as the shape of the week at a glance: which days are
busy, which are clear, whose events they are. Day view should read as the
family's day side-by-side: each member's lane scannable top to bottom, with
spare width put to work instead of left blank. Both remain touch-first: a child
at a kitchen tablet can find their color, their name, and their next event.

## 3. Week View: Zoom Out

- **One-line day headers.** "Mon 6" plus inline member dots, today marked with
  the accent ring/fill on the date numeral. No stacked day-name / large-numeral
  / dots / TODAY-pill tower. Header height drops from ~170px to roughly 44px.
- **Denser hour rows on large screens.** Hour row height drops from 80px to
  48-56px on `lg+` (exact value tuned during screenshot review), putting 12-14
  hours on screen. Event blocks keep readable two-line title + time at this
  density; below that they truncate gracefully.
- **Auto-scroll to the useful window:** the current time when the visible
  week includes today, otherwise the earliest event across the visible week.
- **Merged toolbar** per foundations: view switcher, date navigation, and
  member filters in one row.
- Event block anatomy (member color bar, title, time, recurrence glyph) is
  unchanged; this pass is density and chrome, not a re-skin.

## 4. Day View: Member Lanes + Mini-Month Rail

### 4.1 Member lanes

- Shared time axis on the left. One vertical lane per family member, each with
  a lane header: avatar, name, member color, in the existing family order.
- Every event has exactly one owning member in the current data model
  (`CalendarEvent.memberId` is required), so lanes partition the day cleanly.
  If a shared/family-wide event concept ships later, an "Everyone" lane is the
  natural extension — follow-up, not this story.
- Lanes replace the current ad-hoc overlap columns on `lg+`. An event lives in
  its member's lane; overlaps within one member's lane stack the existing way.
- Empty lanes still render (a clear lane is information), but at minimum
  comfortable width.
- All-day events render in an all-day band above the time grid, each chip
  aligned under its member's lane.
- Hour row height follows the Week view's large-screen density.

### 4.2 Mini-month rail

Small families leave real horizontal space unused even with lanes. That spare
width hosts a **mini-month rail** (~300px) on the right:

- A compact month grid: member-colored dots on days with events, today ringed,
  the currently viewed date highlighted.
- Tapping a day stays in Day view and navigates to that date. It is a
  navigator, not a second calendar: no event titles, no extra panels.

**Show/hide mechanics:**

- Auto-shows only when spare width genuinely exists: viewport width minus
  (time axis + minimum comfortable lane width x lane count) leaves room for
  the rail. A 3-member family on a 13-inch laptop gets it; a 6-member family
  on the same screen does not, because lanes need the space more.
- A toolbar toggle can hide it regardless; the preference persists in the
  calendar Zustand store (UI state only).
- The rail appearing or disappearing only releases width back to the lanes;
  it never reflows the lanes into a different design.
- Never shown on mobile.

## 5. Month and Schedule Views

Inherit the foundations chrome (slim header, merged toolbar) and nothing else
in this pass. Any deeper Month/Schedule composition work is a follow-up story.

## 6. Alternatives Considered

### Multi-pane planning workspace - Rejected

A Google-Calendar-style desktop shell: persistent mini-month plus filter
sidebar across all views, inline event detail pane, peek panels for Meals and
Chores. Uses width most aggressively, but it turns the shared family screen
into admin software, duplicates the summarize-and-route role the Home hub spec
reserves for Home, and is a far heavier build than a polish pass. The Day
view's mini-month rail deliberately borrows the one high-value element of this
concept without adopting its posture.

### Day view as a single zoomed canvas - Rejected

Keeping the current single-column Day canvas and just tightening density. It
stays structurally poor: overlap columns are arbitrary rather than meaningful,
and it answers "what is happening" but not "who is doing what," which is the
family question.

## 7. Accessibility

- Lane headers expose accessible names ("Ethan's schedule"); color is never
  the only member identifier.
- Mini-month days expose full date labels and at-least-44px targets.
- Denser week rows must keep event text at readable sizes (no sub-12px text).
- Keyboard: mini-month is focusable and arrow-navigable for laptop use;
  auto-scroll does not steal focus.
- Reduced motion respected for any scroll or rail transitions.

## 8. Acceptance Criteria

- [ ] Week view at 1440x900 shows at least 12 hour rows with one-line day
      headers and a single toolbar row.
- [ ] Week view auto-scrolls to the current time (today) or first event.
- [ ] Day view on `lg+` renders one lane per member with avatar + name +
      color headers.
- [ ] Events appear in the correct member lane; within-lane overlaps still
      stack correctly; all-day events render in the all-day band aligned to
      their member's lane.
- [ ] Mini-month rail auto-shows only when lane math leaves spare width, can
      be hidden via a persisted toolbar toggle, and navigates Day view on tap.
- [ ] Month and Schedule views inherit the foundations chrome unchanged
      otherwise.
- [ ] Mobile Calendar (all views) is visually and behaviorally unchanged.
- [ ] Screenshot review at 1024px, 1440px, and a large-display width covers:
      busy week, sparse week, 3-member day with rail, 6-member day without
      rail, and long event titles. Issues found are iterated before done.

## 9. Out of Scope

- Month/Schedule composition redesign.
- Drag-to-create, pinch-to-zoom, or other gesture stories.
- Event detail/edit dialog changes.
- Home hub (issue #278).
- Backend changes.
