# Home Dashboard Redesign — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the mobile home screen's 2-column module grid with the "The Now" moment-aware dashboard from the design spec.

**Architecture:** A composition of pure derivation logic (hero-state machine, relative-time formatting), data hooks (`useDashboardEvents`, `useHeroState`), and presentational components (`DashboardHeader`, `MemberChipRow`, `HeroCard`, `TodayList`, `ComingUp`). The existing `HomeDashboard` becomes a viewport-gated container in the current FE app shell — mobile renders the new dashboard when `activeModule === null`, while non-mobile keeps the current "null -> calendar" behavior.

**Tech Stack:** React 19, TypeScript, Vite, TanStack Query, Tailwind v4 (oklch CSS vars), Vitest, Playwright, Biome, date-fns. No new dependencies.

**Spec:** `docs/superpowers/specs/2026-04-25-home-dashboard-redesign-design.md`

**Repo:** All work in `frontend/` (separate git repo). Plan path is in root repo for cross-repo reasoning; implementation lives in the FE repo.

---

## Pre-flight

Before starting:

- [ ] Read `docs/superpowers/specs/2026-04-25-home-dashboard-redesign-design.md` end-to-end. Every implementation decision must trace back to it.
- [ ] Read `frontend/CLAUDE.md` — pay attention to date/time pitfalls (`@/lib/time-utils`), test gotchas (gcTime, store reset, async form fields), and z-index hierarchy.
- [ ] Confirm `frontend/` working tree is clean and up to date with `origin/main`.
- [ ] Confirm `mobile-persistent-bottom-nav` is already merged on FE `main`. If it is not, stop and implement that story first; this plan assumes nav-based mobile module switching already exists.
- [ ] Create a feature branch in the FE repo: `git checkout -b feat/home-dashboard-redesign`
- [ ] **Do not** introduce new design tokens. All colors, spacing, and type sizes come from existing Tailwind utilities and CSS variables in `src/index.css`.

## File structure

New files (all under `frontend/src/`):

```
components/home/
├── home-dashboard.tsx                    # MODIFIED — viewport-gated container
├── home-dashboard.test.tsx               # MODIFIED — integration tests
├── index.ts                              # MODIFIED — barrel
├── components/
│   ├── index.ts                          # NEW
│   ├── dashboard-header.tsx              # NEW
│   ├── dashboard-header.test.tsx         # NEW
│   ├── member-chip-row.tsx               # NEW
│   ├── member-chip-row.test.tsx          # NEW
│   ├── hero-card.tsx                     # NEW
│   ├── hero-card.test.tsx                # NEW
│   ├── today-list.tsx                    # NEW
│   ├── today-list.test.tsx               # NEW
│   ├── coming-up.tsx                     # NEW
│   ├── coming-up.test.tsx                # NEW
├── hooks/
│   ├── index.ts                          # NEW
│   ├── use-dashboard-events.ts           # NEW
│   ├── use-dashboard-events.test.tsx     # NEW
│   ├── use-hero-state.ts                 # NEW
│   └── use-hero-state.test.tsx           # NEW
└── lib/
    ├── hero-state.ts                     # NEW (pure)
    ├── hero-state.test.ts                # NEW (pure)
    ├── relative-time.ts                  # NEW (pure)
    └── relative-time.test.ts             # NEW (pure)
```

Modified files outside `components/home/`:

```
e2e/mobile-home-dashboard.spec.ts         # NEW Playwright spec
```

Each task below produces one or more focused commits using conventional-commit format. Use **`feat(home):`** for net-new behavior, **`refactor(home):`** for the gating refactor, **`test(home):`** for test-only commits, and **`docs(home):`** for inline docs / story link updates.

---

## Task 1 — Pure hero-state machine

**Files:**
- Create: `src/components/home/lib/hero-state.ts`
- Test: `src/components/home/lib/hero-state.test.ts`

The state machine from spec §4.2, as a pure function with no React, no date-fns, no I/O. Discriminated union output.

- [ ] **Step 1: Write the failing test**

