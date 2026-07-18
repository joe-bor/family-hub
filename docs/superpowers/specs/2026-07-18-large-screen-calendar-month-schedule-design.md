# Large-Screen Calendar: Month and Schedule - Design Spec

**Date:** 2026-07-18
**Status:** Draft for review
**Story:** `docs/product/backlog/large-screen-ux/large-screen-calendar-month-schedule.md`
**Builds on:** `2026-07-06-large-screen-foundations-design.md` (shipped),
`2026-07-06-large-screen-calendar-design.md` (shipped)
**Scope:** Month and Schedule calendar views at 1024px and above. Mobile
Calendar and the 769-1023px range are unchanged.

## 1. Context

The large-screen epic shipped in FE `0.3.23`. Week received a density and
chrome pass; Day received member lanes plus an optional mini-month rail. Month
and Schedule inherited the foundations chrome and nothing else, by explicit
decision: the shipped Calendar spec Section 5 states that deeper Month and
Schedule composition work is "a follow-up story." This is that story. It is not
a reopening of the shipped epic, which remains closed.

Inspection of the current implementation found the following.

### 1.1 Month (`views/monthly-calendar.tsx`)

Mobile routes to a separate `MobileMonthlyView`, so `MonthlyCalendar` never
renders at or below 768px.

- The grid is `grid-cols-7` with `min-h-[60px] sm:min-h-[100px]` cells and
  `shrink-0` inside an `overflow-auto` wrapper. It does not fill height. At
  1440x900 the month occupies roughly 600px and the remaining viewport is
  empty.
- `eventsToShow = isMobile ? 2 : 3`. The desktop value is a constant at 1024,
  1280 and 1440 alike, so density never responds to the space available. The
  `isMobile` branch is dead code: the component never renders below 769px, so
  the value is always 3 and the `2` has never reached a user. Removing
  `useIsMobile()` from this component is therefore safe.
- `+N more` is a plain `<div>`. It is not interactive. The only route to hidden
  events is clicking the cell, which leaves Month entirely.
- The day cell is a `<div onClick>`: not focusable, no role, no accessible
  name, no keyboard path.
- Member dots are 12px with a `title` attribute only.
- Multi-day events render as one independent, fully rounded chip per day, each
  repeating the title. Nothing marks a run's start or end.
- All-day events are marked by a literal `"● "` prefix concatenated into the
  title string. Recurring events carry no glyph, because Month does not use
  `CalendarEventCard`.
- Today is ringed and tinted. There is no selected-day state and no weekend
  treatment.
- The query range is `startOfMonth`-`endOfMonth`
  (`calendar-module.tsx`), but the grid renders leading and trailing days from
  adjacent months. Those cells are structurally incapable of showing events.
- There is no test file for the component.

### 1.2 Schedule (`views/schedule-calendar.tsx`)

`ScheduleCalendar` is rendered from three call sites in `calendar-module.tsx`:
the mobile switch (`case "schedule"` and its `default`) and the desktop switch.
One component serves both. It is also the mobile smart default for first-time
users, making it the highest-traffic mobile surface in the app.

- `mx-auto max-w-3xl` centres a 768px column, leaving roughly 600px of dead
  margin at 1440. This is a live violation of foundations spec Section 3.3,
  which states that a narrow centred column with dead margins "is no longer an
  acceptable default for any module index screen."
- The window is a fixed 14 days from `currentDate`. Days with zero events are
  dropped from the output entirely, so a multi-day quiet stretch renders as two
  adjacent populated days with no indication of the gap.
- Today is tinted and Tomorrow is labelled. Past days receive no distinct
  treatment.
- Prev/next moves the date by 7 days while the window spans 14, so every page
  repeats half the previous page. The toolbar label for this view is the
  constant string `"Upcoming"` (`utils/context-label.ts`), so paging produces
  no visible change of context.
- The empty state fires only when all 14 days are empty, and renders a locally
  duplicated inline `Calendar` SVG rather than the lucide icon. Its copy
  ("Events for the next 2 weeks will appear here") is also wrong for the
  filter-induced empty case, which fires when every member is deselected.
- `calendar-module.tsx` requests `currentDate` to `addDays(currentDate, 14)`,
  which is 15 days, while the component renders `i = 0..13`. One extra day is
  fetched and discarded. Harmless, and left alone, but recorded so the window
  arithmetic in Section 5.2 is not read as contradicting the query.
- `views/schedule-calendar.test.tsx` has 7 tests, three of which assert the
  combined banner strings that Section 5.1 dismantles at `lg+`.

### 1.3 Shared

Loading and error states live once in `renderCalendarView()` as centred plain
text shared by all four views. There are no skeletons. `utils/day-rail.ts`
already exports `buildMonthMatrix()` and `selectMonthDayDots()`; Month
duplicates the matrix logic inline.

## 2. Design Goal

