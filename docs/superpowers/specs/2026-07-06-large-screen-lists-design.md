# Large-Screen Lists - Design Spec

**Date:** 2026-07-06
**Status:** Draft for review
**Story:** `docs/product/backlog/large-screen-ux/large-screen-lists.md`
**Depends on:** `2026-07-06-large-screen-foundations-design.md`
**Scope:** Lists module on large screens (1024px+). Mobile Lists unchanged.

## 1. Context

Lists is the least adapted module on large screens. The index renders as a
narrow centered column (`max-w-2xl`) of list cards with dead margins on both
sides, and opening a list **replaces the whole screen** with another narrow
column, phone-style, behind a back button. On a laptop or kitchen tablet the
user pays full-screen navigation cost to glance at a grocery list.

Lists are also the module most likely to stay open on a shared device: the
grocery list lives on the kitchen tablet while people add to it across the day.

## 2. Design Goal

Stop navigating; show both panes. A family member walks up to the tablet, sees
every list at a glance on the left, and the active list's items on the right,
ready to check off or extend without any navigation. Mobile keeps its focused
drill-in flow.

## 3. Chosen Direction: Two-Pane Master/Detail

### 3.1 Layout

- **Left rail (~340px):** the lists index. Each list row keeps the existing
  card information (name, item counts, category signal) in a rail-friendly
  density. "New List" lives at the top of the rail. Touch targets stay large;
  this is a shared-device surface, not a file tree.
- **Right pane (remaining width):** the selected list's detail: items,
  sections/categories, add-item flow, and list options. Item rows may cap at a
  comfortable reading width inside the pane rather than stretching edge to
  edge.
- The selected rail row shows a clear selected state (member-color-neutral
  accent, not just a subtle background shift).

### 3.2 Behavior

- First list auto-selected on entry so the detail pane is never empty.
- Selecting a list swaps only the right pane; no screen transition.
- The cross-module handoff intent (e.g. "open this grocery list" from Meals)
  selects that list in the pane instead of pushing a full screen.
- Creating a list selects it when created. Deleting the selected list falls
  back to the first remaining list, or the empty state when none remain.
- Empty states: no lists at all shows the existing create-first-list state
  across the full content area; a selected list with no items shows the
  existing empty-list state in the right pane.
- Below 1024px (and on mobile) the current index -> detail navigation remains
  exactly as it is today, including the slide transition and back handling.

## 4. Alternatives Considered

### Widened full-screen detail - Rejected

Keep the drill-in navigation but let the detail screen use more width
(multi-column item sections, wider rows). Cheaper, but it still hides every
other list behind a back button, still wastes the index screen, and does
nothing for the leave-it-open kitchen-tablet posture that Lists naturally has.

## 5. Accessibility

- The rail is a navigation-of-content structure: expose it as a list with the
  selected item's state announced ("Groceries, selected, 4 items remaining").
- Keyboard: rail items focusable in order; selection follows activation, not
  focus.
- Back-handler semantics on large screens: with both panes visible there is no
  detail "screen" to back out of; the back handler no longer intercepts.
- Touch targets at least 44px throughout both panes.

## 6. Acceptance Criteria

- [ ] At 1024px+ Lists renders rail + detail simultaneously; no full-screen
      swap when selecting a list.
- [ ] First list auto-selected; create/delete selection fallbacks behave as
      specified.
- [ ] Cross-module list handoff selects the list in the pane.
- [ ] Existing item flows (add, check, sections, options, managed categories)
      work unchanged inside the right pane.
- [ ] Below 1024px and on mobile, the existing drill-in flow is visually and
      behaviorally unchanged.
- [ ] Screenshot review at 1024px and 1440px covers: several lists, one list,
      empty list selected, and the no-lists empty state.

## 7. Out of Scope

- List sharing/permissions, reordering lists, or new list features.
- Changes to add-item sheets or category management flows beyond where they
  render.
- Mobile behavior.
- Backend changes.
