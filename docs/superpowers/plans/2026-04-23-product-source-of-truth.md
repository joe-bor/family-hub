# Product Source of Truth Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish a three-tier product source of truth for Family Hub (PRD / roadmap / backlog) rooted in a new `joe-bor/family-hub` git repo, with GitHub Issues for task-level work and a cross-repo Project board. Archive the old `docs/prompts/` handoff files with verified provenance.

**Architecture:** Markdown-primary, agent-agnostic. Three tiers of docs in `docs/product/`, canonical `AGENTS.md` at repo root (with `CLAUDE.md` symlinked to it), GitHub Issues per task linked back to story files via a `Story:` convention, and a cross-repo user-level Project for live task status.

**Tech Stack:** Markdown, git, GitHub CLI (`gh`), GitHub Projects (v2), YAML frontmatter. No code, no CI changes to FE/BE.

**Spec:** `docs/superpowers/specs/2026-04-23-product-source-of-truth-design.md`

**Execution note:** If this implementation plan conflicts with the design spec, follow this plan. It has been corrected to match the observed local workspace state after partial implementation.

**Status ownership model:**
- Story status lives only in `docs/product/backlog/<epic>/<story>.md` frontmatter.
- Task status lives only in the GitHub Project `Family Hub` issue item's `Status` field.
- `docs/product/roadmap.md` is a summary/index, not a second status tracker.
- Labels are for filtering only, not progress tracking.

---

## File Structure Overview

**Expected end state in `joe-bor/family-hub` root repo:**
- `.gitignore`
- `AGENTS.md` (renamed from existing `CLAUDE.md`, with new section appended)
- `CLAUDE.md` (symlink → `AGENTS.md`)
- `docs/product/prd.md` (imported and refreshed from `~/Downloads/family-calendar-prd.md`)
- `docs/product/roadmap.md` (summary/index seeded from Claude `MEMORY.md`, if available, + merged PR history)
- `docs/product/backlog/google-calendar/incremental-sync.md`
- `docs/product/backlog/google-calendar/write-back.md`
- `docs/product/backlog/mobile-ux/*.md` (one per item in `docs/mobile-ux-polish-backlog.md`)
- `docs/prompts/_archive/*` (verified prompts moved here)
- `docs/prompts/_needs_review.md` (unverified prompts surfaced for user)

**Created in existing FE (`joe-bor/FamilyHub`) and BE (`joe-bor/family-hub-api`) repos:**
- `.github/ISSUE_TEMPLATE/story-task.md` (Issue body template)

**Modified:**
- `MEMORY.md` (Claude-only legacy context outside this repo; point to new product tree; shrink inline roadmap to a pointer)
- `frontend/CLAUDE.md` (add one-line pointer to root `docs/product/`)
- `backend/family-hub-api/CLAUDE.md` (same)

**Deleted (after migration):**
- `docs/mobile-ux-polish-backlog.md` (content migrated into `backlog/mobile-ux/`)

**External actions:**
- Create private GitHub repo `joe-bor/family-hub`
- Create user-level Project "Family Hub"
- Seed labels in all three repos

---

## Current Local State

Observed in this workspace before completing the migration:

- Root `family-hub/` is already a local git repo on branch `main`
- `.gitignore`, `README.md`, root `CLAUDE.md`, `docs/`, and `docker-compose.yml` already exist and are staged
- No commits exist yet in the root repo
- No `origin` remote is configured yet in the root repo
- `AGENTS.md` does not exist yet
- `docs/product/` does not exist yet
- `docs/prompts/_archive/` and `docs/prompts/_needs_review.md` do not exist yet
- `MEMORY.md` is not in this repo; if using Claude Code, treat it as an external legacy input for migration only

The tasks below are written as the corrected resume path from this partially implemented state. Do not rerun greenfield bootstrap steps like `git init` if the state above is already true.

---

## Prerequisites

- [ ] `gh` CLI authenticated as `joe-bor` with repo + project scopes:

```bash
gh auth status
gh auth refresh -s project -s read:project -s write:project
```

Expected: logged in to github.com as `joe-bor` with the listed scopes.

- [ ] Confirm working directory is the family-hub root:

```bash
cd /Users/joe.bor/code/family-hub && pwd
```

Expected: `/Users/joe.bor/code/family-hub`

- [ ] Confirm existing PRD is readable:

```bash
test -r ~/Downloads/family-calendar-prd.md && echo OK
```

Expected: `OK`

---

## Task 1: Normalize existing root repo bootstrap and push to GitHub

**Files:**
- Create: `/Users/joe.bor/code/family-hub/.gitignore`
- Create: `/Users/joe.bor/code/family-hub/README.md` (minimal, just a title + pointer to `AGENTS.md`)

- [ ] **Step 1.1: Verify the existing root git repo state**

```bash
cd /Users/joe.bor/code/family-hub
git rev-parse --show-toplevel
git branch --show-current
```

Expected: root resolves to `/Users/joe.bor/code/family-hub` and current branch is `main`. If this already passes, do **not** rerun `git init`.

- [ ] **Step 1.2: Verify / normalize `.gitignore`**

`/Users/joe.bor/code/family-hub/.gitignore` should include at minimum:

```
# Sibling repos — each managed independently
frontend/
backend/

# Editor / OS
.DS_Store
.idea/
.vscode/
*.swp

# Node
node_modules/

# Env
.env
.env.local
```

If extra local-only ignores already exist (for example `.claude/` or `.playwright-mcp/`), keep them.

- [ ] **Step 1.3: Verify `README.md` exists**

`/Users/joe.bor/code/family-hub/README.md` should contain:

```markdown
# Family Hub

Root workspace for the Family Hub product. Frontend (`joe-bor/FamilyHub`) and backend (`joe-bor/family-hub-api`) are separate repos cloned as sibling directories under this root.

## Start here
- [AGENTS.md](AGENTS.md) — agent entry point (canonical, agent-agnostic)
- [docs/product/prd.md](docs/product/prd.md) — product vision, personas, scope
- [docs/product/roadmap.md](docs/product/roadmap.md) — shipped / active / planned summary
- [docs/product/backlog/](docs/product/backlog/) — active and planned stories
```

