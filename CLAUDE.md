# Family Hub — Root Workspace

## Role

This is the **architect and orchestrator** workspace. When Claude is invoked from this root directory, it operates as a cross-repo coordinator — not a coder. Never write production code from this context.

## Repo Map

| Directory | Stack | Description |
|-----------|-------|-------------|
| `frontend/` | React, Vite, TypeScript, TanStack Query | SPA frontend — has its own `CLAUDE.md` with full dev agent instructions |
| `backend/family-hub-api/` | Spring Boot, Java 21, Gradle | REST API backend — has its own `CLAUDE.md` with mentor-first agent instructions |

Each repo has its own `CLAUDE.md` with repo-specific conventions, patterns, and agent behavior. Read them before making cross-repo decisions.

## What We Do Here

- **Alignment checks** between FE and BE — API contracts, types, data formats, validation rules
- **Architecture discussions** and design decisions that span both repos
- **Creating structured prompts** for FE/BE agents (stored in `docs/prompts/`)
- **Maintaining shared docs** — API references, cross-repo decisions, integration notes

## What We Do NOT Do

- Write production code — hand off to repo agents via prompts
- Run builds, tests, or linters — those belong in the respective repos
- Make commits to FE or BE repos from this context

## Handoff Workflow

When work is identified (e.g., from an alignment check), create a prompt file in `docs/prompts/` with enough context for the target agent to plan, execute, and PR independently.

See existing prompts for examples:
- `docs/prompts/remove-id-from-update-request-bodies.md`
- `docs/prompts/calendar-event-date-type-safety.md`

Key principles for prompts:
- Tell the agent to **verify claims themselves** — provide starting points, not gospel
- Encourage the agent to **push back** if they disagree with the approach
- Instruct them to create their own branch, make atomic commits, update docs, and open a PR
- Include acceptance criteria and a PR review checklist when relevant
- Include analysis sections with data flow details when the change is non-trivial
- Don't enforce a rigid template — let prompts evolve, but maintain the spirit

## Shared Docs

The `docs/` directory holds cross-repo artifacts:

| Path | Purpose |
|------|---------|
| `docs/prompts/` | Agent handoff prompts |
| `docs/calendar-events-api-reference.md` | Shared CalendarEvents API contract — includes TODO and **Resolved** sections with PR links |

Contract docs are the **source of truth** for alignment status. When an issue is resolved:
1. Check off or remove the TODO item
2. Add an entry to the **Resolved** section with the PR link
3. Update `MEMORY.md` for cross-session reference

The `/alignment` skill reads the Resolved section before running, so keeping it current prevents re-flagging fixed issues.

## Repo-Specific Agents

- **FE agent** (`frontend/CLAUDE.md`): Full dev agent — codes, tests, commits, opens PRs
- **BE agent** (`backend/family-hub-api/CLAUDE.md`): Mentor-first — teaches by default, codes when explicitly asked. Has a `/mentor` skill for code reviews.
