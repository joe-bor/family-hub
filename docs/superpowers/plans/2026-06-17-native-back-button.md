# Native Hardware Back-Button Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** In the installed Android PWA, make a single hardware/gesture back press dismiss the top-most open overlay or step up to Home, and only exit on a confirmed second press at Home — driving the *existing* close handlers so the sibling story's transitions animate the back for free.

**Architecture:** A LIFO **dismiss registry** (`useBackStack`, Zustand) that open surfaces register into via `useBackHandler(enabled, handler)`, plus a single **interceptor** (`useAndroidBackButton`) mounted once in the authenticated shell that owns one re-pushed history sentinel and runs the cascade on each `popstate`: dismiss top handler → up to Home → exit-hint → second-press exit. No router, no new motion, no new dependency.

**Tech Stack:** React 19, Vite, Zustand, Web History API, the existing `toast` + `pwa.ts` helpers, Vitest + Testing Library. `cn()` from `@/lib/utils`.

**Spec:** `docs/superpowers/specs/2026-06-17-native-back-button-design.md`
**Story:** `docs/product/backlog/mobile-ux/native-back-button.md`

---

## Execution context

- Implement in the **FamilyHub FE repo** (`frontend/` in the workspace; standalone `joe-bor/FamilyHub`). All paths below are relative to the FE repo root.
- Branch: `feat/native-back-button`. Regular merge commits (not squash) per FE `CLAUDE.md` versioning; these are `feat:` commits (minor bump).
- Single-file test run: `npx vitest run <path>`. Full check before PR: `npm run lint && npm test -- --run`.
- Test gotchas (`frontend/CLAUDE.md`): all Zustand stores (incl. the new `useBackStack`) auto-reset between tests via `src/test/setup.ts`; use **fake timers** for the 2s exit window; `matchMedia` is mocked globally — override per-test where the gate matters.

## File structure

| File | Responsibility |
|---|---|
| `src/stores/back-stack-store.ts` (create) | `useBackStack` — LIFO registry of `{ id, handler }` dismiss entries |
| `src/stores/index.ts` (modify) | Barrel export `useBackStack` |
| `src/hooks/use-back-handler.ts` (create) | `useBackHandler(enabled, handler)` — per-surface registration seam (stale-closure-safe) |
| `src/hooks/use-android-back-button.ts` (create) | Single `popstate`/sentinel interceptor + cascade + exit hint + Android gate |
| `src/hooks/index.ts` (modify) | Barrel export both hooks |
| `src/App.tsx` (modify) | Mount `useAndroidBackButton(isAuthenticated && setupComplete)` |
| `src/components/ui/mobile-sheet.tsx`, `src/components/ui/side-sheet.tsx` (modify) | Register their close handler while open |
| `src/components/lists-view.tsx`, `src/components/recipes-view.tsx` (modify) | Register `setSelected…(null)` while a detail is open |
| `src/components/calendar/components/{event-detail-modal,event-form-modal,edit-scope-dialog}.tsx`, `src/components/settings/{family-settings-modal,google-calendar-picker-modal}.tsx`, `src/components/shared/sidebar-menu.tsx` (modify) | Register the close handler on each raw `<Dialog>` |

---

## Setup

- [ ] **Branch from latest `main`**

```bash
git checkout main && git pull
git checkout -b feat/native-back-button
```

- [ ] **Read the spec and story** linked above so the contract is fresh.

---

## Task 1: `useBackStack` dismiss registry

**Files:**
- Create: `src/stores/back-stack-store.ts`
- Test: `src/stores/back-stack-store.test.ts`
- Modify: `src/stores/index.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, expect, it } from "vitest";
import { useBackStack } from "./back-stack-store";

describe("useBackStack", () => {
  it("peek returns undefined when empty", () => {
    expect(useBackStack.getState().peek()).toBeUndefined();
  });

  it("register returns unique ids and peek returns the most recent (LIFO)", () => {
    const a = () => {};
    const b = () => {};
    const idA = useBackStack.getState().register(a);
    const idB = useBackStack.getState().register(b);
    expect(idA).not.toBe(idB);
    expect(useBackStack.getState().peek()?.handler).toBe(b);
  });

  it("unregister by id restores the previous top", () => {
    const a = () => {};
    const b = () => {};
    useBackStack.getState().register(a);
    const idB = useBackStack.getState().register(b);
    useBackStack.getState().unregister(idB);
    expect(useBackStack.getState().peek()?.handler).toBe(a);
  });
});
```

