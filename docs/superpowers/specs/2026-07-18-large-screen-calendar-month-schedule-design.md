# Large-Screen Calendar: Month and Schedule - Design Spec

**Date:** 2026-07-18
**Status:** Reviewed; implementation-ready
**Story:** `docs/product/backlog/large-screen-ux/large-screen-calendar-month-schedule.md`
**Plan:** [Implementation plan](../plans/2026-07-18-large-screen-calendar-month-schedule.md)
**Builds on:** `2026-07-06-large-screen-foundations-design.md` (shipped),
`2026-07-06-large-screen-calendar-design.md` (shipped)
**Scope:** Month and Schedule calendar views at 1024px and above. Mobile
Calendar and the 769-1023px range are unchanged.

### Review amendments (2026-07-20)

The story, spec, plan, frontend `origin/main`, and Issue #293 were reviewed as
one execution chain. Thirteen material corrections were made before implementation:

1. The measured weeks wrapper is a valid `role="rowgroup"`, not an unroled
   child forbidden by the earlier acceptance wording. WAI-ARIA permits rows to
   be owned through a rowgroup, which preserves both valid grid semantics and
   the required observation boundary.
2. Month's 28px chips and overflow label are visual summaries, while the whole
   day gridcell is the single 44px-or-larger control. This preserves density
   without relaxing the PRD's touch-target requirement for the child persona.
3. The chip weld uses the rendered border, padding and column gap as one
   geometry model and cells do not clip the chip layer. The former plan's 4px
   bleed inside `overflow-hidden` cells could not physically bridge a 4px gap.
4. Schedule remains full width through the product's 2560px target. Readable
   line length is enforced inside event rows rather than with a 1400px outer
   cap, which would leave about 1048px of unused width on the target display.
5. Adjacent-month emphasis applies to the cell chrome and date label, never by
   lowering opacity on the whole cell. The widened query now puts real event
   text in those cells, so blanket opacity would undermine the same WCAG AA
   contrast requirement the design promises everywhere else.
6. PRD Section 7.1.1 now records the large-screen Month interaction explicitly:
   an empty day opens Day view, while a populated day opens the complete day
   summary and then offers event-detail and Day-view actions. This is the
   product consequence of keeping the full day cell as the one large target;
   the older unconditional “tap day” sentence could not coexist with that
   reviewed accessibility model.
7. Both Month row and column gaps are 8px, matching the PRD's minimum spacing
   between adjacent targets. With 1px borders and 4px horizontal padding, the
   corresponding per-side weld bleed is therefore 9px, not 7px.
8. Dense Month titles are 14px, repeat on every run segment, and may truncate
   to one line; the popover exposes the complete title. Schedule event titles
   are 20px. This turns “full title on every day” into a testable content rule
   without contradicting fixed 28px slots or the PRD typography scale.
9. WCAG AA acceptance is scoped to the app's currently supported light theme.
   Frontend `origin/main` defines only `:root` colour tokens and never mounts
   its exported theme provider; implementing and validating a dark theme would
   be a separate cross-app feature, not Month/Schedule polish. Missing-member,
   adjacent-month and past-day contrast remain explicit gates in the supported
   theme, and dark-theme support is recorded as a prerequisite follow-up.
10. Zero-member and zero-selection Schedule branches remain defensive
    component states with direct component-test evidence, not fabricated
    app-level screenshots. The released API rejects registration without a
    member, and the authenticated shell routes a family with no remaining
    members to onboarding before Calendar mounts. The reachable visual matrix
    instead includes a missing-member Month response fixture to exercise the
    required fallback tokens while a valid family member keeps Calendar
    mounted.
11. PRD breakpoint ranges now classify 768px as mobile and start tablet at
    769px. The shipped `useIsMobile()` contract is `innerWidth <= 768` and this
    story explicitly preserves that boundary; the earlier PRD classified 768px
    as tablet, contradicting both the running product and its screenshot gate.
