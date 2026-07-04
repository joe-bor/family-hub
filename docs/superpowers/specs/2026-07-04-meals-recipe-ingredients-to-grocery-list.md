# Meals Recipe Ingredients To Grocery List

**Date:** 2026-07-04
**Status:** Spec ready for plan review
**Owner:** Family Hub product / planning
**Related backlog story:** `docs/product/backlog/module-foundations/meals-recipe-ingredients-to-grocery-list.md`
**Related designs:** [Meals And Recipes Foundation](2026-06-01-meals-and-recipes-foundation-design.md), [Lists Family-Managed Categories](2026-06-22-lists-family-managed-categories-design.md), [Focused Meal Planning Sessions](2026-06-28-focused-meal-planning-sessions.md)

## Problem

Focused meal planning just shipped (`0.3.21`), so a parent can now fill a week of dinners in one flow. The very next thing that parent does on a Sunday night is build the grocery list — and today that means opening every planned recipe, reading its ingredients, and hand-copying each line into a Lists checklist. Recipes already store ingredients (`RecipeDetail.ingredients: string[]`), and Lists now has family-managed grocery categories, so the raw material for a one-tap handoff exists in both modules but nothing connects them.

This story adds an **Add ingredients to grocery list** action to the visible Meals week. It gathers ingredients from recipe-backed planned meals, lets the parent review, edit, and select the rows, then appends the selected rows to a chosen grocery list in one confirmed action. It is the recipe-to-list ingredient handoff already named as a follow-on candidate in the Meals/Recipes foundation design, and it is the strongest cross-module daily-use payoff available now.

The feature must be **honest**: it only surfaces ingredients that a recipe actually stores, it never fabricates ingredients for quick meals, it never presents a partial write as success, and it disables writes offline with clear copy.

## Goals

- Let a parent generate a reviewed ingredient checklist from the recipe-backed meals planned in the visible, editable Meals week.
- Preserve each recipe's ingredient wording exactly; do no parsing, splitting, quantity extraction, normalization, or de-duplication.
- Let the parent edit and remove candidate rows, and select which rows to append, before anything is written.
- Never surprise the parent with quick meals: quick meals (and recipe meals with no stored ingredients) appear in an explicit "no recipe ingredients" section with manual add rows, not auto-generated guesses.
- Let the parent append to an existing grocery list, defaulting to the only grocery list when exactly one exists, or create a new grocery list in the flow.
- Append all selected rows in one confirmed, transactional action; offer **View list** on success.
- Add a **generic** backend bulk list-item append endpoint (no meal or recipe concepts leak into the Lists API) so the client makes one write instead of N independent writes.
- Disable writes offline with honest copy; never present a partial or failed write as success.

## Non-goals

- Pantry, inventory, or "already have it" tracking.
- Price, nutrition, or store integrations.
- Quantity parsing, unit math, or ingredient normalization.
- Automatic de-duplication or merging of ingredient rows.
- Automatic category assignment or category guessing for appended items.
- AI meal planning or quick-meal ingredient guessing.
- A reverse Lists-to-Meals flow, or any meal/recipe awareness inside the Lists API.
- Editing the source recipes from the review flow (recipe editing stays in Recipes).
- Offline writes / an outbox (offline writes remain deferred per PRD §7.5.3).

## Product Invariants

### D1. Derive only from the visible, editable week

The action and its candidate rows are derived exclusively from the currently visible Meals board for a **current or future** week. Past weeks never expose the action, matching the existing read-only past-week guardrail and the `Fill empty slots` visibility rule. The action operates on the visible week only; it never reaches into other weeks.

### D2. Ingredients are read verbatim from recipe detail

A candidate ingredient row is exactly one string from a recipe's `RecipeDetail.ingredients[]`, copied verbatim. The client performs no splitting, trimming beyond what the user types, quantity parsing, or normalization. If a recipe stores `"2 cups jasmine rice"`, the row text is `"2 cups jasmine rice"`.

### D3. Quick meals never generate ingredients

A quick meal (`sourceType: "quick"`) has no backing recipe and therefore contributes **zero** auto-generated rows. Quick meals are listed by name in a "no recipe ingredients" section so the parent sees them and understands nothing was invented for them, and may add rows manually if they choose.

### D4. Review before write

No list item is created until the parent confirms **Add to list**. Editing text, removing rows, toggling selection, and choosing/creating the target list are all local until that single confirmation.

### D5. One confirmed, all-or-nothing append

Selected rows append through one backend transaction. Either every selected row is created or none is. A failed append is never shown as success, and there is no partial-append state for the client to reconcile.

### D6. The Lists API stays generic

