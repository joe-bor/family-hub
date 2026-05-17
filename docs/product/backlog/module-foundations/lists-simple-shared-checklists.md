---
id: lists-simple-shared-checklists
title: Lists simple shared checklists
epic: module-foundations
status: done
priority: P1
created: 2026-05-06
updated: 2026-05-17
issues:
  - BE #44
  - FE #163
prs:
  - BE PR #45
  - FE PR #164
---

## Context

This shipped story turned the `Lists` tab from a placeholder into the second real module surface after `Chores`: a lightweight shared-checklist space for groceries, packing, prep work, and other ad-hoc family coordination.

This story is also the product-language guardrail after the chores MVP: `Lists` is not a second chores surface. It should stay simpler, faster, and less structured than `Chores`.

## Scope

- Replace placeholder/sample list content in the shipping path with persisted family-scoped lists
- Support creating a named list
- Support adding checklist items to a list
- Support checking and unchecking items with immediate UI feedback
- Support editing and deleting checklist items
- Support more than one active list for different household contexts

## Out of Scope

- Person assignment or ownership per item
- Due dates, urgency ordering, or chore-style responsibility semantics
- Recurrence, reminders, or notifications
- Meal planning, pantry inventory, or grocery-store integrations
- Dashboard integration
- Photos / Chores / Meals implementation work

## Dependencies

- `module-foundations/chores-core-loop.md` should already be shipped so `Lists` can stay clearly distinct from `Chores`
- FE and BE ownership should be split only after the spec defines the smallest stable contract

## Acceptance Criteria

- [x] `Lists` reads from persisted backend data; placeholder/sample data is not used in the production path
- [x] A family can create a named list and see it in the mobile module surface
- [x] A family can add, edit, check, uncheck, and delete checklist items
- [x] Multiple lists can coexist without collapsing back into a single chore-style board
- [x] Completed items remain clearly distinct without turning `Lists` into an assigned-responsibility workflow
- [x] Empty states are intentional for both "no lists yet" and "list has no items"

## Notes

- Keep `Lists` deliberately lighter than `Chores`. If implementation starts adding assignees, due dates, or progress-by-person lanes, the story has drifted.
- The next planning step after this shipped story is to define the `Meals` week-ahead planning slice without collapsing it into `Lists` or `Chores`.
