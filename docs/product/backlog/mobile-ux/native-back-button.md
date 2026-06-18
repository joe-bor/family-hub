---
id: mobile-native-back-button
title: Native hardware back-button handling (double-tap to exit PWA)
epic: mobile-ux
status: planned
priority: P2
created: 2026-06-16
updated: 2026-06-17
issues:
  - FE #231
prs: []
spec: ../../../superpowers/specs/2026-06-17-native-back-button-design.md
plan: ../../../superpowers/plans/2026-06-17-native-back-button.md
---

## Context

In the installed PWA on Android, the hardware/gesture back button currently exits the
app immediately (browser default) instead of navigating within the app. Make back behave
like a native app: a single back press dismisses an open overlay or navigates one in-app
step; only a second back press at a root view exits, with a brief "Press back again to
exit" hint in between.

Surfaced from phone dogfooding (Galaxy S10 / Chrome), 2026-06-16.

## Acceptance Criteria

- [ ] In the installed Android PWA, a single back press first dismisses the top-most open
      bottom sheet, dialog, or sidebar (most-recently-opened closes first); if none is open
      and the active module is not Home, it goes up to Home instead of exiting
- [ ] At Home with nothing open, the first back press shows a transient "Press back again to
      exit" toast; a second back press within ~2s exits the app
- [ ] Gated to Android standalone (display-mode: standalone, not iOS, coarse pointer);
      browser tabs, iOS standalone (edge-swipe), and desktop installed PWAs are unaffected
- [ ] No back-button trap loops or history leaks; refresh lands on Home with the history
      buffer intact (no router/URL view state to desync)
- [ ] Drives the existing close handlers only — the sibling native-feel transitions animate
      the back with no new motion code

## Notes

Design resolved in [`2026-06-17-native-back-button-design.md`](../../../superpowers/specs/2026-06-17-native-back-button-design.md).
Because Family Hub has no router (modules switch via `useAppStore.setActiveModule()`, per the
persistent-bottom-nav spec) and dismiss state is scattered across local `useState` and two
Zustand stores, the design uses a single `popstate`-sentinel interceptor plus a LIFO registry
of "dismissable now" handlers — not route history. The in-app model is **single-root**:
overlays/details first, then up to Home, then double-press-to-exit — deliberately not a
"previous tab" history stack (which would be polluted by the draft-driven `setActiveModule`
jumps). It drives the existing close handlers, so the `native-feel-interaction-polish.md`
transitions animate the back for free. Pairs with `optional-haptics.md`.
