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

### D6. An empty catalog cannot be grouped

Grouped display is valid only while the family + kind catalog contains at least one category. Grocery and To-do lists default to `grouped` only when their catalog is non-empty; if a family has deleted every category, newly created lists of that kind start `flat`. General always starts `flat`. A request to enable `grouped` against an empty catalog is rejected, and recreating a category never regroups existing lists.

## Domain And Persistence Model

### Category model

The durable application category shape is:

- `id`
- `familyId`
- `kind`
- `name`
- `sortOrder`
- `createdAt`
- `updatedAt`

The shipped `seeded` field is removed from BE entities/DTOs and FE types because it no longer carries behavior. V17 retains the physical database column as an unused rollback-compatibility tombstone so the previously released BE can still start if a deployment must roll back. A later migration may drop the column only after that release is no longer a supported rollback target.

Names are trimmed, non-empty, at most 100 characters, and unique case-insensitively within `(familyId, kind)`. `Documents` and `documents` therefore conflict, while the same name may exist under another kind or family.

`sortOrder` is a dense zero-based sequence within one family + kind catalog. A newly created category appends last. Successful delete compacts the remaining sequence. Deleting the final category also switches every matching list still using `grouped` to `flat` in the same transaction; recreating a category later does not regroup those lists.

### Catalog serialization boundary

Every transaction that can read and then change a catalog invariant acquires the same transaction-scoped database lock keyed by `(familyId, kind)` before reading category rows. The lock must exist even when the catalog is empty; locking only current category rows is insufficient because two creates could otherwise both append at position zero.

This lock applies to category create, rename, delete, and reorder; list creation and grouped-mode changes; and item create/update when a category is assigned. All such paths acquire the catalog lock first, then any category, item, or list rows in deterministic order. The database case-insensitive unique index remains the authority for duplicate-name races; an application pre-check is only for early feedback.

### List and item model

- All three list kinds may use `grouped` or `flat` display mode.
- Grocery and To-do lists default to `grouped` while their catalog is non-empty and to `flat` while it is empty.
- General lists continue defaulting to `flat`.
- All three kinds accept a nullable item `categoryId`, validated against the authenticated family and the list's kind.

### V17 expand/compatibility migration

Do not modify the already-released `V12__create_lists_tables.sql`. Add `V17__enable_family_managed_list_categories.sql` to roll the schema forward:

1. Drop the category-kind check that permits only `GROCERY` and `TODO`, then recreate it for `GROCERY`, `TODO`, and `GENERAL`.
2. Drop the constraint forcing General lists to `FLAT`.
3. Drop the constraint forcing General items to have `category_id IS NULL`.
4. Drop the case-sensitive category-name unique constraint and replace it with a unique expression index over `(family_id, kind, lower(btrim(name)))`.
5. Retain `seeded` unchanged as an unused compatibility tombstone. New application code neither maps nor returns it.

Existing lists, items, preferences, and categories remain intact. Before creating the unique index, the migration checks for normalized duplicates within a family + kind and aborts with a diagnostic if any exist rather than choosing or deleting data. The released application only created fixed starter rows, but V12's case-sensitive constraint alone does not prove the stronger invariant. On a fresh database Flyway applies V12 through V17; on an existing V16 database it applies only V17. Both paths reach the same schema, including the compatibility column.

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

The category DTOs are distinct by purpose:

- `ListCategoryOption`: `id`, `kind`, `name`, `sortOrder`. This is embedded in list detail and returned anywhere an item needs a selectable category.
- `ListCategoryManagementEntry`: all `ListCategoryOption` fields plus `itemCount`.
- `ListCategoryCatalog`: `kind`, `groupedListCount`, ordered `categories: ListCategoryManagementEntry[]`.
- `CategoryDeleteResult`: `uncategorizedItemCount`, `flattenedListCount`.

No category DTO contains `familyId` or `seeded`.

#### Read catalog

`GET /api/lists/categories?kind={kind}` returns `200 OK` with the authenticated family's `ListCategoryCatalog` for one required kind. Each management entry includes:

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

The created category appends to the current catalog. The endpoint returns `201 Created`, a `Location` header for `/api/lists/categories/{categoryId}`, and the created `ListCategoryManagementEntry` with its authoritative zero `itemCount`.

#### Rename

`PATCH /api/lists/categories/{categoryId}`

```json
{
  "name": "Travel documents"
}
```

