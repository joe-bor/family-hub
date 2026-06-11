# Lists Mobile Pass: Compact List Options — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** On mobile, move the list-detail option controls (categories, completed visibility, family default, clear completed) out of the always-visible header card into a half-height `MobileSheet` behind an icon trigger, so list items start within the first screen.

**Architecture:** Extract the controls block from `list-detail-view.tsx` into a `ListOptionsControls` component (pure presentation + the same mutation callbacks). `ListDetailView` renders it inline on desktop and inside a `MobileSheet` (`initialHeight="half"`, trigger: `SlidersHorizontal` icon button) on mobile via `useIsMobile()`. No semantic changes to preferences or mutations.

**Tech Stack:** React 19, TypeScript, Tailwind CSS v4, vaul (via existing `MobileSheet`), Vitest, Testing Library, Playwright.

**Spec:** `docs/superpowers/specs/2026-06-11-mobile-lists-options-density.md`
**Story:** `docs/product/backlog/mobile-ux/mobile-module-content-polish.md` (finding F3)

---

This is an **FE-only** plan. Execute it inside the `frontend/` repo on a fresh feature branch such as `fix/mobile-lists-options-density`. All paths below are relative to `frontend/`. Use **regular merge commits** (release-please) and conventional commit messages.

## File Structure

| File | Action | Purpose |
|------|--------|---------|
| `src/components/lists/list-options-controls.tsx` | Create | Extracted controls block (selects, checkbox, clear button) |
| `src/components/lists/list-detail-view.tsx` | Modify | Two homes: inline (desktop) / sheet + trigger (mobile) |
| `src/components/lists/list-detail-view.test.tsx` | Create or extend | Lock mobile sheet flow + desktop inline rendering |

---

### Task 1: Extract `ListOptionsControls`

Pure extraction — markup and behavior copied verbatim from `list-detail-view.tsx:131-218`.

- [ ] **Step 1: Create `src/components/lists/list-options-controls.tsx`** with props:

  ```tsx
  interface ListOptionsControlsProps {
    list: ListDetail; // same type listQuery.data.data has today
    hasPreferences: boolean;
    familyShowCompletedDefault: boolean;
    completedControlsDisabled: boolean;
    completedOverrideValue: "family-default" | "show" | "hide";
    completedFallbackMessage: string;
    clearCompletedDisabled: boolean;
    onUpdateList: (request: {
      categoryDisplayMode: ListCategoryDisplayMode;
      showCompletedOverride: boolean | null;
    }) => void;
    onUpdatePreferences: (request: { showCompletedByDefault: boolean }) => void;
    onClearCompleted: () => void;
  }
  ```

  Move the two selects (`list-detail-view.tsx:131-187`), the family-default checkbox + clear-completed row (`:190-218`) into it unchanged, swapping direct `mutate` calls for the callback props. Match the exact `ListDetail` type name from `@/lib/types` (verify before writing).

- [ ] **Step 2: Replace the inline block in `list-detail-view.tsx`** with `<ListOptionsControls …/>` (desktop path only for now — keep it rendering unconditionally in this step). Pass `onUpdateList={(req) => updateList.mutate(req)}` etc.

- [ ] **Step 3: Verify no behavior change**

  ```bash
  cd frontend
  npm run test -- --run src/components/lists
  npm run lint
  ```

  Expected: PASS (pure refactor).

- [ ] **Step 4: Commit**

  ```bash
  git add src/components/lists/list-options-controls.tsx src/components/lists/list-detail-view.tsx
  git commit -m "refactor(lists): extract list options controls component"
  ```

---

### Task 2: Mobile sheet + trigger

