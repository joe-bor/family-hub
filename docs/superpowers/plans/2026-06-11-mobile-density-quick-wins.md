# Mobile Density Quick Wins — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the redundant Chores period heading on mobile and convert the Recipes library card to a horizontal thumbnail row on mobile, so both modules show meaningfully more content per phone screen.

**Architecture:** Two independent FE-only changes. Chores: a `showHeading` prop on `ChoreScopeColumn`, driven by `useIsMobile()` in `ChoresView`, with `aria-label` preserving the section's accessible name. Recipes: CSS-first responsive restyle of `RecipeLibraryCard` (`flex-row` base, `md:flex-col`) plus a matching skeleton tweak — no new components, no JS branching. Desktop unchanged in both modules.

**Tech Stack:** React 19, TypeScript, Tailwind CSS v4, Vitest, Testing Library, Playwright.

**Spec:** `docs/superpowers/specs/2026-06-11-mobile-density-quick-wins.md`
**Story:** `docs/product/backlog/mobile-ux/mobile-module-content-polish.md` (findings F4 + F5)

---

This is an **FE-only** plan. Execute it inside the `frontend/` repo on a fresh feature branch such as `fix/mobile-density-quick-wins`. All paths below are relative to `frontend/`. Use **regular merge commits** (release-please) and conventional commit messages.

## File Structure

| File | Action | Purpose |
|------|--------|---------|
| `src/components/chores/chores-scope-column.tsx` | Modify | `showHeading` prop + `aria-label` fallback |
| `src/components/chores-view.tsx` | Modify | Pass `showHeading={!isMobile}` |
| `src/components/chores/chores-scope-column.test.tsx` | Create | Lock heading visibility + accessible name |
| `src/components/recipes/recipe-library-card.tsx` | Modify | Horizontal row on mobile, vertical card on `md:` |
| `src/components/recipes/recipe-library-card.test.tsx` | Modify | Update structure assertions |
| `src/components/recipes-view.tsx` | Modify | Skeleton mirrors mobile row shape |

---

### Task 1: Chores — drop the period heading on mobile (F4)

- [ ] **Step 1: Write the failing test** — create `src/components/chores/chores-scope-column.test.tsx`:

  ```tsx
  import { describe, expect, it } from "vitest";
  import { render, screen } from "@/test/test-utils";
  import type { ChoreScopeBoard } from "@/lib/types";
  import { ChoreScopeColumn } from "./chores-scope-column";

  const board: ChoreScopeBoard = {
    scope: "TODAY",
    periodStartDate: "2026-06-11",
    summary: { total: 2, remaining: 1 },
    assignees: [],
  };

  describe("ChoreScopeColumn", () => {
    it("shows the period heading by default (desktop)", () => {
      render(<ChoreScopeColumn scope={board} />);
      expect(
        screen.getByRole("heading", { name: "Today" }),
      ).toBeInTheDocument();
    });

    it("hides the heading but keeps the accessible name when showHeading is false", () => {
      render(<ChoreScopeColumn scope={board} showHeading={false} />);
      expect(screen.queryByRole("heading", { name: "Today" })).toBeNull();
      // section still announces its period
      expect(
        screen.getByRole("region", { name: "Today" }),
      ).toBeInTheDocument();
      // summary line survives
      expect(screen.getByText(/1 left of 2/i)).toBeInTheDocument();
    });
  });
  ```

  Adjust the `ChoreScopeBoard` fixture shape to the real type in `src/lib/types` if fields differ (check `summary`/`assignees` field names before running).

- [ ] **Step 2: Run and confirm it fails**

  ```bash
  cd frontend
  npm run test -- --run src/components/chores/chores-scope-column.test.tsx
  ```

  Expected: FAIL — `showHeading` prop does not exist yet.

- [ ] **Step 3: Implement in `chores-scope-column.tsx`**
  - Add `showHeading?: boolean` (default `true`) to `ChoreScopeColumnProps`.
  - When `showHeading` is true: render the existing `<header>` (h2 + summary) and keep `aria-labelledby` on the `<section>`.
  - When false: drop the `<h2>`, render the summary line alone at the top of the card (`<p className="mb-4 text-sm text-muted-foreground">{remaining} left of {total}</p>`), and switch the `<section>` to `aria-label={heading}` (remove `aria-labelledby`).

- [ ] **Step 4: Wire up in `chores-view.tsx`** — at the `ChoreScopeColumn` usage (`chores-view.tsx:164-171`), pass `showHeading={!isMobile}` (`isMobile` already exists in scope).

