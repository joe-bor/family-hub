# Optional Haptic Feedback Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A subtle vibration fires on a per-device-configurable set of touch interactions in the installed Android PWA — primary-action **taps**, task/list **completions**, and hardware **back**-dismiss — routed through a single helper, opt-in (default off), and a silent no-op everywhere `navigator.vibrate` is absent (iOS Safari, desktop). Build **only** on the two existing seams (`usePressable().onPointerDown` from PR #230, the `useAndroidBackButton` `onPop` cascade from PR #234); add no new motion/press/back infrastructure and no new dependency.

**Architecture:** One imperative `haptics` object (`src/lib/haptics.ts`) is the *only* caller of `navigator.vibrate`; it reads a persisted per-device Zustand store (`useHapticsPreference`) via `getState()` and gates on capability → master → category → throttle, returning silently on any failing gate. A capability-gated Preferences section (new `Switch` primitive over the already-installed `@radix-ui/react-switch`) drives the store. Three fire-sites consume existing seams: `usePressable().onPointerDown` → `tap()`; the two completion rows → `success()`; the two **handled** branches of the back-button `onPop` cascade → `back()`.

**Tech Stack:** React 19, Vite, Zustand + `persist`, the Web Vibration API, `@radix-ui/react-switch` (already in `package.json`), Vitest + Testing Library. `cn()` from `@/lib/utils`.

**Spec:** `docs/superpowers/specs/2026-06-18-optional-haptics-design.md`
**Story:** `docs/product/backlog/mobile-ux/optional-haptics.md`

---

## Execution context

- Implement in the **FamilyHub FE repo** (`frontend/` in the workspace; standalone `joe-bor/FamilyHub`). All paths below are relative to the FE repo root.
- Branch: `feat/optional-haptics`. Regular merge commits (not squash) per FE `CLAUDE.md` versioning; these are `feat:` commits (minor bump).
- Single-file test run: `npx vitest run <path>`. Full check before PR: `npm run lint && npm test -- --run`.
- **Test gotchas (`frontend/CLAUDE.md`):**
  - `src/test/setup.ts`'s `resetAllStores()` resets stores **by explicit name** (not generically), so **`useHapticsPreference` must be added there** (Task 1) or its `enabled`/`categories` leak across tests in a file.
  - For the integration fire-sites (Tasks 5–7), assert via **`vi.spyOn(haptics, "tap"|"success"|"back")`** on the real imported namespace — do **not** `vi.mock("@/lib/haptics", …)`. `vi.mock` of `@/` alias modules frequently fails to bind because `setup.ts` preloads the `@/` graph; namespace `vi.spyOn` is the reliable seam.
  - For the `haptics` unit test (Task 2), drive the **real** store with `useHapticsPreference.setState(...)` and mock `navigator.vibrate` with `vi.fn()`; **reset every gate input** (`navigator.vibrate` presence, store state, and the throttle clock) in `beforeEach`, not just the one a given test mutates.
  - `navigator.vibrate` is **not** in jsdom, so `canVibrate()` is `false` by default in tests — the correct no-op baseline. Define `navigator.vibrate = vi.fn()` per-test where you need the capable path; `delete (navigator as …).vibrate` to test the incapable path.

## File structure

| File | Responsibility |
|---|---|
| `src/stores/haptics-store.ts` (create) | `useHapticsPreference` (persisted, per-device) + `HAPTICS_STORAGE_KEY` + `HapticCategory` |
| `src/stores/index.ts` (modify) | Barrel export `useHapticsPreference`, `HAPTICS_STORAGE_KEY`, `HapticCategory` |
| `src/test/setup.ts` (modify) | Add `useHapticsPreference` reset to `resetAllStores()` (by explicit name) + clear its key |
| `src/lib/haptics.ts` (create) | `canVibrate()` + centralized `haptics` helper — the only `navigator.vibrate` caller |
| `src/components/ui/switch.tsx` (create) | shadcn-style wrapper over `@radix-ui/react-switch` |
| `src/components/settings/preferences-sheet.tsx` (modify) | Capability-gated "Haptics" section (master + 3 sub-switches) |
| `src/hooks/use-pressable.ts` (modify) | `haptics.tap()` in `onPointerDown` (replace the placeholder comment) |
| `src/components/lists/list-item-row.tsx` (modify) | `haptics.success()` on the completing transition |
| `src/components/chores/chore-row.tsx` (modify) | `haptics.success()` on the completing transition |
| `src/hooks/use-android-back-button.ts` (modify) | `haptics.back()` in the two handled `onPop` branches (before `rebuffer()`) |

