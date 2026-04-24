# Product Source of Truth

## Problem

Family Hub has no single source of truth for what is being built. Product intent, roadmap, status, and task-level work are scattered across:

- A PRD in `~/Downloads/family-calendar-prd.md` (outside the project, outdated in several sections)
- A compact roadmap table in `MEMORY.md` (feature-level only, limited detail)
- ~15 design documents in `docs/` (per-feature, no cross-cutting status)
- 34+ handoff prompt files in `docs/prompts/` (task-level, completed and in-flight mixed together)
- Two separate GitHub repos (`FamilyHub`, `family-hub-api`) with zero Issues filed
- The developer's head (highest-bandwidth, lowest-durability storage)

The root workspace (`family-hub/`) is not a git repo. FE and BE are separate repos cloned as subdirectories. Any docs at the root are unversioned.

Agents spinning up cold cannot answer basic orientation questions — "What is Family Hub? What's shipped? What's in flight? What's next? Why?" — without extensive exploration and re-derivation. This is the core friction driving this work.

## Goals

- **Agent-first, agent-agnostic.** Any agent (Claude Code, Codex, Gemini, Cursor, or a human) reads the same files. No Claude-specific hooks or skills are the source of truth.
- **Three tiers that change at their natural rates.** Product (stable), roadmap (medium churn), backlog (high churn). Work-item tasks live on GitHub.
- **Queryable both ways.** An agent starting from an Issue can find the story; an agent starting from a story can find open Issues and merged PRs.
- **Minimal tooling.** Markdown + `gh` CLI + frontmatter. No custom generators or sync scripts in this phase.
- **Preserve existing work.** No re-creation of prompt files that already tracked implementations that merged and were reviewed; archive them with provenance instead.

### Non-goals

- Not migrating FE and BE into a monorepo (separate project; not in scope here).
- Not auto-generating the roadmap from story frontmatter (deferred; manual maintenance for now).
- Not mirroring every backlog story as a GitHub Issue (only task-level work becomes Issues).
- Not introducing GitHub Milestones (roadmap.md + story status serve that role).
- Not adding CI checks that enforce the structure (trust and convention for now).

## Design

### 1. Repository setup — Option 1 (root git, FE/BE untouched)

The root `family-hub/` directory becomes its own git repo: **`joe-bor/family-hub`**. FE and BE repos remain independent; they continue to be cloned as sibling directories inside this root.

**Tracked in the root repo:**
- `docs/` (product docs, design docs, API references)
- `AGENTS.md` (canonical agent entry point)
- `CLAUDE.md` (symlink to `AGENTS.md`)
- `docker-compose.yml`
- Any future root-level scripts or orchestration files
- `.gitignore`, `README.md`

**Ignored:**
- `frontend/` (separate repo: `joe-bor/FamilyHub`)
- `backend/` (contains `family-hub-api/`, a separate repo)
- `node_modules/`, OS cruft (`.DS_Store`), editor files

No changes are made to FE or BE repos, their CI, release-please config, GHCR image builds, or deploy scripts. In-flight PRs on either side are unaffected.

### 2. Three-tier documentation structure

```
docs/
├── product/
│   ├── prd.md                          # Stable product doc — refactored from existing PRD
│   ├── roadmap.md                      # Epic/story status — one page, tables per epic
│   └── backlog/
│       ├── <epic-id>/
│       │   └── <story-id>.md           # Story files with YAML frontmatter
│       └── ...
├── prompts/
│   ├── _archive/                       # Verified prompts land here
│   └── _needs_review.md                # Prompts with no matching merged PR
├── calendar-events-api-reference.md    # Stays — shared contract doc
├── deployment-guide.md                 # Stays
├── *-design.md                         # Stays — design docs linked from stories
├── alignment-reports/                  # Stays
├── diagrams/                           # Stays
└── superpowers/specs/                  # Stays — spec artifacts

AGENTS.md                               # Repo root — agent-agnostic entry point
CLAUDE.md                               # Symlink → AGENTS.md
```