12. The PRD's canonical Calendar view model and mobile default now match the
    shipped product: Day, Week, Month, and Schedule are all first-class views;
    first-time mobile users see Schedule, then their chosen view persists per
    device. The old Day/Week/Month-only lists and “default to Day” text
    contradicted the shipped type, switcher, store/module behavior, and existing
    mobile-settings product decision. This matters here because shared
    `ScheduleCalendar` regressions affect the first Calendar view a first-time
    mobile user encounters.
13. PRD scope now distinguishes shipped cached read-only offline viewing from
    the offline writes/outbox/background-sync capability that remains out of
    scope. The former “Offline Mode requires active internet” line contradicted
    the shipped IndexedDB query persistence recorded elsewhere in the same PRD
    and roadmap, and obscured why Month restoration is an acceptance gate.

Proof: [WAI-ARIA grid pattern](https://www.w3.org/WAI/ARIA/apg/patterns/grid/),
[WCAG 2.2 target-size guidance](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum),
[frontend light-only tokens at the audited baseline](https://github.com/joe-bor/FamilyHub/blob/f9dc7e8070457f965b823baee4e3746486afa438/src/index.css),
[frontend zero-member onboarding guard at the audited baseline](https://github.com/joe-bor/FamilyHub/blob/f9dc7e8070457f965b823baee4e3746486afa438/src/App.tsx),
[frontend 768px mobile boundary at the audited baseline](https://github.com/joe-bor/FamilyHub/blob/f9dc7e8070457f965b823baee4e3746486afa438/src/hooks/use-is-mobile.ts),
[frontend mobile Schedule smart default at the audited baseline](https://github.com/joe-bor/FamilyHub/blob/f9dc7e8070457f965b823baee4e3746486afa438/src/components/calendar/calendar-module.tsx),
[frontend four-view switcher at the audited baseline](https://github.com/joe-bor/FamilyHub/blob/f9dc7e8070457f965b823baee4e3746486afa438/src/components/calendar/components/calendar-view-switcher.tsx),
[frontend Calendar view type at the audited baseline](https://github.com/joe-bor/FamilyHub/blob/f9dc7e8070457f965b823baee4e3746486afa438/src/lib/types/calendar.ts),
[frontend persisted read-cache contract at the audited baseline](https://github.com/joe-bor/FamilyHub/blob/f9dc7e8070457f965b823baee4e3746486afa438/src/lib/offline/dehydrate.ts),
[released API registration constraint](https://github.com/joe-bor/family-hub-api/blob/v1.9.0/src/main/java/com/familyhub/demo/dto/RegisterRequest.java),
PRD Sections 1, 7.1.1, 7.5.1, 9.1, 9.4, 9.5 and 11.2, and the shipped
foundations spec Section 3.3.

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
users, making compact-rendering preservation a release-critical risk even
though no traffic ranking is asserted without telemetry.

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

Month and Schedule ship as two separate phases on one branch, Month first, each
ending at its own screenshot gate. Commits within a phase are frequent; the
phase boundary, not the commit count, is what provides the risk isolation.

Because `ScheduleCalendar` is shared with mobile, every Schedule change is
gated on `useIsLargeScreen()` (`LARGE_SCREEN_BREAKPOINT = 1024`), and the
Schedule phase is not complete until mobile rendering has been proven unchanged
by screenshot hash comparison against `origin/main`, the method used for the
shipped Calendar work in FE PR #285.

## 4. Month View

### 4.1 Grid sizing

At `lg+` the grid fills the height available to it instead of sizing to
content.

- Row height = (weeks-container height - inter-row gaps) / week count, where
  week count is **4, 5 or 6**. Four occurs for a non-leap February beginning on a
  Sunday: `buildMonthMatrix` pads only to a multiple of seven, so 28 days from
  a Sunday produce exactly four rows. February 2026 is such a month, and is the
  only one between 2024 and 2030 — which makes it exactly the case that ships
  broken and is found late. It is a required entry in the screenshot matrix and
  in the capacity unit tests.
- A floor of `MONTH_MIN_ROW_HEIGHT = 96px` applies. If the computed height is
  below the floor, the floor wins and the grid scrolls.
- Height is observed with a `ResizeObserver` on the **weeks container**, not on
  the grid wrapper. The weekday header is a sibling of the weeks container and
  is not laid out by the row-height calculation, so observing an element that
  includes it would overflow the grid by the header's height and leave a
  permanent scrollbar.
- The observed element unmounts while the loading skeleton is shown, so the
  observer must be attached via a **callback ref**. An effect keyed on the week
  count alone would never re-run when the container remounts, pinning row
  height at the floor for the life of the view — and no unit test can catch it,
  because `ResizeObserver` is a no-op mock in this repo.
- The observer must write row height to state only when the value changes, so
  that observation cannot re-trigger itself.
- Semantically, that measured weeks container is `role="rowgroup"`. The grid
  therefore owns the weekday `row` and the observed `rowgroup`; the rowgroup
  owns the week `row` elements. This is the WAI-ARIA-supported structure that
  keeps the measurement boundary without inserting an unroled child in the
  grid.

### 4.2 Cell slot capacity

Capacity is counted in **slots**, not events, because Section 4.3 can reserve a
slot that holds no event. Slot capacity is derived from measured row height,
never hardcoded.

```
usableHeight = rowHeight
             - MONTH_CELL_BORDER_Y
             - MONTH_CELL_PADDING_Y
             - MONTH_NUMERAL_BLOCK
             - MONTH_HEADER_SLOT_GAP
slotCapacity = floor((usableHeight + MONTH_CHIP_GAP)
                     / (MONTH_CHIP_HEIGHT + MONTH_CHIP_GAP))
```

Starting constants, tunable during the screenshot gate in the same way
`DENSE_HOUR_ROW_HEIGHT` was tuned for the shipped Week work:

| Constant | Value | Note |
| --- | --- | --- |
| `MONTH_CELL_BORDER_Y` | 2px | Two 1px borders |
| `MONTH_CELL_PADDING_Y` | 8px | Two 4px paddings; rendered from this constant |
| `MONTH_NUMERAL_BLOCK` | 20px | Fixed-height header row; rendered from this constant |
| `MONTH_HEADER_SLOT_GAP` | 2px | Between header and first slot |
| `MONTH_CHIP_HEIGHT` | 28px | Visual slot height; not a touch target, see Section 7 |
| `MONTH_CHIP_GAP` | 2px | |
| `MONTH_MIN_ROW_HEIGHT` | 96px | |
| `MONTH_ROW_GAP` | 8px | PRD minimum between vertically adjacent targets |
| `MONTH_COLUMN_GAP` | 8px | PRD minimum between horizontally adjacent targets |

These are one geometry model, not accounting-only knobs. The cell's border,
padding, fixed numeral row and header-to-slot gap must render from these values.
Changing a capacity constant without changing the corresponding CSS would
overstate what fits and is forbidden.

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
  visual summary in the last slot, where **`N` is the number of events not
  rendered** — counted over events only, so reserved blank slots never inflate
  it. `N` is therefore computed from what was actually rendered, not by
  subtracting from `eventCount`. The summary is not a second control; activating
  the day cell opens the full list.

The `MONTH_MIN_ROW_HEIGHT` floor guarantees `slotCapacity >= 2` at every
viewport where this composition is active. It does **not** guarantee a visible
event chip: if the only covering run occupies a reserved lane at or beyond the
visible prefix, the prefix can contain a blank plus `+N more`. That rare case is
an accepted consequence of exact within-row lane alignment. It is covered by a
counterexample unit test, the day cell still announces its event count, and the
same 44px-or-larger cell opens the complete popover. The implementation must
not claim that the first visible slot always contains an event.

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
hardcoded 3. That is an accepted trade for legible visual slots and a complete
popover path where today's `+N more` is inert. If the screenshot gate judges it
too tight, the levers in priority order are: reduce the **rendered** numeral
block, reduce the **rendered** padding, then raise `MONTH_MIN_ROW_HEIGHT` and
accept scrolling for six-week months at that viewport. Reducing
`MONTH_CHIP_HEIGHT` below the Section 7 floor or changing accounting without
matching DOM geometry is not a lever.

### 4.3 Multi-day continuation

A multi-day event renders one chip per day, as today, but the run is welded
visually and every chip carries the full title.

- The chip's outer corner is rounded only at the run's true start and true end.
  All inner corners are square.
- Chips bleed across the cell border, horizontal padding and half the column
  gap so adjacent segments physically touch. With 1px borders, 4px horizontal
  padding and an 8px column gap, the per-side bleed is
  `1 + 4 + (8 / 2) = 9px`. The chip width expands by the sum of its active
  left/right bleeds; a negative margin alone does not enlarge the border box.
- Day cells keep their chip layer `overflow: visible`; the outer grid clips
  horizontal overflow. Sunday suppresses outward left bleed and Saturday
  suppresses outward right bleed, so a run that crosses a week boundary ends
  square at the row edge without causing page overflow. Browser geometry must
  assert that adjacent chip rectangles touch and that the page has no
  horizontal scrollbar.
- A week boundary needs no special case **for corner geometry**: the last cell
  of a row keeps a square right edge because the run continues, and the first
  cell of the next row keeps a square left edge. It is not free for *vertical*
  alignment — see the limitation below.
- The full title string renders on every day of the run in a one-line 14px
  visual summary and may ellipsize when the cell is narrow. The complete title
  is always available in the full-day popover. There is no arrow glyph, no
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

The dense in-grid chip is visual and `aria-hidden`; it includes the member's
visible first name before the event title so colour is secondary for sighted
users. If stale data references a member no longer in the family, the same
position reads `Unknown` and the popover action reads `Unknown member` rather
than silently dropping the identity cue. The day cell announces the date and
event count and references its visually-hidden member summary with
`aria-describedby`; the popover is the
screen-reader path to complete event and member names.
Inside the popover, a multi-day event that is not on its first day includes the
span in its accessible name: `"Spring break, all day, day 3 of 5"`.

### 4.4 Day cell interaction

| Trigger | Result |
| --- | --- |
| Click a day with events, including a chip or `+N more` | Open the day's overflow popover |
| Click a day with no events | Navigate to Day view for that date |
| `Enter` on a focused day with events | Open the day's overflow popover |
| `Enter` on a focused day with no events | Navigate to Day view for that date |

The overflow popover:

- Lists every event for that day, not only the hidden ones, so that the popover
  is a complete answer to "what is on this day."
- Renders each event as a button opening the existing `EventDetailModal`.
- Contains an "Open in Day view" action.
- Closes on `Escape`, on outside click, and before either action runs. Escape
  returns focus to the day cell. An outside pointer dismissal leaves focus on
  the newly clicked target instead of pulling it back to the old cell. Event
  selection suppresses popover focus restoration so the modal can take focus;
  closing the modal returns focus to the originating day cell. "Open in Day
  view" suppresses restoration because the Month grid unmounts. This
  close-reason contract avoids fighting the user's pointer action or the
  modal-dialog autofocus model.
- Is transient. It is not the "permanent inline event-detail pane" excluded
  from the shipped epic, which describes a pane occupying width in every view.

`+N more` is visible text inside the cell, not a nested control. The gridcell's
activation is the one pointer and keyboard path, avoiding overlapping small hit
targets while keeping the dense visual summary.

### 4.5 Keyboard and screen reader

- The grid is a single tab stop using roving `tabindex`. Exactly one day cell
  carries `tabindex="0"` at any time; all others carry `tabindex="-1"`.
- `ArrowLeft` / `ArrowRight` move by one day, `ArrowUp` / `ArrowDown` by one
  week, `Home` / `End` to the first and last day of the focused week, and
  `PageUp` / `PageDown` to the previous and next month.
- Initial roving selection is `currentDate`. Pointer activation makes that day
  the roving selection. Toolbar Previous, Next, Today, or another external
  `currentDate` update resets the roving selection to the new `currentDate`,
  closes an open Month popover, and **does not move DOM focus away from the
  toolbar or external control**. Only an in-grid keyboard traversal requests
  focus after render.
- Moving past a grid edge changes `currentDate` to the adjacent month and
  places focus on the day the movement landed on. Because that re-renders the
  grid and re-runs the query, the implementation must restore focus to the
  landed-on day after re-render, not reset it to the first cell.
- The day cell is a focusable `<div role="gridcell">` carrying the roving
  `tabindex`, inside `role="row"` within a `role="rowgroup"` owned by
  `role="grid"`. It is deliberately **not** a `<button>` so it can carry grid
  semantics. Dense chips and `+N more` are presentational children, not nested
  controls. Keyboard activation is handled by `onKeyDown`; pointer activation
  is handled by the cell. The grid carries an `aria-label` naming the visible
  month.
- Day accessible name: `"March 8, 2026, 6 events"`, or
  `"March 8, 2026, no events"`. Today additionally carries
  `aria-current="date"`.
- Event chips and `+N more` are `aria-hidden` visual content, never individual
  tab stops or pointer targets. Keyboard and pointer users both activate the
  day cell and then use the popover's 44px-or-larger event actions.
- `Space` behaves identically to `Enter` on a focused day cell, and does not
  scroll the page.
- The weekday header is part of the grid, not a sibling of it: it renders as a
  `role="row"` whose seven children are `role="columnheader"`. Its semantic
  sibling is the observed `role="rowgroup"`, whose children are the week rows.
  The grid therefore owns only a row or rowgroup, with every gridcell owned by
  a row. Today the header is a separate `grid grid-cols-7` sibling of the day
  grid, so this is a structural change, not just attributes.
- A day cell's `aria-label` is authoritative for the cell. Chips inside it are
  excluded from the cell's accessible-name computation so that a busy day is
  not announced twice; each interactive event row inside the popover has its
  own complete accessible name.
- Focus is visible on the focused day cell and on event/Day actions inside the
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

The consequence is that `dateRange` gains `isLargeScreen` as a dependency.
TanStack Query keys on the resulting `{startDate, endDate}` parameters, not on
the breakpoint boolean itself. The key therefore differs only when the compact
and expanded ranges differ; a grid-aligned month such as February 2026 reuses
one key. A cached query may also satisfy the expanded range without a network
request. Both behaviours are accepted and are tested explicitly.

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
- Days outside the visible month keep reduced cell chrome and a muted date
  label and are announced as such in their accessible name: `"February 26,
  2026, no events, outside March"`. The cell itself never receives reduced
  opacity: adjacent-month event chips, markers and member summaries stay at
  full contrast now that Section 4.6 supplies real data for those dates.
- The `"● "` string prefix is removed from all-day titles and replaced by a
  rendered visual marker. The dense marker is `aria-hidden`; the popover event
  action includes "all day" in its accessible name.
- Recurring events gain the `Repeat` glyph already used by `CalendarEventCard`
  in Week and Day. It is also `aria-hidden`; the popover event action includes
  "repeats" in its accessible name.
- Member identity in a cell is conveyed by chip colour plus the day-cell dot
  summary, and the popover shows/names each member, so colour is never the only
  channel.
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
  Section 3.3 violation. The day group and event-row surface stay full width
  through the PRD's 2560px target; there is no fixed outer cap that recreates
  centred dead margins. Event-row text uses a readable internal measure
  (`max-width: 72ch`) while time/member affordances can use the remaining row
  width.
- Event rows keep their existing anatomy (colour left border, title, time,
  location, member avatar) and gain the member's **name** as visible text.
  Large-screen event titles use the PRD's 20px Calendar-event size; secondary
  time, location and member metadata uses at least 14px.

**The coloured left border does not currently render, and this story fixes it.**
`schedule-calendar.tsx` passes both `colorMap[member.color].bg` and
`colorMap[member.color].light` into `cn()`. Both are `bg-*` utilities
(`bg-[#b95443]` and `bg-[#fbe9e6]`), so `twMerge` treats them as conflicting
and keeps only the last. The `border-l-4` therefore renders in the default
border colour for every member. The fix is to set the border colour from the
member's `hex` via an inline style, which `twMerge` cannot collapse.

**The fix applies to the `lg+` branch only.** Applying it to the shared compact
path would make a member-coloured border appear on mobile where a grey one
ships today — a visible mobile change, which this story's contract forbids, and
one that would also destroy the byte-identical parity check that guards the
rest of the Schedule work. The compact path keeps the bug; see Section 11.
Adding the member's name is what makes member identity robust at `lg+`; fixing
the border is what makes the colour channel real alongside it.
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

Visible and accessible range labels must disambiguate boundaries: same-month
ranges may use `Tue 10 – Thu 12`; cross-month ranges include both month names;
cross-year ranges include both years. The accessible label always uses full
weekday, month, day and year for both endpoints.

When every day in the window is empty, the existing whole-view empty state is
shown instead of a single 14-day gap row.

Whole-view copy identifies why the visible result is empty:

- no family members: `No family members yet` with an add-member prompt;
- if a zero-selection state is delivered defensively:
  `Select at least one profile to view events`, matching PRD Section 7.1.2,
  with a choose-member prompt. The shipped filter pills immediately repair an
  empty selection to “all,” so making zero selection persist is not claimed by
  this story and remains the cross-view follow-up in Section 11;
- events exist in the 14 rendered days but the active member/all-day filters
  hide them all: `No events match your filters` with an adjust-filter prompt;
- no events exist in the rendered window: `No upcoming events` with the honest
  14-day-window description.

The filter-induced case is based on events in the rendered window before
filters are applied. It must not treat an event on the query's inclusive
fifteenth boundary day as evidence for the 14-day rendered window.

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
- Schedule's compact inline `Calendar` SVG and empty-state markup remain
  byte-identical. The new large branch uses the shared `CalendarEmptyState`;
  replacing the compact SVG would invalidate the parity contract.
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
- Offline reads are unchanged for Schedule. Month distinguishes a real cached
  empty response from no query data **after persisted-query restoration has
  completed**. `eventsResponse === undefined` while `useIsRestoring()` is true
  is an indeterminate hydration state and shows the skeleton, never cold-cache
  copy. Only restoration-complete plus absent query data is a cold cache.
  `events.length === 0` is not a cache-presence test. April 2026 covers a
  differing compact/expanded key, February 2026 covers key reuse, and a cached
  empty response must render the normal empty grid rather than "isn't cached"
  copy. An offline cold-start test must prove persisted data wins after
  asynchronous restoration without flashing the uncached message.

## 7. Accessibility Summary

- **Touch-target rule.** Every interactive element stays at least 44px in its
  smallest dimension. Month's dense 28px chip and `+N more` slot are visual
  summaries rather than controls; the 96px-or-taller day gridcell is the single
  target and opens the full-day popover. Popover actions and Schedule event rows
  are at least 44px, and adjacent day targets are separated by 8px in both
  axes. This retains useful density without weakening the PRD's stricter
  touch-first requirement. WCAG 2.5.8 would permit a 24px target or an
  equivalent-control exception, but Family Hub deliberately keeps its stronger
  product rule.
- Colour is never the only channel: visible member names prefix Month chip
  titles and appear on Schedule rows, the Month cell references its hidden
  member summary with `aria-describedby`, and popover actions include complete
  member names. All-day and recurring states are named by popover actions, and
  continuation is carried by geometry plus an explicit day-of-span popover
  name.
- No information is conveyed by opacity alone.
- Text contrast meets WCAG AA in the currently supported light theme.
- Reduced motion is respected for the popover transition.
- Auto-focus never moves focus without a user action. Escape returns to the day
  cell; an outside pointer dismissal leaves focus on the clicked target; event
  selection transfers focus into the modal and returns to the originating cell
  only when that modal closes.

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

**Schedule with a widened outer cap - rejected.** Raising `max-w-3xl` to 1400px
would pass a 1440px screenshot while recreating more than 1000px of combined
dead width at 2560px. The selected design keeps the row surface full width and
constrains only its text measure, so the household board scales without turning
a short title into an unreadably long line.

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
- [ ] The reserved-lane counterexample `[blank, event, single-day]` at capacity
      2 is tested and deliberately yields `[blank, +2 more]`; the full day
      remains discoverable and operable through the gridcell and popover.
- [ ] The `+N more` count equals the number of events not rendered, and is
      unaffected by reserved blank slots.
- [ ] A four-row month (February 2026) renders correctly and its capacity is
      covered by unit test.
- [ ] Activating any day with events opens a popover listing all of that day's
      events; selecting one closes the popover and opens `EventDetailModal`
      without stealing focus back from the modal; closing the modal restores
      the originating day cell. `Escape` from the popover returns to the cell;
      outside pointer dismissal leaves focus on the clicked target.
- [ ] A multi-day run renders with rounded outer corners only at its true start
      and end, square inner corners throughout, and the complete title string
      on every day in a one-line 14px summary that may ellipsize; the popover
      exposes the untruncated title.
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
      cells inside the same `role="grid"` as the day rows; the grid owns only
      rows or rowgroups, and the observed weeks rowgroup owns the day rows.
- [ ] `Enter` and `Space` open the popover on a day with events and navigate to
      Day view on a day without; `Space` does not scroll.
- [ ] Day cells expose the accessible names specified in Section 4.5, and today
      carries `aria-current="date"`.
- [ ] Weekend, today and selected-day treatments are visually distinguishable
      from one another and from the default cell.
- [ ] All-day and recurring events carry visible markers with no `"● "` title
      prefix; the popover action's accessible name carries "all day" and
      "repeats" without duplicate marker announcements.
- [ ] Multi-day chip rectangles physically meet across an inner cell boundary,
      row-edge bleed is suppressed, and the page has no horizontal overflow.
- [ ] Visible member names prefix Month chip titles; per-day member dots expose
      a visually-hidden member summary referenced by the gridcell's
      `aria-describedby`; no member information is conveyed by colour or
      `title` alone.
- [ ] Adjacent Month gridcells have at least 8px row and column gaps, while
      continuation-chip weld geometry still touches across the horizontal gap.
- [ ] Month at 769-1023px is visually and behaviourally unchanged.
- [ ] Mobile Month (`MobileMonthlyView`) is visually and behaviourally
      unchanged.

Schedule:

- [ ] At 1440x900, 1920x1080 and 2560x1440 Schedule's day-group and event-row
      surfaces span the available width with no centred narrow column or fixed
      outer cap; event text keeps a readable internal measure.
- [ ] The date gutter renders the relative label, date and event count, and is
      sticky within its own day group.
- [ ] Event rows display the member's name as visible text in addition to
      colour and avatar; event titles are 20px and secondary metadata is at
      least 14px.
- [ ] At `lg+` the coloured left border renders in the member's colour,
      verified by asserting the resolved `borderLeftColor` equals that member's
      `colorMap` hex. Below 1024px the border is unchanged, bug included.
- [ ] A run of event-free days renders as one gap row naming the range, with
      `Nothing scheduled`; it is not focusable, and cross-month/year ranges are
      unambiguous in visible and accessible labels.
- [ ] A window with no events renders the whole-view empty state, not a gap row.
- [ ] Whole-view Schedule empty copy distinguishes no family members, active
      filters hiding all events in the rendered window, and a genuinely
      event-free window; a defensive zero-selection branch is also present but
      neither zero-member Calendar nor persistent zero selection is an
      app-reachable in-scope path, so both defensive branches have component
      evidence rather than fabricated browser screenshots.
- [ ] Day groups before today are de-emphasised without reducing event text
      contrast.
- [ ] The 14-day window and 7-day prev/next step behave exactly as before.
- [ ] Mobile Schedule rendering is byte-identical to `origin/main`, proven by
      screenshot hash comparison. Note that FE PR #285 found this comparison
      only stabilises once font loading is matched between the two captures;
      budget for that rather than treating an early mismatch as a regression.
- [ ] Schedule at 769-1023px is visually and behaviourally unchanged.
- [ ] When active member/all-day filters hide every event in the rendered
      window, the empty-state copy describes a filter result, not "events for
      the next 2 weeks will appear here".

Shared:

- [ ] Month and Schedule render skeletons while loading and a per-view error
      state with retry; Week and Day keep the existing shared fallback.
- [ ] Every interactive element is at least 44px. Month event chips and `+N`
      summaries render at `MONTH_CHIP_HEIGHT` (28px) but are presentational;
      the 96px-or-taller day gridcell is their single interactive target.
- [ ] Text contrast meets WCAG AA in the currently supported light theme,
      including the missing-member fallback where chips render
      `bg-muted text-foreground`, never `bg-muted text-white`.
- [ ] Adjacent-month cell chrome is reduced without applying opacity to event
      content; event titles and markers in those cells retain full contrast.
- [ ] Month offline state waits for persisted-query restoration, distinguishes
      no query data from a successfully cached empty response, and covers both
      differing-range April 2026 and range-reuse February 2026 without an
      uncached-message flash during an offline cold start.
- [ ] Week and Day views are visually and behaviourally unchanged, and the
      existing `day-rail.test.ts` cases still pass unmodified in substance after
      the `buildMonthMatrix` move.
- [ ] Screenshot review at 375x812, **768 (mobile-boundary check)**, **769**,
      1024x768, 1280x800, 1440x900, 1920x1080 and 2560x1440 covers loading,
      empty, error, offline
      (including the Month cold-cache case from Section 6), dense, recurring,
      multi-day, overflow, **four-row February 2026**, very long titles, and a
      reachable missing-member fallback. Defensive zero-member/zero-selection
      Schedule states are covered at component level. Issues found are iterated
      before done.

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

- **Dark theme is not currently an app surface.** The audited frontend baseline
  exports a `ThemeProvider` wrapper but never mounts it and defines no `.dark`
  token block. This story verifies WCAG AA in the supported light theme rather
  than pretending to test an unreachable theme. Add dark tokens, mount the
  provider, and run app-wide contrast regression testing in a dedicated theme
  story before extending this acceptance contract to dark mode.

- **Persistent zero-selection filtering remains broken across views.** The PRD
  permits clearing every profile and requires the empty selection to persist,
  but shipped `FamilyFilterPills` immediately reselects all members whenever
  the selection becomes empty. Fixing that initialization/state behavior would
  change every Calendar view and every breakpoint, contradicting this story's
  preservation boundary. The new Schedule branch handles zero selection
  defensively, but a dedicated cross-view filter story must make the state
  reachable and verify the canonical PRD message everywhere.
- **A reserved-lane overflow cell can show no event chip.** Exact within-row
  alignment can place the day's only covering run beyond the visible prefix.
  In that case the cell deliberately shows reserved space plus `+N more`; its
  accessible event count and full-day popover remain complete. Maximising a
  visible chip would require moving the run out of its lane and breaking the
  weld, or adopting the spanning-bar lane packer rejected in Section 8.
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
- **Schedule's coloured left border stays broken below 1024px.** The `twMerge`
  collapse described in Section 5.1 is fixed only in the `lg+` branch, because
  fixing it on the shared compact path would be a visible mobile change. Mobile
  and tablet keep a default-coloured border. The fix is a small follow-up whose
  review should be about that visual delta specifically.

## 12. Delivery Notes

Two phases on one branch, Month first, each with its own screenshot review
before it is considered done. Schedule work does not begin until the Month gate
passes. The Schedule commit must include the
mobile parity evidence described in Section 3 before it is called complete.