- [ ] **Step 2: Run it — expect FAIL** (`Cannot find module './back-stack-store'`)

```bash
npx vitest run src/stores/back-stack-store.test.ts
```

- [ ] **Step 3: Implement**

```ts
import { create } from "zustand";

export interface BackHandlerEntry {
  id: number;
  handler: () => void;
}

interface BackStackState {
  stack: BackHandlerEntry[];
  register: (handler: () => void) => number;
  unregister: (id: number) => void;
  peek: () => BackHandlerEntry | undefined;
}

let nextId = 0;

/**
 * LIFO registry of "dismissable now" handlers. Open surfaces register via
 * useBackHandler; useAndroidBackButton peeks the most-recent on hardware back.
 */
export const useBackStack = create<BackStackState>((set, get) => ({
  stack: [],
  register: (handler) => {
    const id = nextId++;
    set((s) => ({ stack: [...s.stack, { id, handler }] }));
    return id;
  },
  unregister: (id) =>
    set((s) => ({ stack: s.stack.filter((e) => e.id !== id) })),
  peek: () => {
    const { stack } = get();
    return stack[stack.length - 1];
  },
}));
```

- [ ] **Step 4: Export from the barrel** — add to `src/stores/index.ts`:

```ts
export { useBackStack } from "./back-stack-store";
```

- [ ] **Step 5: Run it — expect PASS**, then commit

```bash
npx vitest run src/stores/back-stack-store.test.ts
git add src/stores/back-stack-store.ts src/stores/back-stack-store.test.ts src/stores/index.ts
git commit -m "feat(back-button): add useBackStack dismiss registry"
```

---

## Task 2: `useBackHandler` registration seam

**Files:**
- Create: `src/hooks/use-back-handler.ts`
- Test: `src/hooks/use-back-handler.test.ts`
- Modify: `src/hooks/index.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { renderHook } from "@testing-library/react";
import { describe, expect, it, vi } from "vitest";
import { useBackStack } from "@/stores";
import { useBackHandler } from "./use-back-handler";

describe("useBackHandler", () => {
  it("registers while enabled and unregisters on unmount", () => {
    const { unmount } = renderHook(() => useBackHandler(true, vi.fn()));
    expect(useBackStack.getState().stack).toHaveLength(1);
    unmount();
    expect(useBackStack.getState().stack).toHaveLength(0);
  });

  it("does not register while disabled", () => {
    renderHook(() => useBackHandler(false, vi.fn()));
    expect(useBackStack.getState().stack).toHaveLength(0);
  });

  it("invokes the latest handler after a re-render (no stale closure)", () => {
    const first = vi.fn();
    const second = vi.fn();
    const { rerender } = renderHook(({ h }) => useBackHandler(true, h), {
      initialProps: { h: first },
    });
    rerender({ h: second });
    useBackStack.getState().peek()?.handler();
    expect(first).not.toHaveBeenCalled();
    expect(second).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 2: Run it — expect FAIL**

```bash
npx vitest run src/hooks/use-back-handler.test.ts
```

- [ ] **Step 3: Implement**

```ts
import { useEffect, useRef } from "react";
import { useBackStack } from "@/stores";

/**
 * Register a dismiss handler on the back-stack while `enabled`. Hardware back
 * (see useAndroidBackButton) pops the most-recently-registered first (LIFO).
 * The registered wrapper always calls the latest `handler` (no stale closures);
 * it re-registers only when `enabled` flips.
 */
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

- [ ] **Step 4: Export from the barrel** — add to `src/hooks/index.ts`:

```ts
export { useBackHandler } from "./use-back-handler";
```

- [ ] **Step 5: Run it — expect PASS**, then commit

```bash
npx vitest run src/hooks/use-back-handler.test.ts
git add src/hooks/use-back-handler.ts src/hooks/use-back-handler.test.ts src/hooks/index.ts
git commit -m "feat(back-button): add useBackHandler registration seam"
```

---

## Task 3: `useAndroidBackButton` interceptor

**Files:**
- Create: `src/hooks/use-android-back-button.ts`
- Test: `src/hooks/use-android-back-button.test.ts`
- Modify: `src/hooks/index.ts`