**Jurisdiction:**

| Tier | File(s) | Changes | Scope |
|------|---------|---------|-------|
| Product | `docs/product/prd.md` | Rarely | Vision, personas, scope, non-goals, architecture |
| Roadmap | `docs/product/roadmap.md` | Monthly | Epics/features with status rollup |
| Backlog | `docs/product/backlog/<epic>/<story>.md` | Weekly | User stories with AC and design links |
| Work items | GitHub Issues | Daily | PR-sized tasks, linked to stories |

Each tier changes at its natural rate without coupling to the others. Agents and humans read whichever tier answers the question at hand.

### 3. Story file schema

Every backlog story has the same YAML frontmatter. The filename stem matches the `id` field; the parent directory matches the `epic` field.

**Example — `docs/product/backlog/google-calendar/incremental-sync.md`:**

```markdown
---
id: google-cal-incremental-sync
title: Incremental sync with sync tokens
epic: google-calendar-read-only
status: in-progress
priority: P1
created: 2026-04-23
updated: 2026-04-23
design_doc: docs/google-calendar-integration-design.md
issues:
  - repo: family-hub-api
    number: 42
  - repo: FamilyHub
    number: 118
prs:
  - repo: family-hub-api
    number: 25
    merged: 2026-03-20
---

## Context
One paragraph: what this story is and why it matters. Link to PRD section if relevant.

## Acceptance Criteria
- [ ] Backend uses `syncToken` from prior full sync
- [ ] Scheduler polls every 5 minutes
- [ ] Token invalidation triggers re-sync (410 Gone handling)
- [ ] FE reflects updates within one poll cycle

## Out of Scope
- Write-back to Google Calendar (→ `google-cal-write-back.md`)
- Webhook-based push notifications (Phase 3)

## Notes
Free-form design notes, open questions, decisions. Optional.
```

**Frontmatter field reference:**

| Field | Required | Values | Notes |
|-------|----------|--------|-------|
| `id` | yes | kebab-case, matches filename | Stable identifier — never rename |
| `title` | yes | string | Human-readable |
| `epic` | yes | kebab-case | Groups stories; corresponds to `backlog/<epic>/` dir |
| `status` | yes | `planned` \| `in-progress` \| `done` \| `deferred` \| `blocked` | See §4 |
| `priority` | yes | `P0` \| `P1` \| `P2` | P0 = MVP-critical, P1 = roadmap, P2 = nice-to-have |
| `created` | yes | `YYYY-MM-DD` | |
| `updated` | yes | `YYYY-MM-DD` | Bump when status changes |
| `design_doc` | optional | path (string) | If a design doc exists |
| `issues` | optional | list of `{repo, number}` | GitHub Issues implementing this story |
| `prs` | optional | list of `{repo, number, merged}` | PRs that landed for this story |

### 4. Status model — five states, human-managed

| State | Meaning |
|-------|---------|
| `planned` | Story written, AC defined, no Issue or PR yet open |
| `in-progress` | At least one associated Issue is open or PR is in flight |
| `done` | All AC checkboxes ticked and all PRs merged |
| `blocked` | External blocker; `## Notes` must document the blocker |
| `deferred` | Consciously postponed; `## Notes` must state the reason |

Status is **human-managed**, not derived from Issue state. Updates to `status:` also bump `updated:` to today. No automation watches GitHub to flip states. If manual maintenance becomes painful, a derivation script can be added later as a separate project.

### 5. Roadmap format

`docs/product/roadmap.md` is a single page with one table per epic. Each row is a story with links back to the story file and to closed/open PRs.

**Example structure:**

