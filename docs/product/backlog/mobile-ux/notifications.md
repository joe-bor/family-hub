---
id: mobile-notifications
title: Event reminder notifications (push or in-app)
epic: mobile-ux
status: planned
priority: P2
created: 2026-04-23
updated: 2026-04-23
issues: []
prs: []
---

## Context

Notify users before upcoming events. Decision pending: web push (PWA) vs in-app reminders only. (From `docs/mobile-ux-polish-backlog.md` item 6a.)

## Acceptance Criteria

- [ ] Design decision documented (push vs in-app vs both)
- [ ] Lead time configurable per event or globally
- [ ] Reminder fires at the right time (verified via automated test)
- [ ] User can disable reminders per event

## Notes

PWA push requires service worker push handling + user permission grant. Scope may expand if going push.