---

## Setup

- [ ] **Branch from latest `main`**

```bash
git checkout main && git pull
git checkout -b feat/optional-haptics
```

- [ ] **Read the spec and story** linked above so the contract is fresh. Confirm `@radix-ui/react-switch` is still in `package.json` (no install needed).

---

## Task 1: `useHapticsPreference` store + test-harness reset

**Files:**
- Create: `src/stores/haptics-store.ts`
- Test: `src/stores/haptics-store.test.ts`
- Modify: `src/stores/index.ts`, `src/test/setup.ts`

- [ ] **Step 1: Write the failing test** — mirrors `calendar-store`'s persist assertion:

```ts
import { beforeEach, describe, expect, it } from "vitest";
import { HAPTICS_STORAGE_KEY, useHapticsPreference } from "./haptics-store";

beforeEach(() => {
  localStorage.clear();
  useHapticsPreference.setState({
    enabled: false,
    categories: { taps: true, completions: true, back: true },
  });
});

describe("useHapticsPreference", () => {
  it("defaults to master off with every category on", () => {
    const s = useHapticsPreference.getState();
    expect(s.enabled).toBe(false);
    expect(s.categories).toEqual({ taps: true, completions: true, back: true });
  });

  it("setEnabled toggles the master flag", () => {
    useHapticsPreference.getState().setEnabled(true);
    expect(useHapticsPreference.getState().enabled).toBe(true);
  });

  it("setCategory toggles one category without touching the others", () => {
    useHapticsPreference.getState().setCategory("taps", false);
    expect(useHapticsPreference.getState().categories).toEqual({
      taps: false,
      completions: true,
      back: true,
    });
  });

  it("persists enabled + categories to localStorage[family-hub-haptics]", () => {
    useHapticsPreference.getState().setEnabled(true);
    useHapticsPreference.getState().setCategory("back", false);
    const persisted = JSON.parse(
      localStorage.getItem(HAPTICS_STORAGE_KEY) ?? "{}",
    );
    expect(persisted.state.enabled).toBe(true);
    expect(persisted.state.categories.back).toBe(false);
  });
});
```

- [ ] **Step 2: Run it — expect FAIL** (`Cannot find module './haptics-store'`)

```bash
npx vitest run src/stores/haptics-store.test.ts
```

- [ ] **Step 3: Implement** — `src/stores/haptics-store.ts` (per spec D2):

```ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

export type HapticCategory = "taps" | "completions" | "back";

interface HapticsPreferenceState {
  enabled: boolean; // master, default false
  categories: Record<HapticCategory, boolean>; // each default true
  setEnabled: (on: boolean) => void;
  setCategory: (category: HapticCategory, on: boolean) => void;
}

/** localStorage key, exported for the test-harness reset (see Task 1 Step 5). */
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

- [ ] **Step 4: Export from the barrel** — add to `src/stores/index.ts` (with the other store groups):

```ts
// Haptics Store
export {
  type HapticCategory,
  HAPTICS_STORAGE_KEY,
  useHapticsPreference,
} from "./haptics-store";
```

- [ ] **Step 5: Register the per-test reset** — `resetAllStores()` in `src/test/setup.ts` resets by explicit name (it is NOT generic), so add `useHapticsPreference` there or its state leaks across a file. Add the import alongside the others and a reset block inside `resetAllStores()`:

```ts
import { HAPTICS_STORAGE_KEY, useHapticsPreference } from "@/stores/haptics-store";
// ...inside resetAllStores(), alongside the other store resets:
useHapticsPreference.setState({
  enabled: false,
  categories: { taps: true, completions: true, back: true },
});
localStorage.removeItem(HAPTICS_STORAGE_KEY);
```

(`beforeEach` already calls `localStorage.clear()`, but the in-memory store state still needs the explicit by-name reset — same reason `useBackStack` was added here in PR #234.)

- [ ] **Step 6: Run it — expect PASS**, then commit

```bash
npx vitest run src/stores/haptics-store.test.ts
git add src/stores/haptics-store.ts src/stores/haptics-store.test.ts src/stores/index.ts src/test/setup.ts
git commit -m "feat(haptics): add useHapticsPreference persisted store"
```

---

## Task 2: `haptics` helper + `canVibrate()`

**Files:**
- Create: `src/lib/haptics.ts`
- Test: `src/lib/haptics.test.ts`

Depends on Task 1 (reads the store via `getState()`).

- [ ] **Step 1: Write the failing test** — mock `navigator.vibrate`, drive the **real** store, reset **every** gate input each test:

```ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { useHapticsPreference } from "@/stores";
import { canVibrate, haptics } from "./haptics";

