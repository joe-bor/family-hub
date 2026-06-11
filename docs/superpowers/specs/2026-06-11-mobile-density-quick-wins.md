# Mobile Density Quick Wins: Chores Period Label + Recipe Card Layout

## Problem

Two per-module density issues from the 0.3.12 Galaxy S10 device pass (story findings **F4** and **F5**) waste mobile screen space on content that should be glanceable:

- **Chores (F4):** the Day/Week/Month segmented switcher at the top of the view is immediately followed by a card whose heading repeats the same period ("Today" / "This Week" / "This Month"). On mobile only one scope is visible at a time, so the heading restates what the switcher already says — a full heading row spent on zero information.
- **Recipes (F5):** each library card renders a full-width `aspect-[4/3]` image. On a ~360px-wide phone that's ~270px of image per card before the title and tags, so a single recipe eats most of the viewport and browsing several recipes means heavy scrolling.

Both are small, independent, self-contained fixes — batched into one story as the highest-ROI quick wins.

## Current FE state (all paths relative to `frontend/`)

- **Chores switcher:** `src/components/chores-view.tsx:120-125` renders `ChoreScopeSwitcher` on mobile only (`isMobile &&`), selecting which single scope board is shown (`chores-view.tsx:65-69`). Desktop shows all three boards in a `lg:grid-cols-3` grid (`chores-view.tsx:163`).
- **Chores card heading:** `src/components/chores/chores-scope-column.tsx:32-42` renders `scopeHeading(scope.scope)` ("Today"/"This Week"/"This Month") as an `<h2>` plus a "`N` left of `M`" summary line. The `<h2>` is also the column's `aria-labelledby` target (`chores-scope-column.tsx:29`).
- **Recipe card:** `src/components/recipes/recipe-library-card.tsx:30` — `aspect-[4/3] w-full` image block with an overlaid favorite heart (`recipe-library-card.tsx:45-58`), then title + tag chips below (`recipe-library-card.tsx:61-78`). Rendered in a single-column `grid gap-3` (`src/components/recipes-view.tsx:303-312`).
- **Recipe loading skeleton:** `src/components/recipes-view.tsx:227-240` mirrors the vertical card shape (`aspect-[4/3]` pulse block).
- **Breakpoint convention:** `useIsMobile()` = `max-width: 768px`; Tailwind `md:` = `min-width: 768px`, so `md:` variants are the CSS mirror of the hook.

## Goals

- Mobile Chores states the active period exactly once (in the switcher).
- Mobile Recipes shows roughly 3–4 recipe cards per screen instead of ~1, keeping the photo for visual scanning.
- Desktop rendering of both modules is unchanged.

## Non-goals

- No new module features, no backend changes.
- No rework of the Chores desktop 3-column board or the chore row/assignee-group internals.
- No Recipes grid/virtualization work, no detail-view changes, no image pipeline changes.
- F1 (shared header), F2 (settings sheets), F3 (lists controls) are separate stories.

## Decision Summary

### D1. Chores: heading hidden on mobile, kept on desktop

`ChoreScopeColumn` gains a `showHeading` prop (default `true`). `ChoresView` passes `showHeading={!isMobile}`. On desktop the three side-by-side boards still need their headings; on mobile the heading row is dropped and the "`N` left of `M`" summary moves up to the top of the card (it carries real information and stays in both layouts).

Accessibility: the `<h2>` is the `aria-labelledby` target for the `<section>`. When the heading is hidden, the section keeps an accessible name via `aria-label={heading}` instead — screen-reader users still hear which period board they're in.

### D2. Recipes: horizontal thumbnail card on mobile (decided with Joe 2026-06-11)

The fork (rectangular thumbnail card vs. compact text list row vs. 2-column grid) was resolved in favor of the **horizontal thumbnail card**: on mobile the card becomes a row — square thumbnail (~96px, `object-cover`, rounded) on the left; title, tag chips, and the favorite heart on the right. Desktop (≥768px) keeps the current vertical card exactly as-is.

Implementation is **CSS-first**: one card component with responsive classes (`flex-row` base, `md:flex-col` etc.), no `useIsMobile()` branching and no second component. The favorite heart moves from an image overlay to an inline element in the text block on mobile (overlay returns on `md:`). The "No photo" fallback becomes a small muted square thumbnail on mobile. Tag chips clamp to a single row on mobile (`overflow-hidden`, single-line flex) so a tag-heavy recipe can't restore the height problem.

The loading skeleton in `recipes-view.tsx` is updated to mirror the new mobile row shape (`md:` restores the vertical skeleton).

## Visual / Layout Contract

- **Chores (mobile):** view top-down: "Chores" title row → Day/Week/Month switcher → scope card opening directly with "`N` left of `M`" + assignee groups. No "Today"/"This Week"/"This Month" text anywhere in the card.
- **Chores (desktop):** unchanged — three columns, each with its heading + summary.
- **Recipe card (mobile, <768px):** horizontal row, `~96px` square thumbnail left (`rounded-lg`, `object-cover`), text block right with title (up to 2 lines, truncate beyond), one row of tag chips, inline favorite heart aligned with the title row. Whole card stays one tap target (existing overlay button pattern). Card height ≈ thumbnail height + padding — roughly 112–120px.
- **Recipe card (desktop, ≥768px):** unchanged — full-width `aspect-[4/3]` image, overlay heart, title + wrapped tags below.

## Acceptance Criteria

- [ ] On a 360–430px viewport, the Chores card for the selected scope shows no period heading; the period appears only in the segmented switcher. The "`N` left of `M`" summary is still visible.
- [ ] Desktop Chores (≥1024px) still shows all three boards with their "Today"/"This Week"/"This Month" headings.
- [ ] Each scope `<section>` still has an accessible name matching its period in both layouts.
- [ ] On a 360–430px viewport, at least 3 recipe cards are fully visible in the library list at default browser font size (assuming ≥3 recipes exist).
- [ ] The mobile recipe card shows thumbnail (or "No photo" fallback), title, tags, and favorite state; tapping anywhere on the card opens the recipe; the favorite heart still reflects favorite state.
- [ ] Desktop recipe cards (≥768px) are visually unchanged.
- [ ] The recipes loading skeleton matches the mobile row shape on mobile.
- [ ] Existing unit and E2E suites pass; `recipe-library-card.test.tsx` updated where it asserts structure.
