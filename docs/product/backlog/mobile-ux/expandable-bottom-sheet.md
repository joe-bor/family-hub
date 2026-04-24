---
id: mobile-expandable-bottom-sheet
title: Expandable bottom sheet pattern for forms
epic: mobile-ux
status: planned
priority: P2
created: 2026-04-23
updated: 2026-04-23
issues: []
prs: []
---

## Context

Full-screen sheets were implemented for event forms (simplest, most reliable). A drag-to-expand bottom sheet (starts half-screen, user pulls up to full) would feel lighter and more modern — worth revisiting once the base mobile experience is solid. (From `docs/mobile-ux-polish-backlog.md` item 4.)

## Acceptance Criteria

- [ ] Event create/edit form opens as half-screen sheet by default
- [ ] Drag-up expands to full screen; drag-down collapses
- [ ] Keyboard interaction does not break the sheet height
- [ ] Replaces the current full-screen sheet on mobile
