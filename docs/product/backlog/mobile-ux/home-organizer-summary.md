---
id: mobile-home-organizer-summary
title: Home organizer summary (state line + "since you last opened" feed)
epic: mobile-ux
status: planned
priority: P2
created: 2026-06-20
updated: 2026-06-21
issues:
  - FE #239
prs: []
spec: ../../../superpowers/specs/2026-06-20-home-organizer-summary-design.md
plan: ../../../superpowers/plans/2026-06-21-home-organizer-summary.md
---

## Context

The shipped mobile home — "The Now" ([home-dashboard-redesign.md](home-dashboard-redesign.md)) — is deliberately calm and **calendar-only**; its design spec excluded chores/lists/meals because the data didn't exist yet. It does now (`Chores`, `Lists`, `Recipes`, `Meals` all ship). This story evolves Home into an organizer summary **without redesigning "The Now"**: it keeps the hero + agenda intact, inserts one quiet **state line** under the hero (chores remaining today + tonight's dinner), and appends a **"Since you last opened" activity feed** for calendar + lists.

The feed answers the question a parent opens the app to ask when away from the always-on hub: *did anyone touch the shared plan?* — directly serving the PRD's shared-visibility pain. Because the family shares a single account, there is no edit attribution: the feed is event-centric ("Event added · Dentist"), never actor-centric.

Design + mechanics: [spec](../../../superpowers/specs/2026-06-20-home-organizer-summary-design.md) (reviewed by an Opus 4.8 spec-review pass on 2026-06-21; the review verified the codebase constraints in spec §10 and led to the lists-from-summary and calendar-occurrence approaches, the fully-specified diff edge cases, and cutting meals from the feed).

## Acceptance Criteria

- [ ] On mobile (≤768px) the hero, member chips, "rest of today," "coming up," and floating "+" are unchanged from the shipped dashboard.
- [ ] A state line under the hero shows chores remaining today + tonight's dinner; each segment omits when empty, the whole line omits when both are empty; it is family-wide (ignores member-chip focus, which still filters the hero + agenda).
- [ ] A "Since you last opened" feed shows recent changes for **calendar + lists only** (chores, meals, and recipes never appear in the feed).
- [ ] Changes are detected on every load and accumulated; the "new since you last looked" divider advances **only** on a meaningful open (cold start, visible after the configured gap, or device-local day rollover); a quick peek neither advances the divider nor drops unseen changes.
- [ ] Calendar changes coalesce (≥2 → one expandable group; recurring series collapse via `recurringEventId`); lists use summary signals only (created/renamed/removed + item-count deltas), no per-item identity diffing in v1.
- [ ] An item added then removed before a meaningful open leaves no phantom row; a long absence reseeds silently rather than dumping a backlog.
- [ ] Client-side diff only — no new backend endpoint; a new `idb-keyval` activity store + `lastSeen`/`hiddenAt` markers, cleared on logout and the 401 handler.
- [ ] No new design tokens; reuses the existing motion system and the shipped press-feedback (#230) + optional-haptics (#236) seams; dates are device-local.

## Notes

- **Successor to** [home-dashboard-redesign.md](home-dashboard-redesign.md) ("The Now") — extends it, does not replace it.
- **Approach 3** (client-side diff behind a normalized `ActivityItem[]` seam). The **backend activity log** and the **AI day-summary / overload-warnings** are explicit future siblings that consume the same seam — out of scope here.
- Verified constraints (spec §10): no entity `updatedAt`; calendar exposes only expanded instances with nullable `id`; `ListSummary` carries no items/timestamps; FE #222's persister is not a reusable KV store; the FE is device-local (no family-tz helper).
- Single shared family account → no edit attribution anywhere; see also the timezone/cross-stack note in [Settings surface decisions](sidebar-settings-story.md).
