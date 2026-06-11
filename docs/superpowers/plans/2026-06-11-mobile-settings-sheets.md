# Settings Dialogs → Bottom Sheets on Mobile — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert `FamilySettingsModal`, `MemberProfileModal`, and `MemberFormModal` to vaul `MobileSheet`s on mobile (centered dialogs on desktop) via one adaptive wrapper, so settings surfaces are viewport-bounded and keyboard-safe on phones.

**Architecture:** New `ResponsiveFormDialog` adapter (`useIsMobile()` → `MobileSheet`, else `Dialog`/`DialogContent`). The three modals swap their `Dialog` scaffolding for the wrapper; in-content header `X` rows become desktop-only since the sheet supplies title + Cancel. Heights: full/full/half. Confirm dialogs untouched. Sheet-over-sheet stacking is a flagged device-verification risk with a documented fallback.

**Tech Stack:** React 19, TypeScript, vaul (existing `MobileSheet`), `@radix-ui/react-dialog`, Tailwind CSS v4, Vitest, Testing Library, Playwright.

**Spec:** `docs/superpowers/specs/2026-06-11-mobile-settings-sheets.md`
**Story:** `docs/product/backlog/mobile-ux/mobile-module-content-polish.md` (finding F2)

---

This is an **FE-only** plan. Execute it inside the `frontend/` repo on a fresh feature branch such as `feat/mobile-settings-sheets`. All paths below are relative to `frontend/`. Use **regular merge commits** (release-please) and conventional commit messages.

## File Structure

| File | Action | Purpose |
|------|--------|---------|
| `src/components/ui/responsive-form-dialog.tsx` | Create | Adaptive Dialog/MobileSheet wrapper |
| `src/components/ui/responsive-form-dialog.test.tsx` | Create | Lock mobile→sheet / desktop→dialog switching |
| `src/components/settings/family-settings-modal.tsx` | Modify | Wrap in `ResponsiveFormDialog` (full) |
| `src/components/settings/member-profile-modal.tsx` | Modify | Wrap in `ResponsiveFormDialog` (full) |
| `src/components/onboarding/member-form-modal.tsx` | Modify | Wrap in `ResponsiveFormDialog` (half) |
| `src/components/settings/*.test.tsx` | Modify | Update structural assertions |
| `e2e/mobile-settings-sheets.spec.ts` | Create | Sheet open/dismiss/keyboard-path E2E |

---

### Task 1: `ResponsiveFormDialog` wrapper

- [ ] **Step 1: Write the failing test** — `src/components/ui/responsive-form-dialog.test.tsx`:

  ```tsx
  import { describe, expect, it, vi } from "vitest";
  import { render, screen } from "@/test/test-utils";
  import { ResponsiveFormDialog } from "./responsive-form-dialog";

  let mockIsMobile = false;
  vi.mock("@/hooks", async (importOriginal) => {
    const actual = await importOriginal<typeof import("@/hooks")>();
    return { ...actual, useIsMobile: () => mockIsMobile };
  });

  describe("ResponsiveFormDialog", () => {
    it("renders a centered dialog with the title on desktop", () => {
      mockIsMobile = false;
      render(
        <ResponsiveFormDialog open onOpenChange={vi.fn()} title="Family Settings">
          <p>content</p>
        </ResponsiveFormDialog>,
      );
      expect(screen.getByRole("dialog", { name: "Family Settings" })).toBeInTheDocument();
      expect(screen.getByText("content")).toBeInTheDocument();
      // sheet chrome absent
      expect(screen.queryByRole("button", { name: /cancel/i })).toBeNull();
    });

    it("renders the mobile sheet with Cancel on mobile", () => {
      mockIsMobile = true;
      render(
        <ResponsiveFormDialog open onOpenChange={vi.fn()} title="Family Settings">
          <p>content</p>
        </ResponsiveFormDialog>,
      );
      expect(screen.getByText("content")).toBeInTheDocument();
      expect(screen.getByRole("button", { name: /cancel/i })).toBeInTheDocument();
    });
  });
  ```

  Reuse the repo's existing vaul/jsdom test shim (see how current `MobileSheet` consumers' tests are set up — e.g. the bottom-nav More-sheet or list-item-sheet tests).