- [ ] **Step 1: Write the failing test** — mock the gate (`@/lib/pwa` + a coarse-pointer `matchMedia`) and the `toast`, spy on history, dispatch `popstate`:

```ts
import { act, renderHook } from "@testing-library/react";
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";

vi.mock("@/lib/pwa", () => ({
  isStandalone: vi.fn(() => true),
  isIOS: vi.fn(() => false),
}));
vi.mock("@/components/ui/toaster", () => ({ toast: vi.fn() }));

import { toast } from "@/components/ui/toaster";
import { isStandalone } from "@/lib/pwa";
import { useAppStore, useBackStack } from "@/stores";
import { useAndroidBackButton } from "./use-android-back-button";

function mockCoarsePointer(coarse: boolean) {
  window.matchMedia = vi.fn().mockImplementation((query: string) => ({
    matches: query.includes("coarse") ? coarse : false,
    media: query,
    onchange: null,
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    addListener: vi.fn(),
    removeListener: vi.fn(),
    dispatchEvent: vi.fn(),
  }));
}

function popstate() {
  act(() => {
    window.dispatchEvent(new PopStateEvent("popstate"));
  });
}

beforeEach(() => {
  vi.mocked(isStandalone).mockReturnValue(true);
  mockCoarsePointer(true);
  useAppStore.setState({ activeModule: null });
  vi.spyOn(window.history, "pushState").mockImplementation(() => {});
  vi.spyOn(window.history, "back").mockImplementation(() => {});
});
afterEach(() => {
  vi.restoreAllMocks();
});

describe("useAndroidBackButton", () => {
  it("is a no-op when disabled", () => {
    renderHook(() => useAndroidBackButton(false));
    expect(window.history.pushState).not.toHaveBeenCalled();
    popstate();
    expect(toast).not.toHaveBeenCalled();
  });

  it("is a no-op on desktop (no coarse pointer)", () => {
    mockCoarsePointer(false);
    renderHook(() => useAndroidBackButton(true));
    expect(window.history.pushState).not.toHaveBeenCalled();
  });

  it("pushes a single sentinel on mount when gated on", () => {
    renderHook(() => useAndroidBackButton(true));
    expect(window.history.pushState).toHaveBeenCalledTimes(1);
  });

  it("dismisses the top registered handler first, leaving the module untouched", () => {
    const handler = vi.fn();
    useBackStack.getState().register(handler);
    useAppStore.setState({ activeModule: "calendar" });
    renderHook(() => useAndroidBackButton(true));
    popstate();
    expect(handler).toHaveBeenCalledTimes(1);
    expect(useAppStore.getState().activeModule).toBe("calendar");
    expect(window.history.back).not.toHaveBeenCalled();
  });

  it("goes up to Home when nothing is registered and not at Home", () => {
    useAppStore.setState({ activeModule: "lists" });
    renderHook(() => useAndroidBackButton(true));
    popstate();
    expect(useAppStore.getState().activeModule).toBeNull();
    expect(window.history.back).not.toHaveBeenCalled();
  });

  it("shows the hint on first root press and exits on the second", () => {
    vi.useFakeTimers();
    renderHook(() => useAndroidBackButton(true));
    popstate(); // first
    expect(toast).toHaveBeenCalledWith(
      expect.objectContaining({ description: "Press back again to exit" }),
    );
    expect(window.history.back).not.toHaveBeenCalled();
    popstate(); // second, within window
    act(() => {
      vi.advanceTimersByTime(1); // flush the deferred history.back()
    });
    expect(window.history.back).toHaveBeenCalledTimes(1);
    vi.useRealTimers();
  });

  it("re-arms after the window elapses", () => {
    vi.useFakeTimers();
    renderHook(() => useAndroidBackButton(true));
    popstate(); // first → arm
    act(() => {
      vi.advanceTimersByTime(2000); // window elapses, disarm
    });
    popstate(); // treated as first again
    expect(toast).toHaveBeenCalledTimes(2);
    expect(window.history.back).not.toHaveBeenCalled();
    vi.useRealTimers();
  });
});
```

- [ ] **Step 2: Run it — expect FAIL**

```bash
npx vitest run src/hooks/use-android-back-button.test.ts
```

- [ ] **Step 3: Implement**

