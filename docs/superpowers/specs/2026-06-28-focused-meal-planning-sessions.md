# Focused Meal Planning Sessions

**Date:** 2026-06-28
**Status:** Draft for product review
**Owner:** Family Hub product / planning
**Related foundation spec:** `docs/superpowers/specs/2026-06-01-meals-and-recipes-foundation-design.md`

## Summary

Add a focused planning session to the existing `Meals` board so a parent can fill the empty meal slots for the week they are already viewing.

The product concept is:

> The normal Meals board is for editing one meal. `Fill empty slots` is for sitting down and planning the week.

This is not a separate meal-planning module. It is a faster planning mode inside `Meals` that finds the empty slots, keeps the user in one focused flow, and lets them fill several meals before saving the plan to the real board.

## Current Product Context

Family Hub already has:

- a top-level `Meals` module
- a weekly board with breakfast, lunch, and dinner slots
- mobile day cards and a larger-screen weekly grid
- empty meal slots that can be tapped to add a meal
- saved recipe-backed meals
- quick meals without a recipe
- extras or sides
- recipe snapshots when a recipe is placed on the board
- move, duplicate, replace, remove, and add-extra behavior
- collision handling when a target slot already has a primary meal
- current and future weeks editable
- past weeks review-only in the normal frontend experience

The gap is not persistence or slot editing. The gap is the focused planning workflow. Today, the user can fill the week one slot at a time, but they still need to hunt for empty slots, repeatedly open and close the composer, and keep the whole weekly plan in their head.

## Problem

Meal planning is usually a session:

- What are the gaps this week?
- Which dinners do we still need?
- What do we already like?
- What quick meals can cover busy nights?
- What should we save as the week plan?

The current board supports the final result, but not the session. It records decisions after the user has already made them.

## Goals

- Help a parent fill empty meal slots for the visible week faster than manual slot-by-slot editing.
- Keep the existing Meals board as the source of truth.
- Make planning ahead work naturally by using the same flow on future weeks.
- Keep quick meals equal to recipe-backed meals.
- Protect existing planned meals from accidental overwrite.
- Let the user make several draft choices before saving anything to the real board.
- Preserve the low-friction Family Hub style: practical, calm, and household-oriented.

## Non-Goals

- A separate Meal Planner module.
- Grocery list generation in this first slice.
- Pantry or inventory tracking.
- Nutrition or calorie planning.
- AI-generated weekly plans.
- Automatic meal assignment.
- Per-person meal plans.
- Meal status such as cooked, skipped, or done.
- Saved multi-session draft plans.
- Copy-week templates in the first slice.
- Recurring meal rules in the first slice.

## User-Facing Concept

Add a button to the Meals week view:

`Fill empty slots`

This is the default v1 label. Product review may still choose different copy later, but implementation planning should treat `Fill empty slots` as the working label.

The button applies to the week currently visible on the board.

- If the user is viewing this week, it fills gaps in this week.
- If the user navigates to next week, it fills gaps in next week.
- If the next week is blank, the session can start from an empty week.
- If the user is viewing a past week, the action is hidden or disabled because past weeks are review-only.

The button should not mean "auto-plan my week." It means:

> Show me the empty meal slots for this week and help me fill them quickly.

## Core User Flow

### 1. User opens Meals

The user sees the normal weekly Meals board.

Example:

- Monday dinner: empty
- Tuesday dinner: spaghetti
- Wednesday dinner: empty
- Thursday dinner: empty
- Friday dinner: pizza night

The existing board remains familiar. Nothing about normal slot editing goes away.

### 2. User taps `Fill empty slots`

The first slice starts a scoped planning session.

The user chooses what kind of empty slots they want to fill.

V1 scope options:

- `Empty dinners` - the default, focused on weeknight planning
- `All empty slots` - breakfast, lunch, and dinner across the visible week
- `Selected days` - all empty meal slots for the days the user chooses

Deferred scope options:

- `Pick meal types`

Dinner remains the default because it is the highest-value planning use case, but v1 should still let the user choose a broader whole-week or whole-day planning session.

### 3. App identifies the planning queue

The app scans the visible week and builds a queue of matching empty slots.

For the example week:

- Monday dinner
- Wednesday dinner
- Thursday dinner

If the user chooses `All empty slots`, the queue can include breakfast, lunch, and dinner slots. If the user chooses `Selected days`, the queue includes the empty breakfast, lunch, and dinner slots for the selected days only.

Existing filled slots remain visible on the board, but they are protected by default. The session should not ask the user to replace Tuesday spaghetti or Friday pizza unless a later explicit action allows replacing filled slots.

