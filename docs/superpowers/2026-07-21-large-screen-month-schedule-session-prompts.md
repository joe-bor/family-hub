# Session Prompts: Large-Screen Calendar Month and Schedule (FE #293)

The [implementation plan](plans/2026-07-18-large-screen-calendar-month-schedule.md)
is 6,416 lines across 18 tasks and roughly 90 steps. It is correct and stays
**unrevised** — this document only slices it into six sessions, each ending at a
hard review gate.

Splitting does not weaken the plan's own sequencing rule. Execution contract
item 17 requires two phases on one worktree branch, Month before Schedule, with
Schedule blocked until the Task 12 screenshot gate passes. Sessions A–D are
Month, E–F are Schedule, and the Task 12 gate is preserved as its own session
boundary.

## Why it splits cleanly

Every task in the plan ends with its own commit step, so every task boundary is
a resumable seam. The six groups below were chosen so that each gate is
something a reviewer can actually judge, and so that risk rises gradually rather
than all at once.

| Session | Tasks | Plan lines | What the gate proves | User-visible change |
| --- | --- | --- | --- | --- |
| **A — Helpers** | 0–4 | ~944 | Layout arithmetic is correct in isolation | None |
| **B — Components** | 5–7 | ~1,085 | Chip, popover and day cell are right on their own | None |
| **C — Month wiring** | 8–10 | ~1,347 | Month is live at `lg+`; tablet path preserved | Month at `lg+` |
| **D — Month gate** | 11–12 | ~1,102 | **The plan's mandated Month phase gate** | Evidence only |
| **E — Schedule build** | 13–15 | ~1,179 | Schedule is live at `lg+`; compact path untouched | Schedule at `lg+` |
| **F — Schedule gate + PR** | 16–17 | ~594 | Full evidence mapping; draft PR opened | Evidence + PR |

Sessions A and B produce **no user-visible change at all**. That is deliberate:
they are fast, low-risk reviews of pure formulas and isolated components before
anything touches the rendered Calendar.

**Escape hatch:** Session C is the heaviest group, and Task 8 alone is 823 lines
with a 374-line keyboard-navigation step. If it runs long, cut it at the 8 | 9
boundary and run Tasks 9–10 as their own session. Do not cut inside Task 8.

## Shared facts every session needs

- **Issue:** https://github.com/joe-bor/FamilyHub/issues/293
- **Worktree:** `frontend/.worktrees/large-screen-month-schedule`
- **Branch:** `feat/large-screen-month-schedule`
- **Baseline:** frontend `origin/main` = `e74c5c737c77aa2285ac83e2c523093cea7a3216`