Month should answer "what does this month look like, and what is on any given
day" without leaving the view, using the whole viewport it occupies. Schedule
should answer "what is coming up, in order" while using the width it has been
given. Both stay a household board on a shared screen, not a planning
workspace: no new panes, no new navigation model, no controls that only make
sense with a mouse.

## 3. Breakpoint and Risk Model

| Viewport | Month | Schedule |
| --- | --- | --- |
| `<= 768px` | `MobileMonthlyView`, untouched | Unchanged rendering |
| `769-1023px` | Current rendering, untouched | Current rendering, untouched |
| `>= 1024px` | Adaptive grid (Section 4) | Gutter composition (Section 5) |

Note the boundary is inclusive: `useIsMobile()` tests `max-width: 768px`, so at
exactly 768px the mobile views render. The tablet range is 769-1023px, matching
foundations spec Section 3.4. Screenshot gates must use 769px (or wider) to
capture the tablet composition; a capture at exactly 768px shows the mobile
view.

Every change in this spec, **including the Month query-range change in Section
4.6**, is gated at 1024px. Nothing below that boundary changes in rendering or
in the data it is given.

Month and Schedule ship as two separate atomic commits on one branch, Month
first. Because `ScheduleCalendar` is shared with mobile, every Schedule change
is gated on `useIsLargeScreen()` (`LARGE_SCREEN_BREAKPOINT = 1024`), and the
Schedule commit is not complete until mobile rendering has been proven
unchanged by screenshot hash comparison against `origin/main`, the method used
for the shipped Calendar work in FE PR #285.

## 4. Month View

### 4.1 Grid sizing

At `lg+` the grid fills the height available to it instead of sizing to
content.

- Row height = (container height - weekday header) / week count, where week
  count is **4, 5 or 6**. Four occurs for a non-leap February beginning on a
  Sunday: `buildMonthMatrix` pads only to a multiple of seven, so 28 days from
  a Sunday produce exactly four rows. February 2026 is such a month, and is the
  only one between 2024 and 2030 — which makes it exactly the case that ships
  broken and is found late. It is a required entry in the screenshot matrix and
  in the capacity unit tests.
- A floor of `MONTH_MIN_ROW_HEIGHT = 96px` applies. If the computed height is
  below the floor, the floor wins and the grid scrolls.
- Container height is observed with a `ResizeObserver` on the grid wrapper. The
  observed height feeds the capacity calculation in Section 4.2. The observer
  must write row height to state only when the value changes, so that
  observation cannot re-trigger itself.

### 4.2 Cell slot capacity

Capacity is counted in **slots**, not events, because Section 4.3 can reserve a
slot that holds no event. Slot capacity is derived from measured row height,
never hardcoded.

```
usableHeight = rowHeight - MONTH_NUMERAL_BLOCK - MONTH_CELL_PADDING_Y
slotCapacity = floor((usableHeight + MONTH_CHIP_GAP)
                     / (MONTH_CHIP_HEIGHT + MONTH_CHIP_GAP))
```

Starting constants, tunable during the screenshot gate in the same way
`DENSE_HOUR_ROW_HEIGHT` was tuned for the shipped Week work:

| Constant | Value | Note |
| --- | --- | --- |
| `MONTH_NUMERAL_BLOCK` | 20px | |
| `MONTH_CELL_PADDING_Y` | 10px | |
| `MONTH_CHIP_HEIGHT` | 28px | Floor, see Section 7 |
| `MONTH_CHIP_GAP` | 2px | |
| `MONTH_MIN_ROW_HEIGHT` | 96px | |