```typescript
// hero-state.test.ts
import { describe, it, expect } from "vitest";
import { deriveHeroState } from "./hero-state";
import type { CalendarEvent } from "@/lib/types";

const at = (h: number, m = 0) =>
  new Date(2026, 3, 25, h, m, 0); // 2026-04-25 local

const event = (overrides: Partial<CalendarEvent>): CalendarEvent => ({
  id: "e1",
  title: "Test",
  date: new Date(2026, 3, 25),
  startTime: "10:00",
  endTime: "11:00",
  memberId: "m1",
  isAllDay: false,
  location: "Gym",
  source: "NATIVE",
  ...overrides,
});

describe("deriveHeroState", () => {
  it("returns ALL_CLEAR_TODAY when no events today", () => {
    expect(deriveHeroState({ todayEvents: [], now: at(9) }))
      .toEqual({ kind: "ALL_CLEAR_TODAY" });
  });

  it("returns RIGHT_NOW for an in-progress timed event", () => {
    const e = event({ startTime: "09:00", endTime: "10:00" });
    expect(deriveHeroState({ todayEvents: [e], now: at(9, 30) }))
      .toEqual({ kind: "RIGHT_NOW", event: e });
  });

  it("returns UP_NEXT for the next future timed event", () => {
    const past = event({ id: "p", startTime: "08:00", endTime: "09:00" });
    const future = event({ id: "f", startTime: "10:00", endTime: "11:00" });
    expect(deriveHeroState({ todayEvents: [past, future], now: at(9, 30) }))
      .toEqual({ kind: "UP_NEXT", event: future });
  });

  it("returns ALL_DAY_ONLY when only all-day events today", () => {
    const ad = event({ isAllDay: true });
    expect(deriveHeroState({ todayEvents: [ad], now: at(12) }))
      .toEqual({ kind: "ALL_DAY_ONLY", event: ad });
  });

  it("returns REST_OF_DAY_CLEAR when today had timed events but all are past", () => {
    const past = event({ startTime: "08:00", endTime: "09:00" });
    expect(deriveHeroState({ todayEvents: [past], now: at(12) }))
      .toEqual({ kind: "REST_OF_DAY_CLEAR" });
  });

  it("ignores all-day events for RIGHT_NOW / UP_NEXT", () => {
    const ad = event({ id: "ad", isAllDay: true });
    const next = event({ id: "n", startTime: "14:00", endTime: "15:00" });
    expect(deriveHeroState({ todayEvents: [ad, next], now: at(12) }))
      .toEqual({ kind: "UP_NEXT", event: next });
  });
});
```

- [ ] **Step 2: Run test, verify it fails**

```bash
cd frontend && npx vitest run src/components/home/lib/hero-state.test.ts
```

Expected: all assertions fail with "deriveHeroState is not a function".

- [ ] **Step 3: Implement minimum to pass**

```typescript
// hero-state.ts
import type { CalendarEvent } from "@/lib/types";

export type HeroState =
  | { kind: "RIGHT_NOW"; event: CalendarEvent }
  | { kind: "UP_NEXT"; event: CalendarEvent }
  | { kind: "ALL_DAY_ONLY"; event: CalendarEvent }
  | { kind: "REST_OF_DAY_CLEAR" }
  | { kind: "ALL_CLEAR_TODAY" };

export interface DeriveHeroStateInput {
  todayEvents: CalendarEvent[];
  now: Date;
}

function eventStart(e: CalendarEvent): Date {
  const [h, m] = e.startTime.split(":").map(Number);
  const d = new Date(e.date);
  d.setHours(h, m, 0, 0);
  return d;
}

function eventEnd(e: CalendarEvent): Date {
  const [h, m] = e.endTime.split(":").map(Number);
  const d = new Date(e.date);
  d.setHours(h, m, 0, 0);
  return d;
}

export function deriveHeroState({ todayEvents, now }: DeriveHeroStateInput): HeroState {
  const timed = todayEvents.filter((e) => !e.isAllDay);
  const allDay = todayEvents.filter((e) => e.isAllDay);

  const inProgress = timed.find((e) => eventStart(e) <= now && now < eventEnd(e));
  if (inProgress) return { kind: "RIGHT_NOW", event: inProgress };

  const next = timed
    .filter((e) => eventStart(e) > now)
    .sort((a, b) => +eventStart(a) - +eventStart(b))[0];
  if (next) return { kind: "UP_NEXT", event: next };

  if (timed.length === 0 && allDay.length > 0) {
    return { kind: "ALL_DAY_ONLY", event: allDay[0] };
  }

  if (timed.length > 0) return { kind: "REST_OF_DAY_CLEAR" };

  return { kind: "ALL_CLEAR_TODAY" };
}
```