For v1, an empty slot means a slot with no primary meal. Slots with a primary meal are excluded, even if they could accept extras or sides. Extras-only slots should not normally exist; if encountered, treat them as non-empty existing data rather than a target for this session.

### 4. Planning session starts

The board remains visible. The current slot is highlighted.

The session shows progress:

`Monday dinner - 1 of 3`

A meal tray stays open with candidate meals.

Candidate sections for the first slice:

- favorite recipes
- recent recipes
- search recipes
- add quick meal

Quick meal entry must be easy and first-class. Recent or repeated quick meals are useful later, but they should not be required in the first slice unless they already exist without new history behavior.

On phone, the session may use a bottom sheet or focused planning panel rather than keeping the full board visible at all times. The phone experience must still keep the current slot, queue progress, and enough week context visible or reachable without exiting the flow.

### 5. User fills slots as drafts

The user taps a candidate meal.

Example:

- Monday dinner gets draft `Tacos`
- app moves focus to Wednesday dinner
- user taps `Leftovers`
- app moves focus to Thursday dinner
- user searches recipes and chooses `Sheet Pan Chicken`

Draft choices appear on the board but are not saved yet.

The session supports:

- skip this slot
- remove a draft choice
- change a draft choice
- cancel the whole session

Swap and duplicate are deferred draft conveniences. They are useful follow-ons, but not necessary to prove the first focused planning session.

The important difference from normal board editing is that the user stays in one flow. They do not repeatedly open and close the same composer or hunt through the board for the next empty slot.

### 6. User reviews before saving

When the queue is filled or the user chooses to finish, the session shows a plain summary:

`3 meals ready to add`

Example:

- Monday dinner: Tacos
- Wednesday dinner: Leftovers
- Thursday dinner: Sheet Pan Chicken

Primary actions:

- `Save to week`
- `Keep editing`
- `Cancel`

Cancel discards drafts and leaves the real board unchanged.

### 7. User saves to the real board

When the user taps `Save to week`, the draft meals become normal planned meals on the existing Meals board.

After saving, the user is back on the normal weekly board. The meals are regular meal blocks and can be edited with the existing interactions.

## V1 Behavior Contract

The first slice has a strict draft contract:

- session choices stay temporary until `Save to week`
- cancel leaves persisted meals unchanged
- normal one-slot editing still works outside the session
- after save, meals are ordinary planned meal blocks
- no separate planning module or second board is introduced

Before saving, the app must re-check that each target slot is still empty. This protects against another device or another interaction filling a slot while the planning session is open.

If any target slot is no longer empty at save time, the app must not silently overwrite it. The user should see the affected slot or slots and choose one of these paths:

- skip the conflicted draft and save the remaining non-conflicted drafts
- return to editing the draft session
- cancel the save and leave the board unchanged

Replacing the newly filled slot or adding the draft as an extra can be considered later, but the first slice should avoid that complexity.

If saving fails partway through, the app must show a clear result instead of implying the whole plan saved. Product preference for v1 is to avoid partial success where practical. If implementation cannot guarantee an all-or-nothing save, the post-save result must clearly identify which meals were saved and which still need attention.

## Future Week Behavior

Future weeks use the same flow.

Example:

1. User opens `Meals`.
2. User taps `Next week`.
3. The next week has no planned meals.
4. User taps `Fill empty slots`.
5. User chooses `Empty dinners` or a broader scope such as `All empty slots`.
6. If the user chooses `Empty dinners`, the app queues all seven dinner slots.
7. If the user chooses `All empty slots`, the app queues breakfast, lunch, and dinner for the blank week.
8. User saves once.

This makes "plan ahead" feel like a natural extension of the current board rather than a separate workflow.

## Current Week Behavior

Current week also uses the same flow, but the queue is usually smaller because some meals may already be planned.

Example:

- Tuesday dinner is already planned.
- Friday dinner is already planned.
- Monday, Wednesday, and Thursday are empty.

`Fill empty slots` queues only Monday, Wednesday, and Thursday by default.

This supports the real household pattern where the week may be partially planned already.

## Existing Meals And Overwrite Rules

Existing planned meals are protected by default.

In the first version:

- `Fill empty slots` should only target empty slots.
- filled slots should remain visible as context
- filled slots should not be replaced during the session
- save-time conflicts must be detected before any overwrite can happen

A later version may add an explicit `include filled slots` or `replace meals` mode, but that is not needed for the first version.

## Candidate Meal Sources

### First Slice

Use sources that already fit the current product:

- favorite recipes
- recent recipes
- recipe search
- quick meal entry

### Later Sources

