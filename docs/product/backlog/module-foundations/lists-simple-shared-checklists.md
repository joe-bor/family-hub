---
id: lists-simple-shared-checklists
title: Lists simple shared checklists
epic: module-foundations
status: in-progress
priority: P1
created: 2026-05-06
updated: 2026-05-07
issues:
  - BE #44
  - FE #163
prs: []
---

## Context

The `Lists` tab is still a placeholder even though the shell now presents it as a first-class destination. With `Chores` shipped as the assigned-responsibility surface, the next module slice should turn `Lists` into the lightweight shared-checklist space for groceries, packing, prep work, and other ad-hoc family coordination.

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

- [ ] `Lists` reads from persisted backend data; placeholder/sample data is not used in the production path
- [ ] A family can create a named list and see it in the mobile module surface
- [ ] A family can add, edit, check, uncheck, and delete checklist items
- [ ] Multiple lists can coexist without collapsing back into a single chore-style board
- [ ] Completed items remain clearly distinct without turning `Lists` into an assigned-responsibility workflow
- [ ] Empty states are intentional for both "no lists yet" and "list has no items"

## Notes

- Keep `Lists` deliberately lighter than `Chores`. If implementation starts adding assignees, due dates, or progress-by-person lanes, the story has drifted.
- The next planning step after this story is to write a dedicated design spec and implementation plan for the smallest real `Lists` loop.
