---
id: mobile-sidebar-settings-onboarding
title: Sidebar + settings + onboarding mobile pass
epic: mobile-ux
status: done
priority: P2
created: 2026-04-23
updated: 2026-06-11
issues:
  - FE #193
prs:
  - FE #194
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

- [x] Onboarding/login are keyboard-safe and use `dvh` + safe-area (no `100vh`, no clipped CTAs). — `dvh` + safe-area on all four onboarding screens, `login-form`, and `App.tsx` shell; mobile-gated autofocus off + non-centered scroll on `login-form`/`onboarding-credentials`/`onboarding-family-name` (commits `aa33983`, `c2a3518`, `d8c546e`).
- [x] One working header settings entry; sidebar rebuilt on Radix (focus-trap, Esc, scrim, swipe-to-close), ≥44px targets, confirmed Sign Out. — dead Settings button removed from `app-header.tsx`; new `side-sheet.tsx` Radix-Dialog primitive + `sidebar-menu.tsx` rebuild with "Menu" label and confirm dialog (commits `70016b3`, `d8c546e`).
- [x] Settings modals scroll and never hide their actions. — `member-profile-modal.tsx` gains `max-h-[90dvh] overflow-y-auto` parity with `family-settings-modal.tsx`.
- [x] Bottom nav reduced to ≤5 slots with a scalable "More" overflow; Photos dropped from the nav. — `mobile-bottom-nav.tsx` now renders 4 primary + a `MobileSheet`-based "More" overflow (Meals, Recipes); Photos removed from the module list (commit `1d16544`).

Implementation merged via **FE PR #194** (2026-06-10), released in `0.3.12` and deployed to production 2026-06-11; CI lint/unit/E2E green. All ten detailed acceptance criteria below are satisfied. The two on-device behaviors jsdom/Playwright cannot assert — swipe-left-to-close on the side sheet, and focused inputs sitting above the soft keyboard — were **verified on a Galaxy S10 / Chrome on 2026-06-11**. The "More" sheet rides the later vaul `MobileSheet` rebuild (FE PR #198, story [expandable-bottom-sheet](expandable-bottom-sheet.md)) and now opens at half height — superseding this spec's hand-rolled-sheet description and its "no vaul migration" non-goal.