- [ ] **Step 4: Run tests, verify they pass**

```bash
npx vitest run src/components/home/lib/hero-state.test.ts
```

Expected: all green.

- [ ] **Step 5: Commit**

```bash
git add src/components/home/lib/hero-state.ts src/components/home/lib/hero-state.test.ts
git commit -m "feat(home): add hero-state derivation"
```

---

## Task 2 — Relative-time formatting

**Files:**
- Create: `src/components/home/lib/relative-time.ts`
- Test: `src/components/home/lib/relative-time.test.ts`

Pure formatter producing "in 38 min", "in 2 hrs", "at 3:00 PM", "ends in 22 min". Built on date-fns (already a dep). Do **not** use `Date.toLocaleTimeString` for the absolute case — use date-fns `format` for stability.

- [ ] **Step 1: Write tests for `formatRelativeStart` and `formatRemainingEnd`**

Cover: 0–60 min → "in N min", 1–6 hrs → "in N hrs", >6 hrs → "at H:MM AM/PM" (12-hour, per PRD §13 default), and edge cases at boundaries (59 min → "in 59 min", 61 min → "in 1 hr").

- [ ] **Step 2: Run tests, verify failure**
- [ ] **Step 3: Implement using date-fns `differenceInMinutes` / `format`**
- [ ] **Step 4: Run tests, verify pass**
- [ ] **Step 5: Commit** — `feat(home): add relative-time formatters`

---

## Task 3 — `useDashboardEvents` hook

**Files:**
- Create: `src/components/home/hooks/use-dashboard-events.ts`
- Test: `src/components/home/hooks/use-dashboard-events.test.tsx`

Composes the existing `useCalendarEvents` from `@/api/hooks/use-calendar.ts`. Returns `{ today, comingUp, isLoading, isError }` where:

- `today` = events where `date` is today (local TZ — use `formatLocalDate` from `@/lib/time-utils`)
- `comingUp` = events with start in `(endOfToday, endOfToday+2 days)`, sorted by start, sliced to 3
- Multi-day events appear in `today` if today is in `[start, end]`

Accept an optional `memberFocusId: string | null` parameter; when set, filter both arrays to events containing that member.

- [ ] **Step 1: Write hook tests using `renderHook` from `@testing-library/react`**

Use the project's existing test utilities. Seed the query client with mock event data; assert the hook returns the correct partition.

⚠️ Use `gcTime: Infinity` on the test QueryClient (per `frontend/CLAUDE.md` — default `gcTime: 0` GCs cache before assertions).

- [ ] **Step 2: Run, verify failure**
- [ ] **Step 3: Implement the hook**

Use `useCalendarEvents({ from, to })` with a single query covering both ranges (avoid double fetching). Derive the `today` / `comingUp` split with `useMemo`.

- [ ] **Step 4: Run, verify pass**
- [ ] **Step 5: Commit** — `feat(home): add useDashboardEvents hook`

---

## Task 4 — `useHeroState` hook

**Files:**
- Create: `src/components/home/hooks/use-hero-state.ts`
- Test: `src/components/home/hooks/use-hero-state.test.tsx`

Wraps `deriveHeroState`. Adds:
- 30-second interval that triggers a re-derivation
- `visibilitychange → visible` listener that triggers re-derivation
- Returns memoized `HeroState`

The hook accepts `todayEvents: CalendarEvent[]` and returns `HeroState`. The clock is internal but injectable via an optional `nowProvider?: () => Date` for tests.

- [ ] **Step 1: Test that the hook**
  - returns the correct initial state for given events + clock
  - re-derives when the interval ticks (use `vi.useFakeTimers()` and advance 30s)
  - re-derives when `visibilitychange` fires while document is visible
  - cleans up interval and listener on unmount