```ts
import { useEffect, useRef } from "react";
import { toast } from "@/components/ui/toaster";
import { isIOS, isStandalone } from "@/lib/pwa";
import { useAppStore, useBackStack } from "@/stores";

const EXIT_HINT_MS = 2000;
const SENTINEL = { __familyHubBack: true } as const;

/** Installed + Android-class + touch: the only context with a hardware back. */
function isAndroidBackContext(): boolean {
  if (typeof window === "undefined") return false;
  const coarse = window.matchMedia?.("(pointer: coarse)").matches === true;
  return isStandalone() && !isIOS() && coarse;
}

/**
 * Hardware/gesture back handling for the installed Android PWA. Mount once in
 * the authenticated shell — it is the single owner of history interception.
 */
export function useAndroidBackButton(enabled: boolean): void {
  const armedRef = useRef(false);
  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  useEffect(() => {
    if (!enabled || !isAndroidBackContext()) return;

    const rebuffer = () => window.history.pushState(SENTINEL, "");
    rebuffer(); // initial sentinel: [appEntry, sentinel]

    const onPop = () => {
      // 1. Overlay/detail layer (LIFO). The surface closes and unregisters
      //    itself via its own useBackHandler effect; we only peek.
      const top = useBackStack.getState().peek();
      if (top) {
        top.handler();
        rebuffer();
        return;
      }
      // 2. Module layer (single-root): step up to Home.
      const { activeModule, setActiveModule } = useAppStore.getState();
      if (activeModule !== null) {
        setActiveModule(null);
        rebuffer();
        return;
      }
      // 3. Home root: double-press to exit.
      if (armedRef.current) {
        armedRef.current = false;
        if (timerRef.current) clearTimeout(timerRef.current);
        // Defer so the exit runs after this popstate settles.
        setTimeout(() => window.history.back(), 0);
        return;
      }
      armedRef.current = true;
      toast({ description: "Press back again to exit", duration: EXIT_HINT_MS });
      timerRef.current = setTimeout(() => {
        armedRef.current = false;
      }, EXIT_HINT_MS);
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

- [ ] **Step 4: Export from the barrel** — add to `src/hooks/index.ts`:

```ts
export { useAndroidBackButton } from "./use-android-back-button";
```

- [ ] **Step 5: Run it — expect PASS**, then commit

```bash
npx vitest run src/hooks/use-android-back-button.test.ts
git add src/hooks/use-android-back-button.ts src/hooks/use-android-back-button.test.ts src/hooks/index.ts
git commit -m "feat(back-button): add useAndroidBackButton popstate interceptor"
```

---

## Task 4: Mount the interceptor in `App.tsx`

**Files:**
- Modify: `src/App.tsx`
- Test: `src/App.shell.test.tsx` (extend)

> The interceptor must be called **above** the early returns (it's a hook). `App.shell.test.tsx`'s `matchMedia` mock returns `true` for non-width queries, so `isStandalone()` and `(pointer: coarse)` both pass and the gate is naturally on in that file — no extra mock needed. Existing shell tests stay green: they never dispatch `popstate`, and the on-mount sentinel `pushState` is benign.

- [ ] **Step 1: Write the failing test** — add to `src/App.shell.test.tsx` (reuses the file's existing `seedAuthStore`/`seedFamilyStore` beforeEach and `setViewportWidth` helper):

```ts
it("hardware back steps up to Home from a module (Android standalone)", async () => {
  setViewportWidth(768);
  useAppStore.setState({ activeModule: "calendar", isSidebarOpen: false });

  render(<FamilyHub />);
  await screen.findByRole("navigation", { name: /primary/i });

  act(() => {
    window.dispatchEvent(new PopStateEvent("popstate"));
  });

  expect(useAppStore.getState().activeModule).toBeNull();
});
```

- [ ] **Step 2: Run it — expect FAIL** (back press does nothing yet → `activeModule` stays `"calendar"`)

```bash
npx vitest run src/App.shell.test.tsx
```

- [ ] **Step 3: Implement** — in `src/App.tsx`, import the hook and call it once, **after** `isAuthenticated`/`setupComplete` are computed (around line 109) and **before** the first early return:

```tsx
import { useAndroidBackButton, useGoogleAuthReturn, useIsMobile } from "@/hooks";
// ...inside FamilyHub(), after `const setupComplete = useSetupComplete();`:
useAndroidBackButton(isAuthenticated && setupComplete);
```

- [ ] **Step 4: Run it — expect PASS**, then commit

```bash
npx vitest run src/App.shell.test.tsx
git add src/App.tsx src/App.shell.test.tsx
git commit -m "feat(back-button): mount the interceptor in the authenticated shell"
```

---

## Task 5: Register the leaf sheet primitives (`MobileSheet`, `SideSheet`)

**Files:**
- Modify: `src/components/ui/mobile-sheet.tsx`, `src/components/ui/side-sheet.tsx`
- Test: `src/components/ui/mobile-sheet.test.tsx` (extend)

> Registering here covers the "More" sheet, every `ResponsiveFormDialog` on mobile (it delegates to `MobileSheet`), the meal sheets, and the sidebar — with no consumer changes.

- [ ] **Step 1: Write the failing test** — add to `src/components/ui/mobile-sheet.test.tsx` (it already imports `MobileSheet`, `renderWithUser`, `screen`):

```tsx
import { act } from "@/test/test-utils";
import { useBackStack } from "@/stores";
import { vi } from "vitest";