Note: these links are forward references until Tasks 2-7 complete. That is acceptable during migration, but do not treat the README as "finished" until those files exist.

- [ ] **Step 1.4: Stage only the tracked root-level files**

```bash
cd /Users/joe.bor/code/family-hub
git add .gitignore README.md CLAUDE.md docs/
# docker-compose.yml may or may not exist at root depending on workspace state
[ -f docker-compose.yml ] && git add docker-compose.yml
git status
```

Expected: `docs/`, `CLAUDE.md`, `.gitignore`, `README.md` all staged (plus `docker-compose.yml` if present). `frontend/` and `backend/` NOT listed (ignored).

- [ ] **Step 1.5: First commit (only if root repo still has no commits)**

```bash
git commit -m "chore: initialize root workspace repo"
```

- [ ] **Step 1.6: Create GitHub repo and push**

```bash
git remote get-url origin >/dev/null 2>&1 || gh repo create joe-bor/family-hub --private --source=. --remote=origin
git push -u origin main
```

Expected: repo exists at `https://github.com/joe-bor/family-hub`, local `main` pushed to `origin/main`.

- [ ] **Step 1.7: Verify**

```bash
gh repo view joe-bor/family-hub --json name,visibility,defaultBranchRef
```

Expected: JSON showing `family-hub`, `PRIVATE`, default branch `main`.

---

## Task 2: Rename CLAUDE.md → AGENTS.md + symlink

**Files:**
- Rename: `CLAUDE.md` → `AGENTS.md`
- Create: `CLAUDE.md` (symlink → `AGENTS.md`)

- [ ] **Step 2.1: Create a feature branch**

```bash
cd /Users/joe.bor/code/family-hub
git checkout -b feat/agents-md-canonical
```

- [ ] **Step 2.2: Rename the file**

```bash
git mv CLAUDE.md AGENTS.md
```

- [ ] **Step 2.3: Create symlink `CLAUDE.md → AGENTS.md`**

```bash
ln -s AGENTS.md CLAUDE.md
git add CLAUDE.md
```

- [ ] **Step 2.4: Verify symlink resolves**

```bash
ls -la CLAUDE.md
cat CLAUDE.md | head -5
```

Expected: `ls -la` shows `CLAUDE.md -> AGENTS.md`. `cat` shows first lines of `AGENTS.md`.

- [ ] **Step 2.5: Commit**

```bash
git commit -m "refactor: make AGENTS.md canonical; symlink CLAUDE.md to it

Agent-agnostic root entry point. Claude Code continues to read CLAUDE.md
via the symlink; future harnesses (Codex, Gemini, Cursor) read AGENTS.md
directly or via their own symlink."
```

---

## Task 3: Rewrite AGENTS.md for the new product structure

The existing `AGENTS.md` (formerly root `CLAUDE.md`) describes the old "handoff via `docs/prompts/`" workflow. Replace that with the new product-tree structure. Keep the Role, Repo Map, and What-We-Do-Not-Do sections mostly intact — they still apply.

**Files:**
- Modify: `/Users/joe.bor/code/family-hub/AGENTS.md`

- [ ] **Step 3.1: Replace `AGENTS.md` with the new content below**

Full file content:

```markdown
# Family Hub — Agent Entry Point

Canonical agent entry for the Family Hub root workspace. `CLAUDE.md` is a symlink to this file. Any harness (Claude Code, Codex, Gemini, Cursor, human) reads the same content.

## Role

This is the **architect and orchestrator** workspace. When invoked from this root directory, the agent operates as a cross-repo coordinator — not a coder. Never write production code from this context.

## Repo Map

| Directory | Stack | Description |
|-----------|-------|-------------|
| `frontend/` | React, Vite, TypeScript, TanStack Query | SPA frontend — has its own `CLAUDE.md` with full dev agent instructions |
| `backend/family-hub-api/` | Spring Boot, Java 21, Maven | REST API backend — has its own `CLAUDE.md` with mentor-first agent instructions |

Each sub-repo has its own `CLAUDE.md` with repo-specific conventions, patterns, and agent behavior. Read them before making cross-repo decisions.

## Sources of truth — start here

| Doc | What | When to read |
|-----|------|--------------|
| `docs/product/prd.md` | Product vision, personas, scope, architecture | Session start, orientation, "what is Family Hub?" |
| `docs/product/roadmap.md` | Epic summary and story index | "What's shipped / active / next?" |
| `docs/product/backlog/<epic>/<story>.md` | Per-story acceptance criteria, design links, Issues, PRs | Working on a specific story |
| GitHub Project **Family Hub** (user-level, cross-repo) | Live task-level state | Picking up in-flight work |

## When picking up a task

1. `gh issue view <N> --repo <repo>` → find the `Story:` URL at the top of the body
2. Read that story file for context, acceptance criteria, and linked design doc
3. Execute the task; open PR with `Closes #<N>`
4. If the story's overall state changes, update only its `status:` and `updated:` in frontmatter. Treat `roadmap.md` as an index, not a second status system.

## Status ownership

- Story status: story file frontmatter in `docs/product/backlog/`
- Task status: GitHub Project `Family Hub` `Status` field
- Labels: filtering only
- `roadmap.md`: summary/index only

## What we do here

- **Alignment checks** between FE and BE — API contracts, types, data formats, validation rules
- **Architecture discussions** and design decisions that span both repos
- **Maintaining product docs** — PRD, roadmap summary, backlog stories, API reference, design docs

## What we do NOT do

- Write production code — that belongs in FE or BE repos
- Run builds, tests, or linters — those belong in the respective repos
- Make commits to FE or BE repos from this context

## Shared docs

| Path | Purpose |
|------|---------|
| `docs/product/` | Product source of truth (PRD, roadmap, backlog) |
| `docs/calendar-events-api-reference.md` | Shared CalendarEvents API contract |
| `docs/*-design.md` | Per-feature design documents (linked from story files) |
| `docs/superpowers/` | Specs and plans for major initiatives |
| `docs/prompts/_archive/` | Historical handoff prompts (read-only reference) |

