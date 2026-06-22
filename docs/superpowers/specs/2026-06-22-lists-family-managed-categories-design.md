# Lists Family-Managed Categories

## Problem

The shipped `Lists` foundation already models categories as family-scoped records shared by list kind, but it deliberately made those records read-only. `Grocery` and `To-do` receive fixed starter categories, while `General` is forced to remain a flat checklist with no category support.

That first slice left the right persistence seam but the wrong long-term product boundary. A family's vocabulary is personal: one household may organize groceries by store aisle, another by store, and a General list may need categories such as `Documents`, `Packing`, or `Ideas`. Families should own the category catalog for each list kind, and every list of that kind should reuse it.

## Goals

- Let a family create, rename, delete, and reorder categories for each list kind.
- Make one ordered category catalog available to every list of the matching kind.
- Let `General` lists opt into the same grouped/flat behavior as `Grocery` and `To-do` without receiving starter categories.
- Preserve the lightweight Lists contract: each item has zero or one category, and category assignment remains optional.
- Support frictionless category creation while adding or editing an item.
- Evolve the shipped database safely with a new Flyway migration and a BE-first release.

## Non-goals

- Multiple tags per item or a many-to-many tagging model.
- Per-list category catalogs or per-list category order.
- Category colors, icons, descriptions, merge, history, or audit trails.
- Drag-and-drop ordering.
- A central category-management surface in Preferences.
- Automatically enabling grouped mode when a category is created.
- Offline category writes.
- Assignees, due dates, reminders, recurrence, or other `Chores` semantics.

## Product Invariants

### D1. Category ownership is family + list kind

A category belongs to exactly one family and one list kind. It is available to every list of that kind in that family and to no other family or kind.

```text
Family
 ├─ Grocery category catalog ──► every Grocery list
 ├─ To-do category catalog   ──► every To-do list
 └─ General category catalog ──► every General list
```

Changing a category name or order affects every matching list. The UI must state this shared scope wherever categories are managed.

### D2. One item has zero or one category

`categoryId` remains nullable and singular. `Uncategorized` is a synthetic UI section for items with `categoryId = null`; it is not a database row.

### D3. Categories are optional structure

Flat view hides category headings without erasing assignments. Grouped view uses the family-defined order, shows only non-empty visible groups, and includes `Uncategorized` only when it contains visible items.

### D4. Starter categories are ordinary family data

New families receive the existing Grocery and To-do starter categories. General starts empty. Starter categories are fully renameable and deletable; the product and API do not distinguish them from categories created later.

### D5. General starts flat and changes only by explicit choice

New General lists default to `flat`. Creating the first General category does not regroup existing lists. A family explicitly selects `grouped` for each General list that should show headings.

## Domain And Persistence Model

### Category model

The durable category shape is:

- `id`
- `familyId`
- `kind`
- `name`
- `sortOrder`
- `createdAt`
- `updatedAt`

The shipped `seeded` field is removed from the database, BE DTOs, and FE types because it no longer carries behavior.

Names are trimmed, non-empty, at most 100 characters, and unique case-insensitively within `(familyId, kind)`. `Documents` and `documents` therefore conflict, while the same name may exist under another kind or family.

`sortOrder` is a dense zero-based sequence within one family + kind catalog. A newly created category appends last. Successful delete compacts the remaining sequence. Deleting the final category also switches every matching list still using `grouped` to `flat` in the same transaction; recreating a category later does not regroup those lists.

### List and item model

- All three list kinds may use `grouped` or `flat` display mode.
- Grocery and To-do lists continue defaulting to `grouped`.
- General lists continue defaulting to `flat`.
- All three kinds accept a nullable item `categoryId`, validated against the authenticated family and the list's kind.

### V17 migration

Do not modify the already-released `V12__create_lists_tables.sql`. Add `V17__enable_family_managed_list_categories.sql` to roll the schema forward:

1. Drop the category-kind check that permits only `GROCERY` and `TODO`, then recreate it for `GROCERY`, `TODO`, and `GENERAL`.
2. Drop the constraint forcing General lists to `FLAT`.
3. Drop the constraint forcing General items to have `category_id IS NULL`.
4. Drop the case-sensitive category-name unique constraint and replace it with a case-insensitive unique index over family, kind, and normalized name.
5. Drop the `seeded` column.

Existing lists, items, preferences, and categories remain intact. On a fresh database Flyway applies V12 and then V17; on an existing database it applies only V17. Both paths reach the same final schema.

### Registration initialization

Category initialization remains part of new-family registration, but it is one-time starter creation rather than an ongoing reconciliation process:

```text
new family registration
  ├─ create Lists preferences
  ├─ create Grocery starter categories
  ├─ create To-do starter categories
  └─ create no General categories
```

There is no category-repair endpoint or background reseeding. Deleting a starter category is permanent unless the family explicitly recreates it.

## API Contract

### Catalog endpoints

Add a dedicated family category-catalog resource under `/api/lists`:

```text
GET    /api/lists/categories?kind={kind}
POST   /api/lists/categories
PATCH  /api/lists/categories/{categoryId}
DELETE /api/lists/categories/{categoryId}
PUT    /api/lists/categories/order
```

All responses follow the existing `ApiResponse<T>` envelope.

#### Read catalog

`GET /api/lists/categories?kind={kind}` returns the authenticated family's ordered catalog for one required kind. The response data includes `kind`, `groupedListCount`, and the ordered `categories`. Each management entry includes:

- `id`
- `kind`
- `name`
- `sortOrder`
- `itemCount` across every matching list in the family

`itemCount` and `groupedListCount` support honest delete confirmation, including the final-category flattening case. They are management metadata and do not need to be added to ordinary list detail responses.

#### Create

`POST /api/lists/categories`

```json
{
  "kind": "general",
  "name": "Documents"
}
```

The created category appends to the current catalog and returns its canonical representation.

#### Rename

`PATCH /api/lists/categories/{categoryId}`

```json
{
  "name": "Travel documents"
}
```

The category is looked up through the authenticated family. Kind and order do not change through this endpoint.

#### Delete

`DELETE /api/lists/categories/{categoryId}` performs one transaction:

```text
lock category catalog
→ count matching items
→ set their category_id to null
→ delete category
→ compact remaining sortOrder values
→ if the catalog is now empty, switch matching grouped lists to flat
→ return exact affected counts
```

Response data:

```json
{
  "uncategorizedItemCount": 7,
  "flattenedListCount": 2
}
```

If any step fails, nothing changes.

#### Reorder

`PUT /api/lists/categories/order`

```json
{
  "kind": "general",
  "expectedCategoryIds": ["documents-id", "packing-id", "ideas-id"],
  "categoryIds": ["packing-id", "documents-id", "ideas-id"]
}
```

The backend locks the family + kind catalog, compares its current ordered IDs with `expectedCategoryIds`, validates that `categoryIds` contains the same IDs exactly once, writes the dense new order, and returns the canonical catalog.

A concurrent create, delete, or reorder changes the expected sequence and produces `409 Conflict`. This prevents a stale device from silently overwriting another device's order without adding a catalog-version column.

### Existing list endpoints

- `GET /api/lists/{id}` returns ordered categories for all list kinds, including General.
- `PATCH /api/lists/{id}` permits `grouped` for General.
- Item create/update accepts General category IDs while retaining family + kind validation.
- The ordinary embedded category response removes `seeded` and does not include `itemCount`.

## Frontend Experience

### List creation copy

Update kind descriptions:

- Grocery: customizable shopping categories with starter examples.
- To-do: customizable planning buckets with starter examples.
- General: flexible checklist; add categories whenever useful.

General still defaults to a flat checklist.

### List Options

Every list kind shows one Categories block:

```text
Categories
View: [ Grouped / Flat ]

[ Manage categories ]
Available across all General lists in your family.
```

For a kind with no categories, `Grouped` is disabled with `Create a category first.` Category creation does not automatically change the display mode.

On mobile, selecting `Manage categories` closes the half-height List Options sheet and opens the full-height manager. This avoids sheet-on-sheet stacking. Desktop uses a centered dialog with the same content component.

### Category manager

The manager is scoped to the current list's kind and shows:

- Shared-scope helper copy.
- Add category.
- Ordered rows with name and item count.
- Rename and delete actions.
- An explicit Reorder mode.

Empty state copy explains that categories can be created here or while adding an item.

Rename happens inline. Delete opens a confirmation such as:

```text
Delete "Documents"?
7 items across your General lists will become Uncategorized.
```

When deleting the final category, the confirmation also says how many matching lists will switch from grouped to flat. Creating a category later does not restore grouped mode automatically.

The success message uses the exact count returned by DELETE rather than assuming the possibly stale count shown before confirmation.

### Reorder mode

Reorder is a local draft with an explicit Save/Cancel boundary:

```text
server order
→ enter Reorder mode
→ move rows locally with accessible up/down controls
→ Save order
→ one PUT request
```

Arrow clicks never send network requests. Create, rename, and delete are disabled during Reorder mode so the baseline remains stable. Save is disabled until the sequence changes and while the request is pending. Cancel discards the draft. Closing with unsaved changes requires discard confirmation.

No drag-and-drop library or gesture is introduced. Up/down controls must be keyboard accessible and meet the existing mobile target-size standard.

### Inline category creation

The Add/Edit Item sheet offers category selection for every list kind:

```text
Category
[ Uncategorized ]

+ New category
```

`New category` expands a small inline name field in the existing sheet. It does not open another overlay. On successful POST, FE adds the returned category to cache and selects it for the item. The item is saved separately with that `categoryId`.

If item save later fails, the newly created category remains valid family catalog data and the item form stays open.

### Grouped and flat rendering

- Grouped view follows catalog order.
- Categories with no visible items are omitted.
- `Uncategorized` is rendered only when it has visible items.
- Flat view hides headings while retaining assignments.
- Completed-item sorting and visibility behavior remain unchanged inside each visible group.

## Frontend Cache Flow

