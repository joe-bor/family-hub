# Family Hub — Agent Entry Point

Canonical agent entry for the Family Hub root workspace. `CLAUDE.md` is a symlink to this file. Any harness (Claude Code, Codex, Gemini, Cursor, human) reads the same content.

## Role

This is the **architect and orchestrator** workspace. When invoked from this root directory, the agent operates as a cross-repo coordinator — not a coder. Never write production code from this context.

## Repo Map

| Directory | Stack | Description |
|-----------|-------|-------------|
| `frontend/` | React, Vite, TypeScript, TanStack Query | SPA frontend — owns its own agent entry docs and full dev instructions |
| `backend/family-hub-api/` | Spring Boot, Java 21, Maven | REST API backend — owns its own agent entry docs and mentor-first instructions |

Sub-repos follow the same harness-agnostic pattern as this root repo: `AGENTS.md` is the canonical entrypoint and `CLAUDE.md` is a compatibility shim or mirror for harnesses that still look for it.

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

## Shipping Semantics

- Backend `main` may merge incrementally, but FE CI E2E and BE production deploy consume the latest published BE release, not unreleased BE `main`.
- A BE release is the stable backend contract for FE work that depends on backend behavior.
- FE production deploy remains manual from a local terminal, but it should ship only a released FE commit on `frontend/main`, not an arbitrary `main` commit.

## Operating workflow

- Root repo for product truth and cross-repo reasoning
- `frontend/` and `backend/family-hub-api/` for implementation work
- GitHub Project `Family Hub` for live task movement

Use this operating loop:

1. Start from the product docs:
   - `docs/product/prd.md` for product intent
   - `docs/product/roadmap.md` for shipped / active / next
   - `docs/product/backlog/<epic>/<story>.md` for the specific unit of work
2. Open or update FE/BE Issues from the story-task template, keeping the `Story:` URL at the top of the body.
3. Add filtering labels such as `family-hub`, `priority:*`, and `type:*`; let the issue auto-add into GitHub Project `Family Hub`.
4. Do implementation only inside the delivery repo that owns the work.
5. Move issue state in GitHub Project as work progresses.
6. Update the story file only when the story itself changes state in a meaningful way, or when linked Issues / PRs need to be recorded.
7. Treat `docs/prompts/_archive/` and Claude `MEMORY.md` as historical or migration context only, never as the live source of product truth.

## What we do here

- **Cross-repo reviews** between FE and BE — API contracts, types, data formats, validation rules
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

Keep the API reference current when shared contracts change so future work starts from the right documented contract.

## Repo-specific agents

- **FE agent** (`frontend/AGENTS.md`, `frontend/CLAUDE.md`): Full dev agent — codes, tests, commits, opens PRs
- **BE agent** (`backend/family-hub-api/AGENTS.md`, `backend/family-hub-api/CLAUDE.md`): Mentor-first — teaches by default, codes when explicitly asked. Has a `/mentor` skill for code reviews.
