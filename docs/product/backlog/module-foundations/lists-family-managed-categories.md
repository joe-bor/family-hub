---
id: lists-family-managed-categories
title: Lists family-managed categories
epic: module-foundations
status: done
priority: P2
created: 2026-06-22
updated: 2026-07-01
issues:
  - BE #60
  - FE #261
prs:
  - BE PR #63
  - BE release PR #62
  - FE PR #268
spec: ../../../superpowers/specs/2026-06-22-lists-family-managed-categories-design.md
plan: ../../../superpowers/plans/2026-06-22-lists-family-managed-categories.md
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
- General grouped/flat support, remaining flat by default; no kind may group an empty catalog
- Inline create-and-select from Add/Edit Item
- Atomic delete-to-`Uncategorized` across matching lists, with final-category flattening
- Family + kind catalog serialization for create/delete/reorder, list mode, and item assignment concurrency
- Flyway V17 expand/compatibility migration and dedicated category API; the unused `seeded` database column remains as a rollback tombstone while application contracts remove it
- BE-first delivery against the dynamically resolved latest published backend release, with no fallback to unreleased `latest`

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
- [ ] Grouped mode is rejected while a catalog is empty; Grocery/To-do list creation falls back to flat in that state.
- [ ] Items keep zero-or-one optional category semantics.
- [ ] Category names are case-insensitively unique per family + kind.
- [ ] Inline creation selects the new category without losing the item draft.
- [ ] Deleting a category atomically moves affected items to `Uncategorized` and reports the count.
- [ ] Deleting the final category atomically switches every matching grouped list to flat.
- [ ] Recreating a category after final deletion does not regroup existing lists.
- [ ] Catalog-dependent writes serialize by family + kind, including empty-catalog creates, and duplicate-name races return `409` rather than `500`.
- [ ] Reorder changes stay local until Save, which sends one concurrency-guarded request with defined invalid, pending, retry, and conflict behavior.
- [ ] Empty categories do not render empty checklist sections.
- [ ] Mobile sheet handoff, focus, keyboard ordering, announcements, and touch targets meet the design contract.
- [ ] Loading, error, stale-form, pending, and offline states preserve drafts and cached read-only Lists access.
- [ ] Existing Lists behavior and data survive both a clean migration and V16-to-V17 upgrade; the released BE remains rollback-compatible.
- [ ] FE integration and E2E resolve the latest published BE release image and fail closed if it is unavailable, never substituting mutable `latest`.

## Delivery Notes

- Create the BE and FE execution Issues only after the root implementation plan exists.
- BE implementation shipped in BE PR #63 and was published as `family-hub-api` `v1.7.0` via BE release PR #62 on 2026-06-27. FE #261 consumed that released contract and shipped in FE PR #268 as `family-hub` `0.3.20` on 2026-07-01.
- Normal FE CI discovers the latest published backend release automatically and must fail closed if it is unavailable.
- Record Issue and PR links in this frontmatter when they exist.