The `/alignment` skill reads the Resolved section of the API reference doc before running, so keeping it current prevents re-flagging fixed issues.

## Repo-specific agents

- **FE agent** (`frontend/CLAUDE.md`): Full dev agent — codes, tests, commits, opens PRs
- **BE agent** (`backend/family-hub-api/CLAUDE.md`): Mentor-first — teaches by default, codes when explicitly asked. Has a `/mentor` skill for code reviews.
```

- [ ] **Step 3.2: Verify symlink still resolves and both files show the new content**

```bash
cat AGENTS.md | head -3
cat CLAUDE.md | head -3
```

Expected: both show `# Family Hub — Agent Entry Point` ... identical.

- [ ] **Step 3.3: Commit**

```bash
git add AGENTS.md
git commit -m "docs(agents): rewrite root entry point for product source-of-truth tree"
```

---

## Task 4: Scaffold `docs/product/` tree

**Files:**
- Create: `docs/product/` directory
- Create: `docs/product/backlog/` directory
- Create: `docs/product/README.md` (one-line pointer, optional — skip if minimalist)

- [ ] **Step 4.1: Create directories**

```bash
cd /Users/joe.bor/code/family-hub
mkdir -p docs/product/backlog
```

- [ ] **Step 4.2: Verify**

```bash
ls -la docs/product/
```

Expected: `backlog/` subdirectory exists.

Directories are empty until Tasks 5-7 populate them. No commit here — will commit with the first populated task.

---

## Task 5: Import and refresh the PRD

Import the existing PRD from `~/Downloads/` and apply surgical edits per spec §9. Do NOT rewrite — preserve voice, structure, and content that is still accurate.

**Files:**
- Create: `/Users/joe.bor/code/family-hub/docs/product/prd.md`

- [ ] **Step 5.1: Copy the PRD into the product tree**

```bash
cp ~/Downloads/family-calendar-prd.md /Users/joe.bor/code/family-hub/docs/product/prd.md
```

- [ ] **Step 5.2: Apply edits** (see sub-steps below, then commit once)

Edits to apply to `docs/product/prd.md`:

**a) Header** — replace the top metadata block:

```markdown
# Product Requirements Document (PRD)
## Family Hub — Family Touchscreen Calendar

**Document Version:** 2.0
**Last Updated:** 2026-04-23
**Project Owner:** Joe
**Document Status:** Living document — see `docs/product/roadmap.md` for current phase.

---
```

**b) §7 Feature Requirements** — add one note at the start of the section (before the numbered feature blocks):

```markdown
> Delivery status lives in `docs/product/roadmap.md`. Story-level execution details live in `docs/product/backlog/`.
```

**c) §6 Product Scope — In/Out rebalance.** Edit the lists:

- In §6 **In Scope (MVP - Phase 1)**: remove the three Task Management lines. Add new lines:
  - ✅ JWT authentication + family registration
  - ✅ Recurring events (RRULE expansion)
  - ✅ Multi-day events (endDate)
  - ✅ Docker deployment + Neon Postgres + DigitalOcean droplet
- In §6 **Out of Scope**: add:
  - ❌ **Task/Todo Management** — deferred post-MVP
  - ❌ **WebSocket real-time sync** — polling via TanStack Query is sufficient
  - ❌ **Two-way Google Calendar sync** — Phase 1 is read-only; write-back is a future epic

**d) §8 Technical Architecture** — update the stack lists to match reality:

- Frontend: replace "Zustand or Jotai" with "Zustand (UI state) + TanStack Query (server state)". Replace "Axios" with "Native fetch via custom `httpClient`".
- Backend: confirm "Spring Boot 3.x, Java 21, Maven" (not Gradle).
- Database: "Neon Postgres (production, external), H2 (development in-memory)".
- Deployment: "DigitalOcean droplet + Docker Compose + GHCR-hosted BE image. FE deployed as static files via `deploy.sh` to nginx."
- Remove or mark the WebSocket section as "Deferred — not in current build."

**e) §12 Development Phases** — replace the entire section with:

```markdown
## 12. Development Phases

Phase planning lives in [`docs/product/roadmap.md`](./roadmap.md). This section historically contained a Dec 2024 sprint plan that is no longer representative of the delivered product.

Current epics (see roadmap for details):
- **Foundation**
- **Calendar core**
- **Google Calendar read-only sync**
- **Google Calendar write-back**
- **Mobile UX polish**
- **Tasks/todos** (post-MVP)
```

**f) §15.3 Related Documents** — append:

```markdown
- **Roadmap:** `docs/product/roadmap.md`
- **Backlog:** `docs/product/backlog/`
- **Agent entry point:** `AGENTS.md` (root of `joe-bor/family-hub`)
- **API contract:** `docs/calendar-events-api-reference.md`
```

**g) Change log** — append a row:

```markdown
| 2.0     | 2026-04-23 | Joe    | Realigned with shipped reality; moved phase plan to roadmap.md; added JWT/recurring/multi-day as shipped; deferred Tasks and WebSocket. |
```

- [ ] **Step 5.3: Verify PRD loads cleanly**

```bash
wc -l /Users/joe.bor/code/family-hub/docs/product/prd.md
grep -n "Version:** 2.0" /Users/joe.bor/code/family-hub/docs/product/prd.md
grep -n "Delivery status lives in" /Users/joe.bor/code/family-hub/docs/product/prd.md
```

Expected: version is 2.0; the new delivery-status note is present once.

- [ ] **Step 5.4: Commit**

```bash
git add docs/product/prd.md
git commit -m "docs(product): import PRD v2.0, realigned with shipped reality

- Imported from ~/Downloads/family-calendar-prd.md
- Added one status pointer to roadmap/backlog instead of per-feature live status lines
- Rebalanced §6 scope (Tasks deferred, JWT/recurring/multi-day shipped)
- Updated §8 architecture to match built stack (TanStack Query, Neon, Docker)
- Replaced §12 sprint plan with pointer to roadmap.md"
```

---

## Task 6: Seed `roadmap.md`

**Files:**
- Create: `/Users/joe.bor/code/family-hub/docs/product/roadmap.md`

- [ ] **Step 6.1: Write the file**