it("registers its close handler on the back-stack while open", () => {
  const onClose = vi.fn();
  renderWithUser(
    <MobileSheet isOpen onClose={onClose} title="Test sheet">
      <button type="button">Inside</button>
    </MobileSheet>,
  );
  expect(useBackStack.getState().stack).toHaveLength(1);
  act(() => {
    useBackStack.getState().peek()?.handler();
  });
  expect(onClose).toHaveBeenCalledTimes(1);
});
```

- [ ] **Step 2: Run it — expect FAIL** (`stack` is empty)

```bash
npx vitest run src/components/ui/mobile-sheet.test.tsx
```

- [ ] **Step 3: Implement — `mobile-sheet.tsx`.** Import the hook and call it in the component body (before the `return`):

```tsx
import { useBackHandler } from "@/hooks";
// ...inside MobileSheet(), with the other hooks:
useBackHandler(isOpen, onClose);
```

- [ ] **Step 4: Implement — `side-sheet.tsx`.** Same, using its `open`/`onOpenChange` props:

```tsx
import { useBackHandler } from "@/hooks";
// ...inside SideSheet(), with the other hooks:
useBackHandler(open, () => onOpenChange(false));
```

- [ ] **Step 5: Run it — expect PASS**, then commit

```bash
npx vitest run src/components/ui/mobile-sheet.test.tsx
git add src/components/ui/mobile-sheet.tsx src/components/ui/side-sheet.tsx src/components/ui/mobile-sheet.test.tsx
git commit -m "feat(back-button): register sheet/side-sheet close handlers"
```

---

## Task 6: Register the lists & recipes detail views

**Files:**
- Modify: `src/components/lists-view.tsx`, `src/components/recipes-view.tsx`
- Test: `src/components/lists-view.test.tsx`, `src/components/recipes-view.test.tsx` (extend)

> Registering the existing `setSelected…(null)` is exactly the sibling spec's D5 seam — the detail-close slide (keyed on selection state) animates the back for free.

- [ ] **Step 1: Write the failing tests** — add to each view's existing test file (reusing its `seedMock…`/`renderWithUser` setup and the file's own open-detail interaction):

```tsx
// lists-view.test.tsx — uses the existing groceryList fixture
import { act } from "@/test/test-utils";
import { useBackStack } from "@/stores";

