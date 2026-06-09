---
id: mobile-sidebar-settings-onboarding
title: Sidebar + settings + onboarding mobile pass
epic: mobile-ux
status: planned
priority: P2
created: 2026-04-23
updated: 2026-06-08
issues:
  - FE #193
prs: []
---

## Context

These screens haven't been specifically optimized for mobile. The sidebar slides in but the settings/onboarding flows could benefit from mobile-native layouts. (Originally from the mobile-ux-polish backlog.)

Scoped and specced on 2026-06-08 into an ergonomics + correctness pass across three surfaces (onboarding/login, sidebar/menu, settings modals) plus a bottom-nav overflow fix. FE-only; no backend.

## Spec & Plan

- Spec: [`docs/superpowers/specs/2026-06-08-mobile-sidebar-settings-onboarding-design.md`](../../../superpowers/specs/2026-06-08-mobile-sidebar-settings-onboarding-design.md)
- Plan: [`docs/superpowers/plans/2026-06-08-mobile-sidebar-settings-onboarding.md`](../../../superpowers/plans/2026-06-08-mobile-sidebar-settings-onboarding.md)
- Execution issue: FE #193

## Acceptance Criteria

Detailed, testable criteria live in the spec. Summary:

- [ ] Onboarding/login are keyboard-safe and use `dvh` + safe-area (no `100vh`, no clipped CTAs).
- [ ] One working header settings entry; sidebar rebuilt on Radix (focus-trap, Esc, scrim, swipe-to-close), ≥44px targets, confirmed Sign Out.
- [ ] Settings modals scroll and never hide their actions.
- [ ] Bottom nav reduced to ≤5 slots with a scalable "More" overflow; Photos dropped from the nav.
