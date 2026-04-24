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
