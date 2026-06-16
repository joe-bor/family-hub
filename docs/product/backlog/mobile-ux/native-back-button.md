---
id: mobile-native-back-button
title: Native hardware back-button handling (double-tap to exit PWA)
epic: mobile-ux
status: planned
priority: P2
created: 2026-06-16
updated: 2026-06-16
issues: []
prs: []
---

## Context

In the installed PWA on Android, the hardware/gesture back button currently exits the
app immediately (browser default) instead of navigating within the app. Make back behave
like a native app: a single back press dismisses an open overlay or navigates one in-app
step; only a second back press at a root view exits, with a brief "Press back again to
exit" hint in between.

Surfaced from phone dogfooding (Galaxy S10 / Chrome), 2026-06-16.

## Acceptance Criteria

- [ ] In standalone (installed) mode, a single back press first dismisses any open
      bottom sheet, dialog, or sidebar; if none is open, it navigates back one in-app step
      (previous view/tab) instead of exiting
- [ ] At a root view with nothing to go back to, the first back press shows a transient
      "Press back again to exit" hint; a second back press within ~2s exits the app
- [ ] Gated to standalone/PWA on devices with a hardware/gesture back (Android); browser
      tabs and iOS standalone (no hardware back — uses edge-swipe) are unaffected
- [ ] No back-button trap loops or history leaks; refresh and deep-link still land on the
      correct view

## Notes

Standard Android pattern: push a sentinel history entry on app/root mount and intercept
`popstate`; on first back at root, re-push the entry and show the toast, allow exit on the
second. Family Hub currently switches modules via `useAppStore.setActiveModule()` with no
router (per the persistent-bottom-nav spec), so "in-app back" needs an explicit
navigation/history stack rather than relying on routes — that's the main design question
for the spec. Pairs with `native-feel-interaction-polish.md` (a back that animates the
outgoing view feels more native than an instant swap).
