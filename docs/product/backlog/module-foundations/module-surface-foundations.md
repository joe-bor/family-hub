---
id: module-surface-foundations
title: Lists / Chores / Meals / Photos module foundations
epic: module-foundations
status: in-progress
priority: P1
created: 2026-05-01
updated: 2026-06-02
issues:
  - BE #52
prs: []
---

## Context

The shell now exposes first-class tabs for `Lists`, `Chores`, `Meals`, and `Photos`. `Home`, `Calendar`, `Chores`, and `Lists` have now crossed the line into real product surfaces; `Meals` and `Photos` remain placeholder destinations that still need product definition and real data ownership.

After `mobile-ux/visual-identity-refinement.md`, this epic moved from vocabulary-setting into phased module delivery. The next product focus is turning the remaining placeholder destinations into real modules, starting with `Meals`.

## Scope

- Clarify the MVP purpose of `Lists`, `Chores`, `Meals`, and `Photos`
- Resolve the current language mismatch between PRD `Tasks` requirements and the shell's separate `Lists` + `Chores` destinations
- Decide the delivery order for the non-calendar modules after visual polish
- Create the first follow-on implementation stories with clear FE / BE ownership

## MVP module definitions

- `Chores` — recurring family routines with present-focused completion by day, week, and month. This is the current MVP chores contract and the home of the PRD's active chores requirements.
- `Lists` — shared ad-hoc checklists for things like shopping, packing, or prep work. Lists are lighter-weight than chores and are not person-assigned by default.
- `Meals` — week-ahead meal planning for family coordination. Start with planning, not recipes, pantry management, or grocery automation.
- `Photos` — family photo library / screensaver-management surface. Do not assume a final storage provider or Google Photos integration yet.

## Recommended delivery order

1. `Chores` core loop (shipped 2026-05-06, superseded 2026-05-17 by recurring routines)
2. `Chores` recurring routines (shipped 2026-05-21)
3. `Lists` simple shared checklists (shipped 2026-05-17)
4. `Meals` week-ahead planning
5. `Photos` library / screensaver administration

## Follow-on implementation stories

- [Chores recurring routines](./chores-recurring-routines.md) — current shipped chores contract
- [Chores core loop (real data, create, complete)](./chores-core-loop.md) — historical shipped one-off chores release, superseded 2026-05-17
- [Lists simple shared checklists](./lists-simple-shared-checklists.md) — shipped 2026-05-17
- `Meals` week-ahead planning — next recommended story to define

## Acceptance Criteria

- [x] Product docs explicitly define what `Lists`, `Chores`, `Meals`, and `Photos` each mean in MVP
- [x] The vocabulary decision is recorded: whether `Chores` is the MVP task-management surface, how `Lists` differs, and what PRD wording needs alignment
- [x] The delivery order after `visual-identity-refinement.md` is documented
- [x] At least the first follow-on module implementation story is created with a clear contract
- [x] Roadmap and backlog reflect this epic as the next focus after visual polish

## Notes

- Keep this epic sliced by module. Do not turn `Lists`, `Chores`, `Meals`, and `Photos` into one giant implementation issue.
- `Chores recurring routines` is now the live chores story. `Chores core loop` remains historical context only.
- `Chores` and `Lists` now anchor the module vocabulary in shipped product behavior; `Meals` should build on that clarity rather than re-open it.
- Remaining mobile-UX polish stories do not block the next module slice once `visual-identity-refinement.md` is complete.
