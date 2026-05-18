# Recurring Chores — Present-Focused Redesign

## Problem

The shipped `Chores` module is a one-off assigned task board with optional due dates. That no longer matches the intended product. The household vision is present-focused recurring routines:

- what routines matter now
- what has already been done in the current timeframe
- what is still left in the rest of the current timeframe

This redesign intentionally replaces the shipped chores contract. It is not a compatibility extension of the current one-off chores model.

## Product Boundary

Family Hub's module vocabulary for MVP becomes:

- `Chores` = recurring household routines only
- `Lists` = ad-hoc shared checklists only
- `Calendar` = scheduled commitments only

Implications:

- `Chores` no longer owns one-off assigned tasks.
- `Lists` remains intentionally lighter than chores and should not absorb assignment, cadence, or responsibility semantics in this story.
- `Calendar` may share time-boundary policy with chores, but chores do not become calendar events or calendar instances.

## Goals

- Replace the one-off chore model with recurring routines only.
- Keep the chores surface fully present-focused for MVP.
- Let the family see both remaining and completed routines within the current scope.
- Use one backend model that supports both mobile single-scope rendering and larger-screen multi-scope rendering.
- Mirror calendar time policy exactly for MVP so week boundaries and date windows remain consistent across modules.

## Non-goals

- One-off assigned chores
- Due dates inside a cadence
- History browsing, missed-history UI, or streaks
- Carrying incomplete routines into the next period
- Rewards, points, reminders, notifications, or gamification
- Calendar rendering of chores as scheduled events
- Re-opening the `Lists` contract

## Product Invariants

### D1. One family account, not multiple app users

The app still uses a single shared family account. Chores are assigned to a family member for responsibility clarity, not for permissions or actor identity.

### D2. Cadence determines scope

Each chore template belongs to exactly one recurring cadence:

- `DAILY`
- `WEEKLY`
- `MONTHLY`

Each cadence appears in exactly one board scope:

- `DAILY` -> `TODAY`
- `WEEKLY` -> `THIS_WEEK`
- `MONTHLY` -> `THIS_MONTH`

No routine appears in more than one scope.

### D3. Completion belongs to the current period, not the template

A recurring routine is never globally complete. It is only complete for the current day, week, or month period.

### D4. Periods reset cleanly

When a new day, week, or month begins, the board resets for the new current period. Incomplete routines do not carry forward.

### D5. MVP remains fully present-focused

The product does not expose prior-period history, missed periods, or retrospective reporting in this slice, even if completion rows exist under the hood.

### D6. Completed routines stay visible, but secondary

Within the current scope, incomplete routines appear first. Completed routines remain visible underneath in a clearly muted state.

### D7. Chores mirror calendar time policy exactly for MVP

Chores use the same effective time rules as calendar:

- same family-local timezone policy
- same week-start definition
- same interpretation of "today", "this week", and "this month"

For MVP, chores should mirror the current calendar week behavior rather than introducing separate chores-only time preferences.

## UX Contract

## Mobile

On mobile, `Chores` uses a scope switcher:

- `Day`
- `Week`
- `Month`

Only one scope is visible at a time.

Within the chosen scope:

- assignee groups are stacked vertically
- each group shows summary progress for that scope
- incomplete routines appear first
- completed routines remain visible beneath incomplete routines in a muted visual treatment

## Larger screens

On larger screens, `Chores` shows three parallel columns:

- `Today`
- `This Week`
- `This Month`

Each column contains assignee groups for that scope. This is the desktop/tablet expression of the same underlying board contract as mobile, not a separate product model.

## Assignee grouping

The board remains assignee-oriented inside each scope. The family should be able to answer:

- what routines belong to this timeframe
- whose routine is it
- what is still remaining

The API should express this as assignee groups rather than UI-specific "lane" wording.

## Empty states

Required cases:

1. no recurring chores exist at all
2. a scope has no routines for the family
3. a scope has routines but everything in that scope is complete

The third case should still show completed routines in that scope rather than switching to a history view.

## Activation Rules

Templates need an explicit activation boundary so mid-period behavior is predictable.

### D8. `activeFrom` is required

Each template has an `activeFrom` date. A template only participates in periods on or after `activeFrom`.

Rules:

- a template created during an already-open period is active immediately for that current period
- the current period is not split; the routine simply becomes outstanding from creation onward
- archived templates stop appearing in the board immediately after archival

This is intentionally simple for MVP. We do not model "start next month instead" or deferred activation scheduling in this slice.

## Member lifecycle

### D9. Active assigned templates block member deletion

If a family member still has active chore templates assigned to them, member deletion should be blocked until those templates are reassigned or archived.

This is the simplest safe MVP rule and avoids orphaned recurring responsibility definitions.

## Data Model

## `chore_template`

The standing recurring definition.

Fields:

- `id`
- `familyId`
- `assignedToMemberId`
- `title`
- `cadence` (`DAILY | WEEKLY | MONTHLY`)
- `activeFrom`
- `archivedAt | null`
- `createdAt`
- `updatedAt`

Notes:

- `title` is plain text and may include emoji inline.
- There is no `dueDate`.
- There is no template-level `completed`.
- There is no one-off mode.

## `chore_period_completion`

The completion record for one template in one specific period.

Fields:

- `id`
- `choreTemplateId`
- `periodStartDate`
- `periodEndDate`
- `completedAt`

Constraint:

- unique on `choreTemplateId + periodStartDate + periodEndDate`

