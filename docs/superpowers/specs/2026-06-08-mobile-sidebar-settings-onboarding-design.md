# Sidebar + Settings + Onboarding Mobile Pass

## Problem

The shell-level navigation got its mobile pass (persistent bottom nav, home redesign), but three surrounding surfaces never did: the **onboarding/login flow**, the **sidebar/menu**, and the **settings modals**. They work, but they're rough enough that two adults using the app daily on phones will hit friction every session:

- Onboarding/login screens use `100vh` and vertical centering, so the primary CTA can sit under the browser chrome and inputs get covered by the keyboard.
- The header carries a dead Settings button next to the working Menu button — two settings-looking affordances, one a no-op.
- The sidebar is a hand-rolled `<aside>` with no focus trap, no Esc-to-close, no swipe, a stale "Calendar Settings" label, and an unconfirmed Sign Out.
- The member-profile modal has no scroll bound, so a long form (avatar + color + email + Google Calendar section) can push its Save/Cancel buttons off-screen.
- The bottom nav renders **7 tabs in a horizontal scroller** (~490px of content on a ~390px screen), so Recipes/Photos are effectively undiscoverable.

This story is an **ergonomics + correctness** pass on those three surfaces. It is not a visual redesign and not a bottom-sheet rebuild.

## Current FE state (relevant constraints)

- **Viewport units:** every onboarding/login screen and the app shell use `min-h-screen`/`h-screen` (`100vh`), never `100dvh`. Only the calendar/home/FAB already handle `env(safe-area-inset-bottom)`.
  - `src/components/onboarding/{onboarding-welcome,onboarding-family-name,onboarding-members,onboarding-credentials}.tsx`
  - `src/components/auth/login-form.tsx`
  - `src/App.tsx` (`h-screen` shell)
- **Keyboard:** `login-form`, `onboarding-credentials`, `onboarding-family-name` center their form with `justify-center` and `autoFocus` the first field — keyboard covers the inputs.
- **Header dead button:** `src/components/shared/app-header.tsx:97-105` renders a `<Settings>` icon `<Button>` with no `onClick`. The working entry is the `Menu` (hamburger) button (`app-header.tsx:48-55`) which calls `openSidebar`. The mobile calendar has its own Menu button in `src/components/calendar/components/mobile-toolbar.tsx:90` (header is suppressed on mobile Calendar per `App.tsx:170`).
- **Sidebar:** `src/components/shared/sidebar-menu.tsx` is a custom `<aside>` (not Radix). `if (!isOpen) return null`; scrim is a `<div onClick>`. No focus trap, no Esc handler. Subtitle reads "Calendar Settings" (`sidebar-menu.tsx:66`). Sign Out calls `logout` directly with no confirm (`sidebar-menu.tsx:47`). It hosts the member list (→ `MemberProfileModal`), `Family Settings` (→ `FamilySettingsModal`), Sign Out, and version.
- **Settings modals:** both use the centered `DialogContent` (`src/components/ui/dialog.tsx:36-49`, `fixed left-1/2 top-1/2 -translate-*`). `FamilySettingsModal` has `max-h-[90vh] overflow-y-auto`; `MemberProfileModal` has **no** max-height/scroll and embeds `GoogleCalendarSection`.
- **Bottom nav:** `src/components/shared/mobile-bottom-nav.tsx` renders 7 tabs (`Home, Calendar, Lists, Chores, Meals, Recipes, Photos`) in `overflow-x-auto` with `min-w-16` each.
- **Existing primitives we will reuse:**
  - `src/components/ui/mobile-sheet.tsx` — `MobileSheet` full-screen bottom sheet (used by chores/lists/recipes/meals/calendar). The bottom-nav "More" overflow reuses this.
  - `@radix-ui/react-dialog` (already wrapped in `dialog.tsx`) — basis for the new left-anchored sidebar primitive.
  - `@radix-ui/react-alert-dialog` is installed but unwrapped; the existing reset-family confirm uses a plain `Dialog`, so Sign Out confirm matches that pattern (no new primitive).
