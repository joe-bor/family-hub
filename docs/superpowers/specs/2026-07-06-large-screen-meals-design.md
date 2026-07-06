# Large-Screen Meals - Design Spec

**Date:** 2026-07-06
**Status:** Draft for review
**Story:** `docs/product/backlog/large-screen-ux/large-screen-meals.md`
**Depends on:** `2026-07-06-large-screen-foundations-design.md`
**Scope:** Meals week board on large screens (1024px+). Mobile Meals unchanged.

## 1. Context

The Meals week grid (meal types as rows, days as columns) is the right shape
for a large screen, but at 1440px it horizontally scrolls — Friday and Saturday
are cut off — while roughly a third of the screen sits empty beside it. The
grid neither fills the width it has nor fits the week it shows. This is layout
math, not a redesign.

## 2. Design Goal

The whole week is visible at once on a laptop or landscape tablet: seven day
columns, three meal rows, no horizontal scrolling. Today is findable at a
glance. Planning stays exactly the flow it is today.

## 3. Chosen Direction: Full-Width Week Board

- **All 7 day columns fit** without horizontal scrolling at 1024px and wider.
  Columns flex to share available width inside a generous cap (foundations
  content strategy); the grid centers under the cap on very wide screens.
- **Today's column highlighted** with the same today treatment Calendar uses
  (accent on the day header, subtle column tint), so the board answers
  "what's tonight?" instantly.
- **Slot cards adapt to column width.** Filled slots keep meal name +
  provenance line; empty slots keep the add affordance. At the narrowest
  columns (1024px), card text truncates rather than wrapping into tall cells.
- **Weekday headers align with Calendar's one-line style** ("Sun 5") for
  cross-module consistency.
- **Foundations chrome:** week title, range navigation, and Fill empty slots
  in one toolbar row.
- Composer/editor sheets, focused planning sessions, and the
  ingredients-to-grocery flow are untouched; only the board layout changes.

## 4. Alternatives Considered

### Keep horizontal scroll with snap and edge affordances - Rejected

Keeping the scrolling grid but adding scroll-snap and clearer cut-off cues.
Cheaper, but a seven-day family plan that hides two days on a 13-inch laptop
is simply wrong; there is enough width, and the point of the board is the
whole week at a glance.

## 5. Accessibility

- Day columns keep proper header associations so a slot reads as "Dinner,
  Wednesday: Turbong Manok".
- Add-meal targets stay at least 44px in the narrowest column layout.
- Truncated meal names keep full names available (accessible label / title).
- Today's highlight is not color-only (header treatment carries it).

## 6. Acceptance Criteria

- [ ] No horizontal scrolling of the week grid at 1024px, 1280px, and 1440px.
- [ ] All 7 days and all 3 meal rows visible; grid centered under its cap on
      very wide screens.
- [ ] Today's column visibly highlighted.
- [ ] All existing flows (add meal, edit, move, fill empty slots, recipe
      links, ingredients-to-grocery) work unchanged.
- [ ] Mobile Meals is visually and behaviorally unchanged.
- [ ] Screenshot review at 1024px, 1440px, and a large-display width covers:
      full week planned, empty week, long meal names, and a past (read-only)
      week.

## 7. Out of Scope

- Meal planning workflow changes (composer, sessions, scope dialogs).
- Nutrition, portions, or other new meal features.
- Mobile behavior.
- Backend changes.
