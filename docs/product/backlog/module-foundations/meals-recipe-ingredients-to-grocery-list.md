---
id: meals-recipe-ingredients-to-grocery-list
title: Meals recipe ingredients to grocery list
epic: module-foundations
status: planned
priority: P2
created: 2026-07-04
updated: 2026-07-04
issues:
  - https://github.com/joe-bor/family-hub-api/issues/68
  - https://github.com/joe-bor/FamilyHub/issues/277
prs: []
spec: ../../../superpowers/specs/2026-07-04-meals-recipe-ingredients-to-grocery-list.md
plan: ../../../superpowers/plans/2026-07-04-meals-recipe-ingredients-to-grocery-list.md
---

## Context

Focused meal planning shipped in `0.3.21`, so a parent can fill a week of dinners in one flow. The next thing they do on a Sunday night is build the grocery list — today by opening every planned recipe and hand-copying ingredients. Recipes already store ingredients (`RecipeDetail.ingredients: string[]`) and Lists now has family-managed grocery categories, so the raw material for a one-tap handoff exists in both modules but nothing connects them.

This story adds an **Add ingredients** action to the visible Meals week. It gathers ingredients from recipe-backed planned meals, lets the parent review, edit, and select rows, then appends the selected rows to a chosen grocery list in one confirmed action. It is the "recipe-to-list ingredient handoff" named as a follow-on candidate in the [Meals And Recipes Foundation](../../../superpowers/specs/2026-06-01-meals-and-recipes-foundation-design.md) design.

Design: [Meals Recipe Ingredients To Grocery List](../../../superpowers/specs/2026-07-04-meals-recipe-ingredients-to-grocery-list.md).

Plan: [Meals Recipe Ingredients To Grocery List Implementation Plan](../../../superpowers/plans/2026-07-04-meals-recipe-ingredients-to-grocery-list.md).

## Scope

- **Add ingredients** entry point inside `Meals` for the visible, editable (current/future) week only.
- Ingredient extraction from recipe-backed planned entries (`sourceType: recipe`), fetching `RecipeDetail` per distinct `recipeId`, preserving ingredient wording verbatim.
- Review sheet grouping rows by meal, with per-row edit, remove, and select; default-selected recipe rows.
- Explicit "no recipe ingredients" section for quick meals and recipe meals with no retrievable ingredients, offering manual add rows (never auto-generated guesses).
- Grocery-list picker: default to the only grocery list, choose among several, or create a new grocery list in-flow; non-grocery lists are never targets.
- One confirmed, transactional bulk append; success offers **View list**.
- A **generic** backend bulk list-item append endpoint (`POST /api/lists/{listId}/items/bulk`) with no meal/recipe/grocery concepts in the Lists API.
- Offline disables writes with honest copy; a partial or failed append is never shown as success.

## Out of Scope

- Pantry, inventory, or "already have it" tracking.
- Price, nutrition, or store integrations.
- Quantity parsing, unit math, or ingredient normalization.
- Automatic de-duplication or merging of rows.
- Automatic category assignment or category guessing for appended items.
- AI meal planning or quick-meal ingredient guessing.
- A reverse Lists-to-Meals flow or any meal/recipe awareness in the Lists API.
- Editing source recipes from the review flow.
- Offline writes / outbox (deferred per PRD §7.5.3).

## Acceptance Criteria

- [ ] The **Add ingredients** action appears on editable current/future Meals weeks when the visible board has at least one recipe-backed planned entry, and never on past weeks.
- [ ] The review sheet groups rows by meal and preserves each recipe's ingredient wording exactly, with no parsing, splitting, or de-duplication.
- [ ] Quick meals (and recipe meals with no stored or no longer retrievable ingredients) appear in a "no recipe ingredients" section with manual add rows, never as auto-generated guesses.
- [ ] The parent can edit and remove candidate rows and select which rows to append before anything is written.
- [ ] The parent chooses an existing grocery list; when exactly one grocery list exists it is defaulted; when none exists the parent can create one; non-grocery lists are never targets.
- [ ] Selected rows append in one confirmed, transactional action via the bulk endpoint, and success offers **View list**.
- [ ] The bulk endpoint is generic (works for any list kind), family-scoped, validates optional categories against family + list kind, is bounded, returns created items in request order, and rolls back entirely on any failure.
- [ ] Offline disables **Add to list** and **Create grocery list** with honest copy; reads still work from cache.
- [ ] A partial or failed append is never presented as success; a created-then-failed-append flow reports failure and offers retry.
- [ ] Backend ships and publishes the endpoint before the frontend ingredient flow is released; FE CI resolves that published semver and fails closed rather than using `latest`.

## Delivery Notes

- Create the backend and frontend execution Issues only after this root implementation plan passes spec-to-plan review.
- Backend delivery adds the generic bulk append endpoint and must publish it in a `family-hub-api` release before frontend released-contract E2E depends on it. The endpoint is additive — no Flyway migration.
- Frontend unit/component work (extraction, review UI) can proceed against MSW before the backend release; the released ingredient flow and its E2E consume the published backend semver.
- Execution Issues: BE [family-hub-api#68](https://github.com/joe-bor/family-hub-api/issues/68) (bulk append endpoint) and FE [FamilyHub#277](https://github.com/joe-bor/FamilyHub/issues/277) (Meals action + review sheet, queued on the BE release). Record PR links here as they land.