Create `/Users/joe.bor/code/family-hub/docs/product/roadmap.md` with exactly this content:

```markdown
# Family Hub — Roadmap

Last updated: 2026-04-23

Use this document as a summary/index. Story status lives in `docs/product/backlog/<epic>/<story>.md`. GitHub Project **Family Hub** is the live task board for issue-level work.

## Shipped

### Foundation

- Family registration + JWT auth — FE + BE (retrofit)
- Docker + GHCR + prod deploy — [deployment-guide](../deployment-guide.md), [docker migration](../deployment-docker-migration.md) · BE #8, #9
- E2E against real backend — FE #105
- BE versioned releases (release-please) — [spec](../superpowers/specs/2026-03-19-be-versioned-releases-design.md) · BE (multiple)

### Calendar core

- CRUD calendar events — [API reference](../calendar-events-api-reference.md) · FE #81–#93, BE #8–#9
- All-day event rendering — [all-day + multi-day design](../all-day-and-multi-day-events-design.md) · FE #110
- Multi-day events (endDate) — same · BE #18, #19, FE #111
- Recurring events (RRULE + expansion) — [recurring events design](../recurring-events-design.md) · BE #20, #21, FE #117

## Active epics

### Google Calendar read-only sync

Completed in this epic:
- OAuth flow — [Google Cal integration design](../google-calendar-integration-design.md) · BE #22
- Calendar selection — same · BE #23
- Full sync — same · BE #24, #25

Remaining story:
- [Incremental sync + scheduler](backlog/google-calendar/incremental-sync.md)

## Planned epics

### Google Calendar write-back

- [Write-back (create/edit/delete to Google)](backlog/google-calendar/write-back.md) — [Google Cal integration design](../google-calendar-integration-design.md)

### Mobile UX polish

- [Home dashboard redesign](backlog/mobile-ux/home-dashboard-redesign.md)
- [Persistent bottom navigation](backlog/mobile-ux/persistent-bottom-nav.md)
- [Visual identity refinement](backlog/mobile-ux/visual-identity-refinement.md)
- [Expandable bottom sheet pattern](backlog/mobile-ux/expandable-bottom-sheet.md)
- [Sidebar + settings + onboarding mobile pass](backlog/mobile-ux/sidebar-settings-onboarding-mobile.md)
- [Notifications (event reminders)](backlog/mobile-ux/notifications.md)
- [Drag-to-create event on time grid](backlog/mobile-ux/drag-to-create.md)
- [Pinch-to-zoom calendar views](backlog/mobile-ux/pinch-to-zoom.md)

## Deferred

### Tasks / todos

Deferred post-MVP per PRD v2.0 realignment. Will be planned when the Google Calendar epic completes.
```

- [ ] **Step 6.2: Commit**

```bash
git add docs/product/roadmap.md
git commit -m "docs(product): seed roadmap summary/index with shipped, active, planned work"
```

---

## Task 7: Create active + future story files

Story files for two Google Calendar stories + eight Mobile UX stories. Shipped stories are NOT backfilled (per spec §Decisions).

**Files:**
- Create: `docs/product/backlog/google-calendar/incremental-sync.md`
- Create: `docs/product/backlog/google-calendar/write-back.md`
- Create: `docs/product/backlog/mobile-ux/*.md` (8 files)

- [ ] **Step 7.1: Create directories**

```bash
cd /Users/joe.bor/code/family-hub
mkdir -p docs/product/backlog/google-calendar docs/product/backlog/mobile-ux
```

- [ ] **Step 7.2: Write `incremental-sync.md`**

Create `docs/product/backlog/google-calendar/incremental-sync.md`:

```markdown
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
```

- [ ] **Step 7.3: Write `write-back.md`**

Create `docs/product/backlog/google-calendar/write-back.md`:

```markdown
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
```

- [ ] **Step 7.4: Write the eight Mobile UX story files**

For each of the 8 items below, create the corresponding file with the shown content.

**`docs/product/backlog/mobile-ux/home-dashboard-redesign.md`:**

```markdown
---
id: mobile-home-dashboard-redesign
title: Home dashboard redesign
epic: mobile-ux
status: planned
priority: P2
created: 2026-04-23
updated: 2026-04-23
issues: []
prs: []
---

## Context

The mobile home screen is a basic 2-column grid of colored module buttons. It works but feels like a prototype. Needs a proper mobile home experience — possibly a dashboard with today's agenda, upcoming chores, quick-add actions, etc. (From `docs/mobile-ux-polish-backlog.md` item 1.)

## Acceptance Criteria

- [ ] Design proposal for a dashboard-style home (agenda + quick actions) reviewed and approved
- [ ] Replaces the 2-column module grid on mobile breakpoints
- [ ] Today's agenda visible at-a-glance
- [ ] At least one quick-add action is reachable in one tap from home

## Notes

Design exploration needed before implementation. May pair well with bottom-nav story.
```

**`docs/product/backlog/mobile-ux/persistent-bottom-nav.md`:**

```markdown
---
id: mobile-persistent-bottom-nav
title: Persistent bottom navigation
epic: mobile-ux
status: planned
priority: P2
created: 2026-04-23
updated: 2026-04-23
issues: []
prs: []
---

## Context

Currently users must return to the Home dashboard to switch modules. A persistent bottom nav (Calendar, Chores, Meals, Lists, Photos) would feel more app-like and reduce friction. (From `docs/mobile-ux-polish-backlog.md` item 2.)

## Acceptance Criteria

- [ ] Bottom nav visible on all module screens at mobile breakpoint
- [ ] Active module indicated
- [ ] Tapping a nav item navigates without full-page reload (SPA routing)
- [ ] Does not compete with module-level headers (layout decision documented)

## Notes

Coordinate with home dashboard redesign; they share mobile shell.
```

**`docs/product/backlog/mobile-ux/visual-identity-refinement.md`:**

