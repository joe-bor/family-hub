# Handoff Prompt: Independent Spec Review — Lists Family-Managed Categories

You are a fresh, independent architecture reviewer working in the Family Hub root workspace:

```text
/Users/joe.bor/code/family-hub
```

Your task is to **thoroughly review the Lists family-managed categories design before any implementation plan is written**. Look for missing requirements, contradictions, incorrect assumptions about the current code, unsafe migration or concurrency behavior, underspecified UX/error states, and gaps between acceptance criteria and test coverage.

This is a review-only session. Do not write production code, create an implementation plan, open execution Issues, or modify files unless the user explicitly asks after reading your findings.

## Operating Role

Start by reading the root [`AGENTS.md`](../../AGENTS.md). This workspace is the architect/orchestrator context:

- Product truth and cross-repo design live here.
- Production code belongs to `frontend/` or `backend/family-hub-api/`.
- Do not run builds, tests, or linters from the root review context.
- Read delivery-repo code and agent instructions as evidence, but keep this pass read-only.
- Treat the story/spec as proposals to verify, not facts to trust.

Use descriptive text or compact ASCII when a visual helps. Do not open a browser-based visual companion.

## Primary Review Target

- Spec: [`docs/superpowers/specs/2026-06-22-lists-family-managed-categories-design.md`](specs/2026-06-22-lists-family-managed-categories-design.md)
- Story: [`docs/product/backlog/module-foundations/lists-family-managed-categories.md`](../product/backlog/module-foundations/lists-family-managed-categories.md)
- Roadmap entry: [`docs/product/roadmap.md`](../product/roadmap.md)
- Design commit: [`8d38558`](https://github.com/joe-bor/family-hub/commit/8d38558)

The spec has been brainstormed and self-reviewed once. Your value is a genuinely fresh pass, not agreement with the prior work.

## Required Context — Read Before Reviewing

Read these sources in order:

1. Root [`AGENTS.md`](../../AGENTS.md).
2. The new story and design spec above, in full.
3. Shipped Lists story: [`docs/product/backlog/module-foundations/lists-simple-shared-checklists.md`](../product/backlog/module-foundations/lists-simple-shared-checklists.md).
4. Original Lists design: [`docs/superpowers/specs/2026-05-06-lists-simple-shared-checklists-design.md`](specs/2026-05-06-lists-simple-shared-checklists-design.md).
5. Relevant roadmap and PRD Lists/module-vocabulary sections.
6. Backend [`AGENTS.md`](../../backend/family-hub-api/AGENTS.md), then the current Lists migration, entities, repositories, services, DTOs, controller, exception mapping, auth registration seeding, and Lists tests.
7. Frontend [`AGENTS.md`](../../frontend/AGENTS.md), then current Lists types, services, hooks/query keys, validation, MSW handlers, list options, list item sheet, grouped-section builder, mobile-sheet/back-overlay behavior, offline validators, and relevant tests/E2E.
8. Recent git history affecting Lists, Flyway migrations, mobile sheets, offline reads, or release semantics.

If any required source is inaccessible, stop and report the blocker rather than reviewing from partial context.

## Approved Product Intent — Do Not Reopen Without Evidence

The review should test whether the spec implements these decisions coherently:

- Categories belong to one `family + list kind` catalog and are reused by every matching list.
- Grocery and To-do get starter categories; General starts with none.
- Starter categories become ordinary editable/deletable family data; `seeded` has no product behavior.
- General remains flat by default and can explicitly opt into grouped display.
- Creating a category never automatically enables grouped mode.
- Each item has zero or one category; multiple tags are out of scope.
- Category management lives in the current list's Options and states that changes affect all lists of that kind.
- Add/Edit Item supports inline category create-and-select.
- Category order is shared by family + kind.
- Reordering uses an explicit local `Reorder` mode with up/down controls; arrow clicks are local-only; Save sends one complete-order request; Cancel discards.
- Delete moves affected items to `Uncategorized` atomically.
- Deleting the final category also switches every matching grouped list to flat atomically; recreating a category does not regroup them.
- Empty category groups remain hidden from the checklist.
- Schema evolution uses a new Flyway migration and leaves released V12 unchanged.
- Delivery is BE-first; FE consumes only a published BE release.

You may challenge one of these decisions only if current code, database behavior, accessibility, security, or an internal contradiction makes it infeasible or unsafe. Clearly separate such evidence from personal preference.

## Required Review Passes

### 1. Product and scope coherence

Verify:

- The problem, goals, non-goals, invariants, API, UX, tests, and acceptance criteria describe the same feature.
- “Family + kind” ownership is consistently reflected everywhere.
- General's empty/flat defaults and grouped opt-in have complete behavior for create, rename, reorder, delete, and final-category deletion.
- `Uncategorized` is consistently synthetic rather than accidentally modeled as a category.
- The story is a coherent implementation unit and not an oversized initiative that needs another split.
- No adjacent feature—tags, per-list catalogs, icons/colors, offline writes, or Chores semantics—has leaked into scope.

### 2. Flyway and persistence safety

Inspect the current backend migration sequence rather than assuming `V17` is still the next available number.

Verify:

- A new migration can safely transform the exact V12 schema while V13–V16 and any newer migrations remain valid.
- Fresh-database (`V1 → latest`) and existing-database (`current → new`) paths converge.
- Dropping the General constraints and `seeded` column is ordered safely relative to JPA startup and released clients.
- The proposed case-insensitive uniqueness mechanism is valid for PostgreSQL, compatible with current test strategy, and handles existing rows.
- Name trimming plus database uniqueness closes race conditions rather than relying only on a pre-check.
- Dense `sortOrder` updates are transactionally safe.
- Composite foreign keys in `shared_list_item` do not make delete-to-null unsafe or impossible.
- Final-category deletion can update items, flatten matching lists, delete the category, compact order, and return exact counts in one transaction.
- Registration initialization cannot resurrect a family-deleted starter category.
- Existing data truly remains valid after the migration.

Use current official Flyway/PostgreSQL documentation if needed, and cite primary sources for any claim that depends on tool/database behavior.

### 3. API, security, and concurrency

Verify:

- Static `/api/lists/categories...` routes coexist safely with existing `/api/lists/{id}` routing.
- Every read/write is scoped through the authenticated family and does not leak cross-family existence or counts.
- Kind validation prevents cross-kind assignment and mutation.
- DTO shapes are explicit and internally consistent, especially:
  - embedded list-detail category option;
  - management catalog entry with `itemCount`;
  - catalog-level `groupedListCount`;
  - delete response counts;
  - reorder request and response.
- `400`, `404`, and `409` semantics fit the backend's current exception/response conventions; identify any missing exception-mapping requirement.
- Duplicate-name races reliably become `409`, not an unhandled `500`.
- Reorder's `expectedCategoryIds` comparison catches concurrent reorder/create/delete after rows are locked.
- Reorder handles empty, single-item, unchanged, duplicated, missing, foreign, and wrong-kind ID sequences.
- Lock acquisition and multi-row updates do not introduce an obvious deadlock or lost-update path.
- Create/delete/reorder maintain dense deterministic ordering under concurrency.
- Usage and grouped-list counts are defined precisely and can be queried without incorrect joins or accidental overcounting.

### 4. Frontend UX, state, and accessibility

Verify against the current components and state model:

- List Options can expose category controls for General without regressing Grocery/To-do or desktop parity.
- The proposed mobile transition from the half-height Options sheet to the full manager avoids sheet stacking and cooperates with the current hardware-back/overlay-dismiss infrastructure.
- Inline creation can preserve the React Hook Form item draft, create the category, select it, and recover cleanly from either category-create or item-save failure.
- Removing `CategoryAwareListKind`/`seeded` and admitting General categories is reflected through types, schemas, fixtures, MSW, and offline validation.
- Category mutations update/invalidate every relevant list-detail query without unnecessary or missing cache updates.
- A stale form referencing a deleted category has an implementable recovery path with current API error information.
- Delete confirmation uses preflight counts honestly while success uses authoritative response counts.
- Final-category deletion visibly reconciles grouped lists switching to flat.
- Reorder mode has a clear baseline, dirty state, Save/Cancel, close-with-unsaved confirmation, pending behavior, retry behavior, and `409` recovery.
- Create/rename/delete cannot mutate the baseline while Reorder mode is active.
- Up/down controls are keyboard accessible, screen-reader clear, correctly disabled at boundaries, and meet touch-target rules.
- Empty, loading, error, pending, and offline states are specified sufficiently.
- No new dependency or unnecessary duplicate component architecture is implied.

### 5. Compatibility and release sequencing

Verify:

- The currently released FE really remains usable when BE removes `seeded` and returns General categories.
- The new FE cannot accidentally ship against an older BE release lacking the endpoints/schema.
- Existing FE E2E release-resolution semantics support the proposed BE-first flow.
- Read-only offline persistence remains valid when category response shapes change.
- The BE work can actually proceed in parallel with current FE work without shared-root or release conflicts.
- Any shared API reference or product documentation that must change is named explicitly.

### 6. Verification and acceptance coverage

Map every non-negotiable requirement to at least one proposed BE or FE test. Check for missing coverage in:

- migration upgrade versus clean migration;
- registration starters and no General starter;
- family/kind isolation;
- duplicate-name race behavior;
- multi-list counts and propagation;
- delete-to-null and final-category flattening;
- reorder batching and concurrency conflicts;
- stale form/cache recovery;
- mobile/desktop/accessibility behavior;
- offline-read regression;
- released-contract E2E.

Flag tests that are impossible, brittle, redundant, or assigned to the wrong layer.

## Review Standards

- Verify claims against current files and git history; do not rely on memory or the spec's own assertions.
- Prefer concrete evidence with clickable absolute file links and tight line references.
- Focus on correctness, security, data integrity, contract clarity, user-visible behavior, and implementability.
- Do not report pure style preferences as findings.
- Distinguish:
  - a **spec defect** that must be corrected before planning;
  - an **implementation detail** that belongs in the later plan;
  - a **future enhancement** outside this story.
- Do not invent findings merely to appear thorough. A clean `PASS` is acceptable when supported by evidence.
- Do not silently edit the spec during review. Recommend exact wording or contract changes for the user to approve.

## Required Output Format

### 1. Verdict

Choose exactly one:

- `PASS — ready for implementation planning`
- `NEEDS REVISION — not ready for implementation planning`

### 2. Findings

List findings first, ordered by severity:

- `P0` — data loss, security breach, or fundamentally invalid architecture.
- `P1` — blocking correctness/contract gap before planning.
- `P2` — important ambiguity or missing edge case that should be resolved.
- `P3` — worthwhile clarification with low implementation risk.

For each finding include:

1. Short title.
2. Evidence with file/line links.
3. Why it matters.
4. Exact recommended spec correction.

If there are no findings, write `No actionable findings.`

### 3. Contract Coverage Matrix

For each approved product decision, mark:

- `Covered`
- `Partially covered`
- `Missing`
- `Contradictory`

Point to the relevant spec sections and tests.

### 4. Assumptions and Questions

List only unresolved items that materially affect planning. Do not repeat resolved decisions or ask preference questions with no engineering impact.

### 5. Planning Readiness Checklist

Confirm or reject:

- Product behavior is internally consistent.
- Migration is safe and implementable.
- API shapes and status semantics are complete.
- Concurrency behavior is implementable.
- FE state/cache/overlay behavior is implementable.
- Acceptance criteria map to tests.
- BE-first release sequencing is viable.
- Scope is ready for one implementation plan with BE and FE delivery issues.

End after the verdict and review. Even on `PASS`, **do not create the implementation plan in this session**. The user will start planning only after reviewing and accepting this audit.
