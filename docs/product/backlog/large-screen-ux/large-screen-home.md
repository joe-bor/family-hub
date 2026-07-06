---
id: large-screen-home
title: Large-screen Home hub
epic: large-screen-ux
status: in-progress
priority: P2
created: 2026-07-05
updated: 2026-07-05
issues:
  - https://github.com/joe-bor/FamilyHub/issues/278
prs: []
spec: ../../../superpowers/specs/2026-07-05-large-screen-home-design.md
---

## Context

The mobile Home surface is now polished around the "what matters now" model.
The next product question is how Family Hub should behave on horizontal tablets,
kitchen-table devices, and eventually a 20-inch-plus home display.

The larger-screen experience should not be a stretched version of mobile Home
and should not default to a dense planning dashboard. It should become an
ambient household display that uses width for hierarchy, context, and
readability.

Design: [Large-screen Home hub](../../../superpowers/specs/2026-07-05-large-screen-home-design.md).

## Scope

- Large-screen Home as the default resting surface for horizontal tablet and
  bigger screens.
- Now-first A1 layout: dominant Now Hero, Today rail, tomorrow peek, and a
  restrained Chores / Meals / Lists state strip.
- Home routes into full modules for deeper work instead of opening inline
  workspaces.
- Fresh launch and long idle return to Home unless an active create/edit flow is
  open.
- Design-heavy screenshot review before the implementation is considered done.

## Out of Scope

- Redesigning Calendar, Meals, Chores, Lists, Recipes, or Photos module
  internals.
- Child-mode Home or family-table mode.
- Weather.
- Generic activity feed.
- Recipes or Photos surfaced on Home by default.
- Full desktop shell redesign.
- Backend changes unless implementation discovers a missing summary contract.

## Acceptance Criteria

- [ ] Larger screens can land on Home instead of being redirected straight to
      Calendar.
- [ ] Home is the resting state on fresh launch and after long idle, unless an
      active create/edit flow is open.
- [ ] The Now Hero is the unmistakable focal point and answers the most
      important current/next schedule question.
- [ ] The Today rail shows rest-of-day context without becoming a mini calendar.
- [ ] The tomorrow peek answers near-future awareness without expanding into a
      full agenda.
- [ ] The state strip includes only Chores, Meals, and Lists.
- [ ] Tapping event surfaces opens full Calendar with date/event context
      preserved.
- [ ] Tapping Chores, Meals, or Lists summaries opens the owning module.
- [ ] Home does not render inline event detail/edit panels or module workspaces;
      deeper work opens Calendar, Meals, Chores, or Lists.
- [ ] No fake weather, Recipes tile, Photos placeholder, or generic activity
      feed appears by default.
- [ ] Mobile Home remains visually and behaviorally unchanged.
- [ ] Required screenshots are produced, critiqued, and iterated until the
      large-screen experience reads as an ambient household display rather than
      a stretched phone screen or generic dashboard.
- [ ] The screenshot review includes an existing mobile Home baseline or
      equivalent regression evidence.

## Follow-Up Stories

- Large-screen Calendar polish.
- Large-screen Meals planning polish.
- Child/table mode.
