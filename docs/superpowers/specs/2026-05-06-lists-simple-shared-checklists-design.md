# Lists Simple Shared Checklists

## Problem

Family Hub's `Lists` tab still behaves like a placeholder even though the mobile shell now presents it as a first-class destination. After `Chores` shipped as the assigned-responsibility surface, the next module slice should make `Lists` into the lightweight shared-checklist space for everyday family coordination.

This story also needs to protect the product boundary that emerged after the chores MVP: `Lists` is not a second chores module. Families should be able to create separate lists for whatever they want, including grocery runs, different stores, movies to watch together, or places to visit, without inheriting assignees, due dates, or chore-style workflow pressure.

## Goals

- Replace placeholder/sample list content with persisted family-scoped list data.
- Make `Lists` feel like a hub of separate family-owned lists rather than a single generic checklist.
- Support list-kind-aware category behavior while keeping the module lighter than `Chores`.
- Ship the smallest stable FE + BE contract for creating lists, adding items, completing items, hiding or showing completed items, and clearing completed items in bulk.
- Establish the product foundation for future category expansion without forcing custom category management into the first slice.

## Non-goals

- Adding assignees, due dates, reminders, recurrence, or chore-style responsibility semantics.
- Building custom family-managed categories in this story.
- Adding notes, quantities, prices, or other structured item metadata.
- Shipping a curated icon picker or a separate icon-management system.
- Building dashboard integration, meal-planning behavior, or photo-module functionality.
- Creating cross-household sharing or collaboration outside the family account.

## Scope

### Included

- Persisted lists owned by the family account.
- More than one active list in the module at once.
- Required list `name` and required list `kind` during create.
- Initial list kinds: `grocery`, `to-do`, and `general`.
- Seeded read-only family category sets for `grocery` and `to-do` lists.
- Optional item category assignment for category-aware list kinds.
- Category-aware list views that can switch between grouped and flat presentation.
- Family-wide completed-item visibility default for the module.
- Per-list completed-item visibility override when a specific list needs different behavior.
- Manual `Remove all completed` action for a list.
- Item create, edit, delete, check, and uncheck behavior.

### Explicitly excluded

- Assignee lanes, per-person progress, or any `Chores` board behavior.
- Real due dates or true urgency scheduling on list items.
- Category editing, custom category creation, or category deletion by families.
- Drag-and-drop reordering or manual sort controls.
- List rename or list deletion flows.
- List templates, duplicate-list actions, or list archiving.
- A required dedicated detail screen for individual items.

## Product Invariants

### D1. One family account owns the lists module

`Lists` follows the same family-account model as the rest of the MVP. Lists, items, and list preferences belong to the family account and are shared household state rather than per-user personal state.

### D2. `Lists` is intentionally lighter than `Chores`

The module must not drift into assignees, due dates, or responsibility tracking. If a proposed interaction starts answering "whose task is this?" or "when is this due?" in a structured way, it belongs in `Chores`, not `Lists`.

### D3. List kind determines whether category behavior exists

List kind is not cosmetic. It controls whether a list gets seeded shared categories:

- `grocery` lists use grocery categories
- `to-do` lists use lightweight urgency categories
- `general` lists stay uncategorized by default

### D4. Categories are shared within a family and list kind, not across all households

Seeded categories are shared across that family's lists of the same kind. A grocery category like `Produce` should behave consistently across grocery lists for one family, but it is not a single global record shared across every Family Hub household.

### D5. Categories are optional helpers, not required structure

Even on category-aware list kinds, an item does not have to belong to a category. The first slice must support uncategorized items so families can move quickly without being forced through extra structure.

### D6. Emoji can live inline in text, but no separate icon system ships yet

List names and item text may include emoji inline. That is enough to support visual use cases for now. A dedicated icon picker or richer icon model can be explored later if the module proves it needs one.

## Decision Summary

### D7. Make the module a hub of separate lists

The default mobile `Lists` experience is a hub of multiple family-owned lists. Each list is a named container with one list kind and its own checklist items.

This keeps the module broad enough for grocery runs, errands, movies, packing, and other family use cases without forcing everything into one endless checklist.

### D8. Gate shared categories by list kind

The initial kind behavior is:

- `grocery`: uses a seeded grocery category set, such as `Produce`, `Dairy`, and `Pantry`
- `to-do`: uses a seeded lightweight urgency category set, starting with `Urgent`, `Soon`, and `Later`
- `general`: no category system in the MVP

This gives grocery and to-do lists a better foundation now while keeping the category model intentionally small.

### D9. Let category-aware lists switch between grouped and flat modes

For `grocery` and `to-do` lists, families can hide categories and use a flat checklist view, or show categories and group items under category sections.

Implications:

- category-aware lists need a visible show/hide category control
- grouped mode should include an `Uncategorized` section when needed
- flat mode hides category structure without changing the underlying item data
- `general` lists do not show category controls

### D10. Use completed visibility defaults plus per-list override

Families get a family-wide default for whether completed items are visible in `Lists`. Any specific list can override that default if the household wants a different behavior for that one list.

Completed items remain part of the list's data model even when hidden from the current view.

### D11. Keep completed styling consistent wherever completed items are shown

Whenever a completed item is visible, it must always show:

- checked checkbox
- strikethrough text
- muted visual treatment

If completed items are shown, incomplete items should appear before completed items within the current grouping context. In grouped mode that means within each category section. In flat mode that means within the single visible checklist.

### D12. Provide a one-tap bulk cleanup path

Long-running lists should not require manual deletion of completed items one by one. Each list needs a manual `Remove all completed` action to clear completed items in bulk.

## Data Contract

### List model

The first shared contract should include:

- `id`
- `familyId`
- `name`
- `kind`
- `categoryDisplayMode`
- `showCompletedOverride | null`
- `createdAt`
- `updatedAt`

Notes:

- `kind` is one of `grocery`, `to-do`, or `general`.
- `categoryDisplayMode` is relevant only for category-aware lists and should support at least `grouped` and `flat`.
- `showCompletedOverride` is tri-state in practice:
  - `null` means use the family default
  - `true` means always show completed items for this list
  - `false` means hide completed items for this list

### List item model

The first shared contract should include:

- `id`
- `listId`
- `text`
- `completed`
- `completedAt | null`
- `categoryId | null`
- `createdAt`
- `updatedAt`

Notes:

- `text` is plain text and may include emoji inline.
- `categoryId` is nullable so category-aware lists can still support uncategorized items.
- `completedAt` supports bulk-clear logic, completed-state rendering, and future history needs without adding per-user identity.

### List category model

The first shared contract should include:

- `id`
- `familyId`
- `kind`
- `name`
- `seeded`
- `sortOrder`

Notes:

- categories are family-scoped and shared across lists of the same `kind`
- the initial category sets are seeded and read-only in this story
- `general` lists do not need category records

### List preferences model

The first shared contract should include:

- `familyId`
- `showCompletedByDefault`

This model exists to carry the family-wide default for completed-item visibility across the entire `Lists` module.

### API shape

The first API should support the core loop:

- `GET /lists`
- `POST /lists`
- `GET /lists/{id}`
- `PATCH /lists/{id}`
- `POST /lists/{id}/items`
- `PATCH /lists/{id}/items/{itemId}`
- `DELETE /lists/{id}/items/{itemId}`
- `POST /lists/{id}/clear-completed`
- `GET /lists/preferences`
- `PATCH /lists/preferences`

Expected responsibilities:

- `GET /lists` returns the family's list hub data needed to render the module landing state.
- `POST /lists` creates a family-owned list with required `name` and required `kind`.
- `GET /lists/{id}` returns the list detail payload needed to render one list, including list metadata, items, and the applicable seeded category set when the list kind supports categories.
- `PATCH /lists/{id}` supports the list-level fields needed in the first slice, specifically category display mode and completed-visibility override.
- `POST /lists/{id}/items` creates a new item with required `text` and optional `categoryId` when the list kind supports categories.
- `PATCH /lists/{id}/items/{itemId}` supports text edits, category changes, and complete / uncomplete updates.
- `DELETE /lists/{id}/items/{itemId}` permanently removes one item.
- `POST /lists/{id}/clear-completed` removes all completed items for that list.
- `GET /lists/preferences` returns the family-wide completed-visibility default.
- `PATCH /lists/preferences` updates that family-wide default.