**Required slots for a cell.** Let `k` be as defined in Section 4.3 rule 4
(the highest index in the week row's multi-day list covering this day), or
`-1` when no multi-day run covers this day. Then:

```
requiredSlots = (k + 1) + singleDayCount
```

`(k + 1)` covers the multi-day runs plus any blank slots reserved beneath them.

**Rendering rule.**

- If `requiredSlots <= slotCapacity`, render every slot.
- Otherwise render the first `slotCapacity - 1` slots and put a `+N more`
  button in the last slot, where **`N` is the number of events not rendered** —
  counted over events only, so reserved blank slots never inflate it. `N` is
  therefore computed from what was actually rendered, not by subtracting from
  `eventCount`.

The `MONTH_MIN_ROW_HEIGHT` floor guarantees `slotCapacity >= 2` at every
viewport where this composition is active, so `slotCapacity - 1` is always at
least 1 and the "no chips, only an overflow row" case cannot occur.

**Testability.** The formula lives in a pure exported helper,
`monthSlotCapacity(rowHeight)`, alongside a pure `planCellSlots()` that takes
the week row's multi-day list, the day, its single-day events and the slot
capacity, and returns the ordered slot plan plus the overflow count. Both are
unit-tested directly. This matters because the measured-height path is **not**
reachable in the repo's Vitest environment as configured:

- `src/test/setup.ts` mocks `ResizeObserver` as a no-op, so its callback never
  fires and a measured height never reaches state.
- jsdom has no layout engine, so element heights are 0.
- `src/test/setup.ts` mocks `matchMedia` to return `matches: false` for every
  query, so `useIsLargeScreen()` is `false` in every unit test unless the test
  overrides it explicitly.

Consequently: the formula and slot planning are proven by unit tests on the
pure helpers; any component test asserting lg+ behaviour must override
`matchMedia` for the `min-width: 1024px` query; and the observer-to-render
wiring is proven in Playwright, not Vitest.

**Expected counts.** With these constants slot capacity is expected to be
around 4 for a five-week month at 1440x900 and around 2 for a six-week month at
1024x768. **These are expectations, not requirements** — they depend on the
measured height of app header, toolbar and weekday header, so a small chrome
change shifts them.

Note honestly that the 1024x768 six-week case may land at or below today's
hardcoded 3. That is an accepted trade for chips that meet the Section 7 floor,
taller and more legible cells, and an overflow control that actually works
where today's `+N more` is inert. If the screenshot gate judges it too tight,
the levers in priority order are: reduce `MONTH_NUMERAL_BLOCK`, reduce
`MONTH_CELL_PADDING_Y`, then raise `MONTH_MIN_ROW_HEIGHT` and accept scrolling
for six-week months at that viewport. Reducing `MONTH_CHIP_HEIGHT` below the
Section 7 floor is not a lever.

### 4.3 Multi-day continuation

A multi-day event renders one chip per day, as today, but the run is welded
visually and every chip carries the full title.

- The chip's outer corner is rounded only at the run's true start and true end.
  All inner corners are square.
- Chips bleed into the cell padding and grid gap with negative horizontal
  margins so that adjacent chips physically touch. Without the bleed the weld
  shows seams at every cell boundary.
- A week boundary needs no special case **for corner geometry**: the last cell
  of a row keeps a square right edge because the run continues, and the first
  cell of the next row keeps a square left edge. It is not free for *vertical*
  alignment — see the limitation below.
- The title prints on every day of the run. There is no arrow glyph, no
  terminator glyph, and no opacity change; corner geometry is the only
  continuation signal. Opacity was rejected because dimming white-on-colour
  risks failing WCAG AA and would make opacity carry meaning.

Ordering, so that a run occupies the same visual line in each cell it touches:

1. Within a week row, multi-day events sort before single-day events.
2. Multi-day events sort by start date ascending, then end date descending,
   then title ascending. This ordering is computed once per week row and is
   identical for every cell in that row.
3. Single-day events follow, sorted by the existing `compareEventsAllDayFirst`.
4. Let `R` be the ordered list from rules 1-2 of multi-day events touching this
   week row. For a given day cell, take the highest index in `R` of an event
   that covers this day; call it `k`. The cell renders slots `0..k`, where slot
   `i` holds `R[i]` if `R[i]` covers this day and is otherwise blank. If no
   event in `R` covers this day, `k` is undefined and the cell reserves nothing.
   Single-day events then start at slot `k + 1`.

   In other words: a cell reserves blank slots only *above* the last multi-day
   run it actually participates in, and never reserves anything at all on days
   no run covers.

**Alignment guarantee and its boundary.** Because `R` is fixed per week row,
a run occupies the same slot index in **every cell within that row**, for any
number of overlapping runs. It is *not* guaranteed to hold the same index
across a week boundary, because `R` is recomputed per row.

Concrete counterexample, on a Sunday-start grid: run X covers Mar 2-4 and run Y
covers Mar 6-10, which do not overlap at all. In row one `R = [X, Y]`, so Y
sits at slot 1. In row two X is absent, so `R = [Y]` and Y sits at slot 0. Y
visibly steps up one slot at the boundary.

Making `R` grid-scoped would fix this but is worse overall: an event landing at
a high grid-wide index would force every cell it touches to reserve every slot
beneath it, wasting most of a cell for one event. The cross-row step is
therefore accepted and recorded in Section 11, and the acceptance criterion is
scoped to within a week row. Full lane packing — the spanning-bar model
rejected in Section 8 — is the real fix if this is ever observed to matter.

Rule 4 keeps runs aligned in the common household cases (zero or one multi-day
event, or several that do not interleave) without a general lane-packing model.
See Section 11 for the accepted limitation.

Accessible name for a chip that is not the run's first day includes the span,
so that the audible rendering is not ambiguous: `"Spring break, all day, day 3
of 5"`.

### 4.4 Day cell interaction

| Trigger | Result |
| --- | --- |
| Click cell background | Navigate to Day view for that date (`selectDateAndSwitchToDaily`, unchanged) |
| Click an event chip | Open `EventDetailModal` for that event (unchanged) |
| Click `+N more` | Open the day's overflow popover |
| `Enter` on a focused day with events | Open the day's overflow popover |
| `Enter` on a focused day with no events | Navigate to Day view for that date |

The overflow popover:

- Lists every event for that day, not only the hidden ones, so that the popover
  is a complete answer to "what is on this day."
- Renders each event as a button opening the existing `EventDetailModal`.
- Contains an "Open in Day view" action.
- Closes on `Escape`, on outside click, and after any action, returning focus
  to the day cell that opened it. Note that the popover has two triggers — the
  `+N more` button for pointer users and the cell itself for keyboard users —
  but one focus-return target. Radix `Popover` returns focus to its `Trigger`
  by default, so if that primitive is used, `onCloseAutoFocus` must be
  overridden to send focus to the day cell in both cases.
- Is transient. It is not the "permanent inline event-detail pane" excluded
  from the shipped epic, which describes a pane occupying width in every view.

`+N more` becomes a `<button>` with an accessible name of the form
`"Show all 6 events for March 8"`.

Mouse-click on the cell background navigates to Day view while `Enter` opens
the popover. This divergence is deliberate: it gives keyboard users a route to
every event without putting several hundred chip elements into the tab order,
and it is recorded here rather than left implicit.

### 4.5 Keyboard and screen reader

- The grid is a single tab stop using roving `tabindex`. Exactly one day cell
  carries `tabindex="0"` at any time; all others carry `tabindex="-1"`.
- `ArrowLeft` / `ArrowRight` move by one day, `ArrowUp` / `ArrowDown` by one
  week, `Home` / `End` to the first and last day of the focused week, and
  `PageUp` / `PageDown` to the previous and next month.
- Moving past a grid edge changes `currentDate` to the adjacent month and
  places focus on the day the movement landed on. Because that re-renders the
  grid and re-runs the query, the implementation must restore focus to the
  landed-on day after re-render, not reset it to the first cell.
- The day cell is a focusable `<div role="gridcell">` carrying the roving
  `tabindex`, inside `role="row"` within `role="grid"`. It is deliberately
  **not** a `<button>`: event chips inside the cell are themselves buttons, and
  a button may not contain a button. Keyboard activation is handled by an
  `onKeyDown` handler on the cell. The grid carries an `aria-label` naming the
  visible month.
- Pointer activation of the cell background uses the existing pattern, where
  chip clicks call `stopPropagation()` so they do not also trigger the cell.
- Day accessible name: `"March 8, 2026, 6 events"`, or
  `"March 8, 2026, no events"`. Today additionally carries
  `aria-current="date"`.
- Event chips inside cells carry `tabindex="-1"` and are not individual tab
  stops. They remain clickable for pointer users and are reachable for keyboard
  users through the popover.
- The `+N more` button **also carries `tabindex="-1"`**, for the same reason:
  otherwise a month with six overflow days would add six tab stops and break
  the single-tab-stop model. Keyboard users reach the popover with `Enter` on
  the cell. Its focus ring still applies after pointer activation.
- `Space` behaves identically to `Enter` on a focused day cell, and does not
  scroll the page.
- The weekday header is part of the grid, not a sibling of it: it renders as a
  `role="row"` whose seven children are `role="columnheader"`. A `role="grid"`
  containing a non-row child is invalid ARIA. Today the header is a separate
  `grid grid-cols-7` sibling of the day grid, so this is a structural change,
  not just attributes.
- A day cell's `aria-label` is authoritative for the cell. Chips inside it are
  excluded from the cell's accessible-name computation so that a busy day is
  not announced twice; the chips' own labels are used when a chip is focused
  inside the popover.
- Focus is visible on the focused day cell and on chips focused inside the
  popover.

### 4.6 Data range

For the monthly view **at `lg+` only**, the query range widens from
`startOfMonth(currentDate)` - `endOfMonth(currentDate)` to
`startOfWeek(startOfMonth(currentDate))` - `endOfWeek(endOfMonth(currentDate))`,
matching the range the grid actually renders. This uses the same endpoint and
the same request shape; only the two date parameters change. It fixes leading
and trailing cells that can currently never show an event, which the taller
cells of Section 4.1 would otherwise make more conspicuous.

The `lg+` gate is required, not cosmetic. `dateRange` is computed in
`calendar-module.tsx` for every breakpoint, and `MobileMonthlyView` also renders
adjacent-month days (dimmed via its `isCurrentMonth` check). Widening the range
unconditionally would therefore populate previously-empty cells **on mobile**,
which this story's contract forbids. Gating on `useIsLargeScreen()` ties the
data range to exactly the breakpoint where the new composition activates, so
composition and data change together and everything below 1024px is untouched.

The consequence is that `dateRange` gains `isLargeScreen` as a dependency, so
the query key differs across the 1024px boundary and crossing it triggers one
refetch. Both ranges stay cached; this is accepted.

`startOfWeek` and `endOfWeek` use `weekStartsOn: 0`, matching the existing
Week view and the Sunday week start confirmed in PRD Section 7.1.1.

### 4.7 Emphasis and markers

- Weekend cells receive a distinct background tint from weekday cells.
- Today keeps its ring and tint and gains `aria-current="date"`.
- **"Selected day" means the day currently holding the grid's roving
  `tabindex`**, and nothing else. It is component-local state; no new Zustand
  store state is introduced, and `currentDate` continues to mean "the month
  being viewed" for this view. The selected day receives a ring visually
  distinct from today's, so that today and selected are separable both when
  they are different days and when they are the same day.
- Days outside the visible month keep their reduced emphasis and are announced
  as such in their accessible name: `"February 26, 2026, no events, outside
  March"`.
- The `"● "` string prefix is removed from all-day titles and replaced by a
  rendered marker element with an accessible label, so the marker is not part
  of the title text.
- Recurring events gain the `Repeat` glyph already used by `CalendarEventCard`
  in Week and Day, with an accessible label.
- Member identity in a cell is conveyed by chip colour plus the member's name
  in the chip's accessible name, so colour is never the only channel.
- The per-day member dots are retained — on an overflowing day they summarise
  every member with an event, which the visible chips alone cannot — but their
  `title`-only labelling is replaced. `title` is not a reliable accessible name
  and is unavailable to touch users, which leaves the dots colour-only today.
  The dot group gains a single visually-hidden summary of the form
  `"Ethan, Joe and Partner have events"`, and the individual dots become
  `aria-hidden`.

## 5. Schedule View

All changes in this section are gated on `useIsLargeScreen()`. Below 1024px the
component renders exactly as it does today.

### 5.1 Composition

At `lg+` the per-group full-width date banner is replaced by a fixed left
gutter beside the event rows.

- Two-column grid: gutter at `SCHEDULE_GUTTER_WIDTH = 168px`, events filling
  the remaining width. The value is tunable during the screenshot gate.
- The gutter carries the relative label (`Today`, `Tomorrow`, or the weekday
  name), the date, and an event count for that day.
- The gutter is sticky within its own day group while that group is scrolling,
  replacing the current `sticky top-0` banner behaviour.
- The `mx-auto max-w-3xl` cap is removed at `lg+`, resolving the foundations
  Section 3.3 violation. A maximum content width of
  `SCHEDULE_MAX_WIDTH = 1400px` prevents rows from becoming unreadably wide on
  very large displays.
- Event rows keep their existing anatomy (colour left border, title, time,
  location, member avatar) and gain the member's **name** as visible text.

**The coloured left border does not currently render, and this story fixes it.**
`schedule-calendar.tsx` passes both `colorMap[member.color].bg` and
`colorMap[member.color].light` into `cn()`. Both are `bg-*` utilities
(`bg-[#b95443]` and `bg-[#fbe9e6]`), so `twMerge` treats them as conflicting
and keeps only the last. The `border-l-4` therefore renders in the default
border colour for every member. The fix is to pass a `border-*` colour rather
than a second `bg-*`; it is a one-line change on a surface this story is
already rebuilding, and leaving it would mean shipping a redesign that still
silently drops a member channel. Adding the member's name is what makes member
identity robust; fixing the border is what makes the colour channel real again.
- Rows keep a minimum height of 56px, above the 44px floor.

### 5.2 Gaps and empty days

Days with no events are no longer silently dropped. A consecutive run of one or
more event-free days inside the window renders as a single gap row:

- Gutter: the date range of the run, e.g. `Tue 10 - Thu 12`, or the single date
  when the run is one day.
- Body: `Nothing scheduled`.
- The gap row is not interactive and is not a tab stop.
- Runs are computed only within the rendered 14-day window and are clipped to
  it; the view never reaches outside the window to extend a run.
- A run at the start of the window (the currently focused date has no events)
  and a run at the end of the window both render as gap rows on the same terms
  as an interior run. There is no special leading or trailing case.

When every day in the window is empty, the existing whole-view empty state is
shown instead of a single 14-day gap row.

### 5.3 Past days

Day groups earlier than today are de-emphasised through the gutter label
treatment and the group's border weight only. Event row text, including title,
time and member name, stays at full contrast. Reduced opacity is not used, for
the same WCAG reason given in Section 4.3.

There is no per-event dimming based on the current time of day. That would
require a live clock and would make component tests time-dependent.

### 5.4 Preserved navigation

The 14-day window length and the 7-day prev/next step are unchanged. The
resulting 50% page overlap is a pre-existing behaviour; the contract for this
story is to preserve navigation and date-focus semantics, so it is recorded as
a known quirk rather than altered. See Section 11.

`getContextLabel` is not modified. It returns the constant `"Upcoming"` for
this view and is shared with the mobile toolbar, so changing it would change
mobile. The gutter dates introduced in Section 5.1 give large-screen users
explicit date context that the label does not carry.

### 5.5 Keyboard and screen reader

- Each day group is a `<section>` with an accessible name of the form
  `"Today, Sunday March 8, 3 events"`.
- Event rows remain buttons in the natural tab order, as today.
- Gap rows are announced as `"Tuesday March 10 to Thursday March 12, nothing
  scheduled"` and are not focusable. The abbreviated visible text
  (`Tue 10 - Thu 12`) is `aria-hidden`, so the long form is read once rather
  than the two being announced in succession.
- The gutter's relative label and date are readable text, not decorative.

## 6. Shared Work

- `MonthlyCalendar` uses the existing `buildMonthMatrix()` instead of
  duplicating the matrix logic inline. Because the function is no longer
  Day-rail-specific once Month consumes it, `buildMonthMatrix()` and
  `selectMonthDayDots()` move from `utils/day-rail.ts` to a new
  `utils/month-matrix.ts`. `utils/day-rail.ts` keeps only the rail width and
  threshold constants (`RAIL_WIDTH`, `MIN_LANE_WIDTH`, `TIME_AXIS_WIDTH`,
  `DESKTOP_NAV_WIDTH`, `RAIL_LAYOUT_SLACK`, `railThresholdPx`).

  This move touches shipped Day-view files, so the plan must cover them
  explicitly: `components/day-mini-month-rail.tsx` updates its import, and the
  `buildMonthMatrix` / `selectMonthDayDots` cases in `utils/day-rail.test.ts`
  move to a new `utils/month-matrix.test.ts`. The changes are mechanical and
  behaviour-preserving, and the existing Day rail tests must still pass
  unmodified in substance. This is a move of code that this story consumes, not
  an unrelated refactor.
- Month and Schedule each render a skeleton while `isLoading`, in place of the
  shared centred "Loading events..." text, and a per-view error state with a
  retry affordance. `renderCalendarView()`'s shared fallback continues to serve
  Week and Day unchanged.
- Schedule's locally duplicated inline `Calendar` SVG is replaced with the
  lucide `Calendar` import already used elsewhere in the module.
- Schedule's empty-state copy is corrected to describe the window it actually
  renders.
- Existing Schedule tests must be migrated, not merely kept passing.
  `views/schedule-calendar.test.tsx` asserts the combined banner strings
  `"Today — Wed, Mar 18"`, `"Tomorrow — Thu, Mar 19"` and `"Friday, Mar 20"`,
  which Section 5.1 dismantles at `lg+` by splitting label, date and count into
  the gutter. Its render helper also uses `width = isMobile ? 390 : 1024`,
  which sits exactly on the lg+ boundary. The plan must state which of those
  cases move into an lg+ describe block with `matchMedia` overridden and which
  stay asserting the unchanged mobile path.
- Offline reads are unchanged for Schedule. For Month there is one new case:
  because Section 4.6 puts `isLargeScreen` into the query key, Month has two
  distinct persisted cache entries. A user whose IndexedDB cache was populated
  below 1024px, or who crosses the boundary while offline, gets a cache miss
  and an empty Month with no network to fill it. The view must render its
  offline-empty state rather than an indefinite skeleton in that case, and the
  case belongs in the offline screenshot matrix.

## 7. Accessibility Summary

- **Touch-target rule and its one exception.** All chrome and controls — the
  toolbar, popover actions, Schedule event rows — stay at least 44px in their
  smallest dimension. In-grid Month event chips are the documented exception,
  with a floor of `MONTH_CHIP_HEIGHT = 28px`.

  This is a deliberate narrowing of the 44px rule and is called out rather than
  buried. It matches both precedent and arithmetic. Foundations spec Sections 5
  and 6 already scope the rule to chrome ("All interactive **chrome** keeps
  44px minimum touch targets"), and the shipped Week view renders event buttons
  at `min-h-[28px]` with `MIN_EVENT_PX = 30`. Arithmetically, 44px chips would
  yield a slot capacity of 2 at 1440x900 and 1 at 1024x768 — fewer events than
  ship today, which would defeat the purpose of the story. A 28px chip matches
  the shipped Week floor exactly, and every chip has a full-size alternative
  path: the day cell itself and the overflow popover, both of which meet 44px.
- Colour is never the only channel: member identity is carried in accessible
  names and in visible member names on Schedule, member dots carry a
  visually-hidden summary, all-day and recurring states have marker elements
  with labels, and continuation is carried by geometry plus an explicit
  day-of-span accessible name.
- No information is conveyed by opacity alone.
- Text contrast meets WCAG AA in both light and dark themes.
- Reduced motion is respected for the popover transition.
- Auto-focus never moves focus without a user action; the popover moves focus
  only because the user opened it, and returns it on close.

## 8. Alternatives Considered

**Month with a fixed comfortable row height - rejected.** One stable row height
at every large width is simpler to build and produces identical rhythm month to
month. It was rejected because it still leaves slack at 1440 and starts
scrolling at 1024x768 for six-week months, so it only partly addresses the
reported problem.

**Month as a pure density map - rejected.** Large numerals, member colour bars
and a per-day count, with reading moved entirely into Day view. This is closest
to PRD Section 7.1.1's description of Month, and is the most touchscreen-honest
option. It was rejected because it cannot answer "what is on Thursday" without
leaving Month, which is the question the household asks most.

**Month with true spanning bars - rejected for this story.** One continuous bar
per week row, split at week boundaries, costing one slot per week rather than
one per day. It reads best and matches mainstream calendars. It was rejected
because it requires a per-week lane-packing model with stable lane assignment,
reserves lane space in cells the event does not touch, and feeds back into the
capacity calculation of Section 4.2 — the largest build in the Month work and
the one most likely to destabilise it. Captured as a follow-up in Section 11.

**Multi-day with title once and arrow glyphs through the run - rejected.**
Printing the title only on the first day and on week-row starts is more
economical, but it produces a wordless chip on the run's final day and makes a
mid-run day unable to answer "what is on Thursday" on its own.

**Schedule as two week columns - rejected.** This week beside next week, with
the whole 14-day window visible without scrolling. It is the only option using
both width and height. It was rejected because reading order stops being
self-evident, narrow columns push rows back into a cramped layout that undoes
the width gain, unbalanced weeks leave one column nearly empty, and it diverges
furthest from the mobile rendering it shares a component with.

**Schedule with a widened column cap - rejected.** Raising `max-w-3xl` alone is
a one-line change, but a 1300px row carrying "Trash out - All day" is a worse
outcome than the dead margin it replaces.

## 9. Acceptance Criteria

Month:

- [ ] At 1440x900 the Month grid fills the available height with no dead space
      below the final week row.
- [ ] At 1024x768 a six-week month fills the available height, with the
      `MONTH_MIN_ROW_HEIGHT` floor respected.
- [ ] `monthSlotCapacity(rowHeight)` and `planCellSlots(...)` exist as pure
      exported helpers with direct unit tests; no hardcoded desktop event count
      remains in the component.
- [ ] `monthSlotCapacity` is monotonic: a larger `rowHeight` never yields a
      smaller capacity, and it is strictly larger across the 1024x768 six-week
      and 1440x900 five-week row heights. (Absolute counts are confirmed at the
      screenshot gate, not asserted here.)
- [ ] `monthSlotCapacity(MONTH_MIN_ROW_HEIGHT) >= 2`, asserted by unit test.
- [ ] `planCellSlots` never returns more slots than the capacity it was given,
      including when Section 4.3 reserves blank slots — verified by a case with
      reserved blanks plus single-day events that would overflow under a
      naive `eventCount` comparison.
- [ ] The `+N more` count equals the number of events not rendered, and is
      unaffected by reserved blank slots.
- [ ] A four-row month (February 2026) renders correctly and its capacity is
      covered by unit test.
- [ ] `+N more` opens a popover listing all of that day's events; selecting one
      opens `EventDetailModal`; `Escape` closes it and returns focus to the day
      cell.
- [ ] A multi-day run renders with rounded outer corners only at its true start
      and end, square inner corners throughout, and the full title on every
      day.
- [ ] A multi-day run crossing a week boundary keeps a square right edge on the
      last cell of the row and a square left edge on the first cell of the next.
- [ ] A multi-day run occupies the same slot index in every cell it touches
      **within a single week row**, for any number of overlapping runs. Slot
      index across a week boundary is explicitly not guaranteed (Section 4.3).
- [ ] At `lg+`, leading and trailing adjacent-month cells display events when
      events exist on those dates.
- [ ] Below 1024px the Month query range is unchanged, so mobile and tablet
      adjacent-month cells stay empty exactly as they do today.
- [ ] The day cell is not a `<button>`; no button is nested inside another
      button anywhere in the Month grid.
- [ ] The grid is one tab stop: tabbing from the toolbar reaches exactly one
      day cell, and tabbing again leaves the grid, on a month containing several
      overflowing days.
- [ ] Arrow keys, `Home`/`End` and `PageUp`/`PageDown` move the focused day as
      specified in Section 4.5, and crossing a grid edge restores focus to the
      landed-on day after the month re-renders.
- [ ] The weekday header renders as a `role="row"` of `role="columnheader"`
      cells inside the same `role="grid"` as the day rows; the grid has no
      non-row children.
- [ ] `Enter` opens the popover on a day with events and navigates to Day view
      on a day without.
- [ ] Day cells expose the accessible names specified in Section 4.5, and today
      carries `aria-current="date"`.
- [ ] Weekend, today and selected-day treatments are visually distinguishable
      from one another and from the default cell.
- [ ] All-day and recurring events carry marker elements with accessible
      labels; no `"● "` prefix remains in any title string.
- [ ] Per-day member dots expose a visually-hidden member summary; no member
      information is conveyed by `title` alone.
- [ ] Month at 769-1023px is visually and behaviourally unchanged.
- [ ] Mobile Month (`MobileMonthlyView`) is visually and behaviourally
      unchanged.

Schedule:

- [ ] At 1440x900 Schedule content spans the available width up to
      `SCHEDULE_MAX_WIDTH`, with no centred narrow column and no dead side
      margins.
- [ ] The date gutter renders the relative label, date and event count, and is
      sticky within its own day group.
- [ ] Event rows display the member's name as visible text in addition to
      colour and avatar.
- [ ] The coloured left border renders in the member's colour, verified by
      asserting the resolved class list contains a member `border-*` colour and
      no conflicting second `bg-*`.
- [ ] A run of event-free days renders as one gap row naming the range, with
      `Nothing scheduled`; it is not focusable.
- [ ] A window with no events renders the whole-view empty state, not a gap row.
- [ ] Day groups before today are de-emphasised without reducing event text
      contrast.
- [ ] The 14-day window and 7-day prev/next step behave exactly as before.
- [ ] Mobile Schedule rendering is byte-identical to `origin/main`, proven by
      screenshot hash comparison. Note that FE PR #285 found this comparison
      only stabilises once font loading is matched between the two captures;
      budget for that rather than treating an early mismatch as a regression.
- [ ] Schedule at 769-1023px is visually and behaviourally unchanged.
- [ ] Filtering out every member renders an empty state whose copy describes a
      filter-induced empty, not "events for the next 2 weeks will appear here".

Shared:

- [ ] Month and Schedule render skeletons while loading and a per-view error
      state with retry; Week and Day keep the existing shared fallback.
- [ ] All chrome and controls are at least 44px; Month event chips are at least
      `MONTH_CHIP_HEIGHT` (28px) and are the only documented exception.
- [ ] Text contrast meets WCAG AA in light and dark themes, including the
      zero-member fallback where chips render `bg-muted` with `text-white`.
- [ ] Week and Day views are visually and behaviourally unchanged, and the
      existing `day-rail.test.ts` cases still pass unmodified in substance after
      the `buildMonthMatrix` move.
- [ ] Screenshot review at 375x812, **768 (mobile-boundary check)**, **769**,
      1024x768, 1280x800 and 1440x900 covers loading, empty, error, offline
      (including the Month cold-cache case from Section 6), dense, recurring,
      multi-day, overflow, **four-row February 2026**, very long titles, and a
      family with zero members. Issues found are iterated before done.

## 10. Out of Scope

- Week and Day view composition; both shipped in FE PR #285.
- Mobile Calendar for any view.
- The 769-1023px range for any view.
- A persistent mini-month or filter sidebar across views.
- A permanent inline event-detail pane.
- Meals or Chores peek panels inside Calendar.
- Shell or navigation-rail redesign, global search, notifications, cross-module
  quick-add.
- Calendar gestures (drag-to-create, pinch-to-zoom).
- Event creation, detail, edit and recurrence dialogs.
- Generalising the Day view mini-month rail to Month or Schedule.
- Backend changes of any kind.
- Offline write support.

## 11. Known Limitations and Follow-Ups

- **Multi-day runs can step a slot at a week boundary.** Because the ordered
  list `R` is computed per week row, a run can hold a different slot index in
  the next row when another run present in the first row does not extend into
  the second. Section 4.3 carries a worked counterexample. Within any single
  week row alignment is exact for any number of overlapping runs. The fix is
  the spanning-bar model rejected in Section 8; pick it up only if this is
  observed to matter in real household use.
- **Schedule page overlap.** Paging moves 7 days through a 14-day window, so
  each page repeats half the previous one, and the `"Upcoming"` toolbar label
  never changes to signal where the user is. Both are preserved deliberately
  under this story's contract. A future story could align the step to the
  window, or give Schedule a date-range label without disturbing the shared
  mobile toolbar.
- **Loading and error state divergence.** Month and Schedule gain per-view
  skeletons and error states while Week and Day keep the shared centred text.
  This is an accepted temporary inconsistency; bringing Week and Day up to the
  same treatment is a small follow-up.
- **Adjacent-month cells stay empty below 1024px.** The data-range fix in
  Section 4.6 is gated at `lg+` so that mobile and the 769-1023px range stay
  untouched, which means both keep rendering structurally-empty leading and
  trailing cells. Fixing them requires accepting a visible mobile change and so
  belongs in its own small story, where that change can be reviewed on its own
  terms rather than riding along with a large-screen polish pass.
- **Month capacity is chrome-sensitive.** Because capacity derives from
  measured row height, a future change to app header or toolbar height silently
  shifts how many events a cell shows. This is the intended behaviour, but it
  means the screenshot gate is the only thing pinning the concrete counts.

## 12. Delivery Notes

Two atomic commits on one branch, Month first, each with its own screenshot
review before it is considered done. The Schedule commit must include the
mobile parity evidence described in Section 3 before it is called complete.
