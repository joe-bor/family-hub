# Large-Screen Home Hub - Design Spec

**Date:** 2026-07-05  
**Status:** Draft for review  
**Story:** `docs/product/backlog/large-screen-ux/large-screen-home.md`  
**Concept:** A1 - Now-first ambient home hub  
**Scope:** Horizontal tablet, laptop/desktop, and 20-inch-plus home display
breakpoints. Mobile Home remains unchanged.

## 1. Context

Family Hub's mobile Home has crossed the line from prototype into a polished,
native-feeling organizer surface. It is centered on "what matters now": a calm
hero, today's schedule, member focus, and lightweight household context.

The product now needs a larger-screen direction for horizontal tablets,
kitchen-table devices, and eventually the dedicated home display described in
the PRD. The larger-screen experience must not simply scale the phone UI. That
would create empty space, weak hierarchy, and a product that feels unfinished on
the hardware it was originally meant to support.

This spec defines large-screen Home as an ambient household display: the room's
resting source of truth. It should answer the next important household question
within two seconds, then route deeper work into the owning module.

## 2. Anchor Decisions

| Decision | Contract |
| --- | --- |
| Default concept | Use A1, the Now-first dashboard. |
| Resting state | On fresh launch and after long idle, large screens return to Home unless an active create/edit flow is open. |
| Home role | Home summarizes and routes. It does not become an inline workspace. |
| Module role | Calendar, Meals, Chores, and Lists remain the places where deeper work happens. |
| Density | Larger screens breathe more; they do not gain extra regions by default. |
| Module summaries | The state strip includes Chores, Meals, and Lists only. |
| Explicit exclusions | No fake weather, Recipes tile, Photos placeholder, generic feed, or child-mode behavior in this story. |
| Polish gate | Screenshots must be produced, critiqued, and iterated before the story is considered done. |

## 3. Design Goal

Large-screen Home is an ambient household display, not a stretched mobile screen
and not a planning dashboard. It uses the mobile Home's "what matters now" model
but gives it a true landscape composition: dominant Now Hero, supporting Today
rail, small tomorrow peek, and a restrained Chores / Meals / Lists state strip.

Success means a family member can glance from across the kitchen and understand
the next important thing. If they walk up and tap, they land in the correct
module with context preserved.

## 4. Alternatives Considered

### A. Ambient Home Hub - Selected

A calm Home surface anchored by the current or next important moment. It uses
width for hierarchy and supporting context. It fits the kitchen-table and wall
display posture best.

### B. Planning Workspace - Rejected as Default

A denser layout with more visible controls for editing calendar, meals, lists,
and chores. Useful for module work, but wrong as the default resting state
because it makes the shared display feel like an admin dashboard.

### C. Family Table Mode - Deferred

A participatory, kid-readable surface with people tiles, shared questions, and
large checkoff actions. This has value, but it is a separate child/table-mode
story. Mixing it into adult ambient Home would weaken both concepts.

## 5. Information Architecture

Large-screen Home has two primary zones:

1. **Now Hero zone** - the dominant left zone.
2. **Today Rail zone** - the supporting right zone.

The composition should feel stable across large breakpoints. Horizontal tablets
use a tighter version of the same layout. Larger displays increase type,
spacing, row comfort, and breathing room rather than adding more panels.

### 5.1 Now Hero

The Now Hero owns the screen emotionally and visually. It answers one question:
what matters most right now?

Examples:

- "Swim lesson at 9:00"
- "Leave for swim in 31 min"
- "Rest of day clear"
- "All clear today"

The hero should use the largest type on the screen, strong contrast, and a
single member-color accent. Member color may appear as a leading bar, dot, or
small chip. It must not wash the whole card or dominate the page.

The hero can include concise supporting metadata: relative time, location, a
short note, or the relevant member. It should not include a dense detail view.

### 5.2 State Strip

The state strip lives in or near the hero zone. It provides household context
without competing with the hero.

Allowed state summaries:

- **Chores:** "3 chores left", "Chores done", or equivalent.
- **Meals:** "Tacos tonight", "Dinner not planned", or equivalent.
- **Lists:** one meaningful list signal such as active grocery items or list
  updates.

The state strip should read as household status, not as a module launcher grid.
Tiles or pills may be tappable, but they must stay visually subordinate.

Excluded from the state strip:

- Recipes.
- Photos.
- Weather until a real weather feature exists.
- Settings.
- Sync management.
- Generic activity feed.

### 5.3 Today Rail

The Today Rail provides rest-of-day awareness. It should show enough schedule
context to orient the household without becoming a mini calendar.

Rules:

- Show the next 3-5 rest-of-day items, excluding the hero subject when the hero
  is already an event.
- Prioritize readable time/title/member structure.
- Keep metadata subordinate.
- Avoid hour-grid lines in Home.
- Avoid weekly scope.
- Avoid controls that make the rail feel like Calendar.