- **Breakpoint:** `useIsMobile()` (`src/hooks/use-is-mobile.ts`) = `max-width: 768px`, reactive via `matchMedia`.

## Goals

- Onboarding/login are comfortable with the soft keyboard open; CTAs and inputs always reachable.
- One working, correctly-labeled settings entry on every mobile surface.
- The sidebar is a real, accessible dialog: focus-trapped, Esc/scrim/swipe to close, safe-area aware, ≥44px targets, with a confirmed Sign Out.
- Settings modals never hide their actions; they scroll and stay above the keyboard.
- The bottom nav shows ≤5 slots with a scalable "More" overflow; deferred surfaces don't occupy a tab.
- Leave desktop behavior unchanged. No backend changes.

## Non-goals

- Converting the settings dialogs to true bottom sheets — that is the separate [expandable-bottom-sheet](../../product/backlog/mobile-ux/expandable-bottom-sheet.md) story. This story only adds scroll + keyboard-safety to the existing modals.
- Migrating `MobileSheet` or the new sidebar onto a shared drawer library (e.g. `vaul`).
- Calendar interactions (drag-to-create, pinch-to-zoom), notifications, route-based navigation.
- Desktop side-rail / tablet shell changes.
- New settings *features* (themes, weather, server-side avatars).
- Re-pointing existing colors/spacing tokens (visual identity is already shipped).

## Decision Summary

### D1. `dvh` + safe-area on auth/onboarding/shell

Replace `min-h-screen`/`h-screen` with `min-h-dvh`/`h-dvh` on the four onboarding screens, `login-form`, and the `App.tsx` shell wrappers. Add `env(safe-area-inset-*)` padding to the onboarding/login containers (top for the back button, bottom for the CTA) and to the sidebar (see D4). Dynamic viewport units track the collapsing mobile toolbar so the CTA stays on-screen.

### D2. Keyboard-safe auth/onboarding forms

`login-form`, `onboarding-credentials`, and `onboarding-family-name` stop force-centering on mobile. On mobile they top-align inside a scroll container so the focused field scrolls above the keyboard; desktop keeps centered composition. `autoFocus` is **disabled on mobile** (gated by `useIsMobile()`) so the keyboard doesn't snap open over a still-centered layout on first paint. Submit buttons remain in normal flow at the bottom of the scroll container (not pinned), so they're reachable by scrolling past the keyboard.

### D3. One settings entry in the header

Remove the dead `<Settings>` button from `app-header.tsx`. The `Menu` (hamburger) button — already wired in both the header and the calendar `MobileToolbar` — becomes the single, consistent entry to the menu/settings surface. The sidebar's stale "Calendar Settings" subtitle is corrected to "Menu" (it hosts family + settings + sign-out, not calendar settings).

### D4. Sidebar rebuilt on Radix Dialog

Replace the hand-rolled `<aside>` with a new reusable **left-anchored** primitive `src/components/ui/side-sheet.tsx` built on `@radix-ui/react-dialog`. This buys focus-trap, Esc-to-close, scrim, scroll-lock, and labelled-dialog semantics for free, matching how `dialog.tsx` already wraps Radix. `SidebarMenu` is rebuilt on it.

The new sidebar must:
- Trap focus and restore it to the trigger on close; close on Esc and on scrim tap (Radix defaults).
- Support **swipe-to-close** (horizontal drag-to-dismiss) via a small touch handler in the primitive — the one piece requiring on-device verification.
- Respect safe-area insets top and bottom (`env(safe-area-inset-top/bottom)`).
- Use ≥44px touch targets for member rows, menu items, and the close button.
- Confirm **Sign Out** through a small `Dialog` confirm (mirroring the existing reset-family confirm), so an accidental tap next to "Family Settings" can't log the family out.
- Keep its existing z-layering relationship: scrim/backdrop at `z-40`, panel at `z-50`, consistent with the documented z-index hierarchy.

### D5. Settings modals — keyboard-safe, scroll-bounded (band-aid)