it("registers a back handler that closes the open list detail", async () => {
  seedMockLists([groceryList]);
  const { user } = renderWithUser(<ListsView />);
  await user.click(await screen.findByRole("button", { name: /Trader Joe's Run/i }));
  expect(useBackStack.getState().stack).toHaveLength(1);
  act(() => {
    useBackStack.getState().peek()?.handler();
  });
  expect(await screen.findByRole("button", { name: /new list/i })).toBeInTheDocument();
});
```

```tsx
// recipes-view.test.tsx — uses the existing testRecipeDetail fixture
import { act } from "@/test/test-utils";
import { useBackStack } from "@/stores";

it("registers a back handler that closes the open recipe detail", async () => {
  seedMockRecipes([testRecipeDetail]);
  const { user } = renderWithUser(<RecipesView />);
  await user.click(
    await screen.findByRole("button", { name: `Open recipe: ${testRecipeDetail.title}` }),
  );
  expect(useBackStack.getState().stack).toHaveLength(1);
  act(() => {
    useBackStack.getState().peek()?.handler();
  });
  expect(await screen.findByRole("button", { name: /add recipe/i })).toBeInTheDocument();
});
```

- [ ] **Step 2: Run them — expect FAIL**

```bash
npx vitest run src/components/lists-view.test.tsx src/components/recipes-view.test.tsx
```

- [ ] **Step 3: Implement — `lists-view.tsx`.** Import the hook and register while a list is selected (place with the other hooks, before `body`):

```tsx
import { useBackHandler } from "@/hooks";
// ...inside ListsView(), after the useState declarations:
useBackHandler(selectedListId !== null, () => setSelectedListId(null));
```

- [ ] **Step 4: Implement — `recipes-view.tsx`.** Same shape, gated on `selectedRecipeId`:

```tsx
import { useBackHandler } from "@/hooks";
// ...inside RecipesView(), after the useState declarations:
useBackHandler(selectedRecipeId !== null, () => setSelectedRecipeId(null));
```

- [ ] **Step 5: Run them — expect PASS**, then commit

```bash
npx vitest run src/components/lists-view.test.tsx src/components/recipes-view.test.tsx
git add src/components/lists-view.tsx src/components/recipes-view.tsx src/components/lists-view.test.tsx src/components/recipes-view.test.tsx
git commit -m "feat(back-button): register list and recipe detail back handlers"
```

---

## Task 7: Register the raw `<Dialog>` sites

**Files:**
- Modify: `src/components/calendar/components/event-detail-modal.tsx`, `src/components/calendar/components/event-form-modal.tsx`, `src/components/calendar/components/edit-scope-dialog.tsx`, `src/components/settings/family-settings-modal.tsx`, `src/components/settings/google-calendar-picker-modal.tsx`, `src/components/shared/sidebar-menu.tsx`
- Test: `src/components/calendar/components/event-detail-modal.test.tsx` (extend — representative)

> These use raw Radix `Dialog` whose open-state lives at the call site, so each needs the one-liner. Pattern: `useBackHandler(<openFlag>, <close>)`.

- [ ] **Step 1: Write the failing test** — extend `event-detail-modal.test.tsx`. Its `vi.mock("@/hooks", …)` is a *full* barrel mock, so it must now also provide a **real** `useBackHandler` (spread the actual module) for the registration to occur:

```tsx
// Replace the existing vi.mock("@/hooks", …) with a partial mock so
// useBackHandler is real while useIsMobile stays forced to desktop:
vi.mock("@/hooks", async (importActual) => ({
  ...(await importActual<typeof import("@/hooks")>()),
  useIsMobile: () => false,
  usePressable: () => ({ className: "", onPointerDown: () => {} }),
}));

// new test (defaultProps already has isOpen: true, onClose: mockClose):
it("registers a back handler that closes the modal", () => {
  render(<EventDetailModal {...defaultProps} />);
  expect(useBackStack.getState().stack).toHaveLength(1);
  act(() => {
    useBackStack.getState().peek()?.handler();
  });
  expect(mockClose).toHaveBeenCalledTimes(1);
});
```

Add the imports this test needs at the top of the file: `import { act } from "@/test/test-utils";` and `import { useBackStack } from "@/stores";`.

- [ ] **Step 2: Run it — expect FAIL**

```bash
npx vitest run src/components/calendar/components/event-detail-modal.test.tsx
```

- [ ] **Step 3: Implement** — add `import { useBackHandler } from "@/hooks";` and one registration line in each component, matching its existing open/close props:

  - `event-detail-modal.tsx`: `useBackHandler(isOpen, onClose);`
  - `event-form-modal.tsx`: `useBackHandler(isOpen, onClose);`
  - `edit-scope-dialog.tsx`: `useBackHandler(isOpen, onClose);`
  - `google-calendar-picker-modal.tsx`: `useBackHandler(open, () => onOpenChange(false));`
  - `family-settings-modal.tsx` (the reset-confirm `<Dialog>`): `useBackHandler(showResetConfirm, () => setShowResetConfirm(false));`
  - `sidebar-menu.tsx` (the sign-out-confirm `<Dialog>`): `useBackHandler(showSignOutConfirm, () => setShowSignOutConfirm(false));`

  (Audit for any further raw `<Dialog>` usages with `rg -n "<Dialog\b" src` and apply the same one-liner.)

- [ ] **Step 4: Run it — expect PASS**, then commit

```bash
npx vitest run src/components/calendar/components/event-detail-modal.test.tsx
git add src/components/calendar/components/event-detail-modal.tsx src/components/calendar/components/event-form-modal.tsx src/components/calendar/components/edit-scope-dialog.tsx src/components/settings/family-settings-modal.tsx src/components/settings/google-calendar-picker-modal.tsx src/components/shared/sidebar-menu.tsx src/components/calendar/components/event-detail-modal.test.tsx
git commit -m "feat(back-button): register raw dialog close handlers"
```

---

## Task 8: Full check, manual device pass, PR

- [ ] **Step 1: Full suite + lint**

```bash
npm run lint && npm test -- --run
```
Expected: PASS.

- [ ] **Step 2: Manual device pass (the real gate)** — Galaxy S10 / Chrome, **installed** PWA:
  - Open a list/recipe detail → back closes it (slides left), back again goes Home (fades), back shows the "Press back again to exit" toast, back again exits.
  - Repeat with the "More" sheet, a form sheet, a confirm-dialog-inside-settings (the confirm closes before its parent), and the sidebar — most-recent always closes first.
  - Refresh mid-session → lands on Home, back still buffered (doesn't instantly exit).
  - Open in a normal browser tab and (if available) iOS standalone → back is unaffected (browser default).

- [ ] **Step 3: Contract checklist** (map each AC → code):
  - Single back dismisses top overlay (LIFO) → Tasks 1/2 + registration Tasks 5/6/7.
  - Up to Home when nothing open & not Home → Task 3 (cascade step 2) + Task 4 mount.
  - First root press shows toast, second within ~2s exits → Task 3 (cascade step 3).
  - Gated to Android standalone; tabs/iOS/desktop unaffected → Task 3 `isAndroidBackContext` + Task 4 `enabled`.
  - No trap/leak; refresh→Home buffered → Task 3 single re-pushed sentinel + unpersisted store.
  - No new motion (sibling transitions animate the back) → Tasks 4/6 drive `setActiveModule(null)`/`setSelected…(null)` only.

- [ ] **Step 4: Open PR** with `Closes #<issue>` and the Story/Spec/Plan links.

```bash
git push -u origin feat/native-back-button
gh pr create --title "feat: native hardware back-button (single-root + dismiss registry)" --body "Implements docs/superpowers/specs/2026-06-17-native-back-button-design.md. Closes #<issue>."
```

---

## Self-review

- **Spec coverage:** registry (D1) → Task 1; `useBackHandler` seam (D1) → Task 2; interceptor + cascade + sentinel + gate (D2/D3/D6) → Task 3; single-mount in shell (D2/D6) → Task 4; leaf-primitive registration (D4) → Task 5; detail-view registration / sibling D5 seam (D4) → Task 6; raw `<Dialog>` registration (D4) → Task 7; exit-hint toast (D5) → Task 3; testing strategy → Tasks 1–7 unit/integration + Task 8 manual.
- **Type consistency:** `useBackStack` exposes `register(handler) → number` / `unregister(id)` / `peek()` used identically in Tasks 1, 2, 3, 5, 6, 7. `useBackHandler(enabled, handler)` signature identical in Tasks 5, 6, 7. `useAndroidBackButton(enabled)` identical in Tasks 3, 4. The exit-hint string `"Press back again to exit"` matches between Task 3 implementation and its test.
- **Placeholders:** none — every step has real code; selectors (`/Trader Joe's Run/i`, `Open recipe: …`, `/new list/i`, `/add recipe/i`) and fixtures (`groceryList`, `testRecipeDetail`) are the ones already used in those test files; the raw-`<Dialog>` open/close props are named per call site with an `rg` audit step.
- **Known sharp edge:** `event-detail-modal.test.tsx` must switch its full `@/hooks` mock to a partial (spread-actual) mock so `useBackHandler` is real — called out explicitly in Task 7 Step 1.
- **No new dependency:** reuses Zustand, `toast`, `pwa.ts`, History API; no router, no motion lib.