```markdown
---
id: mobile-visual-identity-refinement
title: Visual identity refinement pass
epic: mobile-ux
status: planned
priority: P2
created: 2026-04-23
updated: 2026-04-23
issues: []
prs: []
---

## Context

The cream/purple/Nunito foundation is solid. A deeper pass should tighten consistent spacing scale, better type hierarchy (heading sizes, weight usage, line heights), and a more cohesive member color palette (the 7 colors work but could feel more unified). During brainstorming, "keep the bones, refine the execution" was chosen over a bigger overhaul — but a more thorough refinement is still wanted. (From `docs/mobile-ux-polish-backlog.md` item 3.)

## Acceptance Criteria

- [ ] Spacing scale documented and applied across screens
- [ ] Type hierarchy documented (heading sizes, weights, line heights) and applied
- [ ] Member color palette reviewed for cohesion; adjustments documented
- [ ] Before/after comparison captured

## Notes

Scope bounded: refinement, not overhaul.
```

**`docs/product/backlog/mobile-ux/expandable-bottom-sheet.md`:**

```markdown
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
```

**`docs/product/backlog/mobile-ux/sidebar-settings-onboarding-mobile.md`:**

```markdown
---
id: mobile-sidebar-settings-onboarding
title: Sidebar + settings + onboarding mobile pass
epic: mobile-ux
status: planned
priority: P2
created: 2026-04-23
updated: 2026-04-23
issues: []
prs: []
---

## Context

These screens haven't been specifically optimized for mobile. The sidebar slides in but the settings/onboarding flows could benefit from mobile-native layouts. (From `docs/mobile-ux-polish-backlog.md` item 5.)

## Acceptance Criteria

- [ ] Sidebar interaction reviewed for mobile ergonomics (edge-swipe, tap-to-close, etc.)
- [ ] Settings screen layout adjusted for thumb reach and readability
- [ ] Onboarding flow reviewed and adjusted for mobile
```

**`docs/product/backlog/mobile-ux/notifications.md`:**

```markdown
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
```

**`docs/product/backlog/mobile-ux/drag-to-create.md`:**

```markdown
---
id: mobile-drag-to-create
title: Drag-to-create event on time grid
epic: mobile-ux
status: planned
priority: P2
created: 2026-04-23
updated: 2026-04-23
issues: []
prs: []
---

## Context

Tap and drag on the time grid to create an event at that time slot. (From `docs/mobile-ux-polish-backlog.md` item 6b.)

## Acceptance Criteria

- [ ] Tap-and-drag on empty time grid creates a new event at the dragged range
- [ ] Release opens the event create form pre-filled with start/end times
- [ ] Works on touch and mouse
- [ ] Does not interfere with existing event tap/edit interactions
```

**`docs/product/backlog/mobile-ux/pinch-to-zoom.md`:**

```markdown
---
id: mobile-pinch-to-zoom
title: Pinch-to-zoom calendar time granularity
epic: mobile-ux
status: planned
priority: P2
created: 2026-04-23
updated: 2026-04-23
issues: []
prs: []
---

## Context

Zoom in/out on calendar views to adjust time granularity. (From `docs/mobile-ux-polish-backlog.md` item 6c.)

## Acceptance Criteria

- [ ] Pinch-out increases time granularity (more detail, e.g. 30-min → 15-min slots)
- [ ] Pinch-in decreases granularity
- [ ] Zoom level persists across navigation
- [ ] Smooth animation; no layout thrash
```

- [ ] **Step 7.5: Delete the old mobile backlog doc**

```bash
cd /Users/joe.bor/code/family-hub
git rm docs/mobile-ux-polish-backlog.md
```

- [ ] **Step 7.6: Verify frontmatter parses as valid YAML**

```bash
cd /Users/joe.bor/code/family-hub
python3 -c "
import sys, yaml, glob
failed = []
for f in sorted(glob.glob('docs/product/backlog/*/*.md')):
    with open(f) as fh:
        content = fh.read()
    if not content.startswith('---\n'):
        failed.append(f + ' (no opening ---)')
        continue
    try:
        fm = content.split('---\n', 2)[1]
        yaml.safe_load(fm)
    except Exception as e:
        failed.append(f'{f}: {e}')
print('OK' if not failed else 'FAIL:\n  ' + '\n  '.join(failed))
sys.exit(0 if not failed else 1)
"
```

Expected: `OK`. If `FAIL`, fix the listed file(s) before committing.

- [ ] **Step 7.7: Commit**

```bash
git add docs/product/backlog/ docs/mobile-ux-polish-backlog.md
git commit -m "docs(product): add initial backlog stories (google-cal + mobile-ux)

- google-cal/incremental-sync (in-progress)
- google-cal/write-back (planned)
- mobile-ux/* (8 stories migrated from mobile-ux-polish-backlog.md)

Per spec §Decisions: shipped work is tracked in roadmap.md only;
story files cover active + future work."
```

---

## Task 8: Open PR and merge Tasks 2–7 to `main`

Everything so far has been on `feat/agents-md-canonical`. Push and open a PR.

- [ ] **Step 8.1: Push the branch**

```bash
cd /Users/joe.bor/code/family-hub
git push -u origin feat/agents-md-canonical
```

- [ ] **Step 8.2: Open the PR**

```bash
gh pr create --repo joe-bor/family-hub \
  --title "feat: product source-of-truth tree (PRD, roadmap, backlog)" \
  --body "$(cat <<'EOF'
Implements docs/superpowers/specs/2026-04-23-product-source-of-truth-design.md.

## Summary
- Rename root CLAUDE.md → AGENTS.md; CLAUDE.md is now a symlink
- Rewrite AGENTS.md for the product source-of-truth structure
- Scaffold docs/product/ (prd, roadmap, backlog)
- Import and refresh PRD to v2.0
- Seed roadmap as a shipped / active / planned summary index
- Add 10 initial story files (2 Google Calendar + 8 mobile UX)
- Migrate and delete docs/mobile-ux-polish-backlog.md

## What this does NOT do (separate tasks in the plan)
- Create the GitHub Project / labels / issue templates (Task 9)
- Archive docs/prompts/ (Task 10)
- Update MEMORY.md + sub-repo CLAUDE.md pointers (Tasks 11-12)

## Test plan
- [ ] Verify CLAUDE.md symlink resolves to AGENTS.md
- [ ] Verify all 10 story files have valid YAML frontmatter
- [ ] Verify roadmap links to existing design docs resolve
EOF
)"
```

