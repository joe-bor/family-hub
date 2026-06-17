# Frontend Handoff Prompt: Native-Feel Interaction Polish

You are the frontend delivery agent working in the `joe-bor/FamilyHub` repo (local path: `frontend/`). Your job is to implement **FamilyHub#229** exactly per its linked Plan and Spec.

Start by reading the issue body and every linked artifact before touching code. Treat the issue as the execution entrypoint. Restate the non-negotiable contract in your work log, then continue without waiting for approval.

## Required artifacts

- **Execution issue:** https://github.com/joe-bor/FamilyHub/issues/229
- **Plan** (implement task-by-task, TDD): https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-06-16-native-feel-interaction-polish.md
  - local (workspace): `../docs/superpowers/plans/2026-06-16-native-feel-interaction-polish.md`
- **Spec** (design rationale, decisions, verified FE anchors): https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/specs/2026-06-16-native-feel-interaction-polish-design.md
  - local: `../docs/superpowers/specs/2026-06-16-native-feel-interaction-polish-design.md`
- **Story** (product context): https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/mobile-ux/native-feel-interaction-polish.md
  - local: `../docs/product/backlog/mobile-ux/native-feel-interaction-polish.md`

The **Plan** is the source of truth for execution: it has the full TDD task breakdown (tasks 1–8) with exact file paths, test code, implementation code, and per-task commits. The **Spec** explains *why* and lists the verified FE integration anchors.

## Use the Superpowers skills from the start

- `superpowers:using-superpowers` — orient at session start.
- `superpowers:subagent-driven-development` — **REQUIRED**: execute the Plan task-by-task with a fresh subagent per task and review between tasks. (If you prefer inline batch execution instead, use `superpowers:executing-plans`.)
- `superpowers:test-driven-development` — the Plan is already red→green→commit per task; honor it.
- `superpowers:using-git-worktrees` — optional, for an isolated workspace.
- `superpowers:systematic-debugging` — if anything behaves unexpectedly.
- `superpowers:verification-before-completion` — run lint + the full suite and confirm output before claiming done.
- `superpowers:requesting-code-review` — before opening the PR.
- `superpowers:finishing-a-development-branch` — to wrap up into the PR.

## Working rules

- Work **only** in the frontend repo. The root repo (`joe-bor/family-hub`) artifacts are **read-only inputs** — do not modify them.
- Follow `frontend/AGENTS.md` exactly (test gotchas, `time-utils`, store reset, E2E helpers).
- **Branch from the latest `main`:**
  ```bash
  git checkout main && git pull
  git checkout -b feat/native-feel-interaction-polish
  ```
- **Atomic conventional commits** — the Plan defines one commit per task (`feat(motion): …`, `feat(ui): …`). Keep them release-please-compatible (`feat:` → minor bump); use regular merge-friendly commits.
- **No new dependency.** Do not add `framer-motion`/`motion`. Reuse `tailwindcss-animate` + the Web Animations API (confirmed absent in the Spec).

## Non-negotiable contract (restate in your work log before coding)

- Two primitives only:
  - `usePressable()` — scale+tint press visual + the single `onPointerDown` **haptic seam** (a future story plugs haptics in here; do not scatter the logic).
  - `ScreenTransition` — Web Animations API **enter-transition** (`fade` | `slide` + `direction`).
- Differentiated transitions:
  - **Fade-through** on module switch — wrap `renderModule(activeModule)` in `src/App.tsx`.
  - **Shared-axis slide** on list/recipe detail — keyed on `selectedListId` / `selectedRecipeId`, `direction` forward (open) / back (close).
- **Press feedback on every tappable** (Button + raw nav/sidebar/row buttons) via `usePressable`.
- **Reduced-motion split:** CSS `motion-safe:` gates the press scale (the tint stays); `ScreenTransition` short-circuits to an instant cut under reduced motion.
- `transform`/`opacity` only, 60fps on the Galaxy S10. **Enter-transition only** (no exit+enter cross-dissolve). Do **not** touch the existing sheet/dialog/sidebar motion.

## End state

- All Plan tasks (1–8) complete; `npm run lint && npm test -- --run` green; manual device pass on Android Chrome (transitions + press feedback + reduced-motion behaviour).
- Open a PR with **`Closes #229`** whose description includes a checklist mapping **every** non-negotiable above to the code and tests that satisfy it.

## Stop conditions

Stop and report instead of improvising if: a linked artifact is inaccessible, or the current FE code no longer matches the anchors the Spec lists under "Current FE state" (e.g., `renderModule`, `selectedListId`/`selectedRecipeId`, the `Button` primitive). Confirm the contract still holds before proceeding.