The API does not need to expose category-management writes in this story. Seeded categories are read-only module foundation data.

## UI Contract

### Hub layout

On mobile, `Lists` opens to a family-owned list hub with:

- page title
- `New List` action
- vertically stacked list cards or rows

Each list summary should show enough information to scan quickly, including:

- list name
- list kind
- lightweight progress or remaining-item summary

### List detail behavior

Inside an individual list, the first slice should support:

- add item
- edit item
- delete item
- check / uncheck item
- category show / hide control for category-aware kinds
- completed visibility behavior based on family default plus optional per-list override
- `Remove all completed` action

### Category-aware rendering

For `grocery` and `to-do` lists:

- grouped mode renders items under category sections
- uncategorized items render under an `Uncategorized` section when present
- flat mode removes category headers from the active view

For `general` lists:

- the list renders as a single flat checklist
- no category show/hide control is shown

### Create flow

Creating a list must ask for:

- `name`
- `kind`

The chosen `kind` should make the downstream behavior legible. If the list is `grocery` or `to-do`, the user should understand that seeded shared categories will be available inside the list. If the list is `general`, the user should understand that it will remain a flat checklist.

Creating or editing an item must ask only for:

- `text` (required)
- `category` (optional when the current list kind supports categories)

No notes, quantity, price, or due-date fields ship in this slice.

### Completed-item behavior

Completed behavior must support both family styles:

1. Families who want to keep completed items visible
   - completed items stay visible with checked, muted, strikethrough treatment

2. Families who want a cleaner active checklist
   - completed items can be hidden from the active view through the family default or a per-list override

### Empty states

Required empty-state cases:

1. No lists exist at all
   - Show a clear family-level empty state with CTA to create the first list.

2. A list exists but has no items
   - Show a clear list-level empty state with CTA to add the first item.

## Acceptance Criteria

- [ ] `Lists` reads from persisted backend data; placeholder or sample list data is not used in the production path.
- [ ] The module opens to a hub of multiple family-owned lists rather than a single combined checklist.
- [ ] A family can create a list with required name and required kind.
- [ ] `Grocery` lists use seeded shared grocery categories for that family.
- [ ] `To-do` lists use seeded shared urgency categories for that family.
- [ ] `General` lists remain uncategorized by default.
- [ ] Category-aware lists can switch between grouped category view and flat hidden-category view.
- [ ] A family can add, edit, delete, check, and uncheck items.
- [ ] Completed items always use checked, muted, strikethrough styling whenever they are visible.
- [ ] The module supports a family-wide completed-visibility default plus per-list override.
- [ ] A list supports `Remove all completed` as a manual bulk-cleanup action.
- [ ] Empty states are intentional for both `no lists yet` and `list has no items`.
- [ ] The module does not reintroduce assignees, due dates, or chore-style responsibility semantics.

## Quality Bar

- FE must stop using placeholder list data in the shipping path.
- Category-aware behavior should feel helpful but optional, not mandatory.
- Create, edit, check, uncheck, delete, and clear-completed actions should feel immediate in the UI, then persist successfully.
- Seeded categories should be shared consistently across that family's lists of the same kind without duplicating per list.
- Completed visibility rules should be predictable across family defaults and list-level overrides.
- The module should remain comfortable on mobile while still degrading cleanly on desktop.

## Testing Expectations

Implementation should cover:

- list create flow with name + kind
- family hub rendering with more than one list
- seeded category behavior for `grocery`
- seeded urgency-category behavior for `to-do`
- `general` list behavior without categories
- grouped vs flat rendering for category-aware lists
- uncategorized item behavior
- item create, edit, delete, check, and uncheck flow
- completed styling when visible
- family-wide completed default plus per-list override behavior
- `Remove all completed` flow
- no-lists empty state
- empty-list state

## Delivery Notes

- This is one product story with one root spec and one root implementation plan.
- Execution should likely split into two delivery issues after planning:
  - BE: lists persistence, seeded categories, preferences, and API
  - FE: lists hub, list detail UI, item interactions, and settings wiring against the released BE contract
- FE shipping should use a released BE contract per the root repo shipping rules.