The backend append endpoint accepts `{ text, categoryId? }` items for any list kind. It has no concept of meals, recipes, ingredients, or grocery-specific behavior. "Groceries" is a frontend product choice about which lists the ingredient flow targets, not a backend rule.

### D7. Offline disables writes honestly

While offline, the review sheet still opens and reads from already-cached board and recipe data, but **Add to list** and **Create grocery list** are disabled with copy that explains why. No write is queued.

## Frontend Experience

### Entry point and visibility

A single **Add ingredients** action lives on the Meals board for the visible week (header or board overflow, alongside `Fill empty slots`).

It is shown only when **both** hold:

1. The visible week is editable (current or future) — reuse the exact predicate that gates `Fill empty slots`.
2. The visible `MealBoard` contains at least one recipe-backed planned entry: any slot `primary` or `extras[]` entry with `sourceType === "recipe"` and non-null `recipeId`.

**Design decision — visibility uses the cheap board signal, not resolved ingredients.** The board snapshot (`MealSlotEntry`) intentionally does not carry ingredients; only `RecipeDetail` does. Resolving "does at least one recipe actually have ingredient lines" would require fetching every planned recipe just to decide whether to render a button, which is wasteful and eager. So visibility is gated on the presence of a recipe-backed entry (cheap, already in the board), and true ingredient resolution happens when the sheet opens. If, after fetching, no recipe yields any ingredient line, the sheet shows an honest empty state plus the quick-meal section rather than hiding after the fact. This satisfies the acceptance intent ("appears when there is plausibly something to gather") without ever fabricating ingredients. The alternative — eager pre-resolution to make the button perfectly precise — is rejected as over-fetching for a visibility check.

Past weeks never show the action.

### Ingredient extraction

On open, the sheet walks the visible board across all days and slots, collecting every planned entry (each slot's `primary` and every entry in `extras[]`), and partitions them:

- **Recipe-backed with ingredients:** entries with `sourceType === "recipe"` and non-null `recipeId`. The client fetches `RecipeDetail` for each **distinct** `recipeId` (deduplicating *fetches* by recipe id through the existing TanStack Query recipe-detail cache — this is fetch reuse, not ingredient de-duplication). Each string in `ingredients[]` becomes one candidate row (D2).
- **No recipe ingredients:** quick meals; recipe entries whose fetched `ingredients[]` is empty; and recipe entries whose recipe was deleted (detail fetch returns 404). These contribute no automatic rows and are surfaced in the "no recipe ingredients" section (D3).

Fetch states: while recipe details load, the sheet shows a loading state; a recoverable fetch error (non-404) for one recipe surfaces that meal in a retryable error row without blocking the rest; a 404 (deleted recipe) is treated as "no recipe ingredients," not an error.

### Review sheet

The sheet groups candidate rows **by meal**, preserving day/slot context and the meal's snapshot title:

```text
Add ingredients to grocery list

Wednesday · Dinner — Sheet Pan Chicken
  [x] 2 chicken breasts            (edit) (remove)
  [x] 1 tbsp olive oil             (edit) (remove)
  [x] 2 cups broccoli florets      (edit) (remove)

Thursday · Dinner — Pasta Primavera
  [x] 8 oz penne                   (edit) (remove)
  [x] 1 zucchini, sliced           (edit) (remove)

No recipe ingredients
  These meals have no saved recipe ingredients. Add anything you need by hand.
  Friday · Dinner — Taco night (quick meal)   [+ Add item]
  Saturday · Dinner — Grandma's stew (recipe has no ingredients)   [+ Add item]
```

Row behavior:

- Every recipe-derived row is **selected by default** with a checkbox; the parent can deselect any row. Only selected rows are appended.
- Every row's text is **editable** inline and any row can be **removed** before append (D4).
- Group-level and sheet-level select-all / clear conveniences are allowed.
- Duplicate ingredient strings from different meals appear as **separate rows** — there is no de-duplication (non-goal). The parent may remove duplicates by hand.
- Manual rows added under the "no recipe ingredients" section are ordinary editable/selectable rows; nothing is pre-filled for them.

**Category decision — v1 appends uncategorized.** Ingredient rows are appended with no `categoryId`. The feature does not guess categories, and manual per-row category assignment is deferred to keep v1 focused and honest (the parent can categorize inside Lists afterward). The backend endpoint still accepts an optional `categoryId` for genericness and future frontend use; the ingredient flow simply never sets it.

### Choose or create the grocery list

The sheet resolves target lists from the family's list summaries, filtered to `kind === "grocery"`:

- **Exactly one grocery list:** preselect it.
- **Multiple grocery lists:** the parent picks one from the grocery lists.
- **Zero grocery lists:** offer **Create a grocery list** — a name field that calls `POST /api/lists` with `{ name, kind: "grocery" }`, then targets the created list.

Only grocery lists are offered; to-do and general lists are never targets for this flow.

### Append and success

**Add to list** is one confirmed action that posts the selected rows to the bulk append endpoint for the chosen list. On success the sheet:

- Merges the returned created items into the cached `ListDetail` for the target list and updates the `ListSummary.totalItems` count.
- Shows success **only after** the single confirmed response, and offers **View list**, which navigates to the target grocery list.

On failure it surfaces the error and does **not** show success. Because the append is one transaction, there is no partial state to repair (D5).

**Create-then-append edge:** if the parent created a new grocery list and the subsequent append fails, the created (empty) list persists but the flow reports the append failure — never success — and offers retry into that list. Creating a list is a separate committed step; only the append determines success messaging.

### Offline

While offline, the sheet opens and reads from cached board and cached recipe details, but **Add to list** and **Create grocery list** are disabled with copy such as: "You're offline. Review your ingredients now and add them to your list when you're back online." No write is queued (D7).

## Backend Contract

### New endpoint: generic bulk list-item append

```text
POST /api/lists/{listId}/items/bulk
```

Request body:

```json
{
  "items": [
    { "text": "2 chicken breasts" },
    { "text": "1 tbsp olive oil", "categoryId": null }
  ]
}
```

- `items`: required, non-empty, at most `MAX_BULK_ITEMS` (**100**, a named tunable constant chosen to cover a full week of recipe-backed dinners with extras while bounding payload).
- Each item mirrors the existing `CreateListItemRequest`: `text` reuses the existing single-item text validation (trimmed, non-blank, existing max length); `categoryId` is optional and nullable.
- The endpoint is generic and carries **no** meal, recipe, or grocery-specific logic (D6). It works for any list kind.

Behavior:

- **Auth + family scope:** the list is looked up through the authenticated family. If `listId` does not belong to the authenticated family, return `404 Not Found` without revealing cross-family existence.
- **Category validation:** each non-null `categoryId` must belong to the same family and match the target list's kind, reusing the existing single-item category-assignment validation and its status codes (wrong-kind → `400`; not found in family → `404`). Before validating any non-null category assignment, acquire the `(familyId, listKind)` catalog lock, consistent with the serialization rule in the Lists family-managed-categories design. When no item carries a category (the ingredient flow's default), no catalog lock is required.
- **Transaction:** validate the whole request first, then insert all items in **request order**, appended after the list's existing items. Any validation or persistence failure rolls back the entire batch — nothing is written (D5).
- **Response:** `201 Created` with the existing `ApiResponse<T>` envelope wrapping `ListItem[]` — the created items only, in the same order as the request `items`. No `Location` header (multiple resources created).