function setVibrate(fn: ((p: number | number[]) => boolean) | undefined) {
  if (fn) {
    Object.defineProperty(navigator, "vibrate", {
      value: fn,
      configurable: true,
      writable: true,
    });
  } else {
    // biome-ignore lint/performance/noDelete: capability removal for the test
    delete (navigator as { vibrate?: unknown }).vibrate;
  }
}

beforeEach(() => {
  vi.useFakeTimers(); // controls the throttle clock (Date.now)
  setVibrate(vi.fn(() => true)); // capable baseline; tests override
  useHapticsPreference.setState({
    enabled: true,
    categories: { taps: true, completions: true, back: true },
  });
});
afterEach(() => {
  vi.useRealTimers();
  setVibrate(undefined);
});

describe("canVibrate", () => {
  it("is true when navigator.vibrate exists, false when absent", () => {
    expect(canVibrate()).toBe(true);
    setVibrate(undefined);
    expect(canVibrate()).toBe(false);
  });
});

describe("haptics", () => {
  it("fires the right pattern per category when capable + opted-in", () => {
    const vibrate = navigator.vibrate as ReturnType<typeof vi.fn>;
    haptics.tap();
    expect(vibrate).toHaveBeenLastCalledWith(10);
    vi.advanceTimersByTime(100);
    haptics.success();
    expect(vibrate).toHaveBeenLastCalledWith([12, 40, 12]);
    vi.advanceTimersByTime(100);
    haptics.back();
    expect(vibrate).toHaveBeenLastCalledWith(8);
  });

  it("is a silent no-op when navigator.vibrate is absent", () => {
    setVibrate(undefined);
    expect(() => haptics.tap()).not.toThrow();
  });

  it("is a no-op when the master toggle is off", () => {
    useHapticsPreference.setState({ enabled: false });
    haptics.tap();
    expect(navigator.vibrate).not.toHaveBeenCalled();
  });

  it("is a no-op when the specific category is off", () => {
    useHapticsPreference.setState({
      enabled: true,
      categories: { taps: false, completions: true, back: true },
    });
    haptics.tap();
    expect(navigator.vibrate).not.toHaveBeenCalled();
  });

  it("throttles repeats inside the window and re-fires after it", () => {
    const vibrate = navigator.vibrate as ReturnType<typeof vi.fn>;
    haptics.tap();
    haptics.tap(); // within THROTTLE_MS → suppressed
    expect(vibrate).toHaveBeenCalledTimes(1);
    vi.advanceTimersByTime(100); // past the window
    haptics.tap();
    expect(vibrate).toHaveBeenCalledTimes(2);
  });
});
```

- [ ] **Step 2: Run it — expect FAIL** (`Cannot find module './haptics'`)

```bash
npx vitest run src/lib/haptics.test.ts
```

- [ ] **Step 3: Implement** — `src/lib/haptics.ts` (per spec D1/D3; capability-only gate, no install/context check):

```ts
import { useHapticsPreference } from "@/stores/haptics-store";

/** Short, subtle patterns (ms). Single pulses for tap/back; a brief double for success. */
const PATTERNS = {
  taps: 10,
  completions: [12, 40, 12],
  back: 8,
} as const;

const THROTTLE_MS = 40; // guard against pointerdown storms / rapid repeats
let lastFireAt = 0;

