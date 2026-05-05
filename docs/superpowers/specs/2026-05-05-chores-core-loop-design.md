# Chores Core Loop

## Problem

Family Hub's `Chores` tab currently looks polished but still runs on placeholder sample data. The shell suggests a real family product, but this surface does not yet provide real household value.

The next product step is turning `Chores` into the first real non-calendar module with the smallest useful loop: create a chore, assign it to a family member, see it on a shared household board, and mark it done.

This story also resolves an important product-language decision: Family Hub does not have separate user identities inside the household. The family shares one account. Chores therefore belong to the family account, can be assigned to a family member for clarity, and are either done or not done without tracking who tapped completion.

## Goals

- Replace placeholder chores with persisted family-scoped data.
- Make `Chores` feel like a mobile-first family board rather than a demo list.
- Keep assignment meaningful for adults and pre-readers without inventing per-user permissions.
- Ship the smallest real FE + BE contract that can support create, complete, and delete.
- Establish the product pattern for the first module beyond Home and Calendar.

## Non-goals

- Adding separate household user identities, permissions, or audit trails.
- Tracking who completed a chore.
- Adding notes, recurrence, streaks, rewards, points, or categories.
- Building custom sorting controls, drag/reorder behavior, or dashboard integration.
- Defining the final product for `Lists`, `Meals`, or `Photos`.
- Introducing reminders or notifications in this story.

## Scope

### Included

- Persisted chores owned by the family account.
- Required single assignee chosen from existing family members.
- Optional due date.
- Shared complete / incomplete state.
- Delete action.
- Mobile-first `Chores` board made of assignee lanes.
- Dynamic progress bar in each lane header.
- Default urgency-first ordering.
- Completed chores visible at the bottom of each lane.
- Create flow via the existing full-screen mobile sheet pattern.

### Explicitly excluded

- Separate icon field or curated icon picker.
- Notes field.
- Delete-by-swipe as a required interaction pattern.
- Detail screen as a required part of the MVP loop.
- Per-lane manual sort controls or saved sort preferences.

## Product Invariants

### D1. One family account, not multiple app users

The app does not model individual signed-in household users for this module. A family uses one shared account. This is not a temporary MVP shortcut; it is the intended product model for chores.

### D2. Assignment is for responsibility, not identity

Each chore is assigned to exactly one family member so the board can answer "whose chore is this?" clearly. Assignment does not imply the system knows who interacted with the row.

### D3. Completion is shared state only

A chore is either complete or incomplete. The product does not track `completedBy`, does not infer which household member tapped the checkbox, and should not suggest that it knows.

### D4. Pre-reader support starts with visual cues, not a new icon system

Titles may include emoji such as `🗑️ Take out trash` or `🪥 Brush teeth`. This is enough for the first slice. A curated icon picker can be explored later if the household needs more structured non-reading cues.

## Decision Summary

### D5. Use assignee lanes as the primary board shape

The default mobile `Chores` experience is a vertical stack of assignee lanes. This best matches the family-board vision, keeps chores tied to a person visually, and still works for adults scanning urgency.

Each lane contains one family member's chores, with the member represented through:

- name
- existing member color / avatar cue
- a dynamic progress bar showing completion progress within that lane

The progress bar is the preferred summary signal over a numeric "2 of 5" count because it is more visual and more helpful for a pre-reader.

### D6. Show only active lanes by default

The default board only renders lanes for family members who currently have incomplete chores.

Implications:

- If a family member has no incomplete chores, they do not get a visible active lane by default.
- If the entire family has no chores at all, the screen uses a family-level empty state.
- If the family has only completed chores, the board switches to an "all caught up" state that still renders completed-only lanes for members with completed chores.

### D7. Order chores by urgency inside each lane

Inside each assignee lane, chores sort in this order:

1. overdue
2. due today
3. future dated
4. no due date
5. completed chores at the bottom of the lane

This urgency-first sort is the fixed default for MVP. Future user-controlled sorting may be explored in a later story, but no sort controls ship in this slice.

### D8. Keep completed chores visible at the bottom of each lane

Completed chores are not hidden behind a toggle. They remain visible at the bottom of the same lane in a clearly distinct visual state.

Required treatment:

- muted tone
- strikethrough or equivalent "done" treatment
- still easy to toggle back to incomplete

This keeps the family board satisfying and legible without adding separate completed-state UI.

### D9. Use the calendar-style mobile sheet for create flow

Creating a chore follows the established mobile interaction pattern already used by Calendar:

- tap `+`
- open full-screen mobile sheet
- fill the small form
- save and return to the board

