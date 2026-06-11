# Lists Mobile Pass: Compact List Options

## Problem

Finding **F3** from the 0.3.12 Galaxy S10 device pass: in the list detail view, the header card stacks two `<select>`s (category display, completed visibility), a family-default checkbox, and a "Remove all completed" button **above the actual list items**. On a phone this controls block fills most of the first screen, pushing the items — the content the family actually came for — below the fold. These are set-once preferences, not per-visit controls; they don't deserve permanent prime real estate on mobile.

## Current FE state (all paths relative to `frontend/`)

- **Controls block:** `src/components/lists/list-detail-view.tsx:112-219` — one `rounded-lg border` card containing:
  - kind label + list name + "Add item" button (`list-detail-view.tsx:113-129`) — this part is fine;
  - a `grid gap-3 sm:grid-cols-2` with the **Categories** select (`:131-151`, grocery/to-do lists only) and the **Completed items** select (`:153-187`, with a fallback message when family preferences are unavailable);
  - a row with the **"Show completed by default"** family checkbox and the **"Remove all completed"** button (`:190-218`).
- On mobile (`<640px`, below `sm:`) the grid is single-column, so the block is two full-width selects + helper text + checkbox row + button — roughly 280–340px of viewport before the first list item.
- **Sheet primitive:** `src/components/ui/mobile-sheet.tsx` — vaul `MobileSheet` with `initialHeight: "half" | "full"`; half sheets auto-expand on input focus. Already the house pattern for short forms (list create, list item, chore create…).
- **Breakpoint:** `useIsMobile()` (`src/hooks/use-is-mobile.ts`) = `max-width: 768px`.

## Goals

- On mobile, list items start within the first screen; the options controls are one tap away, not permanently rendered.
- All four controls (categories, completed visibility, family default, clear completed) remain fully functional with unchanged semantics, including the preferences-unavailable fallback messaging.
- Desktop list detail is unchanged.

## Non-goals

- No changes to list preference semantics, API calls, optimistic updates, or `buildListSections` logic.
- No redesign of the lists overview (`lists-view.tsx`), item rows, or the create/edit item sheets.
- No new persistence (the sheet's open state is ephemeral UI state).

## Decision Summary

### D1. Extract a `ListOptionsControls` component, render it in two homes

The four controls (categories select, completed select, family-default checkbox, clear-completed button) move into a single extracted component `src/components/lists/list-options-controls.tsx`, taking the props it needs (`list`, `preferences` state, the mutation callbacks). It is rendered:

- **Desktop (≥768px):** inline in the header card, exactly where the controls live today — no visual change.
- **Mobile (<768px):** inside a `MobileSheet` (`initialHeight="half"`, title "List options") opened from a compact icon button.

This is one source of truth for the controls; the two homes are layout-only. Gating uses `useIsMobile()` (matches the chores-view pattern) — render either the inline block or the trigger + sheet.

### D2. Trigger: icon button in the list header row

The mobile trigger is a ghost icon `Button` (lucide `SlidersHorizontal`, `aria-label="List options"`, ≥44px target) placed in the header card's title row next to "Add item". The header card on mobile shrinks to: kind label, list name, "Add item", options trigger.

### D3. "Remove all completed" stays inside the options sheet

It's a bulk action, not a browse-time control, and it already has a disabled state when there's nothing to clear. Keeping all four controls together in the sheet preserves the current grouping and avoids a second affordance. The sheet stays open after the action (user sees the disabled state flip), closing via the standard sheet gestures/Cancel.

## Visual / Layout Contract

- **Mobile list detail, top-down:** back button → header card (kind label, name, Add item, options icon) → first list item. The header card contains **no selects, no checkbox, no clear button**.
- **Options sheet (mobile):** half-height `MobileSheet`, title "List options", containing in order: Categories select (when `kind !== "general"`), Completed items select (+ fallback helper text when preferences unavailable), "Show completed by default" checkbox, "Remove all completed" button (destructive-outline, full-width). Field labels and helper copy identical to today.
- **Desktop (≥768px):** pixel-identical to current layout.

## Interaction Model

- Tap options icon → half sheet opens. Changing a select fires the same `updateList.mutate` / `updatePreferences.mutate` calls immediately (no Save step — unchanged semantics). Sheet dismisses via flick-down/scrim/Esc/Cancel.
- List re-renders behind the sheet as options change (optimistic updates already handle this).

## Out of Scope

- Lists overview, item CRUD sheets, list create flow.
- Backend, desktop layout, preference semantics.

## Acceptance Criteria

- [ ] On a 360–430px viewport, the list detail header card shows only kind label, list name, "Add item", and the options trigger; at least the first list item is visible without scrolling (for a list with items).
- [ ] The options trigger opens a half-height sheet containing all four controls with today's labels, helper text, and disabled states (including the preferences-unavailable fallback).
- [ ] Changing categories/completed options from the sheet updates the visible list exactly as the inline controls do today.
- [ ] "Remove all completed" works from the sheet and disables when there are no completed items.
- [ ] The categories select is absent for `general` lists in both homes.
- [ ] Desktop (≥768px) list detail renders the controls inline, visually unchanged.
- [ ] Existing lists unit + E2E suites pass (updated where they assert the old inline-on-mobile structure).