Successful category create, rename, delete, or reorder must converge both cache surfaces:

```text
category mutation succeeds
  ├─ refresh/update current kind's management catalog
  └─ refresh cached list-detail queries
```

The active list may be updated immediately from the mutation response where practical, but invalidation remains the correctness backstop because one mutation affects multiple lists.

The Lists summary cache does not carry category data and does not require category-specific updates.

## Error And Concurrency Behavior

### Backend statuses

- `400 Bad Request`: missing/invalid kind, blank or overlong name, wrong-kind assignment, malformed reorder membership.
- `404 Not Found`: category does not exist within the authenticated family. Do not reveal cross-family records.
- `409 Conflict`: case-insensitive duplicate name or stale reorder baseline.

### Frontend recovery

- Inline-create failure preserves the item draft and entered category name.
- Rename/delete failure keeps the manager open and leaves server-backed state intact.
- Network/server failure while saving order preserves the local draft so the user may Retry or Cancel.
- Reorder `409` reloads the server order, exits the stale draft, and explains that categories changed elsewhere.
- A stale item form referencing a deleted category refreshes the list, selects `Uncategorized`, and asks the user to save again.
- Mutation controls are disabled only for the relevant pending request.

Delete remains fully transactional; there is no partially deleted state for FE to repair.

## Compatibility And Shipping

- V17 is additive and preserves current data.
- The existing FE remains usable against the new BE: it ignores General category behavior and does not rely on `seeded` at runtime.
- New FE must consume a published BE release containing V17 and the category endpoints, never unreleased BE `main`.
- Offline Lists remain readable from the existing persisted TanStack Query cache. Category writes require connectivity and do not introduce an offline outbox.

## Testing Expectations

### Backend

Cover:

- Flyway upgrade through V17 on PostgreSQL-compatible schema.
- New-family Grocery/To-do starter creation and empty General catalog.
- No automatic recreation of deleted starters.
- Category CRUD for all three kinds.
- Family and kind isolation.
- Trimmed, case-insensitive name uniqueness.
- General grouped/flat updates and General item category assignment.
- Catalog item counts across multiple lists.
- Atomic delete-to-Uncategorized behavior, final-category flattening, and returned counts.
- Dense order after create/delete/reorder.
- Reorder success with one transaction.
- Stale `expectedCategoryIds`, missing/duplicate/foreign IDs, and concurrent-change `409` behavior.
- Controller response statuses and authenticated integration flow.

### Frontend

Cover:

- Types, service methods, hooks, and MSW handlers for the released contract.
- Revised list-kind copy.
- General category management and explicit grouped opt-in.
- Shared category visibility across two lists of the same kind.
- Inline create-and-select while preserving the item draft.
- Empty-category and `Uncategorized` rendering.
- Rename/delete cache convergence.
- Delete confirmation count and returned success count.
- Reorder arrow clicks produce no request; Save produces exactly one complete-order request.
- Cancel and dirty-close behavior.
- Network failure preserves reorder draft; `409` refreshes canonical order.
- Accessibility for manager labels, focus, confirmation, keyboard order controls, and touch targets.
- Regression coverage for Grocery/To-do, completed visibility, clear-completed, and offline reads.

### Released-contract E2E

Against the published BE release, verify a family can:

1. Create a General list.
2. Create a category inline and save an item into it.
3. Opt the list into grouped view.
4. Observe the category in a second General list.
5. Reorder and persist through reload.
6. Rename and observe propagation.
7. Delete and observe affected items under `Uncategorized`.
8. Delete the final category and observe matching grouped lists switch to flat.

## Delivery Strategy

This is one root product story with two delivery issues created after the implementation plan exists:

1. BE: V17, category catalog API, list/item rule changes, and tests.
2. FE: management UI, inline creation, General grouping, cache behavior, and released-contract E2E.

The BE issue may execute in parallel with current FE-only work such as Home organizer summary FE #239 because it lives in the backend repository. FE category implementation remains queued until the BE release is published.

Before each implementation PR, map every non-negotiable requirement to code and tests. The FE PR must reference the released BE version used for E2E.

## Acceptance Criteria

- [ ] A family can create, rename, delete, and reorder categories for Grocery, To-do, and General.
- [ ] Categories and their order are shared across every family list of the matching kind.
- [ ] Category names are case-insensitively unique per family + kind.
- [ ] Grocery and To-do retain editable starter categories; General starts empty.
- [ ] General remains flat by default and can explicitly opt into grouped view.
- [ ] Each item has zero or one optional category.
- [ ] Add/Edit Item supports inline create-and-select without losing the item draft.
- [ ] Deleting a category atomically moves affected items to `Uncategorized` and reports the exact count.
- [ ] Deleting the final category atomically switches every matching grouped list to flat and reports the exact count.
- [ ] Reorder arrows are local-only; Save sends one concurrency-guarded request.
- [ ] Empty category groups remain hidden from the checklist.
- [ ] Existing Lists completion and visibility behavior remains intact.
- [ ] BE ships and publishes the stable contract before FE implementation is released.
