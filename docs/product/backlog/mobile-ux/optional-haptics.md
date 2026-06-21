---
id: mobile-optional-haptics
title: Optional haptic feedback (Vibration API) with settings toggle
epic: mobile-ux
status: done
priority: P2
created: 2026-06-16
updated: 2026-06-20
issues:
  - FE #235
prs:
  - FE #236
spec: ../../../superpowers/specs/2026-06-18-optional-haptics-design.md
plan: ../../../superpowers/plans/2026-06-18-optional-haptics.md
---

**Shipped** in `0.3.18` — [FE #236](https://github.com/joe-bor/FamilyHub/pull/236), closed [FE #235](https://github.com/joe-bor/FamilyHub/issues/235). Default off, per-device toggles (Taps / Completions / Back); bottom-sheet-snap + destructive-confirm pulses deferred post-v1.

## Context

Add subtle, optional haptic feedback on key touch interactions (primary-action taps,
completing a chore/list item, bottom-sheet snap, destructive-action confirm) to make the
app feel more tactile and native. Exposed as an opt-in toggle in Preferences / sidebar.

**Supersedes** the "No haptics (PWA limits)" exclusion in the home-dashboard redesign spec
(`docs/superpowers/specs/2026-04-25-home-dashboard-redesign-design.md`). That note is
outdated for the primary dogfood device: `navigator.vibrate()` is supported in Android
Chrome PWAs, so haptics are viable there as a progressive enhancement.

Surfaced from phone dogfooding (Galaxy S10 / Chrome), 2026-06-16.

## Acceptance Criteria

- [x] A "Haptic feedback" toggle in Preferences (default off — confirm in spec), persisted
      alongside existing preferences
- [x] When enabled, a short vibration fires on the v1 interaction set — primary action taps
      and task/list completion (master + per-category toggles: Taps / Completions / Back);
      bottom-sheet snap and destructive-confirm/`warning()` deferred post-v1
- [x] Capability-detected: silently no-ops where `navigator.vibrate` is unavailable
      (iOS Safari, desktop) and whenever the toggle is off
- [x] Patterns are short and subtle (single brief pulses, not long buzzes)
- [x] Centralized helper (e.g. `haptics.tap()/success()/warning()`) instead of scattered
      `navigator.vibrate` calls

## Notes

Design resolved in [`2026-06-18-optional-haptics-design.md`](../../../superpowers/specs/2026-06-18-optional-haptics-design.md):
default **off**; **per-device** localStorage (`family-hub-haptics`), not synced; **independent**
of `prefers-reduced-motion` (the toggle is the single source of truth); **master + per-category**
toggles (Taps / Completions / Back). `warning()`/destructive-confirm and the bottom-sheet-snap
pulse are **deferred** (out of scope for v1). Built entirely on the existing PR #230
(`usePressable().onPointerDown`) and PR #234 (`useAndroidBackButton` `onPop`) seams — no new
press/motion/back infrastructure.

Vibration API only (`navigator.vibrate(ms)`); no iOS support, so frame strictly as
progressive enhancement. Open decisions for the spec: per-device setting (localStorage)
vs. synced preference — per-device is simpler and aligns with the device-member-association
direction; and whether to couple to `prefers-reduced-motion` (likely keep independent — the
toggle is the single source of truth). Guard against overuse; too-frequent haptics feel
worse than none. Pairs with `native-feel-interaction-polish.md` — visual + tactile
acknowledgment of the same press.