This lets the system mark a routine complete once per current period and naturally reset in the next period without scheduled reset jobs.

## Derived board item

The board item is a computed read model, not a persistent entity.

Fields:

- `templateId`
- `title`
- `cadence`
- `assignedToMemberId`
- `completed`
- `completedAt | null`

`completed` is derived by checking whether a matching `chore_period_completion` exists for the current scope period.

## API Contract

## Read

`GET /chores/board`

Returns all three scopes in one payload for both mobile and larger screens.

```ts
type ChoresBoardResponse = ApiResponse<{
  timezone: string;
  today: ChoreScopeBoard;
  thisWeek: ChoreScopeBoard;
  thisMonth: ChoreScopeBoard;
}>;

interface ChoreScopeBoard {
  scope: "TODAY" | "THIS_WEEK" | "THIS_MONTH";
  periodStartDate: string; // yyyy-MM-dd
  periodEndDate: string;   // yyyy-MM-dd
  summary: {
    total: number;
    completed: number;
    remaining: number;
  };
  assignees: ChoreAssigneeGroup[];
}

interface ChoreAssigneeGroup {
  member: {
    id: string;
    name: string;
    color: string;
  };
  summary: {
    total: number;
    completed: number;
    remaining: number;
  };
  chores: ChoreBoardItem[];
}

interface ChoreBoardItem {
  templateId: string;
  title: string;
  cadence: "DAILY" | "WEEKLY" | "MONTHLY";
  assignedToMemberId: string;
  completed: boolean;
  completedAt: string | null;
}
```

Notes:

- The response returns all scopes every time.
- Mobile uses one scope at a time from this payload.
- Larger screens render all three scopes from this payload.
- The API returns assignee groups, not UI-named lanes.

## Template mutations

### Create

`POST /chores/templates`

Request:

- `title`
- `assignedToMemberId`
- `cadence`
- `activeFrom`

### Update

`PATCH /chores/templates/{id}`

Editable fields:

- `title`
- `assignedToMemberId`
- `cadence`
- `activeFrom`
- `archived`

For MVP, archive is preferred over delete for routine definitions. Hard delete is not required in this slice.

## Current-period completion mutations

### Complete current period

`PUT /chores/templates/{id}/current-period-completion`

### Uncomplete current period

`DELETE /chores/templates/{id}/current-period-completion`

These writes must be stale-period safe.

Request body:

```ts
{
  scope: "TODAY" | "THIS_WEEK" | "THIS_MONTH";
  periodStartDate: string; // yyyy-MM-dd
}
```

Rules:

- The client sends the period it believes it is mutating.
- The server recomputes the current valid period for the template.
- If the request period does not match the current valid period, the server rejects the write as stale.

This prevents writing the wrong completion row when a screen stays open across midnight, week rollover, or month-end.

Suggested mutation response:

```ts
type ChoreCurrentPeriodStateResponse = ApiResponse<{
  scope: "TODAY" | "THIS_WEEK" | "THIS_MONTH";
  periodStartDate: string;
  periodEndDate: string;
  item: ChoreBoardItem;
}>;
```

## Ordering Rules

Within each assignee group:

1. incomplete routines
2. completed routines

Inside each completion state bucket, use one stable deterministic order for MVP:

1. `createdAt` ascending
2. `title` ascending as a secondary tie-breaker

This replaces the old due-date urgency ordering.

## Validation Rules

- `title` required, max 100 characters
- `assignedToMemberId` required and must belong to the authenticated family
- `cadence` required and must be one of `DAILY`, `WEEKLY`, `MONTHLY`
- `activeFrom` required and date-only
- active assigned templates block member deletion
- current-period completion requests must match the server-computed valid current period

## Side Effects On Existing Product Docs

This story replaces the current shipped chores story semantics. The following docs will need intentional alignment once implementation planning starts:

- `docs/product/prd.md`
- `docs/product/backlog/module-foundations/chores-core-loop.md`
- `docs/product/backlog/module-foundations/module-surface-foundations.md`

Key alignment changes:

- chores are no longer one-off due-date tasks
- recurring chores are now the MVP chores contract
- `Show Completed` toggle language should be removed in favor of visible-but-secondary completed routines inside the active scope

## Acceptance Criteria

- [ ] `Chores` only supports recurring templates with `DAILY`, `WEEKLY`, or `MONTHLY` cadence
- [ ] One-off assigned chores are removed from the chores contract
- [ ] `Lists` remains the ad-hoc one-off checklist surface
- [ ] `GET /chores/board` returns `today`, `thisWeek`, and `thisMonth` in one payload
- [ ] mobile shows one scope at a time via `Day | Week | Month`
- [ ] larger screens show all three scopes in parallel columns
- [ ] each routine appears in exactly one scope based on cadence
- [ ] weekly and monthly routines can be completed anytime in the current period
- [ ] incomplete routines do not carry forward into the next period
- [ ] completed routines remain visible but clearly secondary inside the current scope
- [ ] no history surface or missed-history UI ships in MVP
- [ ] chores mirrors calendar time policy exactly for MVP
- [ ] current-period completion writes are stale-period safe
- [ ] member deletion is blocked when active recurring chore templates are still assigned

## Delivery Notes

- This is a contract reset, not a compatibility migration.
- Because there are no real users to preserve, FE and BE can replace the current chores model aggressively rather than dual-supporting old and new behavior.
- Implementation should happen in the owning FE and BE repos after a plan is written from this spec.