This avoids inventing a new creation pattern and keeps the new module aligned with the rest of the app.

## Data Contract

### Chore model

The first shared contract should include:

- `id`
- `familyId`
- `title`
- `assignedToMemberId`
- `dueDate | null`
- `completed`
- `completedAt | null`
- `createdAt`
- `updatedAt`

Notes:

- `familyId` establishes tenancy; chores belong to the family account.
- `assignedToMemberId` is required and must reference an existing family member in the same family.
- `title` is plain text and may include emoji inline.
- `dueDate` is date-only, not date-time.
- `completedAt` exists to support completed-state handling and future history/sorting needs without adding per-user identity.

### API shape

The first API should support the full core loop:

- `GET /chores`
- `POST /chores`
- `PATCH /chores/{id}`
- `DELETE /chores/{id}`

Expected responsibilities:

- `GET /chores` returns the family's chores needed to render the board.
- `POST /chores` creates a family-owned chore with required `title` and `assignedToMemberId`, plus optional `dueDate`.
- `PATCH /chores/{id}` supports toggling completion and updating editable chore fields needed by the first slice.
- `DELETE /chores/{id}` permanently removes the chore.

The API does not expose or accept any `completedBy` concept.

## UI Contract

### Board layout

On mobile, `Chores` is a household board with:

- page title
- `+` action
- vertical list of assignee lanes

Each lane header shows:

- family member name
- member color/avatar cue
- dynamic progress bar

Each lane body shows:

- incomplete chores first
- completed chores at the bottom
- due-date emphasis when relevant

### Chore row behavior

Each row should have:

- large tap-friendly complete toggle
- title
- due-date label only when present

Visual rules:

- overdue chores are clearly emphasized
- due-today chores are clearly emphasized, but less strongly than overdue chores
- completed chores remain visible and distinct

The row does not need to show who completed the chore because that concept does not exist.

### Create flow

The create sheet includes only:

- `title` (required)
- `assignee` (required)
- `due date` (optional)

No notes field ships in this slice.

On save:

- the sheet closes
- the new chore appears immediately in the correct lane
- the chore is placed in the correct urgency order

### Delete behavior

Delete is part of the first slice, but the spec does not require swipe gestures. The default affordance should be a simple overflow/menu action on the chore row that is reachable on mobile and desktop alike.

### Empty states

Required empty-state cases:

1. No chores exist at all
   - Show a clear family-level empty state with CTA to create the first chore.

2. All chores are completed
   - Show a lightweight family-level "all caught up" message and render completed-only lanes for members with completed chores.

3. A member has no incomplete chores
   - No active lane is rendered for that member by default.

## Acceptance Criteria

- [ ] `Chores` reads from persisted backend data; generated sample chores are not used in the production path.
- [ ] The board is family-owned, with one required assignee per chore and no per-user identity model.
- [ ] The system does not track or imply `completedBy`.
- [ ] Default mobile view renders assignee lanes for members with incomplete chores.
- [ ] Each visible lane includes member identity cues plus a dynamic progress bar.
- [ ] Chores inside each lane sort by overdue, due today, future dated, no due date, then completed at the bottom.
- [ ] The family can create a chore with required title, required assignee, and optional due date via a full-screen mobile sheet.
- [ ] Emoji in titles are supported as a lightweight child-visibility cue.
- [ ] Completing or uncompleting a chore updates the UI immediately and persists successfully.
- [ ] Delete exists in the first slice and removes the chore successfully.
- [ ] Completed chores remain visible at the bottom of the relevant lane in a distinct visual state.
- [ ] Empty states are intentional for "no chores yet" and "everything completed."

## Quality Bar

- FE must stop using generated sample chores in the shipping path.
- Create, toggle, and delete should feel immediate in the UI, then persist to the backend.
- Due-date handling should be date-only and timezone-safe for normal family use.
- Mobile touch targets should remain comfortable for both adults and kids.
- The lane layout should degrade cleanly on desktop even though mobile is the designed surface.

## Testing Expectations

Implementation should cover:

- urgency ordering
- grouping by assignee
- required-field validation for create
- complete / uncomplete flow
- delete flow
- completed-at-bottom behavior
- family empty-state behavior
- all-completed state behavior

## Delivery Notes

- This is one product story with one root spec and one root implementation plan.
- Execution will likely split into two delivery issues after planning:
  - BE: chores persistence + API
  - FE: chores board + create/toggle/delete wiring against the released BE contract
- FE shipping should use a released BE contract per the root repo shipping rules.
