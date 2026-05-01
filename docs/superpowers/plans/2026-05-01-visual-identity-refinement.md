# Visual Identity Refinement Pass Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Tighten Family Hub's authenticated mobile visual execution across shell, Home, Calendar, and current placeholder module screens without changing product behavior or module meaning.

**Architecture:** This is a frontend-only refinement pass built around shared visual primitives first, then shell chrome, then shipped product surfaces, then light alignment of placeholder module screens. The work should flow from common tokens and reusable UI patterns outward so Home, Calendar, and the tab destinations inherit the same spacing, typography, and palette decisions.

**Tech Stack:** React 19, TypeScript, Vite, Tailwind v4, oklch CSS variables, Vitest, Playwright, Biome.

**Spec:** `docs/superpowers/specs/2026-05-01-visual-identity-refinement-design.md`

**Repo:** All implementation work lives in `frontend/`. This plan lives in the root repo for cross-repo coordination only.

---

## Pre-flight

- [ ] Read `docs/superpowers/specs/2026-05-01-visual-identity-refinement-design.md` end-to-end.
- [ ] Read `frontend/AGENTS.md` and respect its date/time, testing, and shipping notes.
- [ ] Confirm the FE issue body includes `Story:`, `Spec:`, and `Plan:` links before coding.
- [ ] Create a feature branch in `frontend/`, for example: `git checkout -b feat/visual-identity-refinement`
- [ ] Keep the story bounded:
  - no routing work
  - no BE/API changes
  - no new module functionality
  - no redesign of onboarding/settings/sidebar behavior

## File structure

Expected FE files to review and likely modify:

```text
frontend/src/
├── index.css
├── components/shared/
│   ├── app-header.tsx
│   └── mobile-bottom-nav.tsx
├── components/ui/
│   ├── button.tsx
│   ├── dialog.tsx
│   ├── input.tsx
│   └── label.tsx
├── components/home/
│   ├── home-dashboard.tsx
│   └── components/
│       ├── dashboard-header.tsx
│       ├── hero-card.tsx
│       ├── member-chip-row.tsx
│       ├── today-list.tsx
│       └── coming-up.tsx
├── components/calendar/
│   ├── components/
│   │   ├── mobile-toolbar.tsx
│   │   ├── mobile-event-sheet.tsx
│   │   ├── mobile-event-detail.tsx
│   │   ├── event-form.tsx
│   │   ├── event-detail-modal.tsx
│   │   ├── calendar-navigation.tsx
│   │   └── calendar-view-switcher.tsx
│   └── views/mobile/
│       ├── mobile-daily-view.tsx
│       ├── mobile-weekly-view.tsx
│       └── mobile-monthly-view.tsx
├── components/
│   ├── lists-view.tsx
│   ├── chores-view.tsx
│   ├── meals-view.tsx
│   └── photos-view.tsx
└── tests / specs to update as needed:
    ├── App.shell.test.tsx
    ├── components/shared/mobile-bottom-nav.test.tsx
    ├── components/home/home-dashboard.test.tsx
    ├── components/calendar/components/mobile-toolbar.test.tsx
    ├── components/calendar/views/mobile/*.test.tsx
    └── e2e/mobile-*.spec.ts
```

Not every file above must change, but the engineer should start from this map instead of exploring from scratch.

## Task 1: Establish the shared visual contract

**Files:**
- Modify: `frontend/src/index.css`
- Modify: `frontend/src/components/ui/button.tsx`
- Modify: `frontend/src/components/ui/input.tsx`
- Modify: `frontend/src/components/ui/label.tsx`
- Modify: `frontend/src/components/ui/dialog.tsx`

- [ ] Audit current mobile spacing, radii, and text-weight usage in the files above plus the spec.
- [ ] Normalize shared primitives to support the documented rhythm:
  - tighten or soften padding choices where they currently fight the spec
  - keep Nunito as-is
  - avoid introducing new custom token names
- [ ] Review the seven member-color tokens in `src/index.css` and harmonize them only if needed for cohesion / contrast.
- [ ] Run focused FE verification:

```bash
cd frontend
npm run lint
```

Expected: no Biome regressions from shared-primitives work.

- [ ] Commit:

```bash
git add src/index.css src/components/ui/button.tsx src/components/ui/input.tsx src/components/ui/label.tsx src/components/ui/dialog.tsx
git commit -m "feat(ui): tighten shared visual primitives"
```

## Task 2: Polish shell chrome

**Files:**
- Modify: `frontend/src/components/shared/app-header.tsx`
- Modify: `frontend/src/components/shared/mobile-bottom-nav.tsx`
- Test: `frontend/src/App.shell.test.tsx`
- Test: `frontend/src/components/shared/mobile-bottom-nav.test.tsx`

- [ ] Apply the shared spacing / type decisions to the mobile header and bottom nav.
- [ ] Keep behavior unchanged:
  - `AppHeader` still hides on mobile Calendar
  - bottom nav still renders the same six tabs
  - nav state still flows through `activeModule`
- [ ] Verify active states, icon sizing, and label emphasis read consistently with the spec.
- [ ] Update or extend tests only where class-driven structure or accessibility expectations need to change.
- [ ] Run focused FE verification:

```bash
cd frontend
npm run test -- --run src/App.shell.test.tsx src/components/shared/mobile-bottom-nav.test.tsx
```

Expected: shell and nav tests pass after the polish.

- [ ] Commit:

```bash
git add src/components/shared/app-header.tsx src/components/shared/mobile-bottom-nav.tsx src/App.shell.test.tsx src/components/shared/mobile-bottom-nav.test.tsx
git commit -m "feat(ui): refine mobile shell chrome"
```

## Task 3: Polish the shipped Home surface

**Files:**
- Modify: `frontend/src/components/home/home-dashboard.tsx`
- Modify: `frontend/src/components/home/components/dashboard-header.tsx`
- Modify: `frontend/src/components/home/components/hero-card.tsx`
- Modify: `frontend/src/components/home/components/member-chip-row.tsx`
- Modify: `frontend/src/components/home/components/today-list.tsx`
- Modify: `frontend/src/components/home/components/coming-up.tsx`
- Test: `frontend/src/components/home/home-dashboard.test.tsx`

- [ ] Apply the spec's spacing and type hierarchy to greeting, chips, hero, list sections, and card internals.
- [ ] Keep the existing hero-state logic, interaction model, and motion behavior intact.
- [ ] Do not add new Home features or new data dependencies.
- [ ] Update tests only if structure / labels shift in a way that breaks existing assertions.
- [ ] Run focused FE verification:

```bash
cd frontend
npm run test -- --run src/components/home/home-dashboard.test.tsx
```

Expected: Home behavioral tests still pass; only visual classes and structure should have changed.

- [ ] Commit:

```bash
git add src/components/home/home-dashboard.tsx src/components/home/components/dashboard-header.tsx src/components/home/components/hero-card.tsx src/components/home/components/member-chip-row.tsx src/components/home/components/today-list.tsx src/components/home/components/coming-up.tsx src/components/home/home-dashboard.test.tsx
git commit -m "feat(ui): refine home dashboard visual rhythm"
```

## Task 4: Polish Calendar mobile surfaces

**Files:**
- Modify: `frontend/src/components/calendar/components/mobile-toolbar.tsx`
- Modify: `frontend/src/components/calendar/components/mobile-event-sheet.tsx`
- Modify: `frontend/src/components/calendar/components/mobile-event-detail.tsx`
- Modify: `frontend/src/components/calendar/components/event-form.tsx`
- Modify: `frontend/src/components/calendar/components/event-detail-modal.tsx`
- Modify: `frontend/src/components/calendar/components/calendar-navigation.tsx`
- Modify: `frontend/src/components/calendar/components/calendar-view-switcher.tsx`
- Modify: `frontend/src/components/calendar/views/mobile/mobile-daily-view.tsx`
- Modify: `frontend/src/components/calendar/views/mobile/mobile-weekly-view.tsx`
- Modify: `frontend/src/components/calendar/views/mobile/mobile-monthly-view.tsx`
- Test: `frontend/src/components/calendar/components/mobile-toolbar.test.tsx`
- Test: `frontend/src/components/calendar/views/mobile/mobile-daily-view.test.tsx`
- Test: `frontend/src/components/calendar/views/mobile/mobile-weekly-view.test.tsx`
- Test: `frontend/src/components/calendar/views/mobile/mobile-monthly-view.test.tsx`

