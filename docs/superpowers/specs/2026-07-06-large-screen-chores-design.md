# Large-Screen Chores - Design Spec

**Date:** 2026-07-06
**Status:** Draft for review
**Story:** `docs/product/backlog/large-screen-ux/large-screen-chores.md`
**Depends on:** `2026-07-06-large-screen-foundations-design.md`
**Scope:** Chores module on large screens (1024px+). Mobile Chores unchanged.

## 1. Context

Chores is the best-adapted module today: on desktop it already renders Today /
This Week / This Month as three scope columns inside a `max-w-6xl` container.
The structure is right; the surface just has not had a large-screen polish
pass. This spec is deliberately a refinement, not a redesign.

## 2. Design Goal

The chores board is the household's "what needs doing" wall, and for kids it is
primarily "what do I do now?" On a shared large screen, Today should carry the
visual weight, a child should find their own name and color in seconds, and
checking something off should be a satisfying, easy tap from across the table.

## 3. Chosen Direction: Refine, Don't Restructure

- **Let the board breathe.** Widen the container beyond `max-w-6xl` toward the
  content strategy in the foundations spec, with comfortable per-column widths
  instead of three cramped equal columns.
- **Weight Today as the primary column.** Slightly wider, stronger heading
  treatment, and first in reading order. This Week and This Month read as
  supporting context. No layout mode change: still three columns.
- **Bigger checkoff targets.** The complete/uncomplete control grows to a
  generous touch size on large screens; the whole row remains tappable where
  it already is.
- **Member identity forward.** Assignee group headers get the avatar + name +
  member color treatment consistent with the Calendar Day lane headers, so
  "find your name" works the same way across modules.
- **Foundations chrome.** Slim header and single toolbar row (title, scope
  context, add-chore action).

## 4. Alternatives Considered

### Member lanes (columns per person) - Rejected

Mirroring the new Calendar Day view: one column per family member with their
chores across all scopes. Attractive for the kid-facing question, but scope
columns answer the household's actual operating question ("what is due when"),
per-person focus already exists inside each column via assignee grouping, and
switching the board's axis is a redesign this pass does not need. If dogfooding
shows the per-person question dominates, a member-lane mode is a clean
follow-up story.

## 5. Accessibility

- Checkoff controls keep accessible completed/not-completed state and grow to
  at least 44px (larger preferred on `lg+`).
- Member identity in group headers is name + avatar + color; color is never
  the only identifier.
- Column headings are proper landmarks/headings so scope structure is
  navigable by assistive tech.

## 6. Acceptance Criteria

- [ ] Board uses the widened container with Today visually primary at 1024px
      and 1440px.
- [ ] Checkoff targets measure at least 44px on large screens.
- [ ] Assignee group headers show avatar + name + member color consistent
      with Calendar Day lanes.
- [ ] Single toolbar row per foundations; no stacked chrome rows.
- [ ] All existing chore flows (create, edit, complete, uncomplete, archive,
      stale-period recovery) behave unchanged.
- [ ] Mobile Chores (scope switcher + single column) is visually and
      behaviorally unchanged.
- [ ] Screenshot review covers: busy board, chores-all-done, empty board, and
      a family with 5+ members.

## 7. Out of Scope

- Member-lane board mode.
- New chore features (streaks, rewards, reminders).
- Mobile behavior.
- Backend changes.