Good follow-ons:

- `Want to make soon` queue
- recent or repeated quick meals
- meals planned in previous weeks
- meals not used recently
- simple tags such as quick, kid-friendly, breakfast, dinner
- theme night defaults
- calendar-aware hints such as busy evenings

## Why This Adds Value Over Manual Slot Editing

Manual slot editing is still useful for changing one meal.

Focused planning sessions add value because they:

- find the empty slots for the user
- turn those slots into a clear planning queue
- keep the meal tray open across multiple choices
- avoid repeated open/save/close interactions
- show progress through the week
- let the user review multiple draft choices before saving
- protect existing meals by default

If the first version does not feel faster than manually tapping slots, it should not ship.

## MVP Recommendation

Build the first version as:

`Fill empty slots` for the visible week, with `Empty dinners` as the default scope.

MVP behavior:

- action appears on editable current and future weeks
- action is hidden or disabled on past weeks
- scope picker supports `Empty dinners`, `All empty slots`, and `Selected days`
- app creates a planning queue from the visible week and chosen scope
- user fills the queue from a persistent meal tray
- recipe-backed meals and quick meals are both supported
- selected meals appear as draft blocks
- user can skip, remove, change, and cancel draft choices
- user saves once to commit the drafted meals to the existing board
- existing planned meals are not overwritten

This is intentionally smaller than copy-week, grocery generation, or AI planning. It proves whether a focused planning session actually helps before expanding the module.

## Delivery Notes

The first version should reuse the existing Meals board as much as possible. A new persisted draft model is not part of the product requirement for v1; draft state can be temporary session state as long as cancel and save-time conflict protection are honored.

`Save to week` should feel like one user action. The implementation plan should decide whether this needs a dedicated batch-save contract or can be achieved safely with existing Meals writes after the user confirms save. That decision must preserve the v1 behavior contract above.

## Future Product Path

If the focused planning session works, it becomes the base for several useful follow-ons:

### Smarter meal ideas

The tray can learn from household behavior:

- meals we make often
- meals we have not had in a while
- good quick dinners
- favorite recipes
- recently added recipes

### Want-to-make-soon queue

Recipes and quick meals can be marked as `Want soon`. Those candidates rise to the top during a planning session.

### Copy and reuse

Future sessions can start from:

- last week
- a previous week
- a normal week template

This should come after the manual-first session proves useful.

### Draft conveniences

Later versions can add:

- swap two drafted meals
- duplicate a drafted meal into another slot
- include filled slots intentionally
- add draft as extra during conflict resolution

### Theme nights

The app can support lightweight patterns:

- Monday leftovers
- Tuesday tacos
- Friday pizza

These should be suggestions or defaults, not rigid automation.

### Calendar-aware hints

Because Family Hub already has calendar context, the planning session can eventually show soft hints:

- busy evening
- good night for leftovers
- no evening events

These hints should assist the user, not auto-plan the week.

### Grocery handoff

After saving the week, a later version could offer:

`Add ingredients to grocery list`

This should remain a handoff from meal planning, not the center of the first planning slice.

## Acceptance Criteria

- A user can start a focused planning session from the visible editable Meals week.
- A user can choose to fill empty dinners, all empty slots, or all empty slots on selected days.
- A user can use the same flow on current and future weeks.
- Past weeks do not allow planning sessions.
- The app shows a clear queue of empty slots to fill.
- The meal tray remains available across multiple slot choices.
- A user can fill multiple slots before saving.
- Draft choices do not change the real board until saved.
- Canceling the session leaves the board unchanged.
- Saving commits the drafted meals to the existing Meals board.
- Existing planned meals are not overwritten by the default flow.
- Save-time conflicts are detected and never silently overwrite filled slots.
- Quick meals and recipe-backed meals both work in the session.
- The session is meaningfully faster than manually opening each empty slot one at a time.

## Product Success Criteria

- A parent can fill a partially empty week's dinners in under three minutes.
- In a test scenario with three empty dinner slots, a parent can add three meals without opening three separate slot composers.
- A parent can choose a broader scope when they want to plan full days instead of only dinners.
- A parent can plan a blank future week at the chosen scope without leaving the Meals board.
- The user understands that drafts are not saved until `Save to week`.
- Users do not accidentally overwrite existing planned meals.
- The flow feels like a focused planning session, not a second meal board.

## Open Questions

- Should product review keep the v1 button label `Fill empty slots`, or choose alternate copy before implementation?
- Should a partially completed draft survive if the user closes the sheet accidentally?
- Should future copy-week behavior be a separate entry point or a later option inside the same planning session?
