# Settings Dialogs → Bottom Sheets on Mobile

## Problem

Finding **F2** from the 0.3.12 Galaxy S10 device pass: the settings dialogs can still be cut off / cramped on small screens. FE #194 added `max-h-[90dvh] overflow-y-auto` scroll bounds as an explicit band-aid, and both the 2026-06-08 sidebar/settings spec (Non-goals) and the expandable-bottom-sheet story flagged the durable fix as a follow-up: convert the centered settings modals to the vaul `MobileSheet`, which is viewport-bounded and keyboard-safe by construction (auto-expand on input focus, safe-area padding, drag/scrim/Esc dismissal).

Decision (made with Joe, 2026-06-11): **convert the settings/member modals to `MobileSheet` on mobile**; desktop keeps centered dialogs; small confirm dialogs stay centered everywhere.

## Current FE state (all paths relative to `frontend/`)

- **`FamilySettingsModal`** — `src/components/settings/family-settings-modal.tsx:116`: centered `DialogContent` with `max-w-lg max-h-[90dvh] overflow-y-auto`. Long content: family-name form, member list with add/edit, danger zone. Nests `MemberFormModal` (`:242-250`) and the reset confirm `Dialog` (`:253-274`).
- **`MemberProfileModal`** — `src/components/settings/member-profile-modal.tsx:141`: centered `DialogContent` with `max-w-sm max-h-[90dvh] overflow-y-auto`. Long content: avatar upload, name, color picker, email, `GoogleCalendarSection` (`:269`).
- **`MemberFormModal`** — `src/components/onboarding/member-form-modal.tsx:85-87`: centered `DialogContent` (`max-w-md`, **no max-height bound**). Short form (name + color). Used by both `FamilySettingsModal` and the onboarding members step.
- **Dialog primitive:** `src/components/ui/dialog.tsx:36-49` — fixed, centered, `z-50`.
- **Sheet primitive:** `src/components/ui/mobile-sheet.tsx` — vaul drawer, `initialHeight: "half" | "full"`, full Radix dialog semantics, input-focus auto-expand, safe-area-aware, `z-50`. On ≥768px it renders a centered `max-w-lg` sheet (placeholder until the big-screen pass) — which is why settings keep the real `Dialog` on desktop instead of relying on the sheet's desktop fallback.
- **Breakpoint:** `useIsMobile()` = `max-width: 768px`.
- **Auth/onboarding full screens** were made keyboard-safe in #194 (dvh + top-align + gated autofocus) — not part of this story beyond a no-regression check.

## Goals

- On mobile, the three settings/member modals are bottom sheets: never taller than the viewport, keyboard never hides a focused field, standard sheet gesture language.
- Desktop keeps today's centered dialogs, pixel-identical.
- One reusable adaptive pattern so future settings surfaces don't re-decide this.

## Non-goals

- No conversion of small confirm dialogs (Reset Family — `family-settings-modal.tsx:253`, Sign Out confirm in the sidebar): two-button confirms fit any viewport and centered is the right idiom for destructive confirmation.
- No redesign of the modal contents (forms, sections, validation, Google Calendar section).
- No desktop sheet design (the `MobileSheet` desktop placeholder remains untouched).
- No `vaul` migration for the sidebar `SideSheet` or other dialogs.

## Decision Summary

### D1. Adaptive wrapper: `ResponsiveFormDialog`

New `src/components/ui/responsive-form-dialog.tsx`: a thin adapter that renders `MobileSheet` when `useIsMobile()` and the existing `Dialog`/`DialogContent` otherwise. API mirrors the dialog idiom the three modals already use:

```tsx
interface ResponsiveFormDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  title: string;                 // sheet header title AND DialogTitle
  initialHeight?: "half" | "full"; // mobile only; default "full"
  dialogClassName?: string;      // desktop DialogContent classes (e.g. "max-w-lg max-h-[90dvh] overflow-y-auto")
  children: ReactNode;
}
```