- [ ] **Step 5: Run tests + lint**

  ```bash
  cd frontend
  npm run test -- --run src/components/chores
  npm run lint
  ```

  Expected: PASS.

- [ ] **Step 6: Visual smoke** — `npm run dev`, DevTools at 390×844: Chores shows switcher → card starting with "N left of M", no "Today" text in the card. At desktop width: three columns with headings intact.

- [ ] **Step 7: Commit**

  ```bash
  git add src/components/chores/chores-scope-column.tsx src/components/chores/chores-scope-column.test.tsx src/components/chores-view.tsx
  git commit -m "fix(mobile): drop redundant chores period heading on mobile"
  ```

---

### Task 2: Recipes — horizontal thumbnail card on mobile (F5)

- [ ] **Step 1: Update/extend the card test** — in `src/components/recipes/recipe-library-card.test.tsx`, keep the behavioral assertions (title rendered, `onSelect` fires on click, favorite aria state, "No photo" fallback) and remove/adjust any assertion that hard-codes the vertical structure (e.g. `aspect-[4/3]` class checks). Behavior must be identical in both layouts, so the test should stay layout-agnostic.

- [ ] **Step 2: Restyle `recipe-library-card.tsx`** — CSS-only responsive change, no `useIsMobile()`:
  - `<article>`: `flex w-full flex-row gap-3 p-3 md:flex-col md:gap-0 md:p-0` (keep existing border/rounded/bg/overflow classes; keep the absolute overlay `<button>` untouched).
  - Image wrapper (line 30): mobile = fixed square thumbnail, desktop = current banner:
    `relative size-24 shrink-0 overflow-hidden rounded-lg bg-muted md:size-auto md:aspect-[4/3] md:w-full md:rounded-none`.
  - "No photo" fallback: on mobile the square shows a smaller label (`text-xs`); same element works at both sizes — verify it doesn't overflow the 96px square.
  - Heart overlay block (lines 45–58): hide on mobile (`hidden md:block` on the absolute wrapper) and render an inline heart in the text block on mobile (`md:hidden`), aligned with the title row, reusing the same `fill-rose-500` favorite classes and aria attributes.
  - Text block (line 61): `flex min-w-0 flex-1 flex-col gap-1.5 md:gap-3 md:p-4`; title `line-clamp-2`; tags row: `flex gap-1.5 overflow-hidden md:flex-wrap md:gap-2` so mobile clamps to one row.

- [ ] **Step 3: Update the loading skeleton** — in `src/components/recipes-view.tsx:227-240`, mirror the new shape: skeleton card = `flex flex-row gap-3 p-3 md:flex-col md:gap-0 md:p-0` with a `size-24 md:size-auto md:aspect-[4/3] md:w-full` pulse block + two text-line pulses.

- [ ] **Step 4: Run tests + lint**

  ```bash
  cd frontend
  npm run test -- --run src/components/recipes src/components/recipes-view.test.tsx
  npm run lint
  ```

  Expected: PASS (skip the second path if `recipes-view.test.tsx` doesn't exist).

- [ ] **Step 5: Visual smoke** — 390×844: ≥3 cards visible with thumbnails, single tag row, inline hearts; tap opens detail. Desktop width: card identical to before (banner image, overlay heart, wrapped tags).

- [ ] **Step 6: Commit**

  ```bash
  git add src/components/recipes/recipe-library-card.tsx src/components/recipes/recipe-library-card.test.tsx src/components/recipes-view.tsx
  git commit -m "fix(mobile): compact horizontal recipe cards on mobile"
  ```

---

### Task 3: Verification + PR

- [ ] **Step 1: Full gate**

  ```bash
  cd frontend
  npm run lint
  npm run test -- --run
  npm run test:e2e
  ```

  Expected: all green. If any recipe/chore E2E spec asserts the old card structure or the "Today" heading on mobile, update it to the new contract.

- [ ] **Step 2: Open the PR**

  ```bash
  git push -u origin fix/mobile-density-quick-wins
  gh pr create --fill
  ```

  PR body includes `Closes #<N>` for the FE issue and a checklist mapping each acceptance criterion to its commit.

---

## Self-Review

**Spec coverage:** D1 → Task 1; D2 → Task 2 (card + skeleton + heart/tag handling). Acceptance criteria all land in Tasks 1–2 with the E2E sweep in Task 3.

**Open verification risks:** density ("≥3 cards visible") and the desktop-unchanged claims are visual — covered by explicit smoke steps, not unit tests. The `ChoreScopeBoard` test fixture shape must be checked against the real type before Step 2 of Task 1.
