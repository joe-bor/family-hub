---
id: lists-family-managed-categories
title: Lists family-managed categories
epic: module-foundations
status: planned
priority: P2
created: 2026-06-22
updated: 2026-06-22
issues: []
prs: []
spec: ../../../superpowers/specs/2026-06-22-lists-family-managed-categories-design.md
---

## Context

The shipped [Lists simple shared checklists](lists-simple-shared-checklists.md) story deliberately deferred custom category management. It nevertheless established categories as family-scoped records shared by list kind, leaving a strong foundation for this follow-on.

Families should own the vocabulary used to organize their lists. Grocery and To-do starter categories should be editable rather than permanent system fixtures, and General should be able to use family-created categories without losing its flat-by-default simplicity.

Design: [Lists family-managed categories](../../../superpowers/specs/2026-06-22-lists-family-managed-categories-design.md).

## Scope

- One family-owned ordered category catalog per list kind
- Category create, rename, delete, and explicit saved reorder
- Editable Grocery and To-do starter categories
- Empty-by-default General category catalog
- General grouped/flat support, remaining flat by default
- Inline create-and-select from Add/Edit Item
- Atomic delete-to-`Uncategorized` across matching lists, with final-category flattening
- Additive Flyway V17 migration and dedicated category API
- BE-first delivery against a published backend release

## Out of Scope

- Multiple tags per item
- Per-list category catalogs or ordering
- Category colors, icons, merge, history, or audit logs
- Drag-and-drop ordering
- Central Preferences entry
- Automatic General grouping
- Offline writes
- Chores-style assignment, scheduling, or recurrence

## Acceptance Criteria

- [ ] A family can manage categories for Grocery, To-do, and General.
- [ ] A category is available to every family list of its kind and never crosses family or kind boundaries.
- [ ] Grocery and To-do receive editable starter categories; General receives none.
- [ ] General lists remain flat by default and may explicitly enable grouped view.
- [ ] Items keep zero-or-one optional category semantics.
- [ ] Category names are case-insensitively unique per family + kind.
- [ ] Inline creation selects the new category without losing the item draft.
- [ ] Deleting a category atomically moves affected items to `Uncategorized` and reports the count.
- [ ] Deleting the final category atomically switches every matching grouped list to flat.
- [ ] Reorder changes stay local until Save, which sends one concurrency-guarded request.
- [ ] Empty categories do not render empty checklist sections.
- [ ] Existing Lists behavior and data survive the V17 upgrade.
- [ ] FE integration and E2E consume a published BE release.

## Delivery Notes

- Create the BE and FE execution Issues only after the root implementation plan exists.
- BE implementation can run in parallel with current FE-only work; FE category implementation waits for the released BE contract.
- Record Issue and PR links in this frontmatter when they exist.