On mobile it maps `open/onOpenChange` to `MobileSheet`'s `isOpen/onClose` and passes `title` + `initialHeight`. On desktop it renders the `Dialog` exactly as today, including a `DialogHeader`/`DialogTitle` built from `title`. Children are the form content, written once.

Consequence for the three modals: their own header rows change on mobile — `MobileSheet` provides the title + Cancel affordance, so the modals' in-content `X` close buttons render only on desktop (the wrapper owns chrome; content stays chrome-free).

### D2. Heights per surface (follows the established short/long rule)

| Surface | Mobile height |
|---|---|
| `FamilySettingsModal` | `full` (long, multi-section) |
| `MemberProfileModal` | `full` (long; Google Calendar section) |
| `MemberFormModal` | `half` (short form; auto-expands on input focus) |

### D3. Stacking: sheet-over-sheet is accepted, with a fallback

`FamilySettingsModal` (sheet) opens `MemberFormModal` (sheet) and the reset confirm (centered dialog). These are independent portaled roots at `z-50`; later-mounted content paints above (the dialog-over-sheet case already ships today in event flows). Sheet-over-sheet must be verified on device. **Fallback if vaul misbehaves with two stacked drawers:** `MemberFormModal` keeps a centered `Dialog` on mobile too (with a `max-h-[90dvh] overflow-y-auto` bound added) — its short form fits a phone viewport, so the fallback still satisfies the story's goal.

### D4. Onboarding usage of `MemberFormModal` converts with it

The onboarding members step gets the same half-sheet on mobile via the shared component — consistent, and the sheet is keyboard-safer than the current unbounded dialog. The #194 onboarding screens themselves are untouched; a no-regression smoke is part of acceptance.

## Visual / Layout Contract

- **Mobile:** each surface is a standard `MobileSheet`: grab handle, header row (Cancel left, title center), scrollable content with safe-area bottom padding. Family Settings/Member Profile open full-height; Add/Edit Member opens half and expands on focus. No in-content `X` button on mobile.
- **Desktop:** all three render the exact current `DialogContent` (classes preserved via `dialogClassName`), including the in-content header rows with `X` close.
- **Confirm dialogs** (reset, sign-out): centered `Dialog` at all widths, unchanged.

## Interaction Model

- Open paths unchanged (sidebar member row → profile; sidebar Family Settings; Add/Edit member buttons; onboarding members step).
- Mobile dismissal: Cancel / scrim / Esc / flick-down (vaul defaults). Pending-mutation behavior unchanged from today (no new dirty-state guards in this story).
- Focusing any input in a half sheet expands it to full (existing `MobileSheet` behavior).

## Out of Scope

- Confirm dialogs, desktop sheet design, sidebar primitive, auth screens, backend.
- New settings features or form-content changes.

## Acceptance Criteria

- [ ] On a 360–430px viewport, Family Settings, Member Profile, and Add/Edit Member open as bottom sheets; no part of the surface renders off-screen, with and without the soft keyboard open.
- [ ] With the keyboard open, the focused field stays visible in all three surfaces (incl. Member Profile's email and the member form's name) — sheet auto-expand engages from half height.
- [ ] Add/Edit Member opens at half height; Family Settings and Member Profile open full height.
- [ ] Mobile dismissal works via Cancel, scrim, Esc, and flick-down; focus returns to the opener.
- [ ] Member Form opened from within Family Settings stacks correctly above it on device (or the documented fallback is applied and noted in the PR).
- [ ] Reset Family and Sign Out confirms remain centered dialogs and work from within/over the sheets.
- [ ] Desktop (≥768px): all three render as centered dialogs, visually unchanged.
- [ ] Onboarding members step still adds/edits members correctly on mobile and desktop; #194 auth/onboarding keyboard behavior unregressed (device smoke).
- [ ] All existing unit + E2E suites pass (updated where they assert dialog vs. sheet roles/structure).
