# Optional Haptic Feedback (Vibration API) — Design

> **Story:** `docs/product/backlog/mobile-ux/optional-haptics.md`
> **Scope:** FE only. One helper + one persisted store + one Switch primitive + a Preferences section. **Reuses the two existing seams** (`usePressable().onPointerDown` from PR #230, `useAndroidBackButton`'s `onPop` cascade from PR #234). No new motion code, no new runtime dependency. Android installed-PWA only; iOS Safari + desktop are silent no-ops.

## Problem

Touch interactions in the installed Android PWA are visually acknowledged (press scale + tint from `usePressable`, screen transitions) but not *tactile*. Native apps add a subtle vibration on key touches — a tap, a completion, a back-dismiss — which makes the app feel physical. Surfaced from phone dogfooding (Galaxy S10 / Chrome, 2026-06-16).

`navigator.vibrate()` does real work only on touch devices like Android Chrome PWAs; iOS Safari omits it entirely and desktop Chrome/Edge/Firefox expose only a silent no-op. So haptics are a **progressive enhancement**: present where capable (Vibration API **and** a coarse pointer), a silent no-op everywhere else. This story **supersedes** the "No haptics (PWA limits)" exclusion in the home-dashboard redesign spec (`docs/superpowers/specs/2026-04-25-home-dashboard-redesign-design.md` — that note predates the Android-Chrome dogfood finding).

This is the behavioral sibling of [native-feel-interaction-polish](2026-06-16-native-feel-interaction-polish-design.md) (PR #230, which left `usePressable().onPointerDown` as an explicit haptic seam) and pairs with [native-back-button](2026-06-17-native-back-button-design.md) (PR #234, whose `onPop` cascade is the single central back-dismiss point). Both PRs deliberately left seams this story consumes — it adds **no** new press/motion/back infrastructure.

## Current FE state (verified 2026-06-18, paths relative to `frontend/`)

- **Tap seam already exists** — `src/hooks/use-pressable.ts`: `usePressable()` returns `{ className, onPointerDown }`; `onPointerDown` is a `useCallback` (`:22-24`) whose body is a literal placeholder comment: *"Haptic seam: the optional-haptics story adds `useHaptics().tap()` here."* It is already spread through `button.tsx`, `mobile-bottom-nav.tsx`, `sidebar-menu.tsx`, and the list/recipe row cards. **One `haptics.tap()` call here covers every primary-action surface with zero call-site churn.**
- **Back/dismiss seam already exists** — `src/hooks/use-android-back-button.ts` (PR #234, on `main`): mounted once in the authenticated shell (`App.tsx:116`, `useAndroidBackButton(isAuthenticated && setupComplete)`). Its `onPop` walks a cascade: **(1)** registered overlay/detail handler → `top.handler()` + rebuffer; **(2)** `activeModule !== null` → `setActiveModule(null)` (up to Home) + rebuffer; **(3)** at Home → first press shows an exit-hint toast, second press within 2s exits. Steps 1 and 2 are the *handled back-dismiss* points; the toast/exit branch is not a dismiss.
- **Capability/platform detection** — `src/lib/pwa.ts`: `isStandalone()` (`:15`), `isIOS()` (`:25`). These detect *install context*, not vibration capability — so this story adds a separate, narrowly-scoped `canVibrate()` (feature detection of `navigator.vibrate`).
- **Persisted-store pattern to mirror** — `src/stores/calendar-store.ts`: Zustand `persist()` with `name: "family-hub-calendar"` (`:234`) and an explicit `partialize` (`:236-240`). Storage keys follow the `family-hub-*` convention (`src/lib/constants.ts:12,18,24`). No generic "preferences" store exists yet — `preferences-sheet.tsx` only manages the BE-backed family timezone, so this story adds a new client-only store.
- **Preferences surface** — `src/components/settings/preferences-sheet.tsx`: a `ResponsiveFormDialog` with a "Family Timezone" section and a disabled "Coming Soon" stub section. Rendered from `src/components/shared/sidebar-menu.tsx:184`. This is where the haptics toggles go.
- **Switch primitive** — **none exists** (`src/components/ui/` has no `switch.tsx`/`toggle.tsx`), but **`@radix-ui/react-switch ^1.3.1` is already a dependency**. So this story adds a thin shadcn-style `Switch` wrapper — no new dependency.
- **Completion call sites** (each a leaf row whose tap toggles done-state; **neither uses `usePressable`**, verified — so a `success()` pulse will not double-fire with a `tap()`):
  - Lists — `src/components/lists/list-item-row.tsx:24`: `onClick={() => onToggle(!item.completed)}`. Completing transition is `!item.completed === true`.
  - Chores — `src/components/chores/chore-row.tsx:42`: `onClick={chore.completed ? onUncomplete : onComplete}`. Completing path is `onComplete` (when `!chore.completed`).
- **Reduced-motion hook** — `src/hooks/use-prefers-reduced-motion.ts` exists. Per the decision below, haptics are **independent** of it (not consumed here).
- **Test harness** — `src/test/setup.ts`'s `resetAllStores()` resets stores **by explicit name** (not generically): `useFamilyStore`, `useCalendarStore`, `useAppStore`, `useAuthStore`, and (post-#234) `useBackStack`. A new store leaks its state across tests until added there. `beforeEach` does `localStorage.clear()`; `matchMedia` is mocked. `navigator.vibrate` is **not** mocked (jsdom lacks it → `canVibrate()` is `false` by default in tests, which is the correct "no-op" baseline).

## Goals

- A subtle vibration fires on a defined, **per-device-configurable** set of touch interactions in the installed Android PWA: primary-action **taps**, task/list **completions**, and hardware **back**-dismiss.
- A **single centralized helper** (`haptics.tap()/success()/back()`) is the only code that ever calls `navigator.vibrate` — no scattered calls.
- Every haptic is a **silent no-op** unless: `navigator.vibrate` exists **AND** the master toggle is on **AND** that interaction's category is on. iOS Safari and desktop are entirely unaffected.
- A persisted **per-device** preference (localStorage), default **off**, exposed in Preferences as a master switch plus per-category sub-switches.
- **Reuse the seams, build no new infrastructure:** `usePressable().onPointerDown` for taps; the `useAndroidBackButton` cascade for back; the two completion rows for completions.
- Short, subtle single/double pulses; guarded against overuse.

## Non-goals

- No `warning()` / destructive-confirm haptic in v1 (deferred — see Out of scope). The helper stays trivially extensible.
- No bottom-sheet-snap pulse (deferred — too frequent, high noise risk).
- No synced/server-side preference — per-device localStorage only.
- No coupling to `prefers-reduced-motion` — the toggle is the single source of truth (haptics are tactile, not visual motion).
- No iOS / desktop behavior change; no new motion, press, or back-button infrastructure (all reused).
- No change to how `activeModule`/detail/overlay state is stored.

---

## Decision Summary

### D1. Centralized helper: imperative `haptics` object (not a hook)

**`src/lib/haptics.ts`** is the single integration point and the only caller of `navigator.vibrate`. It is an **imperative object** (like the existing `toast`), not a `useHaptics()` hook, because two of the three fire-sites are **not** React render contexts:

- `useAndroidBackButton`'s `onPop` fires from a `popstate` **event listener**.
- Completions fire from row `onClick` handlers.

An imperative helper reading `useHapticsPreference.getState()` works uniformly in all three contexts (hook seam, event listener, click handler) with no hook-context constraints.

```ts
// src/lib/haptics.ts
import { useHapticsPreference } from "@/stores/haptics-store";

/** Short, subtle patterns (ms). Single pulses for tap/back; a brief double for success. */
const PATTERNS = {
  taps: 10,
  completions: [12, 40, 12],
  back: 8,
} as const;

const THROTTLE_MS = 40; // guard against pointerdown storms / rapid repeats (overuse AC)
let lastFireAt = 0;

/**
 * Detect a device that can actually deliver haptics: the Vibration API AND a
 * touch-primary (coarse) pointer. Android Chrome → true; iOS Safari → false (no
 * `navigator.vibrate`); desktop Chrome/Edge/Firefox → false too, because they
 * define a *no-op* `navigator.vibrate` on mouse-primary hardware.
 */
export function canVibrate(): boolean {
  if (typeof navigator === "undefined" || typeof navigator.vibrate !== "function") {
    return false;
  }
  return window.matchMedia?.("(pointer: coarse)").matches === true;
}

function fire(category: keyof typeof PATTERNS): void {
  if (!canVibrate()) return;
  const { enabled, categories } = useHapticsPreference.getState();
  if (!enabled || !categories[category]) return;
  const now = Date.now();
  if (now - lastFireAt < THROTTLE_MS) return; // coalesce rapid repeats
  lastFireAt = now;
  navigator.vibrate(PATTERNS[category]);
}

/** The only sanctioned way to vibrate. Each method is a no-op unless capable + opted-in. */
export const haptics = {
  tap: () => fire("taps"),
  success: () => fire("completions"),
  back: () => fire("back"),
};
```

- **Gate order is capability → master → category → throttle.** Any failing gate is a silent return; nothing throws on unsupported platforms.
- **`canVibrate()` lives in `haptics.ts`** (not `pwa.ts`): it is vibration-capability + touch-primary detection (`navigator.vibrate` **and** `(pointer: coarse)`), distinct from `pwa.ts`'s install-context detection, and is only consumed by haptics + the Preferences visibility gate.
- **One shared throttle** across categories is sufficient: completion rows don't fire `tap()` (verified — they lack `usePressable`), and `back()` fires from `popstate`, so categories don't collide within 40ms in practice. (Per-category throttling is YAGNI.)
- **Patterns are deliberately subtle:** `tap` 10ms, `back` 8ms (single brief pulses); `success` a short `[12, 40, 12]` double so a completion feels distinct from a tap. Final values are confirmed on-device (the manual gate), not asserted as "feel" in tests.

### D2. Preference store: `useHapticsPreference` (persisted, per-device, default off)

**`src/stores/haptics-store.ts`** — Zustand `persist`, mirroring `calendar-store`'s pattern. Master `enabled` defaults **off**; the three category switches default **on** (so flipping the master on enables everything, and the user dials back the noisy ones).

```ts
// src/stores/haptics-store.ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

export type HapticCategory = "taps" | "completions" | "back";

interface HapticsPreferenceState {
  enabled: boolean;                                 // master, default false
  categories: Record<HapticCategory, boolean>;      // each default true
  setEnabled: (on: boolean) => void;
  setCategory: (category: HapticCategory, on: boolean) => void;
}

/** localStorage key, exported for the test-harness reset (see D5). */
export const HAPTICS_STORAGE_KEY = "family-hub-haptics";

export const useHapticsPreference = create<HapticsPreferenceState>()(
  persist(
    (set) => ({
      enabled: false,
      categories: { taps: true, completions: true, back: true },
      setEnabled: (on) => set({ enabled: on }),
      setCategory: (category, on) =>
        set((s) => ({ categories: { ...s.categories, [category]: on } })),
    }),
    {
      name: HAPTICS_STORAGE_KEY,
      partialize: (s) => ({ enabled: s.enabled, categories: s.categories }),
    },
  ),
);
```

Exported from the `src/stores/index.ts` barrel. The helper (D1) reads it via `getState()` (non-reactive, correct for fire-and-forget); the Preferences UI (D4) reads/writes it reactively.

### D3. Activation gate: capability + touch-primary (no install/context gate in the helper)

> **Correction (2026-06-19, post-implementation review of FE PR #236):** the original draft of this section claimed `navigator.vibrate` is *absent* on desktop, so a bare `typeof navigator.vibrate === "function"` check would exclude it. That premise is **wrong** — desktop Chrome/Edge/Firefox define a silent **no-op** `navigator.vibrate`, so the bare check is `true` there and the Preferences section rendered on desktop. `canVibrate()` therefore also requires `(pointer: coarse)`. The text below is the corrected design.

The helper gates **only** on `canVibrate()` + opt-in — it does **not** check `isStandalone()`/`isIOS()`. Rationale:

- `canVibrate()` is `true` only with the Vibration API **and** a coarse (touch) pointer. iOS Safari is excluded (no `navigator.vibrate`); desktop Chrome/Edge/Firefox are excluded by the coarse-pointer requirement (their `navigator.vibrate` is a no-op). The capability check *is* the platform gate for vibration.
- On Android Chrome **in a browser tab** (not installed), `navigator.vibrate` exists and the pointer is coarse, so haptics fire; that is harmless and arguably desirable. Requiring standalone would add complexity for no user benefit.
- This keeps the helper a pure function of capability + preference — trivially testable, with one capability branch and no install-context branches.

(Contrast: `useAndroidBackButton` *does* gate on standalone+Android+coarse, because hardware-back interception is only meaningful in the installed app. The `haptics.back()` call sits *inside* that already-gated cascade, so back-haptics inherit that gate for free; taps/completions are gated on capability + coarse pointer only.)

### D4. Preferences UI: capability-gated section with master + sub-switches

Add a **"Haptics"** section to `preferences-sheet.tsx`, rendered **only when `canVibrate()` is true** — hidden entirely on iOS/desktop so there is no dead/confusing toggle for an unsupported capability.

```tsx
// shape (preferences-sheet.tsx)
const hapticsSupported = canVibrate();
const enabled = useHapticsPreference((s) => s.enabled);
const categories = useHapticsPreference((s) => s.categories);
const setEnabled = useHapticsPreference((s) => s.setEnabled);
const setCategory = useHapticsPreference((s) => s.setCategory);
// ...
{hapticsSupported && (
  <section className="space-y-4">
    <h3 className="…uppercase tracking-wider">Haptics</h3>
    {/* master row: <Switch checked={enabled} onCheckedChange={setEnabled} /> */}
    {enabled && (
      <div className="pl-… space-y-1">
        {/* Taps / Completions / Back, each a labeled <Switch> bound to categories[x] */}
      </div>
    )}
  </section>
)}
```

- The sub-switches render only while the master is on (collapse when off).
- Uses a new **`src/components/ui/switch.tsx`** — a thin shadcn wrapper over `@radix-ui/react-switch` (already installed), styled with `cn()` + theme tokens to match the existing UI primitives. Each toggle has an associated `<Label>`/`aria-label` for accessibility and testability.
- `canVibrate()` is evaluated at render (capability is static at runtime), so no reactivity is needed for visibility.

### D5. Test-harness reset (the critical leak gotcha)

`src/test/setup.ts`'s `resetAllStores()` resets by **explicit name**. Add `useHapticsPreference` to it (and clear its key), or its `enabled`/`categories` leak across tests in a file:

```ts
useHapticsPreference.setState({
  enabled: false,
  categories: { taps: true, completions: true, back: true },
});
localStorage.removeItem(HAPTICS_STORAGE_KEY);
```

### D6. Integration wiring (4 leaf edits, all reuse existing seams)

| Seam | Edit | Fires |
|---|---|---|
| `src/hooks/use-pressable.ts` | replace the placeholder comment in `onPointerDown` with `haptics.tap()` | every `usePressable` surface — buttons, nav, sidebar rows, list/recipe cards |
| `src/components/lists/list-item-row.tsx` (`:24`) | fire `haptics.success()` only on the completing transition (`!item.completed`) before `onToggle` | list-item completion |
| `src/components/chores/chore-row.tsx` (`:42`) | fire `haptics.success()` only on the `onComplete` path (`!chore.completed`) | chore completion |
| `src/hooks/use-android-back-button.ts` (`onPop`) | `haptics.back()` in the two **handled** branches (overlay dismiss + up-to-Home), **before** `rebuffer()`; **not** in the exit-hint/exit branch | hardware back-dismiss |

No call-site churn beyond these four files: the tap path is fully covered by the single seam, and `success()`/`back()` attach at the existing user-action handlers.

---

## File map

| File | Action | Purpose |
|---|---|---|
| `src/lib/haptics.ts` | **Create** | `canVibrate()` + centralized `haptics` helper (only `navigator.vibrate` caller) |
| `src/stores/haptics-store.ts` | **Create** | `useHapticsPreference` (persisted, per-device) + `HAPTICS_STORAGE_KEY` |
| `src/stores/index.ts` | Modify | Export `useHapticsPreference`, `HAPTICS_STORAGE_KEY`, `HapticCategory` |
| `src/components/ui/switch.tsx` | **Create** | shadcn wrapper over `@radix-ui/react-switch` |
| `src/test/setup.ts` | Modify | Add `useHapticsPreference` to `resetAllStores()` by name + clear its key |
| `src/hooks/use-pressable.ts` | Modify | `haptics.tap()` in `onPointerDown` (the seam) |
| `src/components/lists/list-item-row.tsx` | Modify | `haptics.success()` on completing transition |
| `src/components/chores/chore-row.tsx` | Modify | `haptics.success()` on completing transition |
| `src/hooks/use-android-back-button.ts` | Modify | `haptics.back()` on handled back-dismiss (steps 1 & 2) |
| `src/components/settings/preferences-sheet.tsx` | Modify | Capability-gated "Haptics" section (master + 3 sub-switches) |

## Testing strategy

- **Unit — `haptics` (`src/lib/haptics.test.ts`):** mock `navigator.vibrate` with `vi.fn()`; drive the store with `useHapticsPreference.setState(...)` (the **real** store, not a `vi.mock` — avoids the alias-preload mock gotcha). Assert:
  - capable + `enabled` + category on → `navigator.vibrate` called with the right pattern (`tap`→`10`, `success`→`[12,40,12]`, `back`→`8`);
  - **silent no-op** when: `canVibrate()` false (delete `navigator.vibrate`), master off, the specific category off;
  - throttle: a second fire within `THROTTLE_MS` is suppressed; after the window it fires again (fake timers or `Date.now` spy).
  - **Reset every gate input in `beforeEach`** — `navigator.vibrate` presence, store state, and `lastFireAt` (re-import/reset) — not just the one a given test mutates (the factory-mock isolation gotcha from PR #234).
- **Unit — `useHapticsPreference` (`haptics-store.test.ts`):** defaults (`enabled` false, all categories true); `setEnabled`/`setCategory` mutate correctly; **persists** to `localStorage["family-hub-haptics"]` (mirror calendar-store's persist assertion).
- **Component — `switch.test.tsx`:** renders a Radix switch, `onCheckedChange` fires, `checked`/`aria-checked` reflect state.
- **Component — `preferences-sheet.test.tsx` (extend):** with a capable device mocked (`navigator.vibrate = vi.fn()` **and** `matchMedia("(pointer: coarse)")` → `matches: true`) → the Haptics section + master switch render; toggling the master writes `enabled` and reveals the 3 sub-switches; toggling a sub-switch writes `categories`; sub-switches hidden when master off. With `navigator.vibrate` undefined **or** a fine pointer → the section is **absent**.
- **Integration:** `usePressable` → `pointerdown` calls `navigator.vibrate` when opted-in (spy); the back seam fires `haptics.back()` on a handled `popstate` (extend the existing `use-android-back-button` test harness, gate mocked on, `navigator.vibrate` spied).
- **Manual device pass (the real acceptance gate):** Galaxy S10 / Chrome, installed PWA — toggle master on → feel a tap on buttons/nav, a double on list/chore completion, a pulse on hardware back-dismiss; turn off individual categories → those stop; master off → nothing; verify iOS Safari and desktop feel nothing and show no Haptics section. **If this device pass can't be run, say so explicitly in the PR for the human to verify.**
- Follow `frontend/CLAUDE.md` gotchas: new store added to `resetAllStores()` by name (D5); reset all mocked gate inputs each test; prefer real store + `setState` over `vi.mock` of `@/` alias modules.

## Out of scope

`warning()` + destructive-confirm wiring; bottom-sheet-snap pulse; synced/server preference; `prefers-reduced-motion` coupling; iOS/desktop haptics; any new motion/press/back infrastructure; per-category throttling; analytics on haptic usage.

## Acceptance criteria

(Mirror of the story, refined by the spec decisions.)

- A "Haptics" master toggle in Preferences, **default off**, persisted per-device in `localStorage["family-hub-haptics"]`, with per-category sub-toggles (Taps / Completions / Back, default on) shown only when the master is on.
- When enabled, a short vibration fires on: primary-action taps (`usePressable`), task/list completion, and hardware back-dismiss — each gated by its category.
- Capability-detected: silently no-ops where the device is not touch-primary or lacks a working Vibration API — iOS Safari (no `navigator.vibrate`) and desktop Chrome/Edge/Firefox (no-op `navigator.vibrate`, fine pointer) — and whenever the master or category toggle is off; the Haptics section is hidden where unsupported.
- Patterns are short and subtle (single brief pulses for tap/back; a brief double for success), throttled against rapid repeats.
- All vibration goes through the single `haptics.tap()/success()/back()` helper — no scattered `navigator.vibrate`.
- Built entirely on the PR #230 (`usePressable`) and PR #234 (`useAndroidBackButton`) seams — no new press/motion/back code.

## Self-review notes

- **Placeholders:** none — every change names a real file/symbol verified 2026-06-18; the helper, store, and gate logic have full sketches.
- **No new dependency:** `@radix-ui/react-switch` already in `package.json`; reuses Zustand `persist`, the existing seams, and the History/Vibration APIs.
- **Decisions resolved (from brainstorming):** default **off**; **master + per-category** (Taps/Completions/Back) toggles; **per-device** localStorage; **independent** of reduced-motion; **back-dismiss included** behind the same toggle as its own category; `warning()`/destructive-confirm and sheet-snap **deferred**.
- **Capability gate honesty:** the helper gates on `canVibrate()` + opt-in only (not standalone). `canVibrate()` requires the Vibration API **and** a coarse pointer, which excludes iOS Safari and all desktop browsers (whose `navigator.vibrate` is a no-op); tab-Android (coarse + working vibrate) haptics are harmless. The back-haptic still inherits `useAndroidBackButton`'s stricter standalone gate by virtue of sitting inside its cascade. (The earlier "absence of `navigator.vibrate` excludes desktop" rationale was corrected post-PR-#236 — see D3.)
- **Seam honesty:** consumes PR #230's `usePressable().onPointerDown` and PR #234's `onPop` cascade; ships zero new motion/press/back infrastructure — exactly the seams those PRs left.
- **No double-pulse:** completion rows (`list-item-row`, `chore-row`) verified to lack `usePressable`, so `success()` does not stack with `tap()`.
- **Test-harness reset (critical):** `src/test/setup.ts` resets by explicit name, so `useHapticsPreference` is added to `resetAllStores()` — without it `enabled`/`categories` leak across a file; and every mocked gate input (`navigator.vibrate`, store, throttle clock) is reset per test.
