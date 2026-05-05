---
id: chores-core-loop
title: Chores core loop (real data, create, complete)
epic: module-foundations
status: in-progress
priority: P1
created: 2026-05-01
updated: 2026-05-05
issues:
  - BE #41
  - FE #158
prs: []
---

## Context

The current `Chores` tab is a polished placeholder built on generated sample data. It proves the shell destination exists, but it does not yet provide real household value.

The first real module slice after visual polish should turn `Chores` into a working family-responsibility surface with the smallest useful loop: see chores, add chores, complete chores.

This story is also the product-language anchor for the old PRD `Tasks` requirements. For MVP, `Chores` is the assigned-responsibility module; `Lists` stays separate as a lighter checklist surface.

## Scope

- Replace sample/generated chores in the shipping path with persisted family-scoped data
- Show incomplete chores grouped by assigned family member
- Allow creating a chore with:
  - title
  - single assignee
  - optional due date
- Allow toggling complete / incomplete with immediate UI feedback
- Allow revealing completed chores without making them the default focus

## Out of Scope

- Recurring chore templates
- Categories or tags
- Multi-assignee chores
- Points, streaks, rewards, or gamification
- Dashboard integration
- Lists / Meals / Photos work

## Dependencies

- Hard prerequisite: `mobile-ux/visual-identity-refinement.md`
- If FE and BE work are split into separate issues, FE shipping must use a released BE contract per root `AGENTS.md`

## Acceptance Criteria

- [ ] `Chores` reads from persisted backend data; generated sample chores are not used in the production path
- [ ] Default mobile view shows incomplete chores grouped by assigned family member
- [ ] Chores are ordered so urgent work is easier to scan (`overdue` first, then `due today`, then future / unscheduled)
- [ ] Parent can create a chore with required title, required single assignee, and optional due date
- [ ] Completing or uncompleting a chore updates the UI immediately and persists successfully
- [ ] A `Show completed` control reveals completed chores in a clearly distinct visual state
- [ ] Empty states are intentional for both "no chores yet" and "nothing left today"

## Notes

- This is the first real module slice because it has the strongest product value-to-scope ratio and already maps to existing PRD task-management requirements.
- Keep the contract crisp. If implementation starts pulling in recurrence, points, or cross-module summaries, the story has drifted.
