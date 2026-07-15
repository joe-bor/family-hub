# Large-Screen Recipes - Design Spec

**Date:** 2026-07-06
**Status:** Approved (spec-to-plan review completed 2026-07-15)
**Story:** `docs/product/backlog/large-screen-ux/large-screen-recipes.md`
**Depends on:** `2026-07-06-large-screen-foundations-design.md`
**Scope:** Recipes index on large screens (1024px+). Mobile Recipes unchanged.

## 1. Context

The Recipes index renders a single mobile-width column on large screens: one
enormous recipe card whose photo fills most of the viewport, with half the
screen as blank margin. Browsing the family's recipes on a laptop means paging
through posters one at a time.

## 2. Design Goal

Recipes becomes browsable: a family member scanning "what should we cook?"
sees a shelf of options at once, each card readable (photo, title, tags,
favorite state) without any card dominating the screen.

## 3. Chosen Direction: Responsive Card Grid

- **Grid of recipe cards** inside a max-width container (~1200px): 2 columns
  at 1024px, 3 at ~1280px, 4 at very wide widths.
- **Capped card images.** Photos render at a fixed aspect ratio (roughly 4:3)
  with `object-cover`, so a card is a card, not a poster. Cards keep title,
  tags, and the existing favorite-state badge. The heart on an index card is
  display-only; changing favorite state remains an action in recipe detail.
- **Image-less recipes** (`imageUrl` is nullable) get a calm placeholder
  panel at the same aspect ratio so the grid stays uniform.
- **Header, search, filters, and Add recipe** sit in the foundations single
  toolbar row above the grid at `lg+`. The search field uses a bounded width;
  Favorites only / All / tag chips use the remaining horizontal space and may
  scroll within that space rather than wrapping the toolbar. At 769-1023px the
  existing two-level header/filter composition may remain. Mobile stays
  unchanged.
- **Empty and no-results states** center within the grid area with the
  existing copy.
- Recipe detail and add/edit flows are untouched in this pass; only the index
  composition changes.

## 4. Alternatives Considered

### Masonry / editorial layout - Rejected

A Pinterest-style variable-height masonry grid. Visually rich for a
photo-heavy library, but it fights scanability (no stable rows to sweep),
complicates keyboard order and virtualization later, and the collection sizes
of a family recipe box do not need it. A uniform grid is calmer and more
child-scannable.

## 5. Accessibility

- Each card keeps its existing semantic article and full-card, accessible
  open-recipe button. The heart is a display-only state badge, not a nested
  interactive control.
- Grid keeps a logical DOM order matching visual order for keyboard flow.
- Images carry the recipe name as alt context; text never renders over images
  without sufficient contrast treatment.
- Interactive toolbar controls are at least 44px at `lg+`; the full-card open
  target remains comfortably larger than that minimum.

## 6. Acceptance Criteria

- [ ] Index renders a 2-4 column grid (by width) inside the max-width
      container at 1024px, 1280px, and 1440px+.
- [ ] Card images render at the capped aspect ratio; no card fills the
      viewport.
- [ ] Search and favorites filtering work unchanged over the grid.
- [ ] At `lg+`, title, bounded-width search, filter chips, and Add recipe fit
      in one toolbar row with 44px interactive targets.
- [ ] Empty library and no-search-results states render centered and intact.
- [ ] Mobile Recipes is visually and behaviorally unchanged.
- [ ] Screenshot review covers: 1 recipe, a dozen recipes, favorites-only
      filter, no-results search, and image-less recipes.

## 7. Out of Scope

- Recipe detail page composition.
- Add/edit recipe flows.
- New features (collections, sorting, import).
- Interactive favoriting from an index card; favoriting stays in detail.
- Mobile behavior.
- Backend changes.
