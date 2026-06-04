# Frontend Handoff Prompt: Recipes And Meals Foundations

You are the frontend delivery agent working in the `joe-bor/FamilyHub` repo.

Start by reading the issue body and all linked root artifacts before touching code. Treat the issue as the execution entrypoint, then restate the non-negotiable contract in your work log before implementation.

## Required Artifacts

- Recipes frontend issue: [FamilyHub#183](https://github.com/joe-bor/FamilyHub/issues/183)
- Meals frontend issue: [FamilyHub#184](https://github.com/joe-bor/FamilyHub/issues/184)
- Story: [module-surface-foundations.md](https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/module-foundations/module-surface-foundations.md)
- Spec: [2026-06-01-meals-and-recipes-foundation-design.md](https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/specs/2026-06-01-meals-and-recipes-foundation-design.md)
- Recipes reference plan: [2026-06-01-recipes-module-foundation.md](https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-06-01-recipes-module-foundation.md)
- Meals reference plan: [2026-06-01-meals-weekly-planning.md](https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-06-01-meals-weekly-planning.md)
- Released backend contract: [family-hub-api v1.5.0](https://github.com/joe-bor/family-hub-api/releases/tag/v1.5.0)
- Backend Recipes delivery: [family-hub-api#52](https://github.com/joe-bor/family-hub-api/issues/52), [PR #53](https://github.com/joe-bor/family-hub-api/pull/53)
- Backend Meals delivery: [family-hub-api#55](https://github.com/joe-bor/family-hub-api/issues/55), [PR #56](https://github.com/joe-bor/family-hub-api/pull/56)

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
- Use atomic commits so each PR is easy to review.
- Keep commit messages compatible with release-please semantics where appropriate.
- Consume only the released backend contract from `family-hub-api` `v1.5.0` or newer. Do not wire frontend behavior against unreleased backend changes.

## Delivery Order

1. Implement [FamilyHub#183](https://github.com/joe-bor/FamilyHub/issues/183) first: standalone `Recipes` module, recipe API client state, recipe UX, and shared `Recipes <-> Meals` handoff state.
2. Implement [FamilyHub#184](https://github.com/joe-bor/FamilyHub/issues/184) second: API-backed `Meals` board, composer, grid/editor behavior, collision prompts, and consumption of the Recipes handoff state.

Do not start Meals by redesigning the Recipes handoff. If the shared app-store draft contract is missing or differs from the issue/spec, stop and report the root-doc drift before continuing.

## Non-Negotiable Contract

### Recipes

- `Recipes` is a standalone top-level module, not hidden inside `Meals`.
- Manual recipe creation requires only a nonblank title.
- URL import lives inside the add flow and successful imports auto-save.
- Recipe detail is cook-first: image, title, ingredients, instructions.
- Favorite and edit actions work on saved recipes.
- Tags and filters stay lightweight.
- `Add to Meals` creates shared handoff state without building the Meals board.
- The reverse `Meals -> Recipes -> back to Meals` draft flow exists for later Meals planning.

### Meals

- `Meals` is household-level weekly planning, not per-person assignment.
- Breakfast, lunch, and dinner are equal first-class slots.
- Empty slots invite adding a meal.
- The composer supports saved recipes, recent/favorite recipes, quick meals, and `Create recipe from this`.
- Recipe-backed planned meals use board-display snapshots and can open the live recipe detail when available.
- Move and duplicate are distinct.
- Slot collisions prompt `Replace primary`, `Add as extra`, or `Cancel`.
- Past weeks are reviewable but not normally editable in the frontend.
- There is no cooked/skipped/done status model.

## End State

- Open a frontend PR for Recipes with `Closes #183`.
- Open a frontend PR for Meals with `Closes #184` after Recipes handoff state exists.
- Each PR final handoff should include a checklist mapping every non-negotiable requirement to code and tests.