Status summary:

- `201 Created`: all items created; ordered `ListItem[]` returned.
- `400 Bad Request`: empty `items`, more than `MAX_BULK_ITEMS`, any blank/overlong `text`, or a `categoryId` of the wrong list kind.
- `404 Not Found`: `listId` not in the authenticated family, or a `categoryId` not found in the authenticated family (do not reveal cross-family records).

The existing single-item `POST /api/lists/{listId}/items` endpoint and all other Lists behavior remain unchanged.

## Data Flow

```text
Meals board (visible week, in cache)
  └─ recipe-backed entries ──► fetch RecipeDetail per distinct recipeId ──► verbatim ingredient rows
  └─ quick / no-ingredient entries ──► "no recipe ingredients" section (manual rows)
        │
        ▼
  review sheet (edit / remove / select)  ── all local ──►  choose or create grocery list
        │
        ▼  Add to list (one confirmed action, online only)
  POST /api/lists/{listId}/items/bulk  { items: [{ text }] }   (one transaction)
        │
        ▼
  ApiResponse<ListItem[]> (ordered)  ──►  update ListDetail + ListSummary cache  ──►  success + View list
```

## Error And Concurrency Behavior

### Backend

- All-or-nothing: a failure at any item writes nothing; there is no partial batch.
- Family and kind isolation for the list and every category reference; cross-family records are never revealed.
- Category-carrying bulk writes acquire the `(familyId, kind)` catalog lock before validating assignments, in the same acquisition order as existing item create/update, so a concurrent category delete cannot produce a dangling assignment.

### Frontend

- Recipe detail fetch: non-404 error → retryable per-meal error row; 404 → treated as "no recipe ingredients."
- Append failure → surface the error, keep the reviewed selection intact for retry, never show success.
- Create-then-append: a created list persisting after an append failure is expected; report the append failure and offer retry into that list.
- Offline → writes disabled with copy; reads remain available from cache.

## Compatibility And Shipping

