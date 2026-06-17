---
id: mobile-native-feel-interaction-polish
title: Native-feel interaction polish (transitions + press feedback)
epic: mobile-ux
status: planned
priority: P2
created: 2026-06-16
updated: 2026-06-16
issues: []
prs: []
spec: ../../../superpowers/specs/2026-06-16-native-feel-interaction-polish-design.md
plan: ../../../superpowers/plans/2026-06-16-native-feel-interaction-polish.md
---

## Context

Close the "web app in a shell" gap with two micro-interaction passes:

1. **Smoother transitions** where switches currently feel harsh — notably module/tab
   changes (the home spec deliberately excluded page-transition slides) and
   sheet/overlay/sidebar open/close.
2. **Obvious press feedback** on interactive elements — immediate active/pressed states
   (subtle scale-down or opacity/contrast shift on tap) so every tap feels acknowledged
   before navigation or data resolves.

Builds on — does not replace — the existing motion vocabulary (three durations / single
easing / `prefers-reduced-motion`) defined in the home-dashboard redesign spec.

Surfaced from phone dogfooding (Galaxy S10 / Chrome), 2026-06-16.

## Acceptance Criteria

- [ ] Audit and smooth the transitions that read as harsh today (module/tab switch, sheet
      and dialog open/close, sidebar) using the existing duration/easing tokens — no new
      competing motion system
- [ ] Every interactive control has a clear pressed/active state that fires immediately on
      touch (not hover-only), giving instant acknowledgment even before navigation/data
      resolves
- [ ] All motion gated behind `prefers-reduced-motion: no-preference`; reduced-motion users
      get instant, non-animated states
- [ ] No added jank: transitions hold 60fps on the target device with no layout thrash

## Notes

Extends the motion system in `docs/superpowers/specs/2026-04-25-home-dashboard-redesign-design.md`
rather than introducing a new one. Pairs with `optional-haptics.md` (tactile half of the
same acknowledgment) and `native-back-button.md` (a back that animates the outgoing view
feels far more native than an instant swap). Keep it subtle — the goal is "acknowledged and
smooth," not flashy. Good candidate to be the first of the three to get a full design spec,
since it touches every screen and is the most visible everyday win.
