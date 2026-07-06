# Large-Screen Foundations - Design Spec

**Date:** 2026-07-06
**Status:** Draft for review
**Story:** `docs/product/backlog/large-screen-ux/large-screen-foundations.md`
**Scope:** Shared shell chrome and layout discipline for all module surfaces on
viewports above the mobile boundary (768px). Mobile shell unchanged.

## 1. Context

On a 13-inch laptop, Family Hub modules render the phone layout with more air.
The most visible shared symptom is vertical chrome: the app header band (family
name, date/time, fake weather, member dots) plus per-module toolbar rows stack
up before any module content appears. In Calendar Week, roughly half the
viewport is chrome and headers; only about four hours of the week are visible.

This spec defines the shared foundation that every module's large-screen pass
builds on. It is deliberately small: slimmer chrome, one toolbar row, and
content-width discipline. Module-specific composition lives in the per-module
specs:

- `2026-07-06-large-screen-calendar-design.md`
- `2026-07-06-large-screen-lists-design.md`
- `2026-07-06-large-screen-chores-design.md`
- `2026-07-06-large-screen-meals-design.md`
- `2026-07-06-large-screen-recipes-design.md`

The large-screen Home hub (`2026-07-05-large-screen-home-design.md`, issue
#278) is a separate in-flight story and is not changed by this spec.

## 2. Design Goal

Chrome should orient, then get out of the way. On large screens the shell above
module content should cost at most ~160px of vertical space, leaving the rest
for the module workspace. Family Hub remains a touch-first shared device
surface: targets stay at least 44px, and density comes from removing redundant
bands, not from shrinking touch targets.

## 3. Chosen Direction

### 3.1 Slimmer app header band

The desktop app header currently shows the family name at display size, the
date and live clock, a hardcoded fake weather chip ("72°"), and member dots.

Changes:

- Remove the fake weather chip from the module shell. This matches the
  large-screen Home spec, which already bans fake weather from Home. If real
  weather ships later, it returns through a real weather story.
- Compress the header to a single compact row: menu affordance, family name at
  a modest size, date/time secondary, member dots retained as the family
  identity signal.
- The header must not grow taller than roughly 64px on large screens.

### 3.2 One toolbar row per module

Modules currently stack a view/section switcher row and a separate
navigation/actions row. On large screens these merge into one toolbar row:

- Left: module view switcher (where the module has views, e.g. Calendar's
  Day / Week / Month / Schedule) or the module title where it does not
  (Lists, Chores, Meals, Recipes).
- Center: contextual navigation (previous / next, Today, current range label).
- Right: filters and primary actions (member filter chips, add buttons).

The toolbar is one row at laptop widths and wider. If a module has too many
controls for one comfortable row at exactly 769-1023px, secondary controls may
collapse behind an overflow control rather than wrapping to a second row.

### 3.3 Content-width discipline

Each module declares an intentional content strategy instead of inheriting a
mobile column:

| Module | Strategy |
| --- | --- |
| Calendar | Full width; the grid is the product. |
| Meals | Full width up to a generous cap; all 7 day columns visible. |
| Lists | Two-pane split filling available width. |
| Chores | Wide board with comfortable column widths. |
| Recipes | Card grid inside a max-width container (~1200px). |

"Centered narrow column with dead margins" is no longer an acceptable default
for any module index screen.

### 3.4 Breakpoint semantics

- Mobile boundary stays `useIsMobile()` at 768px. Nothing below it changes.
- Large-screen compositions (two-pane Lists, Day member lanes, merged toolbar)
  activate at the Tailwind `lg` boundary (1024px) unless a module spec says
  otherwise.
- Between 769px and 1023px (portrait tablet), modules keep their current
  structure but inherit the slimmer header and single toolbar row.

## 4. Alternatives Considered

### Full desktop shell redesign - Rejected

Rebuilding navigation into a persistent expanded sidebar or top-level app bar
with global search and quick-add. It would modernize the shell but is a much
larger project, is explicitly out of scope in the Home hub spec, and does not
fix the actual reported problem, which is per-module wasted space. The existing
compact left nav rail already works well on large screens.

## 5. Accessibility

- All interactive chrome keeps 44px minimum touch targets.
- Keyboard focus visible on all toolbar controls for laptop use.
- The merged toolbar preserves the existing accessible names and roles of the
  controls it absorbs; merging is layout, not semantics.
- Member dots keep accessible labels; color is never the only identifier.

## 6. Acceptance Criteria

- [ ] Fake weather chip no longer renders anywhere in the module shell.
- [ ] App header band is a single compact row (roughly 64px or less) on large
      screens.
- [ ] Calendar chrome above the time grid (header + toolbar) totals ~160px or
      less at 1440x900.
- [ ] Each module renders one toolbar row on `lg+`, not two stacked rows.
- [ ] No module index screen renders as a narrow centered column with dead
      side margins on large screens.
- [ ] Mobile shell (bottom nav, mobile header) is visually and behaviorally
      unchanged.
- [ ] All chrome touch targets remain at least 44px.

## 7. Out of Scope

- Home hub layout (issue #278 owns it).
- Navigation rail redesign, global search, notifications.
- Any real weather feature.
- Module content composition (owned by per-module specs).

## 8. Delivery Notes

This is the first story implemented in the large-screen pass; the module
stories assume it has landed. All large-screen work ships from a single FE
branch (`feat/large-screen-module-polish`) as atomic commits per surface, with
screenshot review before each surface is called done.