The category is looked up through the authenticated family. Kind and order do not change through this endpoint. Success returns `200 OK` with the updated `ListCategoryManagementEntry` and authoritative `itemCount`.

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

Success returns `200 OK` with `CategoryDeleteResult`. The operation acquires the family + kind catalog lock before counting or changing rows, so item counts, final-category detection, flattening, deletion, and order compaction describe one serialized state.

#### Reorder

`PUT /api/lists/categories/order`

```json
{
  "kind": "general",
  "expectedCategoryIds": ["documents-id", "packing-id", "ideas-id"],
  "categoryIds": ["packing-id", "documents-id", "ideas-id"]
}
```

The backend acquires the family + kind catalog lock, compares its current ordered IDs with `expectedCategoryIds`, validates that `categoryIds` contains the same IDs exactly once, writes the dense new order, and returns `200 OK` with the canonical `ListCategoryCatalog`.

A concurrent create, delete, or reorder changes the expected sequence and produces `409 Conflict`. This prevents a stale device from silently overwriting another device's order without adding a catalog-version column.

Empty, single-category, and unchanged complete orders are valid idempotent requests and return `200`. Validation order is intentional: after locking, a mismatch between the current IDs and `expectedCategoryIds` returns `409`; when the baseline matches, duplicated, missing, extra, foreign-family, or wrong-kind IDs in `categoryIds` return `400` without revealing which inaccessible ID exists.

### Existing list endpoints

- `GET /api/lists/{id}` returns ordered `ListCategoryOption` values for all list kinds, including General.
- List creation acquires the catalog lock before choosing the default display mode. Grocery and To-do use `grouped` only for a non-empty catalog; General uses `flat`.
- `PATCH /api/lists/{id}` permits `grouped` for every kind only when the catalog is non-empty. It acquires the same catalog lock and returns `409 Conflict` with `Create a category first.` if the catalog is empty.
- Item create/update accepts General category IDs while retaining family + kind validation and acquiring the catalog lock before validating a non-null assignment.
- The ordinary embedded category response is `ListCategoryOption`; it has neither `seeded` nor `itemCount`.

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

On mobile, selecting `Manage categories` closes the half-height List Options sheet completely before opening the full-height manager. The transition suppresses the Options sheet's normal focus restoration while handing off, registers only one overlay/back handler at a time, and focuses the manager heading or first actionable control after it opens. Closing the manager returns focus to the visible List Options trigger. Desktop uses a centered dialog with the same content component and equivalent focus behavior.

### Category manager

The manager is scoped to the current list's kind and shows:

- Shared-scope helper copy.
- Add category.
- Ordered rows with name and item count.
- Rename and delete actions.
- An explicit Reorder mode.

Empty state copy explains that categories can be created here or while adding an item.

The catalog is an online-only management query and is not persisted for offline use. While offline, `Manage categories` and inline `New category` are unavailable with explanatory copy; previously cached list detail and its embedded category options remain readable. The manager defines a loading state, a catalog-load error with Retry, input-adjacent create/rename errors, and delete errors that preserve the confirmation context for retry.

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

Arrow clicks never send network requests. Create, rename, and delete are disabled during Reorder mode so the baseline remains stable. Save is disabled until the sequence changes and while the request is pending. During the PUT, move controls, Save, Cancel, close, and all other mutations are disabled. A failed request retains the draft and exposes Retry or Cancel; a successful response replaces the baseline with the returned canonical order. Cancel discards the draft. Closing with unsaved changes requires discard confirmation.

No drag-and-drop library or gesture is introduced. Each control has an accessible name such as `Move Documents up`, is disabled at its boundary, retains useful focus after a move, and announces the new position through an `aria-live` status. Controls are keyboard operable and at least 44 by 44 CSS pixels.

### Inline category creation

The Add/Edit Item sheet offers category selection for every list kind:

```text
Category
[ Uncategorized ]

+ New category
```

`New category` expands a small inline name field in the existing sheet. It does not open another overlay. On successful POST, FE adds the returned category to cache and selects it for the item. The item is saved separately with that `categoryId`.

Inline-create failure preserves both the item draft and entered category name. If item save later fails, the newly created category remains valid family catalog data and the item form stays open with the created category still selected.

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

Catalog invalidation is scoped to the affected family session and kind. List-detail invalidation covers every cached list of that kind, not only the active list. Final-category deletion must visibly reconcile any open matching list after the authoritative response so its grouped/flat control and checklist render the server's flat state.

The Lists summary cache does not carry category data and does not require category-specific updates.

## Error And Concurrency Behavior