- [ ] **Step 8.3: Merge via squash or merge commit (user choice)**

```bash
gh pr merge --repo joe-bor/family-hub --merge --delete-branch
git checkout main
git pull --ff-only
```

Expected: branch deleted, local `main` fast-forwarded.

---

## Task 9: Create cross-repo GitHub Project + labels + Issue templates

**Files:**
- Create: `joe-bor/FamilyHub/.github/ISSUE_TEMPLATE/story-task.md`
- Create: `joe-bor/family-hub-api/.github/ISSUE_TEMPLATE/story-task.md`

External: create Project, seed labels, configure auto-add.

- [ ] **Step 9.1: Create the Project**

```bash
gh project create --owner joe-bor --title "Family Hub"
```

Note the Project number from the output (e.g., `#1`). Record it — used in Step 9.3.

- [ ] **Step 9.2: Set Project status column names**

Default Project v2 has a Status field with `Todo`, `In Progress`, `Done`. Add `In Review`:

```bash
# Look up the project number programmatically
PROJECT_NUMBER=$(gh project list --owner joe-bor --format json \
  | jq -r '.projects[] | select(.title=="Family Hub") | .number')
echo "Project number: $PROJECT_NUMBER"
FIELD_ID=$(gh project field-list "$PROJECT_NUMBER" --owner joe-bor --format json \
  | jq -r '.fields[] | select(.name=="Status") | .id')
echo "Add 'In Review' option to Status field between 'In Progress' and 'Done' via the web UI:"
echo "  https://github.com/users/joe-bor/projects/$PROJECT_NUMBER/settings/fields/$FIELD_ID"
```

> **User action required:** The `gh` CLI cannot reorder Project v2 field options as of `gh` 2.x. Open the URL printed above and add the `In Review` option, dragging it between `In Progress` and `Done`. An agent executing this plan should stop and hand off to the user for this step.

- [ ] **Step 9.3: Configure Project auto-add rule**

> **User action required:** CLI lacks coverage for Project v2 workflows as of `gh` 2.x. Configure via web UI:

- Settings → Workflows → **Auto-add to project** → enable → set filter to `is:issue label:family-hub` across repos `joe-bor/FamilyHub`, `joe-bor/family-hub-api`, `joe-bor/family-hub`.
- Also enable the built-in **Auto-archive items** workflow with threshold `30 days` for items in `Done`.

An agent executing this plan should stop and hand off to the user for these toggles, then resume at Step 9.4.

- [ ] **Step 9.4: Seed labels in all three repos**

Create a small script at `/tmp/seed-labels.sh`:

```bash
#!/bin/bash
set -euo pipefail

REPOS=(joe-bor/family-hub joe-bor/FamilyHub joe-bor/family-hub-api)

declare -a LABELS=(
  "family-hub|BFD4F2|Triggers auto-add to the Family Hub Project"
  "priority:P0|E11D48|MVP-critical"
  "priority:P1|F59E0B|Roadmap"
  "priority:P2|84CC16|Nice-to-have"
  "type:feature|0E8A16|New capability"
  "type:bug|D73A4A|Defect"
  "type:chore|CFD3D7|Maintenance"
  "type:refactor|5319E7|Code change without behavior change"
  "type:docs|0075CA|Documentation"
  "needs-design|C2E0C6|Blocks start until design is resolved"
  "blocked|000000|Externally blocked"
)

for repo in "${REPOS[@]}"; do
  echo "=== $repo ==="
  for entry in "${LABELS[@]}"; do
    IFS='|' read -r name color desc <<< "$entry"
    gh label create "$name" --color "$color" --description "$desc" --repo "$repo" 2>&1 \
      | grep -v "already exists" || true
  done
done
```

Run it:

```bash
chmod +x /tmp/seed-labels.sh
/tmp/seed-labels.sh
```

Expected: each label created in each repo (or silently skipped if already present).

- [ ] **Step 9.5: Create Issue template in FE repo**

In the `frontend/` working directory (this is a separate git repo — `joe-bor/FamilyHub`):

```bash
cd /Users/joe.bor/code/family-hub/frontend
git checkout main && git pull
git checkout -b chore/add-issue-template
mkdir -p .github/ISSUE_TEMPLATE
```

Create `/Users/joe.bor/code/family-hub/frontend/.github/ISSUE_TEMPLATE/story-task.md`:

```markdown
---
name: Story task
about: A task implementing a story in the Family Hub product backlog
title: "[FE] "
labels: ["family-hub"]
---

Story: https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/<epic>/<story>.md

## Task
What this specific task accomplishes.

## Acceptance
- [ ] ...

## Notes
Implementation notes, links.
```

Commit + PR + merge:

```bash
git add .github/ISSUE_TEMPLATE/story-task.md
git commit -m "chore: add story-task issue template"
git push -u origin chore/add-issue-template
gh pr create --fill
gh pr merge --squash --delete-branch
git checkout main && git pull --ff-only
```

- [ ] **Step 9.6: Create Issue template in BE repo**

Same procedure in `backend/family-hub-api/`:

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
git checkout main && git pull
git checkout -b chore/add-issue-template
mkdir -p .github/ISSUE_TEMPLATE
```

Create `/Users/joe.bor/code/family-hub/backend/family-hub-api/.github/ISSUE_TEMPLATE/story-task.md` — same content as FE but with `title: "[BE] "`:

```markdown
---
name: Story task
about: A task implementing a story in the Family Hub product backlog
title: "[BE] "
labels: ["family-hub"]
---

Story: https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/<epic>/<story>.md

## Task
What this specific task accomplishes.

## Acceptance
- [ ] ...

## Notes
Implementation notes, links.
```

```bash
git add .github/ISSUE_TEMPLATE/story-task.md
git commit -m "chore: add story-task issue template"
git push -u origin chore/add-issue-template
gh pr create --fill
gh pr merge --squash --delete-branch
git checkout main && git pull --ff-only
```

- [ ] **Step 9.7: Smoke test the whole flow**

> Note: labels in this system are for broad filtering only. The authoritative link from an issue back to product context is the `Story:` URL in the issue body.

```bash
# Return to root repo
cd /Users/joe.bor/code/family-hub