- The bulk endpoint is additive; it does not change existing Lists endpoints, DTOs, or the database schema. No migration is required (list items and the optional `categoryId` already exist).
- The existing frontend continues to work unchanged; only the new Meals action consumes the new endpoint.
- Per Family Hub shipping semantics, the frontend ingredient flow depends on the **published** backend release that adds the bulk endpoint. Frontend unit/component work (extraction logic, review UI) can proceed against MSW before that release; released-contract E2E must consume the published backend semver and record it. Frontend CI resolves the latest published backend release and fails closed rather than falling back to `latest`.

## Testing Expectations

### Backend

- Success: multiple items created, returned in request order, appended after existing items, with correct `totalItems`.
- Empty `items` → `400`.
- Over-`MAX_BULK_ITEMS` → `400`; the boundary (exactly `MAX_BULK_ITEMS`) succeeds.
- Blank / overlong `text` → `400`, whole batch rolled back.
- List not in authenticated family → `404`; cross-family list not revealed.
- Optional `categoryId`: valid same-kind category assigned; wrong-kind category → `400`; foreign/nonexistent category → `404`; a bad category mid-list rolls back the whole batch.
- Catalog-lock acquisition on category-carrying bulk writes matches existing item-create ordering (a concurrent final-category delete cannot leave a dangling assignment).
- Generic-endpoint proof: the same endpoint appends to a to-do or general list, demonstrating no grocery/meal coupling.

### Frontend

- Action visibility: shown for editable current/future weeks with ≥1 recipe-backed entry; hidden on past weeks and when the board has no recipe-backed entries.
- Extraction: recipe-backed entries produce verbatim rows grouped by meal; distinct recipe ids fetched once; quick meals and empty-ingredient / deleted recipes land in the "no recipe ingredients" section with no auto rows.
- Review: edit row text, remove a row, deselect rows; only selected rows are sent; duplicates are preserved (no de-dupe).
- List selection: single grocery list preselected; multiple grocery lists selectable; zero → create-grocery-list path; non-grocery lists never offered.
- Append: one bulk request with selected rows (uncategorized); success updates `ListDetail`/`ListSummary` cache and offers View list; failure is not shown as success; create-then-append failure reports failure and offers retry.
- Offline: writes disabled with copy; reads still work from cache.

### Released-contract E2E

Against the published backend release, verify a family can:

1. Create recipes that store ingredients and plan recipe-backed dinners in the current week, plus one quick meal.
2. Open **Add ingredients**, see rows grouped by meal with verbatim ingredient text, and see the quick meal under "no recipe ingredients."
3. Edit one row and remove one row.
4. Select a grocery list (or create one when none exists).
5. **Add to list** once, see success and **View list**, and find exactly the selected rows appended to that grocery list in order.

## Delivery Strategy

This is one root product story with two delivery issues created only **after** this spec passes plan review:

1. **BE:** `POST /api/lists/{listId}/items/bulk` generic bulk append — validation, transaction, ordered response, and tests. Additive; no migration.
2. **FE:** Meals **Add ingredients** action, ingredient extraction/fetching, review sheet (edit/remove/select), grocery-list picker/create handoff, bulk-append mutation + cache updates, and released-contract E2E.

The BE issue can execute in parallel with FE-only work because it lives in the backend repository. FE ingredient-flow release remains queued until the BE release is published; FE CI then resolves that published semver automatically. Before each implementation PR, map every non-negotiable requirement to code and tests. The FE PR must reference the released BE version used for E2E.

## Acceptance Criteria

- [ ] The **Add ingredients** action appears on editable current/future Meals weeks when the visible board has at least one recipe-backed planned entry, and never on past weeks.
- [ ] The review sheet groups rows by meal and preserves each recipe's ingredient wording exactly, with no parsing, splitting, or de-duplication.
- [ ] Quick meals (and recipe meals with no stored or no longer retrievable ingredients) appear in a "no recipe ingredients" section with manual add rows, never as auto-generated guesses.
- [ ] The parent can edit and remove candidate rows and select which rows to append before anything is written.
- [ ] The parent chooses an existing grocery list; when exactly one grocery list exists it is defaulted; when none exists the parent can create one; non-grocery lists are never targets.
- [ ] Selected rows append in one confirmed, transactional action via `POST /api/lists/{listId}/items/bulk`, and success offers **View list**.
- [ ] The bulk endpoint is generic (works for any list kind), family-scoped, validates optional categories against family + list kind, is bounded, returns created items in request order, and rolls back entirely on any failure.
- [ ] Offline disables **Add to list** and **Create grocery list** with honest copy; reads still work from cache.
- [ ] A partial or failed append is never presented as success; a created-then-failed-append flow reports failure and offers retry.
- [ ] Backend ships and publishes the endpoint before the frontend ingredient flow is released; FE CI resolves that published semver and fails closed rather than using `latest`.
