---
id: mobile-home-dashboard-redesign
title: Home dashboard redesign
epic: mobile-ux
status: done
priority: P2
created: 2026-04-23
updated: 2026-05-17
issues:
  - FE #149
prs:
  - FE PR #154
spec: ../../../superpowers/specs/2026-04-25-home-dashboard-redesign-design.md
---

## Context

This shipped story replaced the old 2-column grid of colored module buttons on mobile Home with a calm, layered, mobile-first dashboard surface for adult users.

Design concept: **"The Now"** — a moment-aware home where the hero announces what matters right now (in progress / next up / all clear), with today's remaining events and a small horizon peek beneath. See the spec for the full design.

## Scope

- **Adult-first:** primary audience is Joe + Partner.
- **Mobile-only:** ≤768px breakpoints. Touchscreen and tablet keep the current home until a dedicated follow-up story.
- **Pure home surface:** no module launcher tiles. Cross-module navigation is owned by the persistent bottom nav.
- **Calendar-only data:** today's events, with up to 3 events in the next 48h beyond today as a horizon peek. No tasks/chores/meals/weather (those data sources don't exist yet).

## Dependencies

- **Hard prerequisite:** `persistent-bottom-nav.md`. This story should not merge to production until mobile nav-based module switching is already in place.
- **Soft:** `visual-identity-refinement.md` (parallel work inherits for free).
- **Soft:** Google Calendar incremental sync epic.

## Acceptance Criteria

### Functional
- [x] Replaces the 2-column module grid as the mobile home surface on breakpoints ≤768px.
- [x] Integrates with the current FE app shell as the mobile `activeModule === null` surface; desktop/non-mobile behavior remains unchanged.
- [x] Hero renders correctly across all five states (RIGHT_NOW, UP_NEXT, ALL_DAY_ONLY, REST_OF_DAY_CLEAR, ALL_CLEAR_TODAY).
- [x] Hero transitions live as time crosses event boundaries (30-second poll + visibility-change recompute).
- [x] Member-chip focus filters Hero, Today list, and Coming-up consistently; single-focus only.
- [x] All-day events pin at the top of Today list.
- [x] Coming-up renders 0–3 events in `(endOfToday, endOfToday+2)`; region omitted when empty.
- [x] "+" opens event-create sheet pre-filled with today + focused member.
- [x] Native and Google-synced events are rendered through the same dashboard states; lack of Google connection never replaces a valid schedule or calm empty state.
- [x] The dashboard does not add dashboard-owned Google connect/sync UI in MVP; connection management remains in settings/member profile.

### Quality
- [x] No new design tokens introduced.
- [x] Motion uses the three-duration / single-easing system in the spec.
- [x] `prefers-reduced-motion` respected.
- [x] All touch targets ≥44px; rows ≥44px tall; chips 36px visual + 8px halo.
- [x] Layout holds 320–768px without horizontal overflow.
- [x] Dashboard and calendar tab are not confusable in screenshots.

### Performance
- [x] Reuses existing events query; no new BE endpoint or duplicate fetch.
- [x] Hero state recompute does not trigger today-list re-render when the subject hasn't changed.

## Follow-up stories to create

These were split out during brainstorming and are not part of this story:

- **Child-mode home** — the 4-year-old's home surface, replacing what they currently have on the touchscreen.
- **Touchscreen / tablet home** — large-screen home surface; current 2-col color grid stays there in the meantime.

## Notes

Detailed design, IA, motion vocabulary, interaction model, and out-of-scope items live in the spec linked in frontmatter.
This story shipped on FE after `persistent-bottom-nav` landed. Follow-up work remains separate for child-mode home and touchscreen / tablet home.