- [ ] **Step 2: Verify failure**
- [ ] **Step 3: Implement using `useState` + `useEffect`**
- [ ] **Step 4: Verify pass**
- [ ] **Step 5: Commit** — `feat(home): add useHeroState hook`

---

## Task 5 — `DashboardHeader` component

**Files:**
- Create: `src/components/home/components/dashboard-header.tsx`
- Test: `src/components/home/components/dashboard-header.test.tsx`

Renders a single line: `<greeting>, <family name>` on the left and formatted date on the right. Greeting derives from current hour (`< 12` morning, `< 17` afternoon, else evening). On narrow viewports (≤360px), the date may wrap below — use `flex-wrap` rather than truncation.

This component is intentionally pure and dashboard-local. It does **not** render Google sync status, retry controls, or connection CTAs in this story.

- [ ] **Step 1: Write component tests** — render morning/afternoon/evening variants, assert text content, and verify the narrow-layout wrapping classes are present.
- [ ] **Step 2: Verify failure**
- [ ] **Step 3: Implement** using existing Tailwind utilities and `cn()` from `@/lib/utils`. No new tokens.
- [ ] **Step 4: Verify pass**
- [ ] **Step 5: Commit** — `feat(home): add DashboardHeader`

---

## Task 6 — `MemberChipRow` component

**Files:**
- Create: `src/components/home/components/member-chip-row.tsx`
- Test: `src/components/home/components/member-chip-row.test.tsx`

Props: `members: FamilyMember[]`, `focusedId: string | null`, `onFocusChange: (id: string | null) => void`.

Renders a horizontally scrollable row (`overflow-x-auto`) of 36px circular avatars. Each chip:
- Background: member color (use the existing color token / value already on `FamilyMember` — do not redefine)
- Initials in white, semibold
- Hit area 44×44 (chip is 36px visual; 4px padding all sides on the button)
- Selected: outer ring `ring-2 ring-[member-color]/80` + `scale-[1.04]`
- When another chip is focused: this chip → `opacity-60`
- `aria-label="Focus on <name>'s events"`, `aria-pressed={focusedId === member.id}`
- Tap toggles focus: if currently focused → call `onFocusChange(null)`; else → call `onFocusChange(member.id)`

- [ ] **Step 1: Tests** — render, simulate clicks, assert callback args; assert aria-pressed; assert focused-vs-others opacity classes.
- [ ] **Step 2: Verify failure**
- [ ] **Step 3: Implement**
- [ ] **Step 4: Verify pass**
- [ ] **Step 5: Commit** — `feat(home): add MemberChipRow`

---

## Task 7 — `HeroCard` component

**Files:**
- Create: `src/components/home/components/hero-card.tsx`
- Test: `src/components/home/components/hero-card.test.tsx`

Props: `state: HeroState`, `member?: FamilyMember`, `now: Date`, `onTap?: () => void`.

Renders the hero per spec §5.2:
- Card chrome: hairline border at low opacity + soft drop shadow (`shadow-md` or equivalent existing utility, no new shadow tokens)
- Padding: 24–28px (`p-6` or `p-7`)
- 4px leading accent bar in member color (only for event states; absent for clear states)
- Time/relative line above title (use `formatRelativeStart` / `formatRemainingEnd` from Task 2)
- Title 24–28px Nunito semibold (`text-2xl font-semibold` or `text-3xl font-semibold` — choose existing scale)
- Location/notes below title at ~60% opacity
- `RIGHT_NOW` only: small breathing dot next to time. Animation: a CSS keyframe with 2s cycle, 0.6 → 1.0 → 0.6 opacity. Define a local `@keyframes home-breathing-pulse` in `src/index.css` if no existing animation covers this, and gate it inside `@media (prefers-reduced-motion: no-preference)`.

States:
- `RIGHT_NOW` → "Now · ends in 22 min" + title + accent
- `UP_NEXT` → "Up next · in 38 min" / "at 3:00 PM" + title + accent
- `ALL_DAY_ONLY` → "Today" + title + accent
- `REST_OF_DAY_CLEAR` → quiet copy "All clear for the rest of today" + soft glyph (lucide `Sparkles` or `Sun`); no accent bar
- `ALL_CLEAR_TODAY` → "Nothing on the calendar today" + glyph; no accent bar

