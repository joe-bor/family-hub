---
id: focused-meal-planning-sessions
title: Focused meal planning sessions
epic: module-foundations
status: planned
priority: P2
created: 2026-06-28
updated: 2026-06-28
issues: []
prs: []
spec: ../../../superpowers/specs/2026-06-28-focused-meal-planning-sessions.md
plan: ../../../superpowers/plans/2026-06-28-focused-meal-planning-sessions.md
---

## Context

The shipped `Meals` module supports week-by-week planning, quick meals, recipe-backed meals, extras, move/duplicate/remove, and review-only past weeks. It still asks parents to fill a week one slot at a time.

This story adds a focused planning session inside the existing Meals board. A parent starts from the visible editable week, chooses which empty slots to fill, drafts several meal decisions in one flow, reviews them, and saves them to the normal board.

Design: [Focused Meal Planning Sessions](../../../superpowers/specs/2026-06-28-focused-meal-planning-sessions.md).

Plan: [Focused Meal Planning Sessions Implementation Plan](../../../superpowers/plans/2026-06-28-focused-meal-planning-sessions.md).

## Scope

- `Fill empty slots` entry point inside `Meals`, not a separate planner module
- Current and future week planning sessions for the visible week
- Scope choices: `Empty dinners`, `All empty slots`, and `Selected days`
- Empty-slot queue with current-slot progress
- Temporary draft choices that project onto the board but do not persist until `Save to week`
- Quick meal and recipe-backed draft choices
- Skip, remove draft, change draft, cancel planning, review, save, and save-conflict paths
- Atomic backend batch save for drafted empty slots
- Save-time conflict handling that never silently overwrites existing meals

## Out of Scope

- Grocery list generation
- Pantry or inventory tracking
- Nutrition planning
- AI-generated plans
- Automatic meal assignment
- Per-person meal plans
- Meal status such as cooked, skipped, or done
- Saved multi-session draft plans
- Copy-week templates
- Recurring meal rules
- Theme-night defaults
- Calendar-aware hints

## Acceptance Criteria

- [ ] A user can start a focused planning session from the visible editable Meals week.
- [ ] A user can choose to fill empty dinners, all empty slots, or all empty slots on selected days.
- [ ] A user can use the same flow on current and future weeks, including a blank future week.
- [ ] Past weeks do not allow planning sessions.
- [ ] The app shows a clear queue of empty slots to fill.
- [ ] The meal tray remains available across multiple slot choices.
- [ ] A user can fill at least three slots before saving without opening three separate slot composers.
- [ ] Draft choices do not change the real board until saved.
- [ ] A user can skip a slot, remove a draft choice, change a draft choice, and cancel the whole session.
- [ ] Canceling the session leaves the board unchanged.
- [ ] Saving commits the drafted meals to the existing Meals board.
- [ ] Existing planned meals are not overwritten by the default flow.
- [ ] Save-time conflicts are detected and never silently overwrite filled slots.
- [ ] On save conflict, the user can skip conflicted drafts and save the rest, keep editing, or cancel the save attempt without discarding the planning session.
- [ ] Quick meals and recipe-backed meals both work in the session.
- [ ] The session is meaningfully faster than manually opening each empty slot one at a time.

## Delivery Notes

- Create the backend and frontend execution Issues only after this root implementation plan passes spec-to-plan review.
- Backend delivery adds the batch-save contract first and must publish a backend release before frontend released-contract E2E depends on it.
- Frontend mock-backed unit/component work may start before that release, but live E2E must record the tested backend semver.
- Record Issue and PR links in this frontmatter when they exist.
