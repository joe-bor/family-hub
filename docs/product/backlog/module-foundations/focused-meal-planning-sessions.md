---
id: focused-meal-planning-sessions
title: Focused meal planning sessions
epic: module-foundations
status: done
priority: P2
created: 2026-06-28
updated: 2026-07-04
issues:
  - https://github.com/joe-bor/family-hub-api/issues/64
  - https://github.com/joe-bor/FamilyHub/issues/262
prs:
  - BE PR #65
  - BE release v1.8.0
  - FE PR #273
  - FE release v0.3.21
spec: ../../../superpowers/specs/2026-06-28-focused-meal-planning-sessions.md
plan: ../../../superpowers/plans/2026-06-28-focused-meal-planning-sessions.md
---

**Shipped** in backend `v1.8.0` and frontend `0.3.21` — BE PR #65, FE PR #273, FE release PR #274.

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

- [x] A user can start a focused planning session from the visible editable Meals week.
- [x] A user can choose to fill empty dinners, all empty slots, or all empty slots on selected days.
- [x] A user can use the same flow on current and future weeks, including a blank future week.
- [x] Past weeks do not allow planning sessions.
- [x] The app shows a clear queue of empty slots to fill.
- [x] The meal tray remains available across multiple slot choices.
- [x] A user can fill at least three slots before saving without opening three separate slot composers.
- [x] Draft choices do not change the real board until saved.
- [x] A user can skip a slot, remove a draft choice, change a draft choice, and cancel the whole session.
- [x] Canceling the session leaves the board unchanged.
- [x] Saving commits the drafted meals to the existing Meals board.
- [x] Existing planned meals are not overwritten by the default flow.
- [x] Save-time conflicts are detected and never silently overwrite filled slots.
- [x] On save conflict, the user can skip conflicted drafts and save the rest, keep editing, or cancel the save attempt without discarding the planning session.
- [x] Quick meals and recipe-backed meals both work in the session.
- [x] The session is meaningfully faster than manually opening each empty slot one at a time.

## Delivery Notes

- Create the backend and frontend execution Issues only after this root implementation plan passes spec-to-plan review.
- Backend delivery added the batch-save contract in BE PR #65 and published it in `family-hub-api` `v1.8.0` on 2026-07-01 before frontend released-contract E2E depends on it.
- Frontend delivery shipped in FE PR #273 and was published in `family-hub` `0.3.21` via FE release PR #274 on 2026-07-04.
- Record Issue and PR links in this frontmatter when they exist.