`MemberProfileModal` gains `max-h-[90vh] overflow-y-auto` parity with `FamilySettingsModal` so the Google Calendar section can never push Save/Cancel off-screen. Both modals must keep their focused input visible when the keyboard opens (the centered `DialogContent` + internal scroll is sufficient once max-height is bounded; verify on device). No structural conversion to sheets in this story.

### D6. Bottom nav ≤5 slots + scalable "More" overflow

Introduce a single ordered, navigable-module list with an explicit primary count. The bar renders **up to 5 slots**: the first N primary modules, and — when more navigable modules exist than fit — a 5th **"More"** slot that opens a `MobileSheet` listing the overflow modules. **Photos is removed from the nav entirely** (indefinitely deferred; its lazy view stays in code but is no longer navigable on mobile).

Concretely, navigable modules become: `Home, Calendar, Lists, Chores, Meals, Recipes` (6). Default split:
- **Primary (bar):** `Home, Calendar, Lists, Chores`
- **More sheet:** `Meals, Recipes`

The split is a single ordering constant — trivially reconfigurable. Selecting a module from More closes the sheet and switches `activeModule`. The "More" slot shows an active state when the current `activeModule` lives in the overflow set. This is the scalable pattern: future modules append to the list and overflow into More automatically.

## Visual / Layout Contract

- **Onboarding/login containers:** `min-h-dvh`, `p-4 md:p-6`, plus top padding `max(1rem, env(safe-area-inset-top))` and bottom padding `max(1rem, env(safe-area-inset-bottom))`. Mobile: content top-aligned in an `overflow-y-auto` column; desktop: keep `justify-center`.
- **Side sheet:** `w-[min(20rem,85vw)]`, anchored left, `100dvh` tall, `env(safe-area-inset-top/bottom)` padding on the header/footer. Slide-in-from-left animation. Scrim `bg-foreground/20 backdrop-blur-sm`.
- **More sheet:** reuse `MobileSheet` with `title="More"`; list overflow modules as full-width rows (icon + label), each ≥44px.
- **Bottom nav:** the 5-slot row no longer needs horizontal scroll on a 360–430px viewport; keep `min-h-14`, `env(safe-area-inset-bottom)`, `z-30`.

## Interaction Model

- Header/calendar Menu button → open side sheet. Esc / scrim / swipe-left → close.
- Member row → `MemberProfileModal`; "Family Settings" → `FamilySettingsModal`; "Sign Out" → confirm dialog → `logout`.
- Bottom-nav primary tab → `setActiveModule(id)`. "More" → open More sheet → pick module → `setActiveModule(id)` + close.
- No swipe-to-change-module; no route changes.

## Out of Scope

- Bottom-sheet rebuild of the settings dialogs (separate story).
- Shared drawer-library migration.
- Backend, routing, desktop/tablet shell, notifications, calendar interactions.
- Reaching the Photos surface on mobile (intentionally unreachable while deferred).

## Acceptance Criteria

- [ ] On a 360–430px viewport with the soft keyboard open, every onboarding and login input can be brought above the keyboard and the primary CTA is reachable; no screen depends on `100vh`.
- [ ] Onboarding/login and the sidebar respect safe-area insets (no content under the notch or home indicator).
- [ ] The header shows exactly one working settings/menu entry; the dead Settings button is gone; the sidebar subtitle no longer says "Calendar Settings".
- [ ] The sidebar is a focus-trapped dialog: Esc closes it, scrim tap closes it, swipe-left closes it, and focus returns to the trigger.
- [ ] Sign Out requires confirmation before logging out.
- [ ] All interactive controls in these surfaces are ≥44px touch targets.
- [ ] `MemberProfileModal` scrolls internally and never pushes Save/Cancel off-screen, including with the Google Calendar section expanded.
- [ ] The bottom nav shows ≤5 slots; navigable modules not in the primary set are reachable through a "More" sheet; the "More" slot reflects active state when an overflow module is active.
- [ ] Photos is not present in the bottom nav (primary or More).
- [ ] Desktop navigation, the desktop header, and all existing unit/E2E suites are unaffected (green).
</content>
