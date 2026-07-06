# Large-Screen Recipes - Design Spec

**Date:** 2026-07-06
**Status:** Draft for review
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
  tags, and the favorite toggle; the favorite target stays comfortably
  tappable on the image corner.
- **Image-less recipes** (`imageUrl` is nullable) get a calm placeholder
  panel at the same aspect ratio so the grid stays uniform.
- **Header, search, and filters** (search field, Favorites only / All) sit in
  the foundations single toolbar row above the grid, with the search field at
  a usable but not full-bleed width.
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

- Cards are single links/buttons with the recipe name as the accessible name;
  the favorite toggle is a separate, labeled control.
- Grid keeps a logical DOM order matching visual order for keyboard flow.
- Images carry the recipe name as alt context; text never renders over images
  without sufficient contrast treatment.
- Touch targets at least 44px, including the favorite toggle.

## 6. Acceptance Criteria

- [ ] Index renders a 2-4 column grid (by width) inside the max-width
      container at 1024px, 1280px, and 1440px+.
- [ ] Card images render at the capped aspect ratio; no card fills the
      viewport.
- [ ] Search and favorites filtering work unchanged over the grid.
- [ ] Empty library and no-search-results states render centered and intact.
- [ ] Mobile Recipes is visually and behaviorally unchanged.
- [ ] Screenshot review covers: 1 recipe, a dozen recipes, favorites-only
      filter, no-results search, and image-less recipes.

## 7. Out of Scope

- Recipe detail page composition.
- Add/edit recipe flows.
- New features (collections, sorting, import).
- Mobile behavior.
- Backend changes.
