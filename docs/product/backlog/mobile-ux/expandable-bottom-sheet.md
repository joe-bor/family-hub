---
id: mobile-expandable-bottom-sheet
title: Expandable bottom sheet pattern for forms
epic: mobile-ux
status: done
priority: P2
created: 2026-04-23
updated: 2026-06-11
issues: []
prs:
  - FE #198
---

## Context

Full-screen sheets were implemented for event forms (simplest, most reliable). A drag-to-expand bottom sheet (starts half-screen, user pulls up to full) feels lighter and more modern. (From `docs/mobile-ux-polish-backlog.md` item 4.)

Implemented 2026-06-11 in FE PR #198. The shared `MobileSheet` primitive was rebuilt on `vaul` (Radix Dialog underneath), keeping its API plus a new `initialHeight: "half" | "full"` prop. One gesture language across all modules: grab handle, tap/drag-up/input-focus expands to full, flick-down/scrim/Esc/Cancel dismisses. Short forms (bottom-nav More, list create, list item, new chore, recipe chooser/import) open at half height; long forms (event, recipe manual/edit, meal composer/editor) open full. The recipes add flow was consolidated into a single sheet that expands in place from chooser to manual form. Desktop (>768px) consumers render a centered `max-w-lg` sheet as a placeholder until the big-screen design pass. Full rationale and verification notes live in the PR body.

Scope decision: the original AC targeted the event form opening half-screen. Superseded — the event form is one of the longest forms, so it stays full height; the half-height default applies to short, single-purpose forms instead.

## Acceptance Criteria

- [x] Shared sheet primitive supports half-height open, drag/tap expand to full, flick-down dismiss, with Radix dialog semantics (focus trap, Esc, scrim, scroll lock, labelled title)
- [x] Short forms (More menu, list create, list item, chore create, recipe chooser/import) open at half height
- [x] Focusing an input auto-expands the sheet to full so the keyboard cannot hide a field
- [x] No action is reachable only by drag gesture (handle is an accessible button; Cancel/Esc/scrim retained)
- [x] Long forms (event, recipe manual/edit, meal composer/editor) keep full height with the same gestures
- [x] Unit + full E2E matrix green; desktop renders a centered sheet
- [x] Real-device pass: half-sheet keyboard open/close — **verified on Galaxy S10 / Chrome 2026-06-11** (auto-expand-on-focus keeps the field above the keyboard; handle drag/tap, flick-to-dismiss, scrim all behaved). iPhone Safari not yet exercised; revisit if an iOS-specific keyboard issue is reported.

## Remaining

- **Done and shipped.** FE release `family-hub-v0.3.12` (release PR #195, merged 2026-06-11) bundled this work with the sidebar/settings/onboarding story; deployed to production 2026-06-11 and smoke-verified on Galaxy S10 / Chrome. The release was cut before the device gate ran, but the gate has since passed retroactively — no fix-forward needed.
- Settings dialogs → bottom sheet conversion stays a separate future story (per the 2026-06-08 sidebar/settings spec) — now captured under [mobile-module-content-polish](mobile-module-content-polish.md).