/** Feature-detect the Vibration API. Android Chrome → true; iOS Safari + desktop → false. */
export function canVibrate(): boolean {
  return typeof navigator !== "undefined" && typeof navigator.vibrate === "function";
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

/** The only sanctioned way to vibrate. Each method no-ops unless capable + opted-in. */
export const haptics = {
  tap: () => fire("taps"),
  success: () => fire("completions"),
  back: () => fire("back"),
};
```

- [ ] **Step 4: Run it — expect PASS**, then commit

```bash
npx vitest run src/lib/haptics.test.ts
git add src/lib/haptics.ts src/lib/haptics.test.ts
git commit -m "feat(haptics): add centralized haptics helper + canVibrate"
```

---

## Task 3: `Switch` UI primitive

**Files:**
- Create: `src/components/ui/switch.tsx`
- Test: `src/components/ui/switch.test.tsx`

> A thin shadcn-style wrapper over the already-installed `@radix-ui/react-switch`, styled like the sibling `label.tsx` (`data-slot`, `cn()`, theme tokens). No new dependency.

- [ ] **Step 1: Write the failing test**

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, expect, it, vi } from "vitest";
import { Switch } from "./switch";

describe("Switch", () => {
  it("renders a switch reflecting checked state", () => {
    render(<Switch checked aria-label="Haptics" onCheckedChange={() => {}} />);
    const sw = screen.getByRole("switch", { name: "Haptics" });
    expect(sw).toBeInTheDocument();
    expect(sw).toBeChecked();
  });

  it("calls onCheckedChange when toggled", async () => {
    const onCheckedChange = vi.fn();
    render(
      <Switch checked={false} aria-label="Haptics" onCheckedChange={onCheckedChange} />,
    );
    await userEvent.click(screen.getByRole("switch", { name: "Haptics" }));
    expect(onCheckedChange).toHaveBeenCalledWith(true);
  });
});
```

- [ ] **Step 2: Run it — expect FAIL**

```bash
npx vitest run src/components/ui/switch.test.tsx
```

- [ ] **Step 3: Implement** — `src/components/ui/switch.tsx`:

```tsx
import * as SwitchPrimitive from "@radix-ui/react-switch";
import type * as React from "react";
import { cn } from "@/lib/utils";

function Switch({
  className,
  ...props
}: React.ComponentProps<typeof SwitchPrimitive.Root>) {
  return (
    <SwitchPrimitive.Root
      data-slot="switch"
      className={cn(
        "peer inline-flex h-5 w-9 shrink-0 cursor-pointer items-center rounded-full border border-transparent shadow-xs transition-colors outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50 data-[state=checked]:bg-primary data-[state=unchecked]:bg-input",
        className,
      )}
      {...props}
    >
      <SwitchPrimitive.Thumb
        data-slot="switch-thumb"
        className="pointer-events-none block h-4 w-4 rounded-full bg-background shadow-sm ring-0 transition-transform data-[state=checked]:translate-x-4 data-[state=unchecked]:translate-x-0"
      />
    </SwitchPrimitive.Root>
  );
}

export { Switch };
```

- [ ] **Step 4: Run it — expect PASS**, then commit

```bash
npx vitest run src/components/ui/switch.test.tsx
git add src/components/ui/switch.tsx src/components/ui/switch.test.tsx
git commit -m "feat(ui): add Switch primitive over @radix-ui/react-switch"
```

---

## Task 4: Capability-gated "Haptics" Preferences section

**Files:**
- Modify: `src/components/settings/preferences-sheet.tsx`
- Test: `src/components/settings/preferences-sheet.test.tsx` (extend, or create if absent)

Depends on Tasks 1 + 3. The section sits between "Family Timezone" and "Coming Soon".

- [ ] **Step 1: Write the failing test** — capability is driven by `navigator.vibrate` presence (the section calls `canVibrate()` at render):

```tsx
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { afterEach, beforeEach, describe, expect, it } from "vitest";
import { useHapticsPreference } from "@/stores";
import { PreferencesSheet } from "./preferences-sheet";
// NOTE: render inside the app's QueryClientProvider wrapper (use the file's
// existing renderWithProviders/renderWithUser helper — PreferencesSheet calls
// useFamilyData()). Shown bare here for brevity.

function setVibrate(on: boolean) {
  if (on) {
    Object.defineProperty(navigator, "vibrate", {
      value: () => true,
      configurable: true,
      writable: true,
    });
  } else {
    // biome-ignore lint/performance/noDelete: capability removal for the test
    delete (navigator as { vibrate?: unknown }).vibrate;
  }
}

afterEach(() => setVibrate(false));

describe("PreferencesSheet — Haptics section", () => {
  it("is absent when navigator.vibrate is unsupported", () => {
    setVibrate(false);
    render(<PreferencesSheet open onOpenChange={() => {}} />);
    expect(screen.queryByRole("heading", { name: /haptics/i })).toBeNull();
  });

  it("shows the master switch and reveals sub-switches when enabled", async () => {
    setVibrate(true);
    render(<PreferencesSheet open onOpenChange={() => {}} />);
    expect(screen.getByRole("heading", { name: /haptics/i })).toBeInTheDocument();

    // sub-switches hidden while master is off
    expect(screen.queryByRole("switch", { name: /taps/i })).toBeNull();

    await userEvent.click(
      screen.getByRole("switch", { name: /enable haptics/i }),
    );
    expect(useHapticsPreference.getState().enabled).toBe(true);

    // sub-switches now visible; toggling one writes its category
    await userEvent.click(screen.getByRole("switch", { name: /taps/i }));
    expect(useHapticsPreference.getState().categories.taps).toBe(false);
  });
});
```

- [ ] **Step 2: Run it — expect FAIL**

```bash
npx vitest run src/components/settings/preferences-sheet.test.tsx
```

- [ ] **Step 3: Implement** — in `preferences-sheet.tsx`: add imports and a capability-gated section. Read the store reactively (the UI mutates it); evaluate `canVibrate()` once at render (capability is static at runtime).

```tsx
import { Vibrate } from "lucide-react"; // add to the existing lucide import
import { canVibrate } from "@/lib/haptics";
import { Switch } from "@/components/ui/switch";
import { type HapticCategory, useHapticsPreference } from "@/stores";

const SUB_TOGGLES: ReadonlyArray<{ key: HapticCategory; label: string }> = [
  { key: "taps", label: "Taps" },
  { key: "completions", label: "Completions" },
  { key: "back", label: "Back" },
];

// ...inside PreferencesSheet(), with the other hooks:
const hapticsSupported = canVibrate();
const hapticsEnabled = useHapticsPreference((s) => s.enabled);
const hapticCategories = useHapticsPreference((s) => s.categories);
const setHapticsEnabled = useHapticsPreference((s) => s.setEnabled);
const setHapticCategory = useHapticsPreference((s) => s.setCategory);
```

Insert the section JSX between the Timezone `</section>` and the Coming Soon `<section>` (mirrors the existing section markup):

```tsx
{hapticsSupported && (
  <section className="space-y-4">
    <h3 className="text-sm font-semibold text-muted-foreground uppercase tracking-wider">
      Haptics
    </h3>
    <Label
      htmlFor="haptics-master"
      className="flex min-h-11 items-center justify-between gap-3 font-medium"
    >
      <span className="flex items-center gap-2">
        <Vibrate className="h-4 w-4" />
        Enable haptics
      </span>
      <Switch
        id="haptics-master"
        aria-label="Enable haptics"
        checked={hapticsEnabled}
        onCheckedChange={setHapticsEnabled}
      />
    </Label>
    {hapticsEnabled && (
      <div className="space-y-1 pl-6">
        {SUB_TOGGLES.map(({ key, label }) => (
          <Label
            key={key}
            htmlFor={`haptics-${key}`}
            className="flex min-h-11 items-center justify-between gap-3 font-normal text-muted-foreground"
          >
            {label}
            <Switch
              id={`haptics-${key}`}
              aria-label={label}
              checked={hapticCategories[key]}
              onCheckedChange={(on) => setHapticCategory(key, on)}
            />
          </Label>
        ))}
      </div>
    )}
  </section>
)}
```

- [ ] **Step 4: Run it — expect PASS**, then commit

```bash
npx vitest run src/components/settings/preferences-sheet.test.tsx
git add src/components/settings/preferences-sheet.tsx src/components/settings/preferences-sheet.test.tsx
git commit -m "feat(haptics): add capability-gated Haptics preferences section"
```

---

## Task 5: Tap haptic via the `usePressable` seam

**Files:**
- Modify: `src/hooks/use-pressable.ts`
- Test: `src/hooks/use-pressable.test.ts` (extend, or create if absent)

> One edit covers every primary-action surface (buttons, bottom-nav, sidebar rows, list/recipe cards) with zero call-site churn — the seam is already spread through them.

- [ ] **Step 1: Write the failing test** — spy on the real `haptics` namespace (not `vi.mock`):

```tsx
import { renderHook } from "@testing-library/react";
import { describe, expect, it, vi } from "vitest";
import { haptics } from "@/lib/haptics";
import { usePressable } from "./use-pressable";

describe("usePressable", () => {
  it("fires haptics.tap() on pointer down", () => {
    const tap = vi.spyOn(haptics, "tap").mockImplementation(() => {});
    const { result } = renderHook(() => usePressable());
    result.current.onPointerDown({} as React.PointerEvent);
    expect(tap).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 2: Run it — expect FAIL** (the body is still the placeholder comment)

```bash
npx vitest run src/hooks/use-pressable.test.ts
```

- [ ] **Step 3: Implement** — in `src/hooks/use-pressable.ts`, import the helper and replace the placeholder comment in `onPointerDown`:

```ts
import { haptics } from "@/lib/haptics";
// ...inside the useCallback body, replacing the "Haptic seam: …" comment:
const onPointerDown = useCallback((_event: PointerEvent) => {
  haptics.tap();
}, []);
```

- [ ] **Step 4: Run it — expect PASS**, then commit

```bash
npx vitest run src/hooks/use-pressable.test.ts
git add src/hooks/use-pressable.ts src/hooks/use-pressable.test.ts
git commit -m "feat(haptics): fire tap haptic from the usePressable seam"
```

---

## Task 6: Success haptic on list & chore completion

**Files:**
- Modify: `src/components/lists/list-item-row.tsx`, `src/components/chores/chore-row.tsx`
- Test: `src/components/lists/list-item-row.test.tsx`, `src/components/chores/chore-row.test.tsx` (extend, or create if absent)

> Both rows fire `success()` **only on the completing transition** — not when un-completing. Verified: neither row uses `usePressable`, so `success()` never double-fires with `tap()`.

- [ ] **Step 1: Write the failing tests** — spy on `haptics.success`:

```tsx
// list-item-row.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, expect, it, vi } from "vitest";
import { haptics } from "@/lib/haptics";
import { ListItemRow } from "./list-item-row";

const baseItem = { id: "1", text: "Milk", completed: false } as const;

describe("ListItemRow haptics", () => {
  it("fires success() when completing an item", async () => {
    const success = vi.spyOn(haptics, "success").mockImplementation(() => {});
    render(
      <ListItemRow item={baseItem} onToggle={vi.fn()} onEdit={vi.fn()} onDelete={vi.fn()} />,
    );
    await userEvent.click(screen.getByRole("button", { name: /milk/i }));
    expect(success).toHaveBeenCalledTimes(1);
  });

  it("does NOT fire success() when un-completing", async () => {
    const success = vi.spyOn(haptics, "success").mockImplementation(() => {});
    render(
      <ListItemRow
        item={{ ...baseItem, completed: true }}
        onToggle={vi.fn()}
        onEdit={vi.fn()}
        onDelete={vi.fn()}
      />,
    );
    await userEvent.click(screen.getByRole("button", { name: /milk/i }));
    expect(success).not.toHaveBeenCalled();
  });
});
```

```tsx
// chore-row.test.tsx — completing path is onComplete (when !chore.completed)
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, expect, it, vi } from "vitest";
import { haptics } from "@/lib/haptics";
import { ChoreRow } from "./chore-row";