`<section aria-label={...}>` reflecting current state (e.g., `"Right now: Soccer practice, ends in 22 minutes"`).

`onTap` only fires for event states. Empty states are no-op visually but should still render the `<section>`.

- [ ] **Step 1: Tests for each of 5 states** — assert correct text, aria-label, accent bar presence, breathing-pulse element only for `RIGHT_NOW`.
- [ ] **Step 2: Verify failure**
- [ ] **Step 3: Implement**
- [ ] **Step 4: Verify pass**
- [ ] **Step 5: Commit** — `feat(home): add HeroCard`

---

## Task 8 — `TodayList` component

**Files:**
- Create: `src/components/home/components/today-list.tsx`
- Test: `src/components/home/components/today-list.test.tsx`

Props: `events: CalendarEvent[]`, `members: FamilyMember[]`, `excludeId?: string`, `onSelect: (event: CalendarEvent) => void`.

- All-day events render at the top as a thin pinned strip (color dot + title, no time)
- Timed events render below, sorted by start time
- `excludeId` removes the hero's subject from the list (do not render it twice)
- Each row: time + title + member dot + truncated location (1 line, `text-ellipsis`)
- Multi-day affix: events spanning multiple days show `→ ends Sat` (when start === today) or `from Mon →` (when end === today)
- Empty case: render nothing — the parent decides what to render in place
- Each row is a `<button>` with min-height 44px and `onClick={() => onSelect(event)}`

- [ ] **Step 1: Tests** — all-day pinning, exclude-id removal, multi-day affix variants, sort order, empty case.
- [ ] **Step 2: Verify failure**
- [ ] **Step 3: Implement**
- [ ] **Step 4: Verify pass**
- [ ] **Step 5: Commit** — `feat(home): add TodayList`

---

## Task 9 — `ComingUp` component

**Files:**
- Create: `src/components/home/components/coming-up.tsx`
- Test: `src/components/home/components/coming-up.test.tsx`

Props: `events: CalendarEvent[]`, `members: FamilyMember[]`, `onSelect: (event: CalendarEvent) => void`.

- If `events.length === 0`, return `null`
- Otherwise: a low-opacity `"Coming up"` heading + up to 3 single-line rows
- Each row: relative date label ("Tomorrow", "Sun") + time + title + member dot
- Use `format(date, "EEE")` from date-fns for non-tomorrow days
- Same row tap behavior as TodayList

- [ ] **Step 1: Tests** — null when empty, max 3 rendered, "Tomorrow" vs weekday labels, click callback.
- [ ] **Step 2: Verify failure**
- [ ] **Step 3: Implement**
- [ ] **Step 4: Verify pass**
- [ ] **Step 5: Commit** — `feat(home): add ComingUp`

---

## Task 10 — Dashboard barrel and hooks barrel

**Files:**
- Create: `src/components/home/components/index.ts`
- Create: `src/components/home/hooks/index.ts`
- Modify: `src/components/home/index.ts`

Export everything new from the per-folder barrels. Keep `home-dashboard` exported from the top-level barrel for unchanged consumers.

- [ ] **Step 1: Write barrels**
- [ ] **Step 2: Verify build** — `npm run build`
- [ ] **Step 3: Commit** — `chore(home): add dashboard barrels`

---

## Task 11 — Compose into `HomeDashboard` in the current FE shell

**Files:**
- Modify: `src/components/home/home-dashboard.tsx`
- Modify: `src/components/home/home-dashboard.test.tsx`
- Reuse from `src/components/calendar/components/`: `add-event-button.tsx`, `event-form-modal.tsx`, `event-detail-modal.tsx` (or their mobile-specific equivalents if current mobile behavior requires them)
- Reuse hooks from `@/api`: `useFamilyName`, `useFamilyMembers`, `useCreateEvent`, `useUpdateEvent`, `useDeleteEvent`

This is the integration step. The current FE shell already treats `activeModule === null` as the mobile home surface and redirects non-mobile `null` to Calendar in `App.tsx`. Do **not** add route semantics and do **not** add a second desktop dashboard path here.