- [ ] **Step 1: Write the failing test** — create/extend `src/components/lists/list-detail-view.test.tsx`. Mock `useIsMobile` (same pattern as `app-header.test.tsx` / login-form tests):

  ```tsx
  let mockIsMobile = true;
  vi.mock("@/hooks", async (importOriginal) => {
    const actual = await importOriginal<typeof import("@/hooks")>();
    return { ...actual, useIsMobile: () => mockIsMobile };
  });
  ```

  Cases (seed a list via the existing lists test utilities/MSW fixtures — mirror whatever `list-item-sheet` or lists hooks tests use for data setup):
  1. **Mobile:** the "Completed items" select is NOT in the document on initial render; a button named "List options" is. Clicking it shows the select inside a dialog; changing it fires the update (assert via the select's new value or the mocked mutation).
  2. **Mobile:** "Add item" button still present in the header.
  3. **Desktop (`mockIsMobile = false`):** the "Completed items" select renders inline without any "List options" trigger.

  > jsdom + vaul: the repo already ships a vaul/jsdom test shim (used by the existing sheet tests — see how `list-item-sheet` or `mobile-bottom-nav` tests render sheets). Reuse that setup; if sheet content portals lazily, use `findBy*`.

- [ ] **Step 2: Run and confirm it fails**

  ```bash
  cd frontend
  npm run test -- --run src/components/lists/list-detail-view.test.tsx
  ```

  Expected: FAIL — no trigger, controls always inline.

- [ ] **Step 3: Implement in `list-detail-view.tsx`**
  - `const isMobile = useIsMobile();` and `const [optionsOpen, setOptionsOpen] = useState(false);`
  - Header row (`:113-129`): next to "Add item", on mobile only, add:

    ```tsx
    {isMobile && (
      <Button
        type="button"
        variant="ghost"
        size="icon"
        aria-label="List options"
        className="h-11 w-11"
        onClick={() => setOptionsOpen(true)}
      >
        <SlidersHorizontal className="h-5 w-5" />
      </Button>
    )}
    ```

  - Render `<ListOptionsControls …/>` inline only when `!isMobile`; on mobile render it inside:

    ```tsx
    <MobileSheet
      isOpen={optionsOpen}
      onClose={() => setOptionsOpen(false)}
      title="List options"
      initialHeight="half"
    >
      <div className="space-y-5">
        <ListOptionsControls … />
      </div>
    </MobileSheet>
    ```

  - Make the clear-completed button full-width inside the sheet (pass a `className`/layout prop or wrap — keep desktop layout untouched).

- [ ] **Step 4: Run the test suite**

  ```bash
  cd frontend
  npm run test -- --run src/components/lists
  npm run lint
  ```

  Expected: PASS.

- [ ] **Step 5: Visual smoke** — `npm run dev` at 390×844: open a grocery list with items → first item visible without scrolling; options icon opens half sheet; toggling "Hide completed" updates the list behind the sheet; "Remove all completed" disables after clearing. Desktop width: controls inline as before, no trigger.

- [ ] **Step 6: Commit**

  ```bash
  git add src/components/lists/list-detail-view.tsx src/components/lists/list-detail-view.test.tsx src/components/lists/list-options-controls.tsx
  git commit -m "fix(mobile): move list options into a half-height sheet"
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

  Expected: all green. If a lists E2E spec drives the inline selects on a mobile viewport, update it to go through the options sheet (and keep a desktop-project assertion for the inline path if one exists).

- [ ] **Step 2: Open the PR**

  ```bash
  git push -u origin fix/mobile-lists-options-density
  gh pr create --fill
  ```

  PR body includes `Closes #<N>` and a checklist mapping each acceptance criterion to its commit.

---

## Self-Review

**Spec coverage:** D1 → Tasks 1–2 (extraction + two homes); D2 → Task 2 Step 3 (trigger); D3 → controls grouped in the sheet including clear-completed. All acceptance criteria covered across Tasks 2–3.

**Open verification risks:** vaul-in-jsdom rendering for the new sheet test — mitigated by reusing the repo's existing sheet-test setup; if content isn't reachable in jsdom, assert the trigger + `MobileSheet` open-state wiring in unit tests and cover select interaction in the mobile E2E project instead. The `ListDetail`/`ListPreferences` prop types must be confirmed against `@/lib/types` before Task 1 Step 1.