- [ ] **Step 2: Run and confirm it fails** (`npm run test -- --run src/components/ui/responsive-form-dialog.test.tsx`).

- [ ] **Step 3: Implement `src/components/ui/responsive-form-dialog.tsx`** per the spec's D1 API:

  ```tsx
  import type { ReactNode } from "react";
  import {
    Dialog,
    DialogContent,
    DialogHeader,
    DialogTitle,
  } from "@/components/ui/dialog";
  import { MobileSheet, type MobileSheetHeight } from "@/components/ui/mobile-sheet";
  import { useIsMobile } from "@/hooks";

  interface ResponsiveFormDialogProps {
    open: boolean;
    onOpenChange: (open: boolean) => void;
    title: string;
    initialHeight?: MobileSheetHeight;
    dialogClassName?: string;
    /** Extra desktop-only header content (e.g. the X close button row). */
    desktopHeaderRight?: ReactNode;
    children: ReactNode;
  }

  export function ResponsiveFormDialog({
    open,
    onOpenChange,
    title,
    initialHeight = "full",
    dialogClassName,
    desktopHeaderRight,
    children,
  }: ResponsiveFormDialogProps) {
    const isMobile = useIsMobile();

    if (isMobile) {
      return (
        <MobileSheet
          isOpen={open}
          onClose={() => onOpenChange(false)}
          title={title}
          initialHeight={initialHeight}
        >
          {children}
        </MobileSheet>
      );
    }

    return (
      <Dialog open={open} onOpenChange={onOpenChange}>
        <DialogContent className={dialogClassName}>
          <DialogHeader>
            <div className="flex w-full items-center justify-between">
              <DialogTitle className="text-xl">{title}</DialogTitle>
              {desktopHeaderRight}
            </div>
          </DialogHeader>
          {children}
        </DialogContent>
      </Dialog>
    );
  }
  ```

  Adjust header markup to match what the settings modals render today (compare against `family-settings-modal.tsx:117-130` so desktop stays pixel-identical).

- [ ] **Step 4: Run the test** — expected PASS — then commit:

  ```bash
  git add src/components/ui/responsive-form-dialog.tsx src/components/ui/responsive-form-dialog.test.tsx
  git commit -m "feat(ui): add responsive form dialog (sheet on mobile, dialog on desktop)"
  ```

---

### Task 2: Convert `FamilySettingsModal` (full)

