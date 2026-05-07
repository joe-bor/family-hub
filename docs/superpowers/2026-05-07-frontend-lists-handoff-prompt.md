# Frontend Handoff Prompt: Lists Simple Shared Checklists

You are the frontend delivery agent working in the `joe-bor/FamilyHub` repo.

Start by reading the issue body and all linked root artifacts before touching code. Treat the issue as the execution entrypoint, then restate the non-negotiable contract in your work log before implementation.

## Required Artifacts

- Frontend issue: [FamilyHub#163](https://github.com/joe-bor/FamilyHub/issues/163)
- Story: [lists-simple-shared-checklists.md](https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/module-foundations/lists-simple-shared-checklists.md)
- Spec: [2026-05-06-lists-simple-shared-checklists-design.md](https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/specs/2026-05-06-lists-simple-shared-checklists-design.md)
- Reference plan: [2026-05-06-lists-simple-shared-checklists.md](https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-05-06-lists-simple-shared-checklists.md)
- Released backend contract: [family-hub-api v1.3.0](https://github.com/joe-bor/family-hub-api/releases/tag/v1.3.0)

## Working Rules

- Work only in the frontend repo, not in the root orchestration repo.
- Follow `frontend/AGENTS.md` and the issue contract exactly.
- Leverage the Superpowers skills from the start.
- Use the skills that fit the moment rather than skipping the workflow:
  - `superpowers:using-superpowers`
  - `superpowers:test-driven-development`
  - `superpowers:systematic-debugging` if anything behaves unexpectedly
  - `superpowers:verification-before-completion`
  - `superpowers:requesting-code-review` before final handoff if available in your environment
  - `superpowers:finishing-a-development-branch` when the implementation is complete
- Use atomic commits so the PR is easy to review. Prefer a few small commits grouped by delivery slice rather than one large commit.
- Keep commit messages compatible with release-please semantics where appropriate.
- Consume only the released backend contract from `family-hub-api` `v1.3.0`. Do not wire frontend behavior against unreleased backend changes.

## Non-Negotiable Contract

- Replace placeholder or sample list content in the production path with persisted `/api/lists` data.
- Open `Lists` as a family-owned hub of multiple lists with a `New List` action and scan-friendly summaries.
- Support creating a list with required `name` and required `kind` of `grocery`, `to-do`, or `general`.
- Honor seeded read-only categories for `grocery` and `to-do`; keep `general` uncategorized by default.
- Support grouped and flat category display modes for category-aware lists, and do not show category controls for `general`.
- Support add, edit, delete, check, and uncheck item flows with immediate UI feedback.
- Honor the family-wide completed-items default plus per-list completed visibility override.
- Whenever completed items are visible, they must stay checked, muted, and struck through.
- Include a manual `Remove all completed` action for each list.
- Provide intentional empty states for both `no lists yet` and `list has no items`.
- Do not reintroduce sample data, assignees, due dates, reminders, recurrence, or any `Chores` semantics.

## Delivery Shape

- Follow the reference plan’s frontend slices in order unless you discover a concrete repo-local reason to adjust.
- Keep the data layer, hub/create flow, detail interactions, and live verification separable in the commit history.
- Add or update tests as each slice lands so the PR shows evidence alongside implementation.

## End State

- End by opening a frontend pull request against `joe-bor/FamilyHub`.
- The PR should use `Closes #163`.
- The final handoff should include a checklist mapping each non-negotiable requirement to code and tests.