- [ ] Tighten toolbar hierarchy, pill rhythm, date emphasis, and sheet/form spacing so Calendar matches Home and the shell.
- [ ] Preserve all current Calendar behavior, including view switching, filters, event CRUD flows, and Google read-only guards.
- [ ] Keep touch targets comfortable; do not make the visual pass more compact at the cost of usability.
- [ ] Run focused FE verification:

```bash
cd frontend
npm run test -- --run \
  src/components/calendar/components/mobile-toolbar.test.tsx \
  src/components/calendar/views/mobile/mobile-daily-view.test.tsx \
  src/components/calendar/views/mobile/mobile-weekly-view.test.tsx \
  src/components/calendar/views/mobile/mobile-monthly-view.test.tsx
```

Expected: mobile Calendar tests stay green after the visual pass.

- [ ] Commit:

```bash
git add src/components/calendar/components/mobile-toolbar.tsx src/components/calendar/components/mobile-event-sheet.tsx src/components/calendar/components/mobile-event-detail.tsx src/components/calendar/components/event-form.tsx src/components/calendar/components/event-detail-modal.tsx src/components/calendar/components/calendar-navigation.tsx src/components/calendar/components/calendar-view-switcher.tsx src/components/calendar/views/mobile/mobile-daily-view.tsx src/components/calendar/views/mobile/mobile-weekly-view.tsx src/components/calendar/views/mobile/mobile-monthly-view.tsx src/components/calendar/components/mobile-toolbar.test.tsx src/components/calendar/views/mobile/mobile-daily-view.test.tsx src/components/calendar/views/mobile/mobile-weekly-view.test.tsx src/components/calendar/views/mobile/mobile-monthly-view.test.tsx
git commit -m "feat(ui): polish mobile calendar surfaces"
```

## Task 5: Align placeholder module screens without redefining them

**Files:**
- Modify: `frontend/src/components/lists-view.tsx`
- Modify: `frontend/src/components/chores-view.tsx`
- Modify: `frontend/src/components/meals-view.tsx`
- Modify: `frontend/src/components/photos-view.tsx`

- [ ] Bring page-title scale, card padding, button treatment, and section spacing into the shared visual rhythm.
- [ ] Do not introduce new product meaning:
  - no new lists semantics
  - no real chores data work
  - no meals model changes
  - no photo-source decisions
- [ ] Run a smoke pass in the browser or via existing mobile specs to ensure these screens still render and switch cleanly from the bottom nav.
- [ ] Commit:

```bash
git add src/components/lists-view.tsx src/components/chores-view.tsx src/components/meals-view.tsx src/components/photos-view.tsx
git commit -m "feat(ui): align placeholder module surfaces"
```

## Task 6: Final verification and evidence capture

**Files:**
- No required repo file changes unless tests expose regressions.

- [ ] Run final FE verification:

```bash
cd frontend
npm run lint
npm run test -- --run src/App.shell.test.tsx src/components/home/home-dashboard.test.tsx src/components/shared/mobile-bottom-nav.test.tsx src/components/calendar/components/mobile-toolbar.test.tsx src/components/calendar/views/mobile/mobile-daily-view.test.tsx src/components/calendar/views/mobile/mobile-weekly-view.test.tsx src/components/calendar/views/mobile/mobile-monthly-view.test.tsx
npm run test:e2e -- mobile-bottom-nav.spec.ts mobile-home-dashboard.spec.ts mobile-calendar.spec.ts
```

Expected:
- lint passes
- focused unit/integration tests pass
- mobile shell / home / calendar E2E smoke remains green

- [ ] Capture before/after mobile screenshots for:
  - Home dashboard
  - Calendar mobile
  - Event sheet or event form
  - Lists
  - Chores
- [ ] Put the screenshot pairings and a short note about any member-palette adjustments into the FE issue or PR body.
- [ ] Final commit if verification fixes were needed:

```bash
git add -A
git commit -m "test(ui): finish visual identity refinement verification"
```

## Exit checklist

- [ ] No FE behavior changed beyond visual treatment.
- [ ] No BE changes were introduced or required.
- [ ] Home, Calendar, and shell chrome now read as one product.
- [ ] Placeholder module screens look coherent without implying settled product meaning.
- [ ] Before/after evidence exists in the FE delivery artifact.