> **Plan header drift.** The plan's "Review baseline" block still names
> `f9dc7e8070457f965b823baee4e3746486afa438`, verified 2026-07-20. Three commits
> have landed since (#292, #296, #298). The Issue body was refreshed on
> 2026-07-21; the plan header was intentionally left alone. Task 0 branches from
> `origin/main` dynamically, so execution is unaffected — but capture every
> preservation baseline from the fresh commit and never reuse a pre-drift image.

### Before Session A

Optional but recommended — the plan mandates one isolated worktree, and the FE
checkout currently carries five stale ones from shipped stories plus an
already-merged branch:

```bash
cd frontend
git worktree list                     # expect 5 leftovers: chores, lists, recipes, meals, quick-capture
git worktree remove .worktrees/<name> # for each shipped story
git worktree prune
```

---

## Session A — Helpers (Tasks 0–4)

```
You are the frontend delivery agent for FamilyHub#293 in the joe-bor/FamilyHub
repo (local path: frontend/).

Read https://github.com/joe-bor/FamilyHub/issues/293 first, then its linked
Story, Spec and Plan. Restate the non-negotiable contract in your work log
before writing code, then continue without waiting for approval.

Use superpowers:subagent-driven-development to execute the plan task-by-task,
and superpowers:test-driven-development — Tasks 3 and 4 are strict
red-green-commit and must not be collapsed.

SCOPE: Plan Tasks 0 through 4 ONLY. Stop after Task 4's commit. Do not begin
Task 5.

Task 0 creates the worktree .worktrees/large-screen-month-schedule on branch
feat/large-screen-month-schedule from origin/main. Note that the plan header's
pinned baseline SHA is stale — origin/main is now e74c5c73. Task 0 branches
from origin/main dynamically, so just confirm HEAD matches the fresh remote.

Read the plan's "Critical environment facts" before Task 1. They are not
optional context: matchMedia is mocked false for every query, ResizeObserver is
a no-op, jsdom has no layout engine, and type errors surface only in
`npm run build` — so run the build before every commit.

GATE — do not hand off until all of these hold:
  1. `npm run lint`, `npm test -- --run` and `npm run build` all exit 0.
  2. `git diff origin/main --stat` shows changes ONLY under
     src/components/calendar/utils/, src/test/test-utils.tsx, and the
     day-rail import/test trims. No view or component file is modified.
  3. There is zero user-visible change at any viewport. Nothing rendered
     differs from origin/main.
  4. monthSlotCapacity is monotonic and
     monthSlotCapacity(MONTH_MIN_ROW_HEIGHT) >= 2, proven by unit test.
  5. planCellSlots never exceeds supplied capacity, and the capacity-2
     [blank, event, single-day] counterexample yields [blank, +2 more].
  6. February 2026 renders exactly four week rows in the matrix unit test.

REPORT: actual test counts, the commit SHA per task, and any point where a plan
snippet conflicted with the fetched baseline. Then STOP for review.
```

---

## Session B — Components (Tasks 5–7)

```
You are the frontend delivery agent for FamilyHub#293, continuing work already
in progress. Read the Issue and its linked Spec and Plan before touching code.

Resume the EXISTING worktree — do not create one and do not re-run Task 0:

  cd frontend/.worktrees/large-screen-month-schedule
  git status --short          # must be clean
  git log --oneline -1        # must be Task 4's commit
  npm test -- --run           # must be green before you start

Use superpowers:subagent-driven-development and
superpowers:test-driven-development. All three tasks are red-green-commit.

SCOPE: Plan Tasks 5, 6 and 7 ONLY — month-event-chip, month-overflow-popover,
month-day-cell. Stop after Task 7's commit. Do not begin Task 8; the grid
rewrite is the next session's work.

Carry these contract items into every component decision:
  - Chips and `+N more` are PRESENTATIONAL summaries, never controls. The
    96px-or-taller day gridcell is their single target. Chips and overflow
    text are aria-hidden.
  - The day cell is a role="gridcell" div, NEVER a <button>. No nested buttons.
  - Colour is never the only channel: every chip visibly prefixes the member's
    first name, with "Unknown" for stale data. Missing-member chips use
    bg-muted text-foreground, never white on the light muted token.
  - Weld geometry is exact: 28px slots, 9px continuation bleed per active side,
    rounded corners only at a run's true start and end.
  - Do not use opacity to carry meaning.

GATE — do not hand off until all of these hold:
  1. `npm run lint`, `npm test -- --run` and `npm run build` all exit 0.
  2. monthly-calendar.tsx is UNMODIFIED. The three components are built and
     exported but not yet consumed, so the app still renders exactly as
     origin/main at every viewport.
  3. Each component has passing tests for its accessible semantics, and the
     chip has explicit weld-geometry and missing-member fallback coverage.
  4. The barrel export in components/index.ts typechecks.

REPORT: test counts, commit SHA per task, and any baseline drift. Then STOP.
```

---

## Session C — Month wiring (Tasks 8–10)

```
You are the frontend delivery agent for FamilyHub#293, continuing work already
in progress. Read the Issue and its linked Spec and Plan before touching code.

Resume the EXISTING worktree:

  cd frontend/.worktrees/large-screen-month-schedule
  git status --short          # must be clean
  git log --oneline -1        # must be Task 7's commit
  npm test -- --run           # must be green before you start

SCOPE: Plan Tasks 8, 9 and 10 ONLY. Stop after Task 10's commit. Do not begin
Task 11; browser proof is the next session.

This is the session where Month becomes user-visible at lg+. Task 8 is the
largest task in the plan (8 steps, including a long keyboard-navigation step) —
work it step-by-step and commit at its natural end, not partway through. If you
run out of room, stop cleanly after Task 8 rather than starting Task 9 badly.

Non-negotiables that are easy to get wrong here:
  - Everything is gated at 1024px via useIsLargeScreen(). Exactly 768px is
    mobile; tablet starts at 769px. Preserve the tablet path first (Task 8
    Step 1) before adding anything.
  - Observe the weeks rowgroup through a CALLBACK ref. The grid owns only the
    weekday row and the observed role="rowgroup"; every gridcell is owned by a
    week row.
  - The grid is ONE tab stop with roving tabindex. Exactly one gridcell has
    tabindex="0". Space activates and must not scroll.
  - Arrow navigation is based on membership in the RENDERED matrix, not
    visible-month equality: March 31 to rendered April 1 must not page the
    month. Crossing the real grid edge changes month and restores focus to the
    landed-on date after rerender.
  - Focus behavior is reason-specific — Escape returns to the originating cell,
    outside-pointer dismissal leaves focus on the newly clicked target, Event
    Detail takes focus and returns it only on close, and opening Day view must
    not focus an unmounted grid.
  - The widened query range is lg+ ONLY. Query identity comes from the actual
    {startDate, endDate} parameters, never the breakpoint boolean.
  - Cached empty data is valid data. `events.length === 0` is NOT a
    cache-presence test, and no cold-cache copy may render until persisted-query
    restoration completes.
  - Pair every setViewportWidth with resetViewportWidth.

GATE — do not hand off until all of these hold:
  1. `npm run lint`, `npm test -- --run` and `npm run build` all exit 0.
  2. Component tests prove that below 1024px both the query range and the
     rendering are unchanged from origin/main.
  3. Exactly one gridcell is tabbable and no dense summary is reachable by tab.
  4. Arrow keys, Home/End and PageUp/PageDown behave per spec, with the
     March 31 to April 1 case explicitly covered.
  5. Month has loading, cached-empty, error/retry and cold-cache states.

REPORT: test counts, commit SHA per task, which contract items have test
coverage versus which are still awaiting browser proof in Session D, and any
baseline drift. Then STOP.
```

---

## Session D — Month phase gate (Tasks 11–12)

```
You are the frontend delivery agent for FamilyHub#293, continuing work already
in progress. Read the Issue and its linked Spec and Plan before touching code.

Resume the EXISTING worktree:

  cd frontend/.worktrees/large-screen-month-schedule
  git status --short          # must be clean
  git log --oneline -1        # must be Task 10's commit
  npm test -- --run           # must be green before you start

SCOPE: Plan Tasks 11 and 12 ONLY. Stop after Task 12. Do not begin Task 13.

THIS IS THE PLAN'S MANDATED HARD GATE. Execution contract item 17 forbids
starting Schedule until the Month screenshot gate passes. Task 12 Step 5 is
literally "Confirm Month is complete" — treat a failure here as a reason to fix
Month, never as a reason to proceed.

Requires Docker and the released backend. Task 11 Step 1 creates
scripts/run-calendar-e2e.sh — a fail-safe runner pinned to the released
bare-semver backend image, using `docker compose up -d --wait` with guaranteed
teardown. Deterministic recurrence fixtures must use syntax accepted by released
API v1.9.0: UNTIL, not COUNT.

Capture preservation baselines from origin/main at e74c5c73 (the plan header's
pinned SHA is stale). Never reuse a pre-drift image.

Screenshots are EVIDENCE TO INSPECT, not artifacts to generate. Task 12 Step 3
and Step 4 require you to actually review the captured matrix against the spec.
Fix real defects you find and rerun the affected gate.

GATE — do not hand off until all of these hold:
  1. Twelve baseline/current pairs are BYTE-IDENTICAL against origin/main:
     Month, Schedule, Week and Day at 375x812, 768x1024 and 769x1024. Print and
     record both SHA-256 values for every pair.
  2. Browser geometry proves adjacent multi-day segments meet within 1px,
     Sunday/Saturday do not bleed outward, and the document has no horizontal
     overflow.
  3. At 1440x900 the final week row reaches the available bottom with no dead
     space; the 1024x768 six-week case respects MONTH_MIN_ROW_HEIGHT with no
     permanent overflow defect.
  4. Measured capacity is recorded as OBSERVED counts, not hardcoded expected
     values, and the five-week April 2026 row at 1440x900 has greater capacity
     than the six-week August 2026 row at 1024x768.
  5. Offline cold-start polls the exact persisted query entries for expanded
     April 2026 (2026-03-29 to 2026-05-02) and grid-aligned February 2026
     (2026-02-01 to 2026-02-28), including cached empty arrays, with no
     "isn't cached" flash during restoration.
  6. Contrast checks composite translucent colours through their ancestor
     layers before calculating the ratio.
  7. Docker teardown succeeded and no container is left running.

REPORT: every SHA-256 pair, the observed capacity counts, actual Playwright and
Vitest counts, screenshots you inspected and any defect you fixed as a result.
Then STOP. Schedule does not begin until this gate is reviewed and accepted.
```

---

## Session E — Schedule build (Tasks 13–15)

```
You are the frontend delivery agent for FamilyHub#293, continuing work already
in progress. Read the Issue and its linked Spec and Plan before touching code.

Resume the EXISTING worktree:

  cd frontend/.worktrees/large-screen-month-schedule
  git status --short          # must be clean
  git log --oneline -1        # must be Task 12's final commit
  npm test -- --run           # must be green before you start

CONFIRM FIRST: the Month phase gate (Task 12) passed and was reviewed. If it did
not, stop and say so — contract item 17 forbids starting Schedule.

SCOPE: Plan Tasks 13, 14 and 15 ONLY. Stop after Task 15's commit. Do not begin
Task 16.

The central risk: ScheduleCalendar is a SINGLE component shared with mobile, and
it is the smart default for first-time mobile users. Its compact rendering is
release-critical. Every change is gated on useIsLargeScreen(), and the compact
branch changes ONLY by mechanical extraction — no behavioural edits.

Non-negotiables:
  - No fixed outer width cap through 2560px. Day groups and event rows use the
    available width; only event TEXT gets a readable internal measure
    (max-width: 72ch).
  - Event titles 20px, secondary metadata at least 14px, rows at least 56px.
  - Fix the coloured left border in the lg+ branch ONLY, using the member hex.
    The compact branch deliberately keeps its shipped grey border.
  - Reason over RENDERED offsets 0..13, not the query-only fifteenth boundary
    day.
  - Empty states must distinguish: no family members; filters hiding every
    rendered event; a genuinely empty 14-day window; and the defensive
    zero-selection branch, which uses the PRD's exact copy
    "Select at least one profile to view events" and stays app-unreachable.
  - Consecutive event-free days collapse into ONE non-focusable gap row.
  - Past-day de-emphasis uses gutter/border styling, never opacity, and retains
    AA contrast.
  - Preserve the 14-day window, 7-day paging step, 50% overlap and the constant
    "Upcoming" toolbar label. These are known limitations, not bugs.

GATE — do not hand off until all of these hold:
  1. `npm run lint`, `npm test -- --run` and `npm run build` all exit 0.
  2. The compact Schedule branch differs from origin/main only by mechanical
     extraction, with no behavioural change.
  3. All four empty-state reasons have component-test evidence, including the
     defensive zero-member and zero-selection branches.
  4. Schedule has loading, empty, filtered-empty, error/retry and populated
     states at lg+.

REPORT: test counts, commit SHA per task, and an explicit statement of what
changed in the compact branch and why it is mechanical. Then STOP.
```

---

## Session F — Schedule gate and PR (Tasks 16–17)

```
You are the frontend delivery agent for FamilyHub#293, completing the story.
Read the Issue and its linked Spec and Plan before touching code.

Resume the EXISTING worktree:

  cd frontend/.worktrees/large-screen-month-schedule
  git status --short          # must be clean
  git log --oneline -1        # must be Task 15's commit
  npm test -- --run           # must be green before you start

SCOPE: Plan Tasks 16 and 17 — Schedule E2E, mobile parity proof, the final
screenshot gate, full verification, and the draft PR.

Requires Docker and the released backend via scripts/run-calendar-e2e.sh.
Capture all preservation baselines from origin/main at e74c5c73.

Use superpowers:verification-before-completion — evidence before assertions,
always. Then superpowers:requesting-code-review before opening the PR.

GATE — do not open the PR until all of these hold:
  1. The twelve preservation pairs are RE-RUN after Schedule and are still
     byte-identical. Print both SHA-256 values for every pair.
  2. Dedicated empty AND populated mobile Schedule captures are byte-identical
     to origin/main, with fixed clock, matched font loading and reduced motion.
  3. Browser evidence shows the 1920 day group wider than 1700px and the 2560
     day group wider than 2300px, while text stays internally constrained.
  4. Sticky proof FIRST asserts the surface is genuinely scrollable, then
     asserts the gutter sticks within its own section.
  5. The 14-day window, 7-day Previous/Next step and 50% overlap are proven
     unchanged by a boundary-seeded Playwright test.
  6. `npm run lint`, `npm test -- --run`, `npm run build` and `npm run test:e2e`
     all pass through the released-backend runner, with ACTUAL counts recorded.
  7. Docker teardown succeeded.
  8. The PR work log maps EVERY execution-contract item to a test name,
     screenshot filename, hash pair or other concrete evidence. No item may be
     supported only by prose.

Open the PR as a DRAFT with "Closes #293", links to Story/Spec/Plan, the
contract-evidence mapping, test counts, hash pairs and screenshot attachments.
After every evidence file is attached, remove the ignored local
.calendar-evidence directory and confirm the worktree is otherwise clean.

REPORT: the PR URL, all hash pairs, actual test counts, and any contract item
you could not evidence. Then STOP.
```

---

## Handoff discipline

Each session ends with a **STOP**. That is the review gate — the point where you
inspect the work before the next session inherits it. Sessions B–F all begin by
asserting the previous session's commit is HEAD and the suite is green, so a
failed handoff surfaces immediately rather than compounding.

If any session reports drift between a plan snippet and the fetched baseline,
resolve it before starting the next one. Plan step 6 of the critical environment
facts is explicit: preserve the spec and Issue requirement rather than weakening
it to match drifted code.