After this task:
- On mobile, `HomeDashboard` renders the new dashboard surface.
- Desktop/non-mobile behavior remains unchanged because `App.tsx` already redirects `activeModule === null` to Calendar.
- No dashboard-specific Google connect / sync UI is introduced. Empty states remain the normal calm hero states.

Composition sketch:

```tsx
const [focusedMemberId, setFocusedMemberId] = useState<string | null>(null);
const familyName = useFamilyName();
const members = useFamilyMembers();
const { today, comingUp, isLoading, isError } = useDashboardEvents({ memberFocusId: focusedMemberId });
const heroState = useHeroState(today);
const focusedMember = focusedMemberId ? members.find((m) => m.id === focusedMemberId) : undefined;
const [selectedEvent, setSelectedEvent] = useState<CalendarEvent | null>(null);
const [isDetailOpen, setIsDetailOpen] = useState(false);
const [isAddOpen, setIsAddOpen] = useState(false);

return (
  <div className="flex-1 overflow-y-auto pb-[calc(env(safe-area-inset-bottom)+72px)]">
    <DashboardHeader familyName={familyName} now={now} />
    <MemberChipRow members={members} focusedId={focusedMemberId} onFocusChange={setFocusedMemberId} />
    <HeroCard state={heroState} member={focusedMember} now={now} onTap={...} />
    <TodayList events={today} members={members} excludeId={heroEventId} onSelect={...} />
    <ComingUp events={comingUp} members={members} onSelect={...} />
    <AddEventButton onClick={() => setIsAddOpen(true)} />
  </div>
);
```

The bottom inset (`pb-[calc(env(safe-area-inset-bottom)+72px)]`) reserves space for the persistent bottom nav. Because nav is a prerequisite, this spacing should be tuned against the real nav implementation, not treated as hypothetical.

`now` should be created once at top of component and passed down — do not create new `Date()` in children. Recompute every 30s via the same mechanism `useHeroState` uses (export `useNow` helper from `use-hero-state.ts`, or co-locate).

The "+" floating action button: reuse the existing `AddEventButton` from calendar rather than creating a dashboard-only FAB.

- [ ] **Step 1: Write integration tests**
  - With mobile shell conditions satisfied, renders DashboardHeader, MemberChipRow, HeroCard, TodayList, and ComingUp from seeded data.
  - With no visible events, renders the calm empty hero state instead of any Google onboarding CTA.
  - Member-chip focus filters TodayList rows.
  - Tapping "+" opens the reused add-event modal flow.
- [ ] **Step 2: Verify failures**
- [ ] **Step 3: Implement the gating + composition**
- [ ] **Step 4: Verify all tests pass**
- [ ] **Step 5: Run full unit suite** — `npm run test -- --run`
- [ ] **Step 6: Commit** — `feat(home): wire mobile home dashboard`

---

## Task 12 — Reduced-motion + a11y verification

**Files:**
- Modify: `src/index.css` (only if a new keyframe was added in Task 7)
- Modify: any component test that needs a `prefers-reduced-motion` assertion

Confirm the breathing pulse (Task 7) and the empty-state breathing fade (if implemented as a CSS animation in HeroCard) are both inside `@media (prefers-reduced-motion: no-preference)` blocks. Confirm chip-focus scale is gated similarly (the ring is fine to keep).

A11y sweep:
- All buttons have accessible names (chips, hero card, list rows, FAB)
- Hero `<section aria-label>` reflects state

Run:
```bash
npm run lint
npm run test -- --run
```

- [ ] **Step 1: Reduced-motion gating verified by reading the compiled CSS or by manual DevTools toggle**
- [ ] **Step 2: A11y attribute sweep, fix any gaps found**
- [ ] **Step 3: Commit** — `chore(home): reduced-motion + a11y polish`

---

## Task 13 — E2E spec

**Files:**
- Create: `e2e/mobile-home-dashboard.spec.ts`

Use `e2e/helpers/api-helpers.ts` to register a family and seed events via the real backend (per `frontend/CLAUDE.md` E2E setup). Use the existing Playwright `mobile` project (or set `viewport: { width: 390, height: 844 }`).