- [ ] **Step 1: Swap scaffolding** — in `family-settings-modal.tsx`, replace the outer `Dialog`/`DialogContent`/`DialogHeader` block (`:115-130`) with:

  ```tsx
  <ResponsiveFormDialog
    open={open}
    onOpenChange={onOpenChange}
    title="Family Settings"
    initialHeight="full"
    dialogClassName="max-w-lg max-h-[90dvh] overflow-y-auto"
    desktopHeaderRight={
      <Button variant="ghost" size="icon" aria-label="Close"
        onClick={() => onOpenChange(false)} className="h-8 w-8">
        <X className="h-4 w-4" />
      </Button>
    }
  >
    {/* existing sections content (current `:132-237`) */}
  </ResponsiveFormDialog>
  ```

  The nested `MemberFormModal` and reset-confirm `Dialog` stay exactly where they are (outside the wrapper, as siblings — unchanged from today's structure).

- [ ] **Step 2: Update `family-settings-modal` tests** — assertions about `DialogTitle`/close button now depend on the mocked `useIsMobile` value; add/keep a desktop-mode case asserting the dialog role + Close button, and a mobile case asserting content renders inside the sheet.

- [ ] **Step 3: Run + commit**

  ```bash
  cd frontend && npm run test -- --run src/components/settings && npm run lint
  git add src/components/settings/family-settings-modal.tsx src/components/settings/*.test.tsx
  git commit -m "feat(mobile): family settings opens as a full-height bottom sheet"
  ```

---

### Task 3: Convert `MemberProfileModal` (full) and `MemberFormModal` (half)

- [ ] **Step 1: `member-profile-modal.tsx`** — same swap as Task 2 at `:141`, with `title` = the current `DialogTitle` text, `initialHeight="full"`, `dialogClassName="max-w-sm max-h-[90dvh] overflow-y-auto"`, preserving the existing desktop header-right content (check `:141-160` for the current header row).

- [ ] **Step 2: `member-form-modal.tsx`** — swap at `:84-94`: `title={mode === "add" ? "Add Family Member" : "Edit Family Member"}`, `initialHeight="half"`, `dialogClassName="w-full max-w-md mx-4 sm:mx-auto max-h-[90dvh] overflow-y-auto"` (adds the previously missing height bound for the desktop dialog). Note: the title is dynamic — confirm the wrapper's `title: string` prop covers it (it does; it re-renders on mode change).

- [ ] **Step 3: Update affected tests** (`member-profile-modal.test.tsx`, any onboarding members-step test that asserts dialog structure).

- [ ] **Step 4: Run + commit**

  ```bash
  cd frontend && npm run test -- --run src/components/settings src/components/onboarding && npm run lint
  git add src/components/settings/member-profile-modal.tsx src/components/onboarding/member-form-modal.tsx src/components/settings src/components/onboarding
  git commit -m "feat(mobile): member profile and member form open as bottom sheets"
  ```

---

### Task 4: E2E + device verification + PR

- [ ] **Step 1: Write `e2e/mobile-settings-sheets.spec.ts`** (mobile project; mirror auth seeding from `e2e/mobile-settings-sidebar.spec.ts` if present, else `registerFamily`/`seedBrowserAuth` per `e2e/helpers/api-helpers.ts`):
  1. Open sidebar → Family Settings → sheet visible (dialog role, name "Family Settings") → Esc dismisses.
  2. Family Settings → Add member → member form sheet/dialog appears above → fill name → submit → new member listed (covers the stacking path in CI's emulated mobile).
  3. Sidebar → member row → Member Profile sheet → scroll to Google Calendar section → Cancel dismisses, sidebar focus restored.

- [ ] **Step 2: Full gate**

  ```bash
  cd frontend
  npm run lint
  npm run test -- --run
  npm run test:e2e
  ```

  Expected: all green; update any settings E2E spec that asserted centered-dialog structure on mobile.

- [ ] **Step 3: Device smoke (required before merge)** — real phone or accurate emulation:
  - Keyboard: focus email in Member Profile and name in the half-height member form → field stays visible (auto-expand engages).
  - **Sheet-over-sheet:** Family Settings → Add member on device. If vaul stacking glitches (background scaling/gesture conflicts), apply the spec's D3 fallback: `MemberFormModal` keeps the centered dialog on mobile (it already gets the `max-h` bound in Task 3) — note the fallback in the PR body.
  - Reset Family confirm opens centered above the sheet and works.
  - #194 no-regression: login + onboarding credentials screens still keyboard-safe.

- [ ] **Step 4: Open the PR**

  ```bash
  git push -u origin feat/mobile-settings-sheets
  gh pr create --fill
  ```

  PR body includes `Closes #<N>`, a checklist mapping each acceptance criterion to its commit, and the device-smoke results (explicitly including the stacking outcome).

---

## Self-Review

**Spec coverage:** D1 → Task 1; D2 → Tasks 2–3 (heights); D3 → Task 4 Step 3 (stacking verification + fallback); D4 → Task 3 Step 2 (shared component converts onboarding usage). All acceptance criteria land in Tasks 2–4.

**Open verification risks (flagged, not placeholders):**
- vaul sheet-over-sheet stacking is the main unknown — explicit device gate with a concrete, already-bounded fallback.
- Keyboard auto-expand can't be asserted in jsdom — covered by the device smoke step.
- Desktop pixel-parity depends on faithfully reproducing each modal's current header row in `desktopHeaderRight` — Task 1 Step 3 calls out comparing against current markup.