### Backend statuses

- `400 Bad Request`: missing/invalid kind, blank or overlong name, wrong-kind assignment, or malformed reorder membership after a matching reorder baseline.
- `404 Not Found`: category does not exist within the authenticated family. Do not reveal cross-family records.
- `409 Conflict`: case-insensitive duplicate name, stale reorder baseline, or an attempt to group a list whose catalog is empty.

Database unique-constraint violations for normalized names must be translated through the existing exception response convention to `409`, including two concurrent requests that both pass an application pre-check. They must never escape as `500`. Count queries are scoped by authenticated family and kind and count distinct matching lists or item rows without multiplying results through joins.

### Frontend recovery

- Inline-create failure preserves the item draft and entered category name.
- Rename/delete failure keeps the manager open and leaves server-backed state intact.
- Network/server failure while saving order preserves the local draft so the user may Retry or Cancel.
- Reorder `409` reloads the server order, exits the stale draft, and explains that categories changed elsewhere.
- Because the current error envelope does not identify which resource caused an item-save `404`, a form with a non-null category performs one list-detail refetch before recovering: if the list is gone, show the list-not-found state; if an edited item is gone, show the item-not-found state; if the selected category is absent, select `Uncategorized`, preserve every other field, and ask the user to save again; otherwise retain the draft and show the original error.
- Outside Reorder mode, mutation controls are disabled only for the relevant pending request. Reorder pending behavior follows the stricter rules above.

Delete remains fully transactional; there is no partially deleted state for FE to repair.

## Compatibility And Shipping

- V17 is an expand/compatibility migration that preserves current data and the unused `seeded` column. The released BE remains a viable rollback target while new BE and FE contracts omit `seeded`.
- The existing FE remains usable against the new BE: it ignores General category behavior, tolerates the expanded list-detail categories, and does not rely on `seeded` at runtime.
- Normal FE CI continues resolving GitHub's latest published BE release on every run and pulls that release's semver image. The image used by FE CI therefore advances only when a BE release PR is merged and its release is published; FE does not pin a version in the workflow for each PR.
- Release resolution fails closed. If the latest published release or its image tag cannot be resolved, FE CI fails instead of falling back to the mutable `latest` image, which represents unreleased BE `main`. The BE production deploy path follows the same fail-closed rule.
- New FE implementation remains queued until the category-capable BE release is published. Its E2E run records the resolved release version and must pass against that released contract.
- Offline Lists remain readable from the existing persisted TanStack Query cache. The new `ListCategoryOption` shape omits `seeded`, while validators continue tolerating legacy cached entries that contain the extra field. The management catalog is not persisted; category writes require connectivity and do not introduce an offline outbox.

## Testing Expectations

### Backend

Cover:

- Separate PostgreSQL integration paths for a clean V1-to-V17 migration and an actual V16-to-V17 upgrade, including the retained `seeded` compatibility column and normalized unique index.
- New-family Grocery/To-do starter creation and empty General catalog; starter rename/delete behaves like ordinary data and registration never recreates a deleted starter.
- Category CRUD for all three kinds.
- Family and kind isolation for every read, mutation, assignment, count, and route.
- Trimmed, case-insensitive name uniqueness, including two concurrent case variants producing one success and one mapped `409`, never `500`.
- General grouped/flat updates and General item category assignment; first category creation does not regroup any list.
- Empty-catalog rules: Grocery/To-do list creation falls back to flat, grouped PATCH returns `409`, and recreating after final deletion does not regroup existing lists.
- Catalog item and grouped-list counts across multiple lists without join overcounting.
- Delete-to-Uncategorized behavior, final-category flattening, order compaction, and returned counts, plus a forced mid-operation failure proving the entire observable change rolls back.
- Dense deterministic order after serialized create/delete/reorder, including concurrent creates and create/delete/reorder contention.
- Reorder empty, single, unchanged, and changed success cases; stale `expectedCategoryIds`; duplicate, missing, extra, foreign, and wrong-kind IDs; and concurrent create/delete/reorder conflicts.
- Catalog-lock coverage for the empty-catalog insert case and consistent acquisition order across category, list-mode, and item-assignment writes.
- Static category routes coexist with `/api/lists/{id}` and return the documented DTOs, statuses, `Location`, envelope, and authenticated behavior.

### Frontend

Cover:

- Types, service methods, query keys/hooks, fixtures, and MSW handlers for all documented DTOs, with static category handlers taking precedence over dynamic list-ID handlers.
- Revised list-kind copy.
- General category management, explicit grouped opt-in, and disabled grouped mode for an empty catalog.
- Shared category visibility across two lists of the same kind.
- Inline create-and-select while preserving the item draft through category-create and later item-save failures.
- Empty-category and `Uncategorized` rendering.
- Rename/delete cache convergence across multiple cached list details, including visible flattening after final-category deletion.
- Delete confirmation uses preflight counts while success uses authoritative response counts.
- Reorder arrow clicks produce no request; Save produces exactly one complete-order request.
- Reorder baseline/dirty state, Save disabled while unchanged, Cancel, dirty-close confirmation, pending control locks, retry, and successful baseline replacement.
- Network failure preserves the reorder draft; `409` refreshes canonical order and explains the conflict.
- Stale item-save `404` recovery distinguishes missing list, missing edited item, missing category, and unrelated failure while preserving the draft.
- Mobile Options-to-manager handoff proves no stacked sheets or duplicate back handlers, correct hardware-back/overlay behavior, and focus restoration; desktop uses the same content without parity regressions.
- Accessible move labels, boundary disabling, keyboard operation, focus retention, live position announcements, and 44-by-44 touch targets in browser/component coverage at the layer that can measure them.
- Manager loading, load-error Retry, mutation error, pending, empty, and offline states; offline mode disables writes while cached list detail remains readable.
- Offline validators accept the new embedded option without `seeded`, tolerate legacy cached options with extra `seeded`, and admit grouped General list detail.
- Regression coverage for Grocery/To-do, completed visibility, and clear-completed.

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

The mobile released-contract path also covers opening the manager from List Options and dismissing it with the hardware-back/overlay mechanism without exposing the closed Options sheet underneath.

### Release contract checks

- FE CI resolves and reports the latest published BE release version and pulls that semver image.
- A missing release or unresolvable release tag fails the workflow; no test or deploy path substitutes `latest`.
- The released-contract E2E cannot pass against an older BE release that lacks the category endpoints, which keeps FE category delivery queued until the BE release exists.

## Delivery Strategy

This is one root product story with two delivery issues created after the implementation plan exists:

1. BE: V17, category catalog API, list/item rule changes, and tests.
2. FE: management UI, inline creation, General grouping, cache behavior, and released-contract E2E.

The BE issue may execute in parallel with current FE-only work such as Home organizer summary FE #239 because it lives in the backend repository. FE category implementation remains queued until the BE release is published. Normal FE CI then picks up that release automatically through its latest-published-release resolution; it does not consume BE `main`.

Before each implementation PR, map every non-negotiable requirement to code and tests. The FE PR must reference the released BE version used for E2E.

## Acceptance Criteria

- [ ] A family can create, rename, delete, and reorder categories for Grocery, To-do, and General.
- [ ] Categories and their order are shared across every family list of the matching kind.
- [ ] Category names are case-insensitively unique per family + kind.
- [ ] Grocery and To-do retain editable starter categories; General starts empty.
- [ ] General remains flat by default and can explicitly opt into grouped view.
- [ ] No kind can enable grouped mode while its catalog is empty; Grocery/To-do list creation falls back to flat in that state.
- [ ] Each item has zero or one optional category.
- [ ] Add/Edit Item supports inline create-and-select without losing the item draft.
- [ ] Deleting a category atomically moves affected items to `Uncategorized` and reports the exact count.
- [ ] Deleting the final category atomically switches every matching grouped list to flat and reports the exact count.
- [ ] Recreating a category after final deletion does not regroup existing lists.
- [ ] All catalog-dependent writes serialize on family + kind, including empty-catalog creates, and duplicate-name races return `409` rather than `500`.
- [ ] Reorder arrows are local-only; Save sends one concurrency-guarded request with defined empty, unchanged, invalid, pending, retry, and conflict behavior.
- [ ] Empty category groups remain hidden from the checklist.
- [ ] Mobile overlay handoff, focus, keyboard controls, announcements, and touch targets meet the documented accessibility behavior.
- [ ] Loading, error, stale-form, pending, and offline states preserve user drafts and cached read-only access as documented.
- [ ] Existing Lists completion and visibility behavior remains intact.
- [ ] Clean and V16-upgrade migrations converge without data loss, and V17 retains the unused `seeded` column for BE rollback compatibility.
- [ ] BE ships and publishes the stable contract before FE implementation is released; FE CI resolves that published semver image and fails instead of falling back to `latest`.