Cover:
1. Dashboard renders on mobile (header text present, member-chip row present, hero card present)
2. Hero shows the seeded event matching current local time as `RIGHT_NOW` or `UP_NEXT`
3. Tapping a member chip filters today list
4. With no events seeded for the day → renders "All clear today" hero state
5. Tapping "+" opens the event-create sheet (existing flow)

- [ ] **Step 1: Write the spec following patterns in `e2e/mobile-calendar.spec.ts`**
- [ ] **Step 2: Run** — `npm run test:e2e -- mobile-home-dashboard.spec.ts`
- [ ] **Step 3: Fix any flakiness using existing helpers (`waitForHydration`, `safeClick`)**
- [ ] **Step 4: Commit** — `test(home): e2e for mobile dashboard`

---

## Task 14 — PR

- [ ] Push branch: `git push -u origin feat/home-dashboard-redesign`
- [ ] Open PR against FE `main`
- [ ] Title: `feat(home): mobile home dashboard redesign — "The Now"`
- [ ] Body: link to spec, link to story, link to root-repo plan, summary of acceptance criteria coverage, screenshots / GIFs of all five hero states + filter behavior
- [ ] Confirm CI passes (lint, tests, E2E against real BE)
- [ ] Request review

---

## Acceptance check — map back to spec §13

Before requesting review, confirm each spec acceptance criterion:

- [ ] Replaces the 2-column module grid as the mobile home surface on breakpoints ≤768px — Task 11
- [ ] Integrates with the current FE app shell as the mobile `activeModule === null` surface; desktop/non-mobile behavior unchanged — Task 11
- [ ] Hero renders the correct state across all five cases — Tasks 1, 7
- [ ] Hero transitions live as time crosses event boundaries — Task 4
- [ ] Member-chip focus filters Hero, Today list, and Coming-up consistently — Tasks 3, 11
- [ ] All-day events pin at the top of Today list; ALL_DAY_ONLY hero state correct — Tasks 1, 8
- [ ] Coming-up renders 0–3 events; region omitted when empty — Tasks 3, 9
- [ ] "+" opens event-create sheet pre-filled — Task 11
- [ ] Native and Google-synced events share the same dashboard states; no dashboard-owned Google onboarding/sync UI added — Tasks 3, 11
- [ ] No new design tokens — verified across all tasks
- [ ] Motion uses three-duration / single-easing — Tasks 6, 7, 12
- [ ] `prefers-reduced-motion` respected — Task 12
- [ ] All touch targets ≥44px — Tasks 6, 7, 8, 9
- [ ] Layout 320–768px without horizontal overflow — Task 13
- [ ] Dashboard and calendar tab not confusable in screenshots — visual review during PR
- [ ] Existing query reused — Task 3
- [ ] Hero recompute does not trigger today-list re-render — Tasks 4, 11 (memoization)

---

## Notes for the implementer

- **Don't introduce new tokens.** If you find yourself wanting a custom color, opacity, or radius value, stop and check the existing scale.
- **Don't subscribe to global state from leaf components.** Hero, list, coming-up, header, and chip-row are all pure props-in. Subscriptions live in `home-dashboard.tsx` and the dashboard-specific hooks.
- **Prefer `useMemo` over `useEffect` + `useState`** for derived values from events.
- **Watch the date pitfalls** documented in `frontend/CLAUDE.md` — never `new Date("YYYY-MM-DD")`, never `toISOString()` for display.
- **Don't add "missed event" or "overdue" UI.** The dashboard is moment-aware, not status-aware (spec §11).
- **Don't add notifications, badges, or weather.** Out of scope (spec §11).
- **Don't invent a family-level Google sync model here.** Connection and manual sync stay in member settings for this story.
- **The `now` clock is one source.** Don't create new `Date()` in multiple components — drift will produce subtle inconsistencies between hero and list ordering.

---

## Out-of-plan reminders (do these in separate stories, not this one)

- Persistent bottom nav (`mobile-ux/persistent-bottom-nav.md`) — hard prerequisite for shipping this to production
- Touchscreen / tablet home — follow-up story to be written
- Child-mode home — follow-up story to be written
- Expandable bottom sheet for event-create — follow-up story; this dashboard's "+" inherits it for free when it lands
