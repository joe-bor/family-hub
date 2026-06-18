# Native Hardware Back-Button — Design

> **Story:** `docs/product/backlog/mobile-ux/native-back-button.md`
> **Scope:** FE only. One hook + one registry + leaf-primitive registration. No new dependency, no router, **no new motion code** (reuses the sibling story's transition seams). Android installed-PWA only.

## Problem

In the installed PWA on Android, the hardware/gesture back button uses the browser default: a single press **exits the app**. Native apps instead treat back as "undo the last navigation" — dismiss the open overlay, step back a view, and only leave the app from a root screen (usually after a confirming second press). Surfaced from phone dogfooding (Galaxy S10 / Chrome, 2026-06-16).

Family Hub has **no router** — modules switch via `useAppStore.setActiveModule()` (per the persistent-bottom-nav spec, `docs/superpowers/specs/2026-04-26-persistent-bottom-nav-design.md`), and overlays/detail views are component state. So "back" can't lean on route history; it needs an explicit in-app notion of "what is dismissable right now" plus a controlled history shim to intercept the hardware button.

This story adds exactly that: a single back interceptor that walks a precedence cascade (overlay → up to Home → exit-with-hint), consuming the close handlers that already exist. It is the behavioral sibling of [native-feel-interaction-polish](../../product/backlog/mobile-ux/native-feel-interaction-polish.md) (which already left the seam) and pairs with [optional-haptics](../../product/backlog/mobile-ux/optional-haptics.md).

## Current FE state (verified 2026-06-17, paths relative to `frontend/`)

The "dismissable universe" a back press must drive is stored **three different ways** — this is the central design constraint:

- **Module state (no router)** — `src/stores/app-store.ts`: `activeModule: ModuleType | null` (`:30`, `null` = Home dashboard, mobile-only), `setActiveModule` (`:54`), `isSidebarOpen` (`:49`) / `closeSidebar` (`:70`). The store uses a **plain `create`** (no `persist` middleware) → `activeModule` resets to `null` (Home) on every refresh.
- **Shared overlay primitives (controlled `open`/`onClose`)**:
  - `src/components/ui/mobile-sheet.tsx` — `MobileSheet({ isOpen, onClose })` (`:35`), vaul `Drawer.Root` with `onOpenChange((open) => { if (!open) onClose() })` (`:89-91`). vaul self-closes on swipe/overlay-tap/Escape, **not** on `popstate`.
  - `src/components/ui/responsive-form-dialog.tsx` — `ResponsiveFormDialog({ open, onOpenChange })` renders `MobileSheet` on mobile (`isMobile && !isMdOrLarger`) and `Dialog` on desktop. It **delegates** to `MobileSheet` on mobile, so it needs no registration of its own.
  - `src/components/ui/side-sheet.tsx` — `SideSheet({ open, onOpenChange })` (`:14-20`), Radix-`Dialog`-based with a left-swipe-to-close gesture.
  - `src/components/ui/dialog.tsx` — `Dialog` is the **raw** `DialogPrimitive.Root` re-export (`:6`); open-state lives at each call site.
- **Store-backed overlays**:
  - Sidebar — `SidebarMenu` (`src/components/shared/sidebar-menu.tsx`) wraps `SideSheet`, driven by `isSidebarOpen`/`closeSidebar`. Also holds a nested sign-out-confirm raw `<Dialog>`.
  - Event detail — `src/stores/calendar-store.ts`: `selectedEvent` (`:17`), `isDetailModalOpen` + `openEventDetail`/`closeEventDetail` (`:220-222`), selector `useEventDetailState` (`:315`). Rendered by `event-detail-modal.tsx` as a raw `<Dialog open={isOpen} onOpenChange={(open) => !open && onClose()}>` (used from both `calendar-module.tsx` and `home-dashboard.tsx`).
- **Local-`useState` detail views / sheets**:
  - `src/components/lists-view.tsx` — `selectedListId` (`:14`); back is `setSelectedListId(null)` (`:34`); already wrapped in `ScreenTransition` keyed on `selectedListId ?? "__list__"` (`:164-170`).
  - `src/components/recipes-view.tsx` — `selectedRecipeId` (`:87`); back is `setSelectedRecipeId(null)` (`:194-197`); already wrapped in `ScreenTransition` (`:175-179`).
  - "More" sheet — `moreOpen` local state in `mobile-bottom-nav.tsx` (`:43`), a `MobileSheet` (`:107-109`).
  - meals/chores open **sheets** (`MobileSheet`, covered by the primitive), not in-place detail views — **but** the meals module also renders raw `<Dialog>` collision/move confirms as *siblings* of the sheet (`meal-composer-sheet.tsx:377` `collisionRequest`, `meal-editor-sheet.tsx:413` `pendingCollision`, `meal-move-picker.tsx:62` `open`), which the sheet primitive does **not** cover and which therefore need explicit registration (see D4).
- **Raw `<Dialog>` / custom-mobile call sites** (open-state at the call site → explicit wiring). Register **once per overlay**: a surface whose *mobile* path is `MobileSheet`/`SideSheet` is already covered by the leaf primitive and must NOT self-register. Sites needing explicit registration: `event-detail-modal.tsx` (its mobile path is the **custom** `MobileEventDetail`, not a sheet — early returns at `:61`/`:79`), `edit-scope-dialog.tsx`, `settings/google-calendar-picker-modal.tsx`, the family-settings reset-confirm (`:255`), the sidebar sign-out-confirm (`:198`), and the meals confirms (`meal-composer-sheet.tsx:377`, `meal-editor-sheet.tsx:413`, `meal-move-picker.tsx:62`). **Excluded:** `event-form-modal.tsx` — its mobile path is `MobileEventSheet` → `MobileSheet` (already covered; registering it too would double-register).
- **PWA + platform detection already exists** — `src/lib/pwa.ts`: `isStandalone()` (`:15`, matches `display-mode: standalone` **or** iOS `navigator.standalone`) and `isIOS()` (`:25`). Note `isStandalone()` is `true` on iOS too, so the Android gate must be `isStandalone() && !isIOS()`.
- **Toast is imperative and already mounted** — `src/components/ui/toaster.tsx`: `toast({ description, duration })` (`:40`) honors a custom `duration` (`:63-69`); `<Toaster />` is at `App.tsx:180`. The codebase already uses custom-duration toasts (the PWA update prompt).
- **App root** — `src/App.tsx`: authenticated shell at `:162-182`; `isAuthenticated` (`:107`), `setupComplete` (`:108`), `isMobile` (`:109`) are all computed **before** the early returns; module render is wrapped in `<ScreenTransition token={activeModule} mode="fade">` (`:170`).
- **Barrels & test infra** — `src/hooks/index.ts` and `src/stores/index.ts` are the export barrels; `src/test/setup.ts`'s `resetAllStores()` resets stores **by explicit name** (not generically — a new store must be added there) and mocks `matchMedia`.

## Goals

- In the installed Android PWA, a single back press dismisses the top-most open overlay/detail; with nothing open, it steps **up to Home**; at Home it shows a transient hint and the **second** press (within ~2s) exits.
- Drive the **existing** close handlers (`onClose`, `onOpenChange(false)`, `setSelectedListId(null)`, `setActiveModule(null)`) — so the sibling story's `activeModule` fade and detail-view slide animate the back **for free**, and this story ships **no transition code**.
- One interception seam: a single hook owns all history/`popstate` manipulation; surfaces only declare "I'm dismissable now."
- No back-trap loops, no history leaks; refresh lands cleanly on Home with the buffer intact.
- Gated precisely to Android standalone; browser tabs, iOS standalone (edge-swipe), and desktop are unaffected no-ops.

## Non-goals

- No router and no change to how `activeModule`/detail/overlay state is stored (the registry is additive).
- No explicit **module-history stack** — the model is single-root ("up to Home"), not "previous tab" (see D2; deferred as a possible future enhancement).
- No new transition/animation code — consumes the seams from `2026-06-16-native-feel-interaction-polish-design.md` (its D5).
- No haptics — [optional-haptics](../../product/backlog/mobile-ux/optional-haptics.md) still owns `usePressable().onPointerDown`, untouched here.
- No back-handling for the pre-auth **login/onboarding** flows (their own step-back UI is a separate future story; v1 mounts the interceptor only in the authenticated shell).
- No iOS / desktop behavior change.

---

## Decision Summary

### D1. Architecture: one interceptor hook + a LIFO dismiss-registry

Because the dismissable state is **scattered** (local `useState` + two stores), a central handler can't *read* it all — so surfaces **register** a dismiss handler while they are open, and the interceptor pops the most-recent one. This collapses the story's open question ("explicit stack vs. priority resolver") into a single registry: a registration-fed LIFO stack *is* the priority resolver.

**`src/stores/back-stack-store.ts`** (Zustand for codebase consistency; note `src/test/setup.ts`'s `resetAllStores()` resets stores by **explicit name**, not generically — so the plan adds `useBackStack` to that reset list, otherwise its `stack` leaks across tests):

```ts
import { create } from "zustand";

export interface BackHandlerEntry { id: number; handler: () => void; }

interface BackStackState {
  stack: BackHandlerEntry[];
  register: (handler: () => void) => number; // returns id
  unregister: (id: number) => void;
  peek: () => BackHandlerEntry | undefined;
}

let nextId = 0;

export const useBackStack = create<BackStackState>((set, get) => ({
  stack: [],
  register: (handler) => {
    const id = nextId++;
    set((s) => ({ stack: [...s.stack, { id, handler }] }));
    return id;
  },
  unregister: (id) => set((s) => ({ stack: s.stack.filter((e) => e.id !== id) })),
  peek: () => { const { stack } = get(); return stack[stack.length - 1]; },
}));
```

**`src/hooks/use-back-handler.ts`** — the per-surface seam. Registers a *stable* wrapper that always calls the latest handler (no stale closures), re-registering only when `enabled` flips:

```ts
import { useEffect, useRef } from "react";
import { useBackStack } from "@/stores";

/** Register a dismiss handler on the back-stack while `enabled`. Hardware back
 *  (see useAndroidBackButton) pops the most-recently-registered first (LIFO). */
export function useBackHandler(enabled: boolean, handler: () => void): void {
  const handlerRef = useRef(handler);
  handlerRef.current = handler;
  const register = useBackStack((s) => s.register);
  const unregister = useBackStack((s) => s.unregister);
  useEffect(() => {
    if (!enabled) return;
    const id = register(() => handlerRef.current());
    return () => unregister(id);
  }, [enabled, register, unregister]);
}
```

Registration is unconditional across platforms (cheap; off-Android the registry is simply never *consumed*), which keeps the hook trivially testable and free of platform branches.

### D2. The back-press cascade (single-root) + sentinel mechanics

**`src/hooks/use-android-back-button.ts`** — mounted **once** in the authenticated shell; the only place history is touched. On mount (gated) it pushes one sentinel so history is `[appEntry, sentinel]`; each hardware back fires one `popstate` (after the browser has already popped the sentinel), which runs the cascade:

```
1. registry non-empty?       → peek().handler()  (close top overlay/detail)  → re-push sentinel
2. else activeModule !== null? → setActiveModule(null)  (up to Home)          → re-push sentinel
3. else (Home, nothing open):
     first press   → toast("Press back again to exit", 2000ms) + arm 2s timer → re-push sentinel
     second press within 2s → history.back()  (actually exit; do NOT re-push)
```

```ts
import { useEffect, useRef } from "react";
import { toast } from "@/components/ui/toaster";
import { isIOS, isStandalone } from "@/lib/pwa";
import { useAppStore, useBackStack } from "@/stores";

const EXIT_HINT_MS = 2000;
const SENTINEL = { __familyHubBack: true } as const;

function isAndroidBackContext(): boolean {
  if (typeof window === "undefined") return false;
  const coarse = window.matchMedia?.("(pointer: coarse)").matches === true;
  return isStandalone() && !isIOS() && coarse; // exclude iOS standalone + desktop PWAs
}

/** Hardware/gesture back for the installed Android PWA. Mount once. */
export function useAndroidBackButton(enabled: boolean): void {
  const armedRef = useRef(false);
  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  useEffect(() => {
    if (!enabled || !isAndroidBackContext()) return;
    const rebuffer = () => window.history.pushState(SENTINEL, "");
    rebuffer(); // initial: [appEntry, sentinel]

    const onPop = () => {
      const top = useBackStack.getState().peek();
      if (top) { top.handler(); rebuffer(); return; }
      const { activeModule, setActiveModule } = useAppStore.getState();
      if (activeModule !== null) { setActiveModule(null); rebuffer(); return; }
      if (armedRef.current) {
        armedRef.current = false;
        if (timerRef.current) clearTimeout(timerRef.current);
        setTimeout(() => window.history.back(), 0); // defer past this popstate
        return;
      }
      armedRef.current = true;
      toast({ description: "Press back again to exit", duration: EXIT_HINT_MS });
      timerRef.current = setTimeout(() => { armedRef.current = false; }, EXIT_HINT_MS);
      rebuffer();
    };

    window.addEventListener("popstate", onPop);
    return () => {
      window.removeEventListener("popstate", onPop);
      if (timerRef.current) clearTimeout(timerRef.current);
    };
  }, [enabled]);
}
```

**Why single-root, not a module-history stack:** Family Hub is a 6-tab bottom-nav app whose mobile root is Home (`null`). "Up to Home, then exit" matches Android's "up to the start destination," needs no history bookkeeping, and is immune to the edge cases a tab-history stack would invite (re-tap dedupe, unbounded growth, and the **draft-driven jumps** — `startMealPlacementFromRecipe`/`startRecipeCreationFromMealSlot` set `activeModule` programmatically and would pollute a literal stack). Reading `getState()` inside `onPop` avoids stale closures.

**Reused motion:** step 2's `setActiveModule(null)` rides the existing `ScreenTransition token={activeModule}` fade (`App.tsx:170`); step 1 closing a lists/recipes detail rides the existing detail slide keyed on selection (`lists-view.tsx:164-170`, `recipes-view.tsx:175-179`). **No motion code in this story** — this is exactly the sibling spec's D5.

### D3. Correctness: no leaks, no traps, refresh-safe

- **One buffer entry, always.** Each handled back pops the sentinel then re-pushes it → history length stays ~2, so there is no growth/leak no matter how many overlays open. `pushState` replaces forward entries, so no forward-leak either.
- **The interceptor never mutates the registry.** It `peek()`s the top handler and calls it; the surface then closes and **unregisters itself** via its own `useBackHandler` effect cleanup. This avoids the interceptor evicting a surface that didn't actually close, and close handlers are idempotent, so the only edge (two `popstate`s fired within one React frame, ~16ms — far below real button cadence) at worst replays a no-op close.
- **Exit is explicit.** Only the confirmed second root-press skips the re-push and calls `history.back()` (deferred via `setTimeout(…, 0)` so it runs after the current `popstate` settles), popping `appEntry` to leave the PWA. Net: a true two-press exit.
- **Refresh / "deep-link".** With no router and an unpersisted `activeModule`, every load is Home with a freshly pushed sentinel — the AC is satisfied by construction; there is no route/URL view state to desync. Overlays are component/store state that also resets on reload, so nothing stale survives.
- **No double-handling with vaul/Radix.** Android hardware back fires `popstate`, not a synthetic `Escape`, so vaul/Radix never self-close on it — our handler is the sole driver. When a surface closes via its *own* affordance (swipe/overlay-tap/Escape), it simply unregisters; no `popstate`, no sentinel churn.
- **Focus.** Closing via back goes through each surface's existing `onClose`, so existing focus-restoration (e.g. `MobileSheet`'s `onCloseAutoFocus` → opener) is preserved.
- **Service worker.** The PWA service worker caches assets; it does not touch `history`/`popstate`, so it cannot interfere.

### D4. Registration wiring (minimize call-site churn)

Register at the **leaf primitives** (covers the bulk with no consumer changes), plus the detail views and the enumerable raw `<Dialog>` sites:

- `MobileSheet` → `useBackHandler(isOpen, onClose)`. Covers the "More" sheet **and** every `ResponsiveFormDialog` on mobile (it delegates to `MobileSheet`, so RFD itself is not touched → no double-registration), plus meal sheets.
- `SideSheet` → `useBackHandler(open, () => onOpenChange(false))`. Covers the sidebar.
- `lists-view.tsx` / `recipes-view.tsx` → `useBackHandler(selectedListId !== null, () => setSelectedListId(null))` (and the recipe equivalent). Exactly the sibling D5 close handlers, so the reverse slide is free.
- **Raw `<Dialog>` / custom-mobile sites** (one line each, placed with the other hooks **above any early returns** to respect the Rules of Hooks): `event-detail-modal.tsx` (`isOpen`/`onClose`), `edit-scope-dialog.tsx` (`isOpen`/`onClose`), `google-calendar-picker-modal.tsx` (`open`/`onOpenChange`), the family-settings reset-confirm (`showResetConfirm`), the sidebar sign-out-confirm (`showSignOutConfirm`), and the meals confirms (`meal-composer-sheet.tsx` `collisionRequest`, `meal-editor-sheet.tsx` `pendingCollision`, `meal-move-picker.tsx` `open`). **Register once:** skip surfaces already covered by `MobileSheet`/`SideSheet` on mobile — notably `event-form-modal.tsx` (mobile → `MobileEventSheet` → `MobileSheet`). Run `rg -n "<Dialog\b" src` to confirm no raw site is missed.

*Considered & rejected:* dispatching a synthetic `Escape` on back to let Radix/vaul self-close — it would cover all dialogs in one place but can't be ordered against the detail/module layers and is focus-brittle. The explicit registry is testable and deterministic.

### D5. Exit hint
`toast({ description: "Press back again to exit", duration: 2000 })` — the 2s duration *is* the double-press window. Reuses the mounted `<Toaster />`; Radix-Toast a11y (role/aria-live) is already wired; no new component.

### D6. Activation gate
`isStandalone() && !isIOS() && matchMedia("(pointer: coarse)")` — installed **and** Android-class **and** touch. Browser tabs, iOS standalone (edge-swipe), and desktop-installed PWAs (mouse "back") are all no-ops. The interceptor is enabled only in the authenticated shell (`enabled = isAuthenticated && setupComplete`); it cleans up its listener and timer on unmount (logout).

---

## File map

| File | Action | Purpose |
|---|---|---|
| `src/stores/back-stack-store.ts` | Create | `useBackStack` LIFO registry of dismiss handlers |
| `src/stores/index.ts` | Modify | Export `useBackStack` |
| `src/test/setup.ts` | Modify | Add `useBackStack` reset to `resetAllStores()` (resets by explicit name, not generically) |
| `src/hooks/use-back-handler.ts` | Create | `useBackHandler(enabled, handler)` per-surface seam |
| `src/hooks/use-android-back-button.ts` | Create | Single `popstate`/sentinel interceptor + cascade + exit hint |
| `src/hooks/index.ts` | Modify | Export both hooks |
| `src/App.tsx` | Modify | `useAndroidBackButton(isAuthenticated && setupComplete)` (call above early returns) |
| `src/components/ui/mobile-sheet.tsx` | Modify | `useBackHandler(isOpen, onClose)` |
| `src/components/ui/side-sheet.tsx` | Modify | `useBackHandler(open, () => onOpenChange(false))` |
| `src/components/lists-view.tsx` | Modify | Register `setSelectedListId(null)` when a list is open |
| `src/components/recipes-view.tsx` | Modify | Register `setSelectedRecipeId(null)` when a recipe is open |
| `src/components/calendar/components/event-detail-modal.tsx` | Modify | `useBackHandler(isOpen, onClose)` above its `:61`/`:79` early returns (custom mobile path) |
| `src/components/calendar/components/edit-scope-dialog.tsx` | Modify | `useBackHandler(isOpen, onClose)` |
| `src/components/settings/google-calendar-picker-modal.tsx` | Modify | `useBackHandler(open, () => onOpenChange(false))` |
| `src/components/settings/family-settings-modal.tsx` | Modify | `useBackHandler` on the reset-confirm (`showResetConfirm`) |
| `src/components/shared/sidebar-menu.tsx` | Modify | `useBackHandler` on the sign-out-confirm (`showSignOutConfirm`) |
| `src/components/meals/meal-composer-sheet.tsx` | Modify | `useBackHandler(collisionRequest !== null, () => setCollisionRequest(null))` |
| `src/components/meals/meal-editor-sheet.tsx` | Modify | `useBackHandler(pendingCollision !== null, () => setPendingCollision(null))` |
| `src/components/meals/meal-move-picker.tsx` | Modify | `useBackHandler(open, () => onCancel())` |

(`event-form-modal.tsx` is intentionally **not** listed — its mobile path is `MobileSheet`, covered by the leaf primitive.)

## Testing strategy

- **Unit (Vitest) — `useBackStack`:** register pushes; `peek` returns the last registered; `unregister` removes by id; LIFO order preserved across multiple registrations.
- **Unit — `useBackHandler`:** registers while `enabled`, unregisters on disable/unmount; a re-render with a new handler still invokes the **latest** handler (stale-closure guard).
- **Unit — `useAndroidBackButton` cascade:** mock the gate to `true` (mock `pwa.isStandalone`/`isIOS` and `matchMedia("(pointer: coarse)")`); spy on `history.pushState`/`history.back`; dispatch `new PopStateEvent("popstate")` and assert each branch:
  1. with a registered handler → handler called + one re-push, `history.back` not called;
  2. empty registry + `activeModule="calendar"` → `setActiveModule(null)` + re-push;
  3. at Home, first pop → `toast` fired + re-push, `history.back` not called; second pop within window (fake timers) → `history.back` called, no re-push; after the 2s timer the arm resets (third pop re-shows the hint).
  - **Gate off:** when not Android-standalone, mount does nothing (no `pushState`, no listener).
- **Integration (Vitest + Testing Library):** render `MobileSheet` open inside a harness that also mounts `useAndroidBackButton(true)` (gate mocked on) → dispatch `popstate` → its `onClose` runs. A refresh-shape test: fresh mount pushes exactly one sentinel; `activeModule` defaults to `null`.
- **Manual device pass (the real gate):** Galaxy S10 / Chrome, installed PWA — open a list detail: back closes it (slides left), back again goes Home (fades), back shows the hint toast, back again exits; repeat with a sheet, a dialog-in-dialog (confirm closes before its parent), and the sidebar; verify a single browser-tab session and iOS are unaffected.
- **E2E (Playwright):** limited — Playwright can't emulate `display-mode: standalone`, so standalone-gated behavior isn't E2E-testable. Optionally assert the **negative**: in a normal (non-standalone) context, `page.goBack()` does not trap. Primary coverage is unit + manual.
- Follow `frontend/CLAUDE.md` test gotchas: `src/test/setup.ts`'s `resetAllStores()` resets stores by explicit name, so **`useBackStack` must be added there** (otherwise its `stack` leaks across tests in a file); use fake timers for the 2s window; the existing `matchMedia` mock in `src/test/setup.ts` must be extended per-test for `(pointer: coarse)` / `(display-mode: standalone)`.

## Out of scope

Router; module-history ("previous tab") stack; login/onboarding back-handling; haptics; iOS / desktop behavior; any new motion system or change to existing transitions; per-overlay history entries (the single-sentinel + registry model replaces them).

## Acceptance criteria

(Mirror of the story.)

- In standalone (installed Android) mode, a single back press first dismisses the top-most open bottom sheet, dialog, or sidebar (LIFO); if none is open and the active module is not Home, it goes **up to Home**; at Home it does not exit on the first press.
- At Home with nothing open, the first back press shows a transient "Press back again to exit" toast; a second back press within ~2s exits the app.
- Gated to Android standalone (`isStandalone() && !isIOS() && pointer:coarse`); browser tabs, iOS standalone (edge-swipe), and desktop are unaffected.
- No back-trap loops or history leaks; refresh lands on Home with the sentinel buffered.
- Drives the existing close handlers only, so the sibling story's `activeModule` fade and detail-view slide animate the back with **no new transition code**.

## Self-review notes

- **Placeholders:** none — every change names a real file/symbol verified 2026-06-17; the three new primitives have full sketches.
- **No new dependency:** reuses Zustand, the existing `toast`, `pwa.ts`, and the History API; no router, no motion lib.
- **Scattered-state insight made explicit:** the registry exists *because* dismiss state is split across local `useState` + two stores; a pure read-existing-state resolver was impossible, so both of the story's Q1 options converge on the registry (which also fixes precedence as LIFO).
- **Single-root resolved unambiguously:** "up to Home, then exit," not a tab-history stack — stated as a non-goal so it isn't mistaken for full history; rationale (draft-driven `activeModule` jumps) recorded.
- **Correctness nailed:** single re-pushed sentinel (no leak), deferred `history.back()` for true two-press exit, refresh→Home by construction (unpersisted store), no `Escape`/`popstate` double-handling.
- **Seam honesty:** consumes the sibling's D5 seam; ships zero motion code and leaves `usePressable().onPointerDown` for optional-haptics untouched.
- **Churn honesty:** leaf-primitive registration covers sheets/sidebar/RFD with no call-site changes; the 8 raw-`<Dialog>`/custom-mobile one-liners (5 calendar/settings/sidebar + 3 meals confirms) are unavoidable because Radix `Dialog` open-state lives at the call site — enumerated, hooks placed above early returns, with an `rg` audit note.
- **Register-once rule (post-review):** a surface whose mobile path is `MobileSheet`/`SideSheet` is auto-covered and must NOT self-register; `event-form-modal` is excluded for exactly this reason, while `event-detail-modal` self-registers because its mobile path is a custom overlay.
- **Test-harness reset (post-review):** `src/test/setup.ts` resets stores by explicit name, so `useBackStack` is added to `resetAllStores()` — without it the registration tests would leak `stack` state across a file.