PROJECT_NUMBER=$(gh project list --owner joe-bor --format json \
  | jq -r '.projects[] | select(.title=="Family Hub") | .number')

# Create a test issue in BE using only pre-seeded labels
gh issue create --repo joe-bor/family-hub-api \
  --title "[BE] Smoke test: issue template + auto-add to Project" \
  --body "Story: https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/google-calendar/incremental-sync.md

This is a smoke test. Close after verifying it auto-added to the Family Hub Project." \
  --label "family-hub,priority:P1,type:chore"
```

Capture the issue number from the create command output (e.g., `https://github.com/joe-bor/family-hub-api/issues/42` → `42`).

Wait ~30 seconds, then verify the Project picked it up (re-compute `PROJECT_NUMBER` because shell state does not persist across Bash tool calls):

```bash
PROJECT_NUMBER=$(gh project list --owner joe-bor --format json \
  | jq -r '.projects[] | select(.title=="Family Hub") | .number') && \
gh project item-list "$PROJECT_NUMBER" --owner joe-bor --limit 10
```

Expected: the smoke-test issue appears in the Project.

Close it once verified (set and use `ISSUE_NUMBER` in the same shell call):

```bash
# Replace <number> with the issue number from the create output
ISSUE_NUMBER=<number> && \
gh issue close "$ISSUE_NUMBER" --repo joe-bor/family-hub-api --comment "Smoke test complete."
```

---

## Task 10: Archive `docs/prompts/` with verification

**Files:**
- Modify: `docs/prompts/*.md` (move verified into `_archive/`, leave unverified in place)
- Create: `docs/prompts/_needs_review.md`

- [ ] **Step 10.1: Create a feature branch**

```bash
cd /Users/joe.bor/code/family-hub
git checkout -b chore/archive-prompts
```

- [ ] **Step 10.2: Create the archive directory**

```bash
mkdir -p docs/prompts/_archive
```

- [ ] **Step 10.3: Classify each prompt file**

This is done one file at a time. For each file in `docs/prompts/*.md`:

1. Read the file (particularly its title, scope, and any mentioned PR numbers).
2. Extract 1–3 keywords from the title.
3. Query merged PRs in both repos:

```bash
gh pr list --repo joe-bor/FamilyHub --state merged --search "<keyword>" --limit 10 --json number,title,mergedAt
gh pr list --repo joe-bor/family-hub-api --state merged --search "<keyword>" --limit 10 --json number,title,mergedAt
```

4. Classify:
   - **Verified** if one or more merged PRs match the prompt's scope confidently.
   - **Unverified** if no match or ambiguous.

- [ ] **Step 10.4: Move verified prompts**

For each verified prompt, prepend a provenance header and move:

```bash
FILE=docs/prompts/<filename>.md
REPO=<FamilyHub or family-hub-api>
PR=<pr-number>

# Prepend provenance line
{
  echo "> Archived 2026-04-23 — implemented by $REPO#$PR"
  echo ""
  cat "$FILE"
} > "/tmp/archived-$(basename $FILE)"
mv "/tmp/archived-$(basename $FILE)" "docs/prompts/_archive/$(basename $FILE)"
rm -f "$FILE"  # the move via temp file + rm keeps things explicit
```

(In practice the agent doing this will script it. The key invariant: every file in `_archive/` has a provenance header.)

- [ ] **Step 10.5: Write `_needs_review.md` for unverified**

For each unverified prompt, append a row to `docs/prompts/_needs_review.md`:

```markdown
# Prompts needing review

These prompts could not be confidently matched to a merged PR. Options for each:
- Match to a PR manually → move to `_archive/` with provenance header
- Recognize as abandoned → delete
- Recognize as still-relevant work → convert into a backlog story under `docs/product/backlog/<epic>/`

| Filename | Reason not matched | Proposed action |
|----------|--------------------|-----------------|
| `prompt-name.md` | No PR matched keywords "X Y Z" | (to decide) |
| ... | ... | ... |
```

Leave unverified prompts in `docs/prompts/` (not in `_archive/`).

- [ ] **Step 10.6: Count each bucket and commit in one shell session**

Shell state does not persist across Bash tool invocations — count and commit must happen in a single call:

```bash
cd /Users/joe.bor/code/family-hub && \
VERIFIED=$(ls docs/prompts/_archive/ 2>/dev/null | wc -l | tr -d ' ') && \
UNVERIFIED=$(find docs/prompts -maxdepth 1 -name '*.md' -not -name '_needs_review.md' | wc -l | tr -d ' ') && \
echo "Verified: $VERIFIED, Unverified: $UNVERIFIED" && \
git add docs/prompts/ && \
git commit -m "chore(prompts): archive verified prompts; surface unverified for review

Verified: $VERIFIED prompts moved to _archive/ with provenance.
Unverified: $UNVERIFIED prompts listed in _needs_review.md for user triage."
```

- [ ] **Step 10.7: Push and merge**

```bash
git push -u origin chore/archive-prompts
gh pr create --fill
gh pr merge --merge --delete-branch
git checkout main && git pull --ff-only
```

- [ ] **Step 10.8: Surface `_needs_review.md` to the user**

After merge, report: "X prompts archived; Y need your review. See `docs/prompts/_needs_review.md`." User handles the Y file triage in a follow-up session.

---

## Task 11: Update `MEMORY.md` to point at the new tree

**Files:**
- Modify: `/Users/joe.bor/.claude/projects/-Users-joe-bor-code-family-hub/memory/MEMORY.md`

- [ ] **Step 11.1: Read the current `MEMORY.md`**

Familiarize with current content so the edit is surgical.

- [ ] **Step 11.2: Replace the "Alignment Status" section**

Replace the entire section starting at `## Alignment Status` through the end of its last subsection (the `### Historical Wiring Fixes (complete)` block), stopping immediately before `## Docker & CI`. Leave `## Docker & CI`, `## Deployment`, `## FE Data Flow Pattern`, and any other sections below intact. Replace with:

```markdown
## Alignment Status & Roadmap

Product source of truth now lives in `joe-bor/family-hub`:

- **PRD:** `docs/product/prd.md`
- **Roadmap:** `docs/product/roadmap.md`
- **Backlog:** `docs/product/backlog/<epic>/<story>.md`
- **Agent entry:** `AGENTS.md` at root (symlink `CLAUDE.md` → `AGENTS.md`)

Live task state: GitHub Project "Family Hub" (user-level, cross-repo). Story status lives in story frontmatter; labels are for broad filtering only.

**Near-term delivery sequence:** Google Calendar read-only sync → Google Calendar write-back → Mobile UX polish → Tasks/todos.

**Fully aligned as of 2026-03-08** — both FE and BE deployed and live. See `docs/calendar-events-api-reference.md` for the shared contract.
```

- [ ] **Step 11.3: Add a new reference entry**

No need to add a new memory file — `MEMORY.md` already links to the relevant files. Just ensure the existing references still point to valid paths after this migration (API reference doc unchanged, deployment guide unchanged).

- [ ] **Step 11.4: Save**

`MEMORY.md` is outside any git repo (lives in `~/.claude/projects/...`). No commit needed; it auto-persists.

---

## Task 12: Update FE and BE CLAUDE.md pointers

**Files:**
- Modify: `/Users/joe.bor/code/family-hub/frontend/CLAUDE.md`
- Modify: `/Users/joe.bor/code/family-hub/backend/family-hub-api/CLAUDE.md`

- [ ] **Step 12.1: Add pointer section to FE CLAUDE.md**

In `frontend/`, on a new branch:

```bash
cd /Users/joe.bor/code/family-hub/frontend
git checkout main && git pull
git checkout -b docs/add-product-pointer
```

Append a section to `frontend/CLAUDE.md` (exact placement depends on existing structure — read first, then insert near the top after any existing intro). Text to add:

```markdown
## Product source of truth (cross-repo)

The Family Hub product vision, roadmap, and backlog live in the root workspace repo `joe-bor/family-hub`, in `docs/product/`. Before starting a story or feature, read the linked story file.

- PRD: `../docs/product/prd.md` (if working in the cloned workspace)
- Roadmap: `../docs/product/roadmap.md`
- Backlog: `../docs/product/backlog/`
- Agent entry: `../AGENTS.md`

Tasks come in as GitHub Issues with a `Story:` line pointing to the story file. Follow that link for context.
```

Commit, push, PR, merge:

```bash
git add CLAUDE.md
git commit -m "docs: add cross-repo pointer to product source of truth"
git push -u origin docs/add-product-pointer
gh pr create --fill
gh pr merge --squash --delete-branch
git checkout main && git pull --ff-only
```

- [ ] **Step 12.2: Same for BE CLAUDE.md**

```bash
cd /Users/joe.bor/code/family-hub/backend/family-hub-api
git checkout main && git pull
git checkout -b docs/add-product-pointer
```

Append the same section (adjusting relative paths if needed — BE is one level deeper than FE):

```markdown
## Product source of truth (cross-repo)

The Family Hub product vision, roadmap, and backlog live in the root workspace repo `joe-bor/family-hub`, in `docs/product/`. Before starting a story or feature, read the linked story file.

- PRD: `../../docs/product/prd.md` (if working in the cloned workspace)
- Roadmap: `../../docs/product/roadmap.md`
- Backlog: `../../docs/product/backlog/`
- Agent entry: `../../AGENTS.md`

Tasks come in as GitHub Issues with a `Story:` line pointing to the story file. Follow that link for context.
```

Commit, push, PR, merge same as Step 12.1.

---

## Final verification

- [ ] **Step F.1: Fresh-clone smoke test**

From a scratch directory, clone all three repos and verify the agent entry flow works:

```bash
mkdir /tmp/family-hub-clone && cd /tmp/family-hub-clone
gh repo clone joe-bor/family-hub
cd family-hub
gh repo clone joe-bor/FamilyHub frontend
gh repo clone joe-bor/family-hub-api backend/family-hub-api
ls AGENTS.md docs/product/prd.md docs/product/roadmap.md
cat CLAUDE.md | head -1
```

Expected: all files exist; `CLAUDE.md` shows `# Family Hub — Agent Entry Point` (via symlink).

- [ ] **Step F.2: Confirm Project + labels populated**

```bash
gh project list --owner joe-bor
gh label list --repo joe-bor/family-hub | grep -E "family-hub|priority:P|type:"
gh label list --repo joe-bor/FamilyHub | grep -E "family-hub|priority:P|type:"
gh label list --repo joe-bor/family-hub-api | grep -E "family-hub|priority:P|type:"
```

Expected: Project exists; all labels present in all three repos.

- [ ] **Step F.3: Announce to user**

Report:

- Root repo URL: `https://github.com/joe-bor/family-hub`
- Project URL: (from Task 9)
- Count of archived prompts / needs-review prompts
- Next actions for user: triage `_needs_review.md`; start using Issues with the story-task template for new work

Cleanup `/tmp/family-hub-clone`:

```bash
rm -rf /tmp/family-hub-clone
```

---

## Commits produced by this plan

A successful execution produces these commits across three repos:

**`joe-bor/family-hub` (new repo):**
1. `chore: initialize root workspace repo`
2. Merged PR: `feat: product source-of-truth tree (PRD, roadmap, backlog)` — contains Tasks 2–7
3. Merged PR: `chore(prompts): archive verified prompts; surface unverified for review` — contains Task 10

**`joe-bor/FamilyHub` (existing FE repo):**
4. Merged PR: `chore: add story-task issue template`
5. Merged PR: `docs: add cross-repo pointer to product source of truth`

**`joe-bor/family-hub-api` (existing BE repo):**
6. Merged PR: `chore: add story-task issue template`
7. Merged PR: `docs: add cross-repo pointer to product source of truth`

---

## Out of scope for this plan

- Backfilling story files for already-shipped work (decided in spec §Decisions)
- Auto-generating roadmap.md from story frontmatter (deferred)
- CI enforcement of frontmatter schema (deferred)
- Monorepo migration (separate project)
- Triaging `_needs_review.md` (user's call, post-merge)
