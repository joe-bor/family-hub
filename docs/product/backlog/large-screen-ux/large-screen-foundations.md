---
id: large-screen-foundations
title: Large-screen shell foundations
epic: large-screen-ux
status: done
priority: P2
created: 2026-07-06
updated: 2026-07-07
issues: []
prs:
  - https://github.com/joe-bor/FamilyHub/pull/282
spec: ../../../superpowers/specs/2026-07-06-large-screen-foundations-design.md
---

## Context

On large screens, stacked chrome (app header band with fake weather, plus two
per-module toolbar rows) eats vertical space before module content starts, and
several modules default to narrow centered columns with dead margins. This
story lands the shared foundation the per-module large-screen stories build
on.

Design: [Large-screen foundations](../../../superpowers/specs/2026-07-06-large-screen-foundations-design.md).

## Scope

- Slim the desktop app header to a single compact row; remove the fake
  weather chip.
- One merged toolbar row per module on large screens.
- Content-width discipline per module (no default narrow centered columns).
- Breakpoint semantics: mobile boundary unchanged at 768px; large-screen
  compositions activate at 1024px.

## Out of Scope

- Home hub (issue #278).
- Navigation rail redesign, global search.
- Per-module content composition (owned by module stories).

## Acceptance Criteria

- [ ] No fake weather chip in the module shell.
- [ ] App header is a single compact row (~64px or less) on large screens.
- [ ] One toolbar row per module on `lg+`.
- [ ] Calendar chrome above the time grid totals ~160px or less at 1440x900.
- [ ] Mobile shell unchanged; all touch targets remain at least 44px.

## Follow-Up Stories

- Large-screen Calendar, Lists, Chores, Meals, Recipes (this epic).