### 5.4 Tomorrow Peek

The tomorrow or near-future peek is intentionally small. It answers "anything
early tomorrow?" without turning Home into an agenda feed.

Rules:

- Show up to 1-3 near-future items.
- Prefer tomorrow morning and high-salience items.
- Omit the region when empty if the screen feels calmer without it.
- Never expand this into a scroll-heavy list.

## 6. Interaction Model

Home summarizes and routes. It should not absorb module workflows.

| Surface | Action |
| --- | --- |
| Now Hero with event | Open full Calendar with the event selected or highlighted. |
| Now Hero empty/all-clear state | No action, or route to Calendar only if an affordance is explicit and quiet. |
| Today Rail event | Open full Calendar focused on that event/date. |
| Tomorrow Peek item | Open full Calendar focused on that date/event. |
| Chores summary | Open full Chores module. |
| Meals summary | Open full Meals module, ideally focused on the current week and relevant meal context. |
| Lists summary | Open full Lists module. |
| Global add action | Open the relevant create flow, but keep it visually secondary. |

The receiving module should preserve context. If the user taps "Swim lesson",
Calendar should not open generically; it should make the tapped event/date
obvious. If the user taps "Dinner not planned", Meals should open in the current
week with dinner planning context if feasible. Home itself must not render inline
event detail panels, edit panels, meal planners, chore boards, list detail panes,
or other module workspaces.

## 7. Resting State And Idle Behavior

Large screens behave more like household appliances than personal browser tabs.

Rules:

- Fresh launch on a large screen should land on Home.
- Long idle should return to Home.
- Active create/edit flows block automatic return.
- Manual module switching remains sticky during an active session.
- Idle return should be gentle and predictable, not surprising.

The idle interval should be decided during implementation, but it must be long
enough not to interrupt normal planning work.

## 8. Responsive Behavior

### 8.1 Horizontal Tablet / Small Landscape

Use a tight A1 split: hero on the left, Today Rail on the right, state strip
inside or beneath the hero. Touch targets remain large because the device may be
used on a counter or kitchen table.

### 8.2 Laptop / Desktop Browser

Use the large-screen Home when width allows, while accepting that the posture may
be more work-oriented. Navigation can be slightly more visible, hover/focus
states matter, and scrolling is acceptable when needed. The design should still
optimize for the kitchen-table direction rather than laptop-first density.

### 8.3 20-Inch-Plus Display

The same regions should breathe more. Increase type scale, margin, row comfort,
and visual confidence. Do not add more default cards, lists, feeds, or a weekly
calendar simply because the screen is larger.

### 8.4 Mobile

Mobile Home remains visually and behaviorally unchanged. Shared logic may be
reused, but the mobile product should not regress.

## 9. Visual And UX Polish Requirements

This story must not ship as "responsive layout done." It should feel like a
designed room object.

### 9.1 Hierarchy

The Now Hero must be the unmistakable focal point. It gets the largest type,
most breathing room, and strongest visual weight. The Today Rail and state strip
are supporting surfaces.

### 9.2 Color Discipline

Member colors appear as accents only: small bars, dots, chips, or rings. They
must not become saturated card backgrounds across the Home surface.

### 9.3 Density

Sparse is intentional. Empty space should feel designed, not leftover. The
implementation should resist filling width with additional cards.

### 9.4 Readability Distances

The screen should work at three distances:

- Across the kitchen: the hero message is readable.
- Standing at the counter: the rail and state strip are scannable.
- Seated at the table: details and actions are comfortable.

### 9.5 Motion

Motion should be quiet:

- Hero state changes may crossfade gently.
- State strip updates should not call attention to themselves.
- Module transitions can be smooth but not theatrical.
- Reduced-motion preferences must be respected.

Avoid decorative motion, animated gradients, background blobs, or visual effects
that make Home feel like a marketing page.

## 10. Content And State Rules

### 10.1 Calendar States

Home should handle:

- Event happening now.
- Upcoming event later today.
- All-day-only day.
- Rest of day clear after earlier events.
- All clear today.
- Sparse day.
- Busy day.
- Long event titles.
- Multi-day or all-day events, using existing calendar semantics.

### 10.2 Chores States

Home should summarize today's chores only:

- Remaining chores.
- All chores done.
- No chores configured.
- Chores data unavailable.

### 10.3 Meals States

Home should summarize the most relevant meal, usually dinner:

- Dinner planned.
- Dinner not planned.
- Meals data unavailable.
- Read-only/past week cases should not matter on Home; tapping routes to Meals.

### 10.4 Lists States

Home should expose one meaningful list signal:

- Active grocery items.
- Recent list update.
- Quiet/clear state.
- Lists data unavailable.

The design must avoid turning Lists into a generic activity feed.

### 10.5 Offline And Cached Data

Home should be honest but not noisy:

- If cached data is available, show it with a small "last updated" or cached
  signal when necessary.
- If data is unavailable because the app has never loaded it offline, show a
  calm unavailable state.
- Write failures belong in the owning module's workflow, not as Home-level
  warnings.

## 11. Navigation And Shell

Home should be reachable as a first-class large-screen surface. Larger screens
should no longer force `activeModule === null` directly to Calendar.

Navigation should be present but visually secondary. A compact rail or compact
top navigation can work, but Home must not become a launcher grid. The design
should make the current surface obvious while letting the hero remain dominant.

The existing fake desktop weather display should not appear in this Home design.
If weather is added later, it must come from a real weather story and real data.

## 12. Accessibility

- All touch targets must be at least 44px, with larger targets preferred for
  kitchen/table use.
- Color must never be the only member identifier.
- Hero content should have an accessible label that reflects the state.
- Tappable summaries should expose clear labels such as "Open Meals. Dinner not
  planned."
- Keyboard focus must be visible for laptop/desktop use.
- Reduced motion must be respected.
- Long text must wrap or truncate intentionally without overlapping adjacent
  regions.

## 13. Visual Review Loop

Screenshot critique is part of the delivery contract.

The implementation must produce screenshots for the required matrix, review them
against the design intent, and iterate before completion. The first screenshot
pass is evidence, not proof.

Required viewport coverage:

- Existing mobile Home baseline, to prove the mobile surface did not regress.
- Horizontal tablet landscape.
- Common laptop/desktop width.
- Simulated 20-inch / large display width.

Required state coverage:

- Busy day.
- Sparse day.
- All-clear day.
- Missing dinner.
- Chores done.
- Lists quiet.
- Offline cached data when available.
- Long event title.

Critique questions:

- Is the Now Hero the unmistakable focal point?
- Can the next important thing be understood in two seconds?
- Does the layout feel intentionally sparse, or accidentally empty?
- Did any region become a card grid or dashboard junk drawer?
- Are Chores, Meals, and Lists present but subordinate?
- Are long titles, missing data, and quiet states still polished?
- Does the large display breathe instead of densifying?
- Is mobile unchanged?

If screenshots reveal awkward spacing, weak hierarchy, cramped text, or
"stretched mobile" energy, implementation must iterate before the PR is
considered ready.

## 14. Acceptance Criteria

### Functional

- [ ] Larger screens can land on Home instead of redirecting directly to
      Calendar.
- [ ] Fresh launch on a large screen lands on Home.
- [ ] Long idle returns to Home unless an active create/edit flow is open.
- [ ] Now Hero renders correct states for current event, next event, all-day-only
      day, rest-of-day clear, and all-clear day.
- [ ] Today Rail shows rest-of-day items without becoming a mini calendar.
- [ ] Tomorrow Peek shows limited near-future context and can be omitted when
      empty.
- [ ] State strip includes Chores, Meals, and Lists only.
- [ ] Tapping an event opens full Calendar with date/event context preserved.
- [ ] Tapping Chores, Meals, or Lists opens the owning module.
- [ ] Home does not render inline event detail/edit panels or module workspaces;
      deeper work opens Calendar, Meals, Chores, or Lists.
- [ ] Mobile Home remains unchanged.

### Design Quality

- [ ] The Now Hero is the primary focal point in required screenshots.
- [ ] Larger widths increase comfort and readability rather than default density.
- [ ] Member colors are accents only.
- [ ] No fake weather, Recipes tile, Photos placeholder, or generic feed appears
      by default.
- [ ] Text does not overlap or overflow awkwardly across required viewport and
      state combinations.
- [ ] Touch targets meet the 44px minimum.
- [ ] Reduced-motion behavior is respected.
- [ ] Screenshot critique has been performed and any identified visual issues
      have been addressed.

### Technical / Delivery

- [ ] The story does not require backend changes unless implementation identifies
      a missing summary contract.
- [ ] Any newly required shared contract is documented before implementation
      proceeds.
- [ ] Visual QA screenshots are attached to the PR or otherwise recorded in the
      work log.
- [ ] The visual QA set includes an existing mobile Home baseline screenshot or
      equivalent regression evidence.

## 15. Out Of Scope

- Redesigning Calendar, Meals, Chores, Lists, Recipes, or Photos internals.
- Child-mode Home.
- Family table mode.
- Weather.
- Photos library or screensaver behavior.
- Recipes surfaced on Home.
- Generic activity feed.
- Full desktop shell redesign.
- Offline writes.
- New backend work unless a summary contract gap is discovered.

## 16. Follow-Up Stories

- **Large-screen Calendar polish:** make the full Calendar module feel excellent
  on horizontal tablet and 20-inch display sizes.
- **Large-screen Meals planning polish:** make weekly meal planning use landscape
  space intentionally.
- **Child/table mode:** explore a more participatory, kid-readable shared screen.