const baseChore = {
  templateId: "t1",
  title: "Dishes",
  cadence: "DAILY",
  completed: false,
} as const;

describe("ChoreRow haptics", () => {
  it("fires success() on the complete path", async () => {
    const success = vi.spyOn(haptics, "success").mockImplementation(() => {});
    render(<ChoreRow chore={baseChore} onComplete={vi.fn()} onUncomplete={vi.fn()} />);
    await userEvent.click(screen.getByRole("button", { name: /mark dishes complete/i }));
    expect(success).toHaveBeenCalledTimes(1);
  });

  it("does NOT fire success() on the uncomplete path", async () => {
    const success = vi.spyOn(haptics, "success").mockImplementation(() => {});
    render(
      <ChoreRow
        chore={{ ...baseChore, completed: true }}
        onComplete={vi.fn()}
        onUncomplete={vi.fn()}
      />,
    );
    await userEvent.click(screen.getByRole("button", { name: /mark dishes incomplete/i }));
    expect(success).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run them — expect FAIL**

```bash
npx vitest run src/components/lists/list-item-row.test.tsx src/components/chores/chore-row.test.tsx
```

- [ ] **Step 3: Implement — `list-item-row.tsx`.** Fire on the completing transition before delegating:

```tsx
import { haptics } from "@/lib/haptics";
// ...in the toggle button's onClick:
onClick={() => {
  if (!item.completed) haptics.success(); // completing transition only
  onToggle(!item.completed);
}}
```

- [ ] **Step 4: Implement — `chore-row.tsx`.** Fire on the `onComplete` path only:

```tsx
import { haptics } from "@/lib/haptics";
// ...in the toggle button's onClick:
onClick={() => {
  if (chore.completed) {
    onUncomplete?.();
  } else {
    haptics.success();
    onComplete?.();
  }
}}
```

- [ ] **Step 5: Run them — expect PASS**, then commit

```bash
npx vitest run src/components/lists/list-item-row.test.tsx src/components/chores/chore-row.test.tsx
git add src/components/lists/list-item-row.tsx src/components/chores/chore-row.tsx src/components/lists/list-item-row.test.tsx src/components/chores/chore-row.test.tsx
git commit -m "feat(haptics): fire success haptic on list and chore completion"
```

---

## Task 7: Back haptic on handled back-dismiss

**Files:**
- Modify: `src/hooks/use-android-back-button.ts`
- Test: `src/hooks/use-android-back-button.test.ts` (extend)

> `haptics.back()` fires in the **two handled branches** of the `onPop` cascade — (1) overlay/detail dismiss and (2) up-to-Home — **before** `rebuffer()`. It does **not** fire in the exit-hint / second-press-exit branch (that is not a dismiss). It sits inside `useAndroidBackButton`'s existing standalone+Android+coarse gate, so back-haptics inherit that stricter gate for free.

- [ ] **Step 1: Write the failing test** — extend the existing harness (gate mocked on, `popstate()`/`mockCoarsePointer` helpers already defined). Spy on `haptics.back`:

```ts
import { haptics } from "@/lib/haptics";

it("fires haptics.back() when dismissing the top overlay", () => {
  const back = vi.spyOn(haptics, "back").mockImplementation(() => {});
  useBackStack.getState().register(vi.fn());
  renderHook(() => useAndroidBackButton(true));
  popstate();
  expect(back).toHaveBeenCalledTimes(1);
});

it("fires haptics.back() when stepping up to Home", () => {
  const back = vi.spyOn(haptics, "back").mockImplementation(() => {});
  useAppStore.setState({ activeModule: "lists" });
  renderHook(() => useAndroidBackButton(true));
  popstate();
  expect(back).toHaveBeenCalledTimes(1);
});

it("does NOT fire haptics.back() on the exit-hint branch (Home root)", () => {
  const back = vi.spyOn(haptics, "back").mockImplementation(() => {});
  useAppStore.setState({ activeModule: null });
  renderHook(() => useAndroidBackButton(true));
  popstate(); // first press at Home → hint toast, not a dismiss
  expect(back).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Run it — expect FAIL**

```bash
npx vitest run src/hooks/use-android-back-button.test.ts
```

- [ ] **Step 3: Implement** — add `import { haptics } from "@/lib/haptics";` and a `haptics.back()` call at the top of each of the two handled branches in `onPop`, before `rebuffer()`:

```ts
// 1. Overlay/detail layer:
if (top) {
  haptics.back();
  top.handler();
  rebuffer();
  return;
}
// 2. Module layer (up to Home):
if (activeModule !== null) {
  haptics.back();
  setActiveModule(null);
  rebuffer();
  return;
}
// 3. Home root (exit-hint / exit): NO haptics.back() — not a dismiss.
```

- [ ] **Step 4: Run it — expect PASS**, then commit

```bash
npx vitest run src/hooks/use-android-back-button.test.ts
git add src/hooks/use-android-back-button.ts src/hooks/use-android-back-button.test.ts
git commit -m "feat(haptics): fire back haptic on handled back-dismiss"
```

---

## Task 8: Full check, manual device pass, PR

- [ ] **Step 1: Full suite + lint**

```bash
npm run lint && npm test -- --run
```
Expected: PASS.

- [ ] **Step 2: Manual device pass (the real acceptance gate)** — Galaxy S10 / Chrome, **installed** PWA:
  - Preferences shows a **Haptics** section; master **off** by default. Flip it on → the 3 sub-switches (Taps / Completions / Back) appear.
  - Feel a subtle **tap** on buttons / bottom-nav / sidebar rows / list & recipe cards; a brief **double** on completing a list item or chore; a **pulse** on a hardware back that dismisses an overlay or steps up to Home (no pulse on the "press back again to exit" hint).
  - Turn off an individual category → those stop; turn the master off → nothing fires.
  - Verify **iOS Safari** and **desktop** feel nothing **and show no Haptics section**.
  - **If this device pass can't be run, say so explicitly in the PR for the human to verify.**

- [ ] **Step 3: Contract checklist** (map each AC → code):
  - Master toggle in Preferences, default off, persisted `localStorage["family-hub-haptics"]`, per-category sub-toggles shown only when master on → Tasks 1 + 4.
  - Vibration on taps / completion / back-dismiss, each gated by category → Tasks 5 / 6 / 7.
  - Silent no-op where `navigator.vibrate` absent or toggle off; section hidden where unsupported → Tasks 2 (gates) + 4 (`canVibrate()` visibility).
  - Short, subtle, throttled patterns → Task 2 (`PATTERNS` + `THROTTLE_MS`).
  - All vibration through the single helper → Task 2 (only `navigator.vibrate` caller) + Tasks 5–7 call only `haptics.*`.
  - Built on the PR #230 + #234 seams, no new press/motion/back code → Tasks 5 (seam comment swap) + 7 (cascade branches).

- [ ] **Step 4: Open PR** with `Closes #<issue>` and the Story/Spec/Plan links.

```bash
git push -u origin feat/optional-haptics
gh pr create --title "feat: optional haptic feedback (Vibration API) with per-device toggles" --body "Implements docs/superpowers/specs/2026-06-18-optional-haptics-design.md. Closes #<issue>."
```

---

## Self-review

- **Spec coverage:** store (D2) → Task 1; helper + `canVibrate()` + capability-only gate (D1/D3) → Task 2; `Switch` primitive (D4) → Task 3; capability-gated Preferences section (D4) → Task 4; tap seam (D6) → Task 5; completion fire-sites (D6) → Task 6; back-dismiss fire-sites (D6) → Task 7; test-harness reset (D5) → Task 1 Step 5; testing strategy → Tasks 1–7 unit/component/integration + Task 8 manual.
- **Type consistency:** `useHapticsPreference` exposes `enabled: boolean` / `categories: Record<HapticCategory, boolean>` / `setEnabled(on)` / `setCategory(category, on)` used identically in Tasks 1, 4. `haptics.tap()/success()/back()` are nullary and used identically in Tasks 5, 6, 7. `canVibrate()` returns `boolean`, used in Tasks 2, 4. `HAPTICS_STORAGE_KEY` shared between Task 1 store and the setup reset. `HapticCategory` shared between store and the Preferences sub-toggle map.
- **Placeholders:** none — every change names a real file/symbol verified against `origin/main` on 2026-06-18; `usePressable`, `useAndroidBackButton`, `list-item-row`, `chore-row`, `preferences-sheet`, `label.tsx`, the stores barrel, and `resetAllStores()` were all read directly, and the `haptics`/store sketches are lifted from the spec.
- **Mocking discipline (the sharp edge):** the integration fire-sites (Tasks 5–7) assert via `vi.spyOn(haptics, …)` on the real namespace, **never** `vi.mock("@/lib/haptics")` — `setup.ts` preloads the `@/` graph so a factory mock would silently fail to bind. The `haptics` unit test (Task 2) uses the **real** store + a `navigator.vibrate` `vi.fn()` and resets all three gate inputs (capability, store, throttle clock) per test.
- **No double-pulse:** `list-item-row` and `chore-row` verified to lack `usePressable`, so `success()` never stacks with `tap()`; completion haptics fire only on the completing transition (`!item.completed` / `onComplete` path).
- **Back-haptic placement:** only the two **handled** `onPop` branches (overlay dismiss, up-to-Home) fire `back()`, before `rebuffer()`; the exit-hint/exit branch deliberately does not. It inherits `useAndroidBackButton`'s standalone+Android+coarse gate.
- **No new dependency:** `@radix-ui/react-switch ^1.3.1` already in `package.json`; reuses Zustand `persist`, the existing seams, and the Vibration API.
```

