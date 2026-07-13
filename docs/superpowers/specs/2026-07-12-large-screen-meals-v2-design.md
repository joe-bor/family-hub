# Large-Screen Meals v2 — Full-Height Board - Design Spec

**Date:** 2026-07-12
**Status:** Draft for review
**Story:** `docs/product/backlog/large-screen-ux/large-screen-meals.md`
**Supersedes (partially):** `2026-07-06-large-screen-meals-design.md` — the v1
full-width board shipped in FE PR #287; this v2 revises the board's vertical
layout, row-header rail, and card anatomy based on review feedback.
**Scope:** Meals week board on large screens (1024px+). Mobile Meals unchanged.

## 1. Context

The v1 large-screen board (FE PR #287) fits all seven day columns without
horizontal scrolling, but it is content-height: on a 900px-tall laptop the
board occupies roughly the top half of the viewport and the bottom half is
empty. The 120px Breakfast/Lunch/Dinner header column also duplicates
information every card already carries in its meal-type eyebrow, spending
width the day columns could use.

## 2. Design Goal

On a large screen the week board *is* the screen: it fills the viewport
height, uses the reclaimed space to show richer meal content (photo, full
title, note, extras), and spends almost no width on row chrome. Today remains
findable at a glance. All planning flows stay exactly as they are.

## 3. Chosen Direction

### 3.1 Geometry — the board owns the viewport

- The Meals section becomes a column layout: WeekHeader toolbar on top, board
  filling all remaining height (`flex-1`; the board keeps its automatic
  minimum height so the 170px row floor can force the module to scroll on
  short viewports instead of being clipped).
- The board's `<table>` is replaced by a CSS grid:
  `grid-template-columns: 36px repeat(7, 1fr)`;
  `grid-template-rows: auto repeat(3, minmax(170px, 1fr))`.
  Table rows cannot reliably stretch children to full cell height; grid can.
- ARIA table semantics are preserved explicitly (`role="table"` /
  `rowgroup` / `row` / `columnheader` / `rowheader`, the "Weekly meals"
  label, and `aria-current="date"` on today's column header) so
  screen-reader announcements are unchanged: a slot still reads as
  "Dinner, Wednesday: Turbong Manok".
- The three meal rows share leftover viewport height equally. Below the
  170px per-row floor (short viewports: half-snapped windows, 768px-tall
  laptops) the board stops shrinking and the page scrolls vertically, as it
  does today. One card layout everywhere; no adaptive collapse.
- The existing centered 1600px width cap stays.
- Below `lg`, the stacked day-card layout is untouched.

### 3.2 Sideways rail — 120px → ~36px

- The row-header column shrinks to ~36px. Labels render vertically
  (bottom-to-top), centered in their row: uppercase, small, muted, tinted
  with the row's meal accent (3.4).
- Implementation note: use `writing-mode: vertical-rl` combined with
  `rotate(180deg)` for cross-browser safety; `sideways-lr` support is still
  patchy.
- The labels remain real row headers for assistive tech.

### 3.3 Banner card (grid only)

A new grid-specific slot card. The existing `MealSlotCard` continues to serve
the mobile day-card layout unchanged; shared accessible-label logic is
extracted rather than duplicated.

Anatomy, top to bottom:

- **Banner** — ~40–45% of the cell height. Recipe image `object-cover`;
  image-less meals get the tinted placeholder band (3.4).
- **Title** — semibold, clamps to two lines (no more single-line
  truncation); full title stays in the accessible label.
- **Notes** — primary note and slot note, line-clamped to fit.
- **Extras** — chips pinned to the card bottom: first two + "+n more".
- The meal-type eyebrow is **removed** in the grid; the rail carries that
  information. (Mobile cards keep their eyebrow.)
- Draft badge, planning-target ring, pending-recipe placement, and all
  click/selection behavior carry over unchanged.

### 3.4 Meal-tint system

- Three soft accent tokens as oklch CSS variables alongside the existing
  palette: breakfast amber, lunch green, dinner indigo. (The app currently
  ships a light palette only — no `.dark` block exists in `index.css` — so
  no dark variants are defined; add them if a dark theme lands.)
- Used in exactly two places: the placeholder band (soft tint + a muted
  lucide glyph — Croissant / Sandwich / CookingPot) and the rail label
  color. Photos remain the visual hero; tints appear only where a photo is
  absent.
- The tinted band must read as "planned, just no photo" — clearly distinct
  from the empty slot's dashed transparency (3.5).

### 3.5 Empty and read-only states

- **Empty editable slot:** one full-cell dashed add-target — plus icon in a
  small filled circle, label "Add breakfast/lunch/dinner". Transparent
  background and dashed border keep it unmistakably different from filled
  cards (solid border, white card, tinted or photo banner).
- **Past read-only weeks:** filled cards render normally minus hover
  affordances; empty slots become a quiet dashed cell with only the muted
  meal label, no CTA.

## 4. Alternatives Considered

- **Compact board + week-status strip** (planned/empty counts, ingredients
  status in the reclaimed space) — rejected for now; adds a new surface to
  design and maintain. Candidate for a future story.
- **Stretch rows but keep compact cards** — rejected; recreates the
  whitespace problem inside each cell.
- **Image-immersive cards** (photo fills the cell, title on a scrim) —
  rejected; manual entries without photos would produce two jarringly
  different card styles side by side.
- **Icon rail / no rail** — rejected in favor of sideways text: icons are
  less immediate than words; deleting the rail entirely repeats the label
  21 times and loses the row anchor on empty weeks.
- **Adaptive card collapse on short viewports** — rejected; two card
  layouts to build and test. Min row height + page scroll is predictable.

## 5. Accessibility

- Grid re-implementation preserves current announcements: meal type, full
  weekday, and untruncated meal title in each slot's accessible label.
- Sideways rail labels remain row headers; day headers keep full weekday
  names and `aria-current="date"` for today.
- Meal tints are never the sole carrier of meaning (rail text + band glyph
  + card structure carry it); tint/glyph contrast meets non-text contrast
  guidance.
- Add-meal targets remain at least 44px in both dimensions.

## 6. Acceptance Criteria

- [ ] At 1440×900 (and 1280×800, 1920×1080) the board's bottom edge lands
      within the viewport padding — no pooled whitespace below the board.
- [ ] Rows never shrink below the min row height; short viewports fall back
      to vertical page scroll with no layout breakage.
- [ ] Rail column is ~36px with vertical labels; no meal-type eyebrows
      inside grid cards.
- [ ] Filled cards show banner (photo or tinted band + glyph), two-line
      title clamp, notes, extras chips.
- [ ] Image-less filled cards are visually distinct from empty slots at a
      glance (tinted solid band vs dashed transparent target).
- [ ] Today's column highlight and `aria-current="date"` preserved.
- [ ] All existing flows (add, edit, move, fill empty slots, recipe links,
      ingredients-to-grocery, planning drafts/targets) work unchanged.
- [ ] Mobile Meals visually and behaviorally unchanged.
- [ ] E2E geometry spec asserts the vertical-fill requirement; the visual
      matrix (full / empty / long names / past week × 1024 / 1440 / 1920)
      is re-captured and reviewed.

## 7. Out of Scope

- Meal planning workflow changes (composer, editor, sessions, scope
  dialogs).
- Week-status strip / board footer.
- Nutrition, portions, or other new meal features.
- Mobile behavior.
- Backend changes.

## 8. Delivery Note

Work continues on the existing FE branch `feat/large-screen-meals`
(PR #287). The currently failing CI checks cover v1 layout details that this
redesign replaces; they are refactored as part of this work rather than
patched beforehand.
