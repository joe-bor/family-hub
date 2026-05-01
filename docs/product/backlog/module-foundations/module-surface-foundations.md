---
id: module-surface-foundations
title: Lists / Chores / Meals / Photos module foundations
epic: module-foundations
status: planned
priority: P1
created: 2026-05-01
updated: 2026-05-01
issues: []
prs: []
---

## Context

The shell now exposes first-class tabs for `Lists`, `Chores`, `Meals`, and `Photos`, but those surfaces are still mostly placeholder or sample-data experiences. `Home` and `Calendar` have crossed the line into real product surfaces; the rest of the module rail has not.

After `mobile-ux/visual-identity-refinement.md`, the next product focus is turning those shell destinations into real modules with documented scope, real data ownership, and implementation-ready slices.

## Scope

- Clarify the MVP purpose of `Lists`, `Chores`, `Meals`, and `Photos`
- Resolve the current language mismatch between PRD `Tasks` requirements and the shell's separate `Lists` + `Chores` destinations
- Decide the delivery order for the non-calendar modules after visual polish
- Create the first follow-on implementation stories with clear FE / BE ownership

## Acceptance Criteria

- [ ] Product docs explicitly define what `Lists`, `Chores`, `Meals`, and `Photos` each mean in MVP
- [ ] The vocabulary decision is recorded: whether `Chores` is the MVP task-management surface, how `Lists` differs, and what PRD wording needs alignment
- [ ] The delivery order after `visual-identity-refinement.md` is documented
- [ ] At least the first follow-on module implementation story is created with a clear contract
- [ ] Roadmap and backlog reflect this epic as the next focus after visual polish

## Notes

- Keep this epic sliced by module. Do not turn `Lists`, `Chores`, `Meals`, and `Photos` into one giant implementation issue.
- `Chores` is the shortest path to a real module because the PRD already contains task-management requirements that can likely anchor its MVP.
