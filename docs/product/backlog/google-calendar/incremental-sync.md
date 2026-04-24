---
id: google-cal-incremental-sync
title: Incremental sync with sync tokens + scheduler
epic: google-calendar-read-only
status: in-progress
priority: P1
created: 2026-04-23
updated: 2026-04-23
design_doc: docs/google-calendar-integration-design.md
issues: []
prs: []
---

## Context

Full sync fetches all events from Google Calendar in one shot (PRs BE #24, #25). Incremental sync persists a `syncToken` after each successful sync and uses it on subsequent syncs to fetch only changes. A background scheduler polls every 5 minutes to keep Family Hub's view fresh without user-triggered refresh.

See [Google Calendar integration design](../../../google-calendar-integration-design.md) for the full data flow and DB schema.

## Acceptance Criteria

- [ ] Successful full sync persists a `syncToken` on `GoogleCalendarSync` (or equivalent table)
- [ ] Subsequent syncs send the stored `syncToken` and process only the delta
- [ ] 410 Gone response from Google → invalidate token, fall back to a fresh full sync, persist new token
- [ ] Background scheduler runs sync every 5 minutes per connected calendar (configurable via application property)
- [ ] Sync failures are logged with correlation to calendar ID; do not crash the scheduler
- [ ] Integration test covers the 410 Gone fallback path
- [ ] FE reflects updates within one poll cycle (no FE changes expected beyond existing TanStack Query cache invalidation)

## Out of Scope

- Webhook / push-channel notifications (future — Phase 3)
- Write-back to Google (→ [`write-back`](write-back.md))
- User-triggered "sync now" button (FE enhancement, separate story if wanted)

## Notes

Design doc describes the sync token lifecycle; confirm DB schema matches before implementation. Claude `MEMORY.md` references this as "PR 5 incremental sync and scheduler next."
