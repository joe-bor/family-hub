# Visual Identity Refinement Pass

## Problem

Family Hub's mobile shell now behaves like a product, not a prototype: bottom nav is shipped, Home is a real dashboard, and Calendar already carries the product's core value. What still feels uneven is the visual execution. Spacing rhythm changes from surface to surface, type hierarchy is inconsistent, and the seven member colors read more like a collection than a system.

This story is the last intentional polish pass before product focus shifts from shell work to real module value.

## Goals

- Make the authenticated mobile experience feel visually coherent across Home, Calendar, and the current shell destinations.
- Document one spacing scale and one type hierarchy that implementation can apply without inventing new design tokens.
- Review the member-color palette for cohesion while preserving quick family-member recognition.
- Capture a clear before/after comparison so the refinement is visible and reviewable.

## Non-goals

- Rebranding Family Hub or changing the cream / purple / Nunito identity.
- Changing information architecture, navigation order, or module semantics.
- Redesigning onboarding, settings, or sidebar ergonomics.
- Adding new module functionality, new data sources, or backend work.
- Introducing a new motion system, new routes, or a new token architecture.

## Scope

### Included surfaces

- Authenticated mobile shell chrome:
  - `AppHeader`
  - `MobileBottomNav`
- Shipped mobile product surfaces:
  - `HomeDashboard`
  - mobile Calendar views + toolbar
  - event sheet / event form / mobile event detail
- Current placeholder shell destinations:
  - `Lists`
  - `Chores`
  - `Meals`
  - `Photos`

### Explicitly excluded

- Login and onboarding
- Settings and sidebar-specific mobile layout work
- Desktop-only refinements
- Module-definition work for Lists / Chores / Meals / Photos

The placeholder module surfaces can receive visual cleanup, but this story must not "settle" their product meaning.

## Decision Summary

### D1. Keep the bones, refine the execution

The story is a refinement pass, not a visual overhaul. Keep the current warm cream background, purple primary, Nunito typography, soft-radius card language, and overall calm tone.

### D2. Use a documented mobile spacing rhythm

Implementation should normalize toward a single 4px-based rhythm using existing Tailwind utilities instead of one-off pixel values.

| Role | Target rhythm | Notes |
|---|---|---|
| Page edge padding | `16px` on narrow mobile, `20px` on roomier mobile | Default page gutters |
| Major section stack | `24px` | Between top-level sections on a screen |
| Card padding | `16px` | Default internal card padding |
| Row / control gap | `12px` | Icon + label rows, card sub-sections |
| Chip / pill gap | `8px` | Dense control clusters |
| Micro spacing | `4px` | Label + helper / compact metadata |

The intent is not "every screen looks the same." The intent is that spacing decisions come from the same small set of values.

### D3. Use a documented type hierarchy

Implementation should tighten weight usage and line-height choices around a small hierarchy:

| Role | Target scale | Typical use |
|---|---|---|
| Display | `28/32`, semibold | Home hero title only |
| Page title | `22-24px`, bold or semibold | Module / screen headings |
| Section title | `16-18px`, semibold | Card headers, list section labels |
| Body | `15-16px`, regular or medium | Default readable copy |
| Meta | `13-14px`, medium | Secondary labels, counts, status text |
| Micro | `11-12px`, medium | Tab labels, tiny metadata only |

Rules:

- Nunito remains the only product typeface.
- Do not use bold as the default answer to visual emphasis.
- Use opacity / tone shifts for secondary information before reaching for smaller and smaller text.

### D4. Harmonize the member palette without changing hue ownership

The member colors should stay recognizably distinct, but they should feel like a family.

Non-negotiables:

- Keep seven clearly different member hues available.
- Normalize lightness / saturation so no single color feels neon next to the others.
- Preserve child-recognizable identity in avatars, filters, event accents, and chips.
- Verify contrast where white text sits on a member color; if a color cannot safely support white text, adjust the treatment rather than shipping poor contrast.

This story does not require a brand-new palette. It requires a tighter, documented version of the current one.

### D5. Polish the product surfaces first; touch placeholders lightly

The visual pass should spend most of its attention on the shipped product surfaces:

- shell chrome
- Home
- Calendar mobile

`Lists`, `Chores`, `Meals`, and `Photos` should be brought into the same visual language, but only through spacing / type / card treatment cleanup. Do not redesign them into final module experiences here.

### D6. Evidence is part of the deliverable

Before closing the story, capture before/after mobile comparisons for at least:

1. Home dashboard
2. Calendar mobile view
3. Event sheet or event form
4. Lists placeholder surface
5. Chores placeholder surface

The comparison can live in the FE issue or PR body, but it must exist.

## Surface Contract

### Shell chrome

- `AppHeader` and `MobileBottomNav` should feel like they belong to the same product layer.
- Icon sizing, label weight, padding, and active-state emphasis should be consistent.
- Bottom-nav labels must remain legible at a glance and preserve minimum touch comfort.

### Home

- Preserve the shipped "The Now" IA and hero-state model.
- Tighten the rhythm between greeting, member chips, hero, Today, and Coming Up.
- Preserve the motion vocabulary already introduced; this story is not a motion rewrite.

### Calendar mobile

- Toolbar rows, view pills, filter controls, date hierarchy, and event-card text should feel lighter and more intentional.
- Mobile sheets and forms should share the same spacing / header / action hierarchy as the rest of the app.
- Do not change Calendar behavior or add new controls in this story.

### Placeholder modules

- Keep existing placeholder IA intact.
- Normalize title size, card padding, section spacing, and button treatment so they no longer look like disconnected demo screens.
- Do not use this polish pass to imply that current placeholder copy or layout is the final product definition for those modules.

## Acceptance Criteria

- [ ] Spacing scale is documented in this spec and applied across authenticated mobile surfaces.
- [ ] Type hierarchy is documented in this spec and applied across authenticated mobile surfaces.
- [ ] Member palette is reviewed for cohesion and any resulting adjustments are captured in implementation notes.
- [ ] Home, Calendar, and shell chrome remain behaviorally unchanged while visually improved.
- [ ] Placeholder module screens receive visual alignment only; no new module behavior is introduced.
- [ ] Before/after comparison is captured for review.

## Implementation Notes

- Expect most FE changes in `frontend/src/index.css`, shared shell components, mobile calendar surfaces, home components, and the four placeholder module views.
- No BE work is required.
- The FE issue created from this story should restate the "refinement, not overhaul" rule explicitly so implementation does not drift into module-definition work.