```markdown
# Family Hub — Roadmap

Last updated: 2026-04-23 · Source of truth: individual story files in `backlog/`.

## Epic: Foundation (shipped)

| Story | Status | Design | PRs |
|-------|--------|--------|-----|
| Family registration + JWT auth | done | — | (retrofit) |
| Docker + GHCR + prod deploy | done | docs/deployment-guide.md | — |
| E2E against real backend | done | — | FE #105 |

## Epic: Calendar core (shipped)

| Story | Status | Design | PRs |
|-------|--------|--------|-----|
| CRUD calendar events | done | docs/calendar-events-api-reference.md | multiple |
| All-day event rendering | done | docs/all-day-and-multi-day-events-design.md | FE #110 |
| Multi-day events (endDate) | done | same | BE #18, #19, FE #111 |
| Recurring events | done | docs/recurring-events-design.md | BE #20, #21, FE #117 |

## Epic: Google Calendar read-only sync (in progress)

| Story | Status | Design | PRs |
|-------|--------|--------|-----|
| OAuth flow | done | docs/google-calendar-integration-design.md | BE #22 |
| Calendar selection | done | same | BE #23 |
| Full sync | done | same | BE #24, #25 |
| Incremental sync + scheduler | in-progress | same | — |

## Epic: Google Calendar write-back (planned)

| Story | Status | Design | PRs |
|-------|--------|--------|-----|
| Write-back to Google (create/edit/delete) | planned | docs/google-calendar-integration-design.md | — |

## Epic: Tasks/todos (deferred)

Deferred post-MVP per PRD realignment (v2.0). Will be planned when scheduled.
```

Seed content is populated from `MEMORY.md` and the shipped PR history at migration time.

### 6. GitHub Issues and Project conventions

**One org-level Project: `Family Hub`** — cross-repo, private. Pulls Issues from three repos: `joe-bor/family-hub` (rare), `joe-bor/FamilyHub` (FE), `joe-bor/family-hub-api` (BE).

