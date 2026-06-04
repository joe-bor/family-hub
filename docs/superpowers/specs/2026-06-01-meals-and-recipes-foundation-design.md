# Meals And Recipes Foundation Design

**Date:** 2026-06-01
**Status:** FE handoff ready
**Owner:** Family Hub product / planning
**Related backlog epic:** `docs/product/backlog/module-foundations/module-surface-foundations.md`

## Summary

`Meals` is the next recommended module foundation after `Chores` and `Lists`, but the weekly planning experience depends on a real reusable recipe source. The right delivery order is therefore:

1. ship a lightweight but real top-level `Recipes` module
2. build `Meals` on top of that recipe foundation

The product split is:

- `Recipes` = reusable library of things the household may cook again
- `Meals` = week-ahead planning board for what the household will eat

This design keeps the current module-foundations focus intact: family coordination first, not pantry management, shopping automation, or a full recipe-platform buildout.

## Handoff Status

As of 2026-06-04:

- Backend `Recipes` issue [family-hub-api#52](https://github.com/joe-bor/family-hub-api/issues/52) and PR [family-hub-api#53](https://github.com/joe-bor/family-hub-api/pull/53) are merged.
- Backend `Meals` issue [family-hub-api#55](https://github.com/joe-bor/family-hub-api/issues/55) and PR [family-hub-api#56](https://github.com/joe-bor/family-hub-api/pull/56) are merged.
- Backend release [family-hub-api v1.5.0](https://github.com/joe-bor/family-hub-api/releases/tag/v1.5.0) is the released contract for frontend real-backend verification.
- Frontend `Recipes` issue [FamilyHub#183](https://github.com/joe-bor/FamilyHub/issues/183) should land first.
- Frontend `Meals` issue [FamilyHub#184](https://github.com/joe-bor/FamilyHub/issues/184) should follow after the Recipes frontend handoff state exists.

## Goals

- Turn the placeholder `Meals` destination into a real family coordination surface
- Introduce `Recipes` as a first-class module instead of hiding recipe data inside `Meals`
- Preserve a clear boundary between reusable recipe content and week-specific meal planning
- Support a visual kitchen-tablet experience without breaking the mobile-first product direction
- Keep scope intentionally lighter than a dedicated meal-tracking or grocery-planning app

## Non-Goals

- Pantry or inventory management
- Grocery list generation in v1
- Meal lifecycle/status tracking such as cooked, skipped, or done
- Per-person meal assignment
- Recurring meal rules or pattern automation
- Copy-day or copy-week planning in the first slice
- Advanced recipe taxonomy or heavy filtering systems

## Product Shape

### Module responsibilities

| Module | Responsibility | v1 purpose |
|---|---|---|
| `Recipes` | Reusable content library | Store recipes that can be created manually or imported from a URL and reused later |
| `Meals` | Weekly planning board | Plan breakfast, lunch, and dinner for the household across a browsable week |

### Core relationship

- `Recipes` ships first because `Meals` needs a real reusable content source.
- `Meals` can still support quick non-recipe entries, but recipe-backed planning is central to the experience.
- Adding a recipe to `Meals` creates a snapshot of that recipe at placement time.
- Later edits to the base recipe do not rewrite previously planned meals.
- New placements always use the latest current recipe state.

### Household and time model

- `Meals` is household-level planning, not person-level planning.
- Breakfast, lunch, and dinner are equal planning surfaces.
- Users can browse week by week.
- Current and future weeks are editable.
- Past weeks are reviewable in the frontend, but not editable through the normal UI.
- `Meals` has no status model, in v1 or later by current product intent.

## Recipes Module

### Role

`Recipes` is a real top-level module, even in its first lightweight version. This avoids building recipe behavior inside `Meals` and then later extracting it into a separate surface.

### Recipes home

- The default home screen is the saved recipe library.
- The default ordering is recently added or recently updated.
- The main entry action is `Add recipe`.
- If the library is empty, the empty state should clearly invite the user to add the first recipe.

### Responsive presentation

- Mobile uses a compact, still-visual presentation.
- Larger screens can expand into a richer card or grid layout.
- The default experience should still feel image-forward and browseable, not like a plain admin list.

### Add recipe flow

The primary add flow branches into:

- `Create manually`
- `Import from URL`

URL import should only appear inside the add flow. It should not compete visually with the recipe library home screen.

### Manual recipe creation

Required:

- name

Optional:

- image
- ingredients
- instructions
- note
- source URL
- tags

The UX should encourage ingredients and instructions so the library remains useful for actual cooking, but the model should allow fast capture when the user wants to save something incomplete and refine it later.

### URL import

- User pastes a recipe URL.
- The system attempts a scrape.
- On success, the recipe is auto-saved immediately.
- On failure, the user sees a clear failure message.
- Cleanup or improvement happens through normal recipe editing later.

The import flow does not require a separate pre-save review step in v1.

URL import must be bounded server-side because it fetches user-provided URLs:

- allow only `http` and `https`
- reject loopback, private, link-local, and otherwise non-public network targets before fetching
- use short connection/read timeouts
- follow only a small bounded redirect chain and re-validate each redirect target
- cap the response body size before parsing
- fail closed with a user-facing import failure, not a partial recipe, when these checks fail

### Recipe detail

Recipe detail should be cook-first, especially on mobile:

1. image
2. title
3. ingredients
4. instructions

Planning actions such as `Add to Meals`, `Favorite`, and `Edit` should be easy to reach without overtaking the cooking content.

`Favorite` is an editable toggle on saved recipes, not only a create-time field. `Edit` updates the saved recipe record through the real recipe editing flow; it is not a meal-planning edit surface.

### Organization and discovery

v1 supports:

- search
- favorites
- light tags
- light filters

Rules:

- favorites use a simple star
- favorites have practical value by surfacing higher in pickers and search
- tags are lightweight and future-friendly, not a heavy taxonomy system
- filters should stay simple and mobile-friendly

### Add to Meals

From recipe detail, `Add to Meals` should:

- default to the current or currently visible week
- let the user choose day and meal slot
- allow switching weeks if needed
- prompt with the normal collision choices if the target slot already has a primary meal

This action creates a meal snapshot from the recipe as it exists at the moment of placement.

## Meals Module

### Role

`Meals` is the household's weekly planning surface. It should feel tangible, visual, and quick to assemble, while remaining clearly separate from recipe management.

### Core board model

Kitchen/tablet vision:

- columns = days of the week
- rows = breakfast, lunch, dinner

Mobile-first implementation:

- keep the same weekly-plan mental model
- allow a more breathable day-card presentation on phone
- let tablet and larger screens open into the more iconic weekly grid

Interaction mechanics can differ by device, but the model should stay consistent.

### Week navigation

The board header should stay minimal:

- previous week
- current week label
- next week

The board itself should do the visual work. There should be no board-level filtering in v1.

### Empty slots

Empty slots should actively invite planning rather than disappear into the background.

Each empty slot should clearly communicate that the user can add a meal there.

### Creating a meal

Primary creation starts by tapping an empty slot.

That opens a composer with:

- a search/type field at the top
- quick-pick sections underneath

Quick-pick sections should include:

- recent recipes
- favorite recipes
- entry point into the full recipe library

### Composer behavior

- Typing fuzzy-matches against saved recipes.
- If the typed text does not match a recipe, the user can still use it immediately as a planned meal.
- The UI should prioritize the quick non-recipe path while also offering `Create recipe from this`.

For quick non-recipe meals:

Required:

- name

Optional:

- image
- note

The user-facing copy should feel like a lightweight quick-meal action rather than exposing internal terminology such as `ad-hoc`.

## Planned Meal Model

### Meal block shape

The manipulable object on the board is the meal block.

- creation begins with tapping an empty slot
- saving creates a meal block in that slot
- once created, the block feels tangible and movable

The board should feel like a planning board of meal cards rather than a spreadsheet of editable cells.

### Primary meal plus extras

Each slot supports:

- one primary meal
- optional extras or sides

Extras can come from:

- saved recipes
- quick non-recipe meals

Visual intent:

- one primary only = simple square-style card
- primary plus extras = bento-style composition

Design for up to 2 visible extras. If more exist, summarize overflow with a compact signal such as `+2 more`.

### Editing a planned meal

Planned meal editing is planning-focused, not recipe-focused.

Supported actions:

- replace meal
- manage extras or sides
- duplicate meal block
- remove meal
- optional meal-specific note support

Recipe editing belongs to `Recipes`, not to the meal block editor.

If a meal is recipe-backed, tapping it from `Meals` should open the actual recipe detail rather than an intermediate duplicated card view.

## Motion And Reuse

### Move behavior

- meal blocks are movable
- drag means move, not copy

Device expectations:

- tablet and larger screens should support direct drag-and-drop well
- mobile should support a hybrid model with drag where it works naturally and reliable action-based move flows as fallback

### Duplicate behavior

Duplication is explicit, not implicit.

v1 includes:

- duplicate meal block

v1 does not include:

- duplicate day
- copy week
- recurring meal rules

Duplicating a meal block copies the full planned unit:

- primary meal snapshot
- extras or sides
- meal-specific note if present

### Collision handling

If any action would put content into a slot that already has a primary meal, prompt the user with:

- `Replace primary`
- `Add as extra`
- `Cancel`

This applies to move, duplicate, and recipe placement from `Add to Meals`. Do not silently replace existing planned content.

## Cross-Module Flows

### From Meals into Recipes

- user can choose `Create recipe from this` while planning
- quick recipe creation can begin inline from `Meals`
- deeper recipe editing should navigate into the real `Recipes` flow

### From Recipes into Meals

- recipe detail exposes `Add to Meals`
- placement defaults to current or visible week
- user chooses day and meal slot
- placement creates a snapshot

### Return behavior

If create/import starts from `Recipes`:

- success lands in recipe detail

If create/import starts from `Meals`:

- success returns to planning with the new recipe available for placement

## Cross-Module Contract

The implementation should treat `Recipes` and `Meals` as separate but intentionally connected modules. The contract between them needs to be explicit before implementation starts so work can move in two focused tracks without hidden assumptions.

Wire values crossing the frontend/backend boundary are lowercase strings. Backend enums may keep uppercase Java constants, but JSON serialization/deserialization should expose lowercase values such as `breakfast`, `lunch`, `dinner`, `recipe`, `quick`, `replace_primary`, and `add_as_extra`.

### Recipes owns

- reusable recipe records
- recipe detail viewing
- recipe editing
- URL import behavior
- favorites, tags, and light filters
- the source data used when creating a recipe-backed meal snapshot

### Meals owns

- week-by-week household planning
- slot-level composition across breakfast, lunch, and dinner
- quick non-recipe meal creation
- move, duplicate, replace, and extras behavior
- review-only treatment of past weeks

### Recipes to Meals contract

`Meals` depends on `Recipes` for these supported behaviors:

- search and selection of saved recipes during meal planning
- favorite recipes surfacing prominently in meal pickers
- recipe detail exposing `Add to Meals`
- recipe placement producing a stable meal snapshot
- later recipe edits not mutating already planned meals

### Meals to Recipes contract

`Recipes` should not depend on `Meals` for its core usefulness, but it must support these planning handoffs:

- a recipe can be added to a meal slot from recipe detail
- quick recipe creation can begin from `Meals`
- deeper recipe editing navigates into the real `Recipes` experience
- successful create/import started from `Meals` returns the user to planning context

## Data And State Rules

### Recipe-backed planned meals

When a saved recipe is placed into `Meals`, the planned item should snapshot the fields needed for stable board display and future week review.

For v1, historical review means stable meal-board fidelity, not a frozen duplicate of the full recipe detail. A recipe-backed planned meal snapshots:

- source type
- recipe id when the source recipe still exists
- title
- image URL
- meal-specific note

Tapping a recipe-backed planned meal opens the live recipe detail when the recipe still exists. If the base recipe is later edited, the board keeps the original snapshot title/image/note, while recipe detail shows the current recipe. If the base recipe is deleted, the board still shows the snapshot and the live recipe-detail action is unavailable.

### Quick meals

Quick meals can exist without a backing recipe.

They are valid first-class planned meals and should not feel like broken or second-rate placeholders in the UI.

### Past weeks

Past weeks should remain browsable for review, but normal editing affordances should not be exposed there. The product should bias toward present and upcoming planning rather than retroactive maintenance.

In v1 this is a frontend product guardrail, not a server-side authorization rule. Backend write endpoints may still accept past-week writes for backfill, recovery, or future administrative workflows unless a later story adds authoritative server enforcement.

## Integration Notes

- Grocery/list generation is out of scope for the first slice.
- The data model should leave room for future recipe-to-list workflows.
- `Meals` should not absorb shopping semantics now just because that linkage may come later.

## Recommended Delivery Sequence

1. `Recipes` module foundation
2. `Meals` planning surface built on top of `Recipes`
3. follow-on backlog items for broader meal reuse or grocery/list integration if still desired

## Implementation Planning Strategy

Implementation planning should be intentionally split into at least two plans rather than one combined plan.

Each delivery plan should also map into repo-owned execution issues:

- a backend issue in `backend/family-hub-api` for schema, services, controllers, and backend tests
- a frontend issue in `frontend` for types, hooks, UI, mocks, and frontend tests
- any root-docs issue only for spec/plan updates, not production implementation

Frontend issues that depend on real backend behavior must state the released backend version they consume. They should not depend on unreleased backend `main`.

Current issue mapping:

- `Recipes` backend: [family-hub-api#52](https://github.com/joe-bor/family-hub-api/issues/52), [PR #53](https://github.com/joe-bor/family-hub-api/pull/53), released in [v1.5.0](https://github.com/joe-bor/family-hub-api/releases/tag/v1.5.0)
- `Recipes` frontend: [FamilyHub#183](https://github.com/joe-bor/FamilyHub/issues/183)
- `Meals` backend: [family-hub-api#55](https://github.com/joe-bor/family-hub-api/issues/55), [PR #56](https://github.com/joe-bor/family-hub-api/pull/56), released in [v1.5.0](https://github.com/joe-bor/family-hub-api/releases/tag/v1.5.0)
- `Meals` frontend: [FamilyHub#184](https://github.com/joe-bor/FamilyHub/issues/184)

### Plan 1: Recipes

This plan should focus on the standalone `Recipes` module while explicitly naming the downstream `Meals` dependency. It should cover:

- recipe library home
- manual recipe creation
- URL import
- recipe detail
- favorites, tags, filters
- `Add to Meals` entry point
- the data and API surface `Meals` will later consume

Expected workflow:

1. implement `Recipes`
2. review
3. apply review changes
4. update docs if implementation changed the contract

### Plan 2: Meals

This plan should focus on `Meals` as a separate implementation track that consumes the already-defined recipe contract. It should cover:

- weekly planning board
- composer flows
- quick meals
- recipe-backed placement
- snapshot behavior
- move, duplicate, and collision handling
- mobile and larger-screen interaction differences

Expected workflow:

1. implement `Meals`
2. review
3. apply review changes
4. update docs if implementation changed the contract

### Planning guardrail

Both plans should reference the existence of the sibling module and the shared contract, but each plan should stay focused on its own owned surface. If implementation reveals a contract gap between `Recipes` and `Meals`, update the spec before continuing so the handoff remains explicit.

## Acceptance Criteria

### Recipes

- A standalone `Recipes` module exists as a top-level surface
- A user can browse a recipe library ordered by recent activity
- A user can manually create a recipe with only a name required
- A user can import a recipe from a URL and have successful imports auto-save
- A user can open a recipe detail screen that is useful for cooking
- A user can edit a saved recipe from recipe detail
- A user can toggle a saved recipe as favorite from recipe detail
- A user can favorite recipes and see those favorites prioritized in selection flows
- A user can use light search, tags, and filters to find recipes
- A user can add a recipe to a meal plan from recipe detail

### Meals

- A user can browse week by week
- A user can plan breakfast, lunch, and dinner with equal first-class treatment
- A user can tap an empty slot to create a planned meal
- A user can choose between saved recipes and quick non-recipe meals
- A planned meal can include one primary item and optional extras
- A planned meal created from a recipe snapshots board-display state at placement time
- A user can move a planned meal and explicitly duplicate a planned meal
- Slot collisions prompt the user instead of silently overwriting content
- Past weeks are reviewable but not normally editable

## Risks And Guardrails

- Do not let `Meals` quietly become a grocery or pantry feature set.
- Do not let `Recipes` bloat into a full recipe-platform scope before `Meals` exists.
- Do not collapse recipe editing and meal editing into the same UI surface.
- Keep the board visual and tactile, but avoid overcommitting to drag-only interactions on mobile.

## Follow-On Backlog Candidates

- duplicate day
- copy week
- recipe-to-list ingredient handoff
- richer recipe tagging or categorization
- more advanced kitchen-display behaviors
