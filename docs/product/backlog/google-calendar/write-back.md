---
id: google-cal-write-back
title: Write-back to Google Calendar (create / edit / delete)
epic: google-calendar-write-back
status: planned
priority: P1
created: 2026-04-23
updated: 2026-04-23
design_doc: docs/google-calendar-integration-design.md
issues: []
prs: []
---

## Context

Phase 1 Google Calendar integration is read-only: events created in Google show up in Family Hub, but events created in Family Hub do not propagate back. Write-back closes the loop so any event created/edited/deleted in Family Hub is reflected in the user's Google Calendar within ~30 seconds.

See [Google Calendar integration design](../../../google-calendar-integration-design.md) for full approach (description-field metadata for multi-person assignees, conflict resolution, etc.).

## Acceptance Criteria

- [ ] BE endpoint `POST /api/events` with a Google-linked calendar creates the event in Google and stores the returned `google_event_id`
- [ ] `PUT /api/events/:id` updates the corresponding Google event
- [ ] `DELETE /api/events/:id` deletes from Google
- [ ] Multi-person profile assignments persist through round-trip sync (description-field metadata per PRD §7.4.2)
- [ ] Failures to call Google (network/auth) do not roll back the local event; a retry/backoff mechanism exists
- [ ] Integration test for round-trip: create in FH → verify in Google → edit in Google → verify in FH → delete in FH → verify gone from Google
- [ ] Design doc updated with the decided conflict-resolution rule (currently "last-write-wins, Google is source of truth")

## Out of Scope

- Recurring event write-back (separate story if scope grows)
- Attendee/email invite support

## Notes

Depends on incremental sync being complete (need a stable sync loop before we add writes that can race with reads). Review the design doc's "description field metadata" section before starting; it has open questions.