- Columns: `Todo` → `In Progress` → `In Review` → `Done`
- Auto-add rule: any Issue with label `family-hub` is auto-added to the Project
- Auto-archive: items in `Done` for more than 30 days (using GitHub Projects' built-in auto-archive workflow; configured at Project setup time)

**Issue body template** (`.github/ISSUE_TEMPLATE/story-task.md`, added to both FE and BE repos):

```markdown
Story: docs/product/backlog/<epic>/<story>.md

## Task
What this specific task accomplishes.

## Acceptance
- [ ] ...

## Notes
Implementation notes, links.
```

**Label taxonomy** (seeded in both FE and BE repos):

| Label | Purpose |
|-------|---------|
| `family-hub` | Triggers Project auto-add |
| `story:<story-id>` | Links Issue to story (e.g., `story:google-cal-incremental-sync`) |
| `epic:<epic-id>` | Rolls up to epic for filtering |
| `priority:P0` / `priority:P1` / `priority:P2` | Mirrors story priority |
| `type:feature` / `type:bug` / `type:chore` / `type:refactor` / `type:docs` | Matches conventional commit types |
| `needs-design` | Blocks start until design is resolved |
| `blocked` | Externally blocked |

### 7. Linking conventions

**Story → Issue:** story frontmatter `issues:` list.
**Issue → Story:** Issue body starts with `Story: docs/product/backlog/<epic>/<story>.md` (path is relative to `joe-bor/family-hub` repo root).
**Issue → PR:** standard `Closes #N` in PR description.
**PR → Story:** traversed via `PR → Issue → Story` chain.
**Story → PR:** story frontmatter `prs:` list (populated on merge).

Labels are the primary query surface: `gh issue list --label "story:google-cal-incremental-sync"` returns every task for a story across both repos.

### 8. Agent entry point — AGENTS.md at repo root

The root `CLAUDE.md` currently contains the architect/orchestrator role guidance, repo map, and handoff workflow. This content moves into a new canonical file `AGENTS.md`. A symlink `CLAUDE.md → AGENTS.md` keeps Claude Code's conventional filename working.

**Migration steps:**
1. `git mv CLAUDE.md AGENTS.md` (after root git init, §10)
2. Append new pointers section to `AGENTS.md` referencing `docs/product/`, `roadmap.md`, `backlog/`, and the GitHub Project
3. `ln -s AGENTS.md CLAUDE.md`
4. `git add CLAUDE.md` (tracks the symlink)

**Unchanged:** `frontend/CLAUDE.md` and `backend/family-hub-api/CLAUDE.md` are repo-scoped, contain different content (FE-specific vs BE-specific agent instructions), and remain as real files inside their respective repos.

**Risk:** if a future harness does not resolve symlinks, we'd switch to duplicated files plus a sync script. No known problems today with Claude Code, Codex, or Cursor.

**New section added to `AGENTS.md` (append, do not replace existing content):**

```markdown
## Agent entry — sources of truth

Start here in any session.

| Doc | What | When to read |
|-----|------|--------------|
| `docs/product/prd.md` | Product vision, personas, scope, architecture | Session start, orientation |
| `docs/product/roadmap.md` | Epic/story status | "What's shipped / in progress / next?" |
| `docs/product/backlog/<epic>/<story>.md` | Per-story AC, design links, Issues, PRs | Working on a specific story |
| GitHub Project "Family Hub" (org-level) | Live task-level state | Picking up in-flight work |

When picking up a task:
1. `gh issue view <N> --repo <repo>` → find the `Story:` line
2. Read that story file for context and AC
3. Read linked `design_doc` if any
4. Execute; open PR with `Closes #N`
```

### 9. PRD refresh — surgical, not rewrite

The existing PRD in `~/Downloads/family-calendar-prd.md` (v1.1, Dec 2024) has solid bones — vision, personas, user stories, architecture — but several sections have drifted from reality. It is imported into `docs/product/prd.md` with targeted edits, not rewritten.

**Edits:**

1. **Header** — version 2.0, date today, status "Living document — see roadmap.md for current phase."
2. **§7 Feature Requirements** — for each feature, add one line:
   `**Implementation status:** Shipped (links) | In progress (→ roadmap) | Deferred | Not started`
3. **§6 Product Scope** — rebalance:
   - Move to Deferred: all of §7.3 Tasks, two-way Google Cal (Phase 1 is read-only), real-time WebSocket sync (polling works)
   - Add to Shipped: JWT auth + family registration, recurring events, multi-day events
4. **§8 Technical Architecture** — align with code:
   - State management: Zustand + TanStack Query (not Jotai / Axios)
   - Database: Neon Postgres in prod, H2 in dev
   - Deployment: DigitalOcean droplet, Docker Compose, GHCR
   - WebSocket section: mark Deferred or remove
5. **§12 Development Phases** — delete and replace with:
   > Phase planning lives in `docs/product/roadmap.md`. This section historically contained a Dec 2024 sprint plan that is no longer representative.
6. **§15.3 Related Documents** — add pointers to `roadmap.md`, `backlog/`, `AGENTS.md`.
7. **Change log** — add v2.0 entry describing the realignment.

The rewritten PRD becomes `docs/product/prd.md`. The `~/Downloads/` copy can be deleted (or archived by the user).

### 10. Prompts archive with verification

For each of the 34 files in `docs/prompts/`:

1. **Extract** feature/scope keywords from filename + title + body of the prompt.
2. **Search** merged PRs in both FE and BE using `gh pr list --repo <repo> --state merged --search "<keywords>"` and by scanning each prompt for explicit PR references.
3. **Classify:**
   - **Verified** — prompt matched to at least one PR that was merged into `main` (this is a solo project; "merged to main" is the bar, not a separate reviewer's approval) → move file to `docs/prompts/_archive/<filename>` and prepend a one-line header:
     `> Archived 2026-04-23 — implemented by <repo>#<pr>`
   - **Unverified** — no confident match or ambiguous → leave in place and add an entry to `docs/prompts/_needs_review.md` with the reason (no matching PR, ambiguous match, prompt appears abandoned, etc.)
4. **Report** — end-of-run summary with verified count, needs-review count, and the `_needs_review.md` list for user triage.

Agents executing this step do not guess: anything that cannot be confidently matched surfaces for human review.

### 11. Root git repo initialization

Once-only setup steps:

1. `cd /Users/joe.bor/code/family-hub && git init`
2. Write `.gitignore`:
   ```
   frontend/
   backend/
   node_modules/
   .DS_Store
   *.swp
   .idea/
   .vscode/
   ```
3. Add and commit all currently-untracked root files in the workspace: `docs/`, `CLAUDE.md`, `docker-compose.yml`, root-level scripts. Excludes `frontend/` and `backend/` (separate repos, ignored) and `MEMORY.md` (lives outside the repo at `~/.claude/projects/...`).
4. Create GitHub repo `joe-bor/family-hub` (private).
5. `git remote add origin git@github.com:joe-bor/family-hub.git`
6. `git push -u origin main`
7. Create the org-level GitHub Project `Family Hub` and connect all three repos.
8. Add the `family-hub` label to all three repos (seeded via `gh label create`).

After this, the product structure migration (sections 2, 3, 5, 8, 9) can proceed as normal branches and PRs against the new `family-hub` repo.

## Migration sequencing (high level)

Implementation order — details go in the writing-plans step:

1. **Root git repo setup** (§11) — unblocks everything else.
2. **AGENTS.md / CLAUDE.md symlink migration** (§8) — small, low risk.
3. **Seed `docs/product/` scaffolding** — empty `prd.md`, `roadmap.md`, `backlog/` directory.
4. **PRD refresh** (§9) — import, edit, commit.
5. **Seed roadmap** (§5) — populate from MEMORY.md + shipped PR history.
6. **Backfill backlog stories for shipped and in-flight epics** — at minimum: Google Calendar epic (in progress), enough historical stories to validate the schema. Not every shipped story needs a retroactive backlog file; aim for the active epics.
7. **GitHub Issues / Project setup** (§6) — create Project, seed labels, add Issue templates to both repos.
8. **Prompts archive execution** (§10) — agent pass with human review gate on `_needs_review.md`.
9. **Update `MEMORY.md`** — point to the new product tree as the source of truth; shrink the inline calendar roadmap table to a pointer.
10. **Update repo-level `frontend/CLAUDE.md` and `backend/family-hub-api/CLAUDE.md`** — add a one-line pointer to `docs/product/` in the root repo for cross-repo context.

## Risks

- **Symlink portability.** Claude Code, Codex, and Cursor all resolve symlinks today. If a future harness does not, we fall back to duplicated files + sync script.
- **Maintenance drift.** Human-managed status in story frontmatter can go stale. The mitigation is discipline: update `status` and `updated` in the same commit that changes Issue/PR state. If this fails repeatedly, add a derivation script.
- **Three GitHub repos in one Project.** Org-level Projects handle this cleanly. The `family-hub` repo will have few or no Issues — it's mostly a docs repo.
- **PRD rewrite scope creep.** Keep edits surgical. Goal is alignment with reality, not a full rewrite. If deeper rework is needed, spin it out as a separate story.
- **Prompts archive false positives.** Bounded by the `_needs_review.md` gate: nothing leaves the working directory without a confident PR match.

## Decisions

- **Backlog coverage:** only in-progress and future work. Shipped epics (Foundation, Calendar core, Google Calendar read-only steps that already merged) are represented in `roadmap.md` with their PR links but do NOT get backfilled story files. Story files are only created for currently-in-flight work and for planned/future stories.
- **`docs/mobile-ux-polish-backlog.md`:** convert each line item into a story under a new `mobile-ux` epic. The original doc is removed once stories are migrated.

## Success criteria

- An agent starting cold (any harness) can answer "What is Family Hub? What's shipped? What's in progress? What's next?" by reading three files: `AGENTS.md`, `docs/product/prd.md`, `docs/product/roadmap.md`.
- Every in-flight task traces: GH Issue → `Story:` path → story file → design doc → PR (via `Closes #N`).
- `docs/prompts/` no longer contains a mix of completed and in-flight work. Everything in it is either archived or explicitly flagged for review.
- The PRD in `docs/product/prd.md` matches shipped reality as of the migration date.
- Status of every open-work story is accurate at the time of each relevant merge.
