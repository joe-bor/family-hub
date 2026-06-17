# Native-Feel Interaction Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a cohesive "acknowledgment layer" — fade-through on module switches, shared-axis slide on list/recipe detail, and immediate scale+tint press feedback on every tappable — leaving single-file seams for the haptics and back-button stories.

**Architecture:** Two reusable primitives. `usePressable()` returns a press-visual class + a `onPointerDown` seam (the haptic hook point). `ScreenTransition` runs an enter-transition (Web Animations API) on a `token` change, in `fade` or `slide` mode. Both are `prefers-reduced-motion`-aware. No new dependency.

**Tech Stack:** React 19, Vite, Tailwind CSS v4, `tailwindcss-animate`, Web Animations API, Vitest + Testing Library. `cn()` from `@/lib/utils`.

**Spec:** `docs/superpowers/specs/2026-06-16-native-feel-interaction-polish-design.md`
**Story:** `docs/product/backlog/mobile-ux/native-feel-interaction-polish.md`

---

## Execution context

- Implement in the **FamilyHub FE repo** (`frontend/` in the workspace; standalone `joe-bor/FamilyHub`). All paths below are relative to the FE repo root.
- Branch: `feat/native-feel-interaction-polish`. Regular merge commits (not squash) per FE `CLAUDE.md` versioning; these are `feat:` commits (minor bump).
- Single-file test run: `npx vitest run <path>`. Full check before PR: `npm run lint && npm test -- --run`.

## File structure

| File | Responsibility |
|---|---|
| `src/hooks/use-prefers-reduced-motion.ts` (create) | `prefers-reduced-motion: reduce` boolean (wraps `useMediaQuery`) |
| `src/hooks/use-pressable.ts` (create) | `PRESSABLE` class + `usePressable()` seam |
| `src/components/shared/screen-transition.tsx` (create) | `ScreenTransition` (fade \| slide, direction, reduced-motion) |
| `src/hooks/index.ts` / `src/components/shared/index.ts` (modify) | Barrel exports |
| `src/components/ui/button.tsx` (modify) | Press feedback on all buttons |
| `src/App.tsx` (modify) | Fade-through around `renderModule` |
| `src/components/lists-view.tsx`, `src/components/recipes-view.tsx` (modify) | Slide on detail open/close |
| `src/components/shared/mobile-bottom-nav.tsx`, `src/components/shared/sidebar-menu.tsx` (modify) | Press feedback on raw tappables |

---

## Setup

- [ ] **Branch from latest `main`**

```bash
git checkout main && git pull
git checkout -b feat/native-feel-interaction-polish
```

- [ ] **Read the spec and story** linked above so the contract is fresh.

---

## Task 1: `usePrefersReducedMotion` hook

**Files:**
- Create: `src/hooks/use-prefers-reduced-motion.ts`
- Test: `src/hooks/use-prefers-reduced-motion.test.ts`
- Modify: `src/hooks/index.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { renderHook } from "@testing-library/react";
import { describe, expect, it, vi } from "vitest";
import { usePrefersReducedMotion } from "./use-prefers-reduced-motion";

function mockMatch(matches: boolean) {
  window.matchMedia = vi.fn().mockImplementation((query: string) => ({
    matches, media: query, onchange: null,
    addEventListener: vi.fn(), removeEventListener: vi.fn(),
    addListener: vi.fn(), removeListener: vi.fn(), dispatchEvent: vi.fn(),
  }));
}

describe("usePrefersReducedMotion", () => {
  it("returns true when reduce is preferred", () => {
    mockMatch(true);
    expect(renderHook(() => usePrefersReducedMotion()).result.current).toBe(true);
  });
  it("returns false otherwise", () => {
    mockMatch(false);
    expect(renderHook(() => usePrefersReducedMotion()).result.current).toBe(false);
  });
});
```

- [ ] **Step 2: Run it — expect FAIL** (`Cannot find module './use-prefers-reduced-motion'`)

```bash
npx vitest run src/hooks/use-prefers-reduced-motion.test.ts
```

- [ ] **Step 3: Implement**

```ts
import { useMediaQuery } from "./use-media-query";

/** True when the user has requested reduced motion. */
export function usePrefersReducedMotion(): boolean {
  return useMediaQuery("(prefers-reduced-motion: reduce)");
}
```

- [ ] **Step 4: Export from the barrel** — add to `src/hooks/index.ts`:

```ts
export { usePrefersReducedMotion } from "./use-prefers-reduced-motion";
```

- [ ] **Step 5: Run it — expect PASS**

```bash
npx vitest run src/hooks/use-prefers-reduced-motion.test.ts
```

- [ ] **Step 6: Commit**

```bash
git add src/hooks/use-prefers-reduced-motion.ts src/hooks/use-prefers-reduced-motion.test.ts src/hooks/index.ts
git commit -m "feat(motion): add usePrefersReducedMotion hook"
```

---

## Task 2: `usePressable` primitive

**Files:**
- Create: `src/hooks/use-pressable.ts`
- Test: `src/hooks/use-pressable.test.ts`
- Modify: `src/hooks/index.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { renderHook } from "@testing-library/react";
import { describe, expect, it } from "vitest";
import { PRESSABLE, usePressable } from "./use-pressable";

describe("usePressable", () => {
  it("gates the scale behind motion-safe but keeps the tint always-on", () => {
    expect(PRESSABLE).toContain("motion-safe:active:scale-[0.97]");
    expect(PRESSABLE).toContain("active:before:opacity-[0.06]");
    // the tint must NOT be motion-gated — it is feedback, not decoration
    expect(PRESSABLE).not.toContain("motion-safe:active:before");
  });
  it("returns the class and a callable pointer-down seam", () => {
    const { result } = renderHook(() => usePressable());
    expect(result.current.className).toBe(PRESSABLE);
    expect(() => result.current.onPointerDown({} as never)).not.toThrow();
  });
});
```

- [ ] **Step 2: Run it — expect FAIL**

```bash
npx vitest run src/hooks/use-pressable.test.ts
```

- [ ] **Step 3: Implement**

```ts
import { useCallback } from "react";
import type { PointerEvent } from "react";

/**
 * Press-feedback visual. The scale is motion-gated (reduced-motion drops it);
 * the tint overlay is always-on (it is feedback). `rounded-[inherit]` so the
 * overlay matches the host element's corners.
 */
export const PRESSABLE =
  "relative transition-transform duration-100 ease-out motion-safe:active:scale-[0.97] " +
  "before:pointer-events-none before:absolute before:inset-0 before:rounded-[inherit] " +
  "before:bg-foreground before:opacity-0 before:transition-opacity before:duration-100 " +
  "active:before:opacity-[0.06]";

export interface Pressable {
  className: string;
  onPointerDown: (event: PointerEvent) => void;
}

/** Single integration point for press visuals and (later) haptics. */
export function usePressable(): Pressable {
  const onPointerDown = useCallback((_event: PointerEvent) => {
    // Haptic seam: the optional-haptics story adds `useHaptics().tap()` here.
  }, []);
  return { className: PRESSABLE, onPointerDown };
}
```

- [ ] **Step 4: Export from the barrel** — add to `src/hooks/index.ts`:

```ts
export { PRESSABLE, usePressable } from "./use-pressable";
```

- [ ] **Step 5: Run it — expect PASS**, then commit

```bash
npx vitest run src/hooks/use-pressable.test.ts
git add src/hooks/use-pressable.ts src/hooks/use-pressable.test.ts src/hooks/index.ts
git commit -m "feat(motion): add usePressable press-feedback primitive"
```

---

## Task 3: Press feedback on `Button`

**Files:**
- Modify: `src/components/ui/button.tsx`
- Test: `src/components/ui/button.test.tsx` (create if absent)

> **Note on `transition-all`:** `buttonVariants` base has `transition-all`; `PRESSABLE` adds `transition-transform`, which tailwind-merge resolves to `transition-transform` (last wins). Accepted tradeoff: hover background-color becomes instant on desktop (irrelevant on the touch target). If you want both, change the base `transition-all` → `transition-colors` and keep `PRESSABLE`'s transform transition.

- [ ] **Step 1: Write the failing test**

```tsx
import { render, screen } from "@testing-library/react";
import { describe, expect, it, vi } from "vitest";
import { Button } from "./button";

describe("Button press feedback", () => {
  it("carries the pressable class", () => {
    render(<Button>Save</Button>);
    expect(screen.getByRole("button").className).toContain("active:scale-[0.97]");
  });
  it("fires the pressable seam and the caller's onPointerDown", () => {
    const onPointerDown = vi.fn();
    render(<Button onPointerDown={onPointerDown}>Save</Button>);
    screen.getByRole("button").dispatchEvent(new Event("pointerdown", { bubbles: true }));
    expect(onPointerDown).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 2: Run it — expect FAIL**

```bash
npx vitest run src/components/ui/button.test.tsx
```

- [ ] **Step 3: Implement** — update the `Button` function body in `src/components/ui/button.tsx`:

```tsx
import { usePressable } from "@/hooks";
// ...
function Button({ className, variant, size, asChild = false, onPointerDown, ...props }: /* same props type */) {
  const Comp = asChild ? Slot : "button";
  const pressable = usePressable();
  return (
    <Comp
      data-slot="button"
      className={cn(buttonVariants({ variant, size }), pressable.className, className)}
      onPointerDown={(event) => {
        pressable.onPointerDown(event);
        onPointerDown?.(event);
      }}
      {...props}
    />
  );
}
```

- [ ] **Step 4: Run it — expect PASS**, then commit

```bash
npx vitest run src/components/ui/button.test.tsx
git add src/components/ui/button.tsx src/components/ui/button.test.tsx
git commit -m "feat(ui): apply press feedback to Button"
```

---

## Task 4: `ScreenTransition` primitive

**Files:**
- Create: `src/components/shared/screen-transition.tsx`
- Test: `src/components/shared/screen-transition.test.tsx`
- Modify: `src/components/shared/index.ts`

- [ ] **Step 1: Write the failing test**

```tsx
import { render } from "@testing-library/react";
import { beforeEach, describe, expect, it, vi } from "vitest";
import { ScreenTransition } from "./screen-transition";

const animateMock = vi.fn();
function setReduce(matches: boolean) {
  window.matchMedia = vi.fn().mockImplementation((query: string) => ({
    matches, media: query, onchange: null,
    addEventListener: vi.fn(), removeEventListener: vi.fn(),
    addListener: vi.fn(), removeListener: vi.fn(), dispatchEvent: vi.fn(),
  }));
}
beforeEach(() => {
  animateMock.mockReset();
  (Element.prototype as unknown as { animate: unknown }).animate = animateMock; // jsdom lacks WAAPI
});

describe("ScreenTransition", () => {
  it("does not animate on first mount, fades on token change", () => {
    setReduce(false);
    const { rerender } = render(<ScreenTransition token="a" mode="fade">A</ScreenTransition>);
    expect(animateMock).not.toHaveBeenCalled();
    rerender(<ScreenTransition token="b" mode="fade">B</ScreenTransition>);
    expect(animateMock).toHaveBeenCalledTimes(1);
    expect(animateMock.mock.calls[0][0][0]).toMatchObject({ transform: "scale(1.012)" });
  });
  it("slides right on forward, left on back", () => {
    setReduce(false);
    const { rerender } = render(<ScreenTransition token="l" mode="slide" direction="forward">L</ScreenTransition>);
    rerender(<ScreenTransition token="d" mode="slide" direction="forward">D</ScreenTransition>);
    expect(animateMock.mock.calls[0][0][0].transform).toBe("translateX(22%)");
    rerender(<ScreenTransition token="l" mode="slide" direction="back">L</ScreenTransition>);
    expect(animateMock.mock.calls[1][0][0].transform).toBe("translateX(-22%)");
  });
  it("skips animation under reduced motion but still swaps content", () => {
    setReduce(true);
    const { rerender, getByText } = render(<ScreenTransition token="a" mode="fade">A</ScreenTransition>);
    rerender(<ScreenTransition token="b" mode="fade">B</ScreenTransition>);
    expect(animateMock).not.toHaveBeenCalled();
    expect(getByText("B")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run it — expect FAIL**

```bash
npx vitest run src/components/shared/screen-transition.test.tsx
```

- [ ] **Step 3: Implement**

```tsx
import { useLayoutEffect, useRef } from "react";
import type { ReactNode } from "react";
import { usePrefersReducedMotion } from "@/hooks";

type Mode = "fade" | "slide";
type Direction = "forward" | "back";

interface ScreenTransitionProps {
  token: string | number | null;
  mode: Mode;
  direction?: Direction;
  children: ReactNode;
}

/** Enter-transition: the incoming screen animates in on `token` change. */
export function ScreenTransition({ token, mode, direction = "forward", children }: ScreenTransitionProps) {
  const ref = useRef<HTMLDivElement>(null);
  const prev = useRef(token);
  const reduce = usePrefersReducedMotion();

  useLayoutEffect(() => {
    if (prev.current === token) return;
    prev.current = token;
    const el = ref.current;
    if (!el || reduce) return;
    const keyframes: Keyframe[] =
      mode === "slide"
        ? [
            { opacity: 0, transform: `translateX(${direction === "back" ? "-22%" : "22%"})` },
            { opacity: 1, transform: "none" },
          ]
        : [
            { opacity: 0, transform: "scale(1.012)" },
            { opacity: 1, transform: "none" },
          ];
    el.animate(keyframes, { duration: mode === "slide" ? 280 : 200, easing: "cubic-bezier(0.2, 0, 0, 1)" });
  }, [token, mode, direction, reduce]);

  return (
    <div ref={ref} className="flex min-h-0 flex-1 flex-col">
      {children}
    </div>
  );
}
```

- [ ] **Step 4: Export from the barrel** — add to `src/components/shared/index.ts`:

```ts
export { ScreenTransition } from "./screen-transition";
```

- [ ] **Step 5: Run it — expect PASS**, then commit

```bash
npx vitest run src/components/shared/screen-transition.test.tsx
git add src/components/shared/screen-transition.tsx src/components/shared/screen-transition.test.tsx src/components/shared/index.ts
git commit -m "feat(motion): add ScreenTransition primitive"
```

---

## Task 5: Fade-through on module switch (`App.tsx`)

**Files:**
- Modify: `src/App.tsx` (the `<main>` at `:168-170`)
- Test: `src/App.shell.test.tsx` (extend)

- [ ] **Step 1: Write the failing test** — add to `src/App.shell.test.tsx`, reusing the file's existing authenticated-shell render/seed setup. Mock WAAPI, then change the module and assert an animation runs:

```tsx
it("animates the module container when the active module changes", async () => {
  const animateMock = vi.fn();
  (Element.prototype as unknown as { animate: unknown }).animate = animateMock;
  renderAuthenticatedShell(); // existing helper in this file
  act(() => { useAppStore.getState().setActiveModule("calendar"); });
  expect(animateMock).toHaveBeenCalled();
});
```

- [ ] **Step 2: Run it — expect FAIL** (`npx vitest run src/App.shell.test.tsx`)

- [ ] **Step 3: Implement** — wrap the module render:

```tsx
import { /* …, */ ScreenTransition } from "@/components/shared";
// ...
<main className="flex-1 min-h-0 flex flex-col overflow-hidden">
  <ScreenTransition token={activeModule} mode="fade">
    {renderModule(activeModule)}
  </ScreenTransition>
</main>
```

- [ ] **Step 4: Run it — expect PASS**, then commit

```bash
npx vitest run src/App.shell.test.tsx
git add src/App.tsx src/App.shell.test.tsx
git commit -m "feat(motion): fade-through on module switch"
```

---

## Task 6: Shared-axis slide on detail open/close

**Files:**
- Modify: `src/components/lists-view.tsx`, `src/components/recipes-view.tsx`
- Test: `src/components/lists-view.test.tsx`, `src/components/recipes-view.test.tsx` (extend)

- [ ] **Step 1: Write the failing tests** — in each view's existing test file (reuse its render/seed setup), mock WAAPI and assert direction:

```tsx
// lists-view.test.tsx — within existing setup that renders ListsView with seeded lists
it("slides the detail in from the right on open", async () => {
  const animateMock = vi.fn();
  (Element.prototype as unknown as { animate: unknown }).animate = animateMock;
  renderListsView(); // existing helper
  await userEvent.click(screen.getByRole("button", { name: /open .*grocer/i })); // an existing list row
  expect(animateMock.mock.calls.at(-1)?.[0][0].transform).toBe("translateX(22%)");
});
```

- [ ] **Step 2: Run them — expect FAIL**

```bash
npx vitest run src/components/lists-view.test.tsx src/components/recipes-view.test.tsx
```

- [ ] **Step 3: Implement — `lists-view.tsx`.** Collapse the early-return into one wrapped ternary. Replace the `if (selectedListId !== null) { return <ListDetailView … /> }` block + the trailing list return with:

```tsx
import { ScreenTransition } from "@/components/shared";
// ...
return (
  <ScreenTransition
    token={selectedListId ?? "__list__"}
    mode="slide"
    direction={selectedListId ? "forward" : "back"}
  >
    {selectedListId !== null ? (
      <ListDetailView
        listId={selectedListId}
        /* …existing props… */
        onBack={() => setSelectedListId(null)}
      />
    ) : (
      <>{/* the existing list/grid JSX exactly as before */}</>
    )}
  </ScreenTransition>
);
```

- [ ] **Step 4: Implement — `recipes-view.tsx`.** It already branches on `selectedRecipeId` (`:168-177`). Wrap that branch region in the same `ScreenTransition` with `token={selectedRecipeId ?? "__list__"}` and `direction={selectedRecipeId ? "forward" : "back"}`. Keep the inner loading/error states unchanged.

- [ ] **Step 5: Run them — expect PASS**, then commit

```bash
npx vitest run src/components/lists-view.test.tsx src/components/recipes-view.test.tsx
git add src/components/lists-view.tsx src/components/recipes-view.tsx src/components/lists-view.test.tsx src/components/recipes-view.test.tsx
git commit -m "feat(motion): shared-axis slide for list and recipe detail"
```

---

## Task 7: Press feedback on remaining tappables

**Files:**
- Modify: `src/components/shared/mobile-bottom-nav.tsx`, `src/components/shared/sidebar-menu.tsx`, `src/components/lists-view.tsx`, `src/components/recipes-view.tsx`
- Test: `src/components/shared/mobile-bottom-nav.test.tsx` (extend)

- [ ] **Step 1: Write the failing test** — `mobile-bottom-nav.test.tsx`:

```tsx
it("gives nav tabs press feedback", () => {
  renderBottomNav(); // existing helper
  expect(screen.getByRole("button", { name: "Calendar" }).className).toContain("active:scale-[0.97]");
});
```

- [ ] **Step 2: Run it — expect FAIL** (`npx vitest run src/components/shared/mobile-bottom-nav.test.tsx`)

- [ ] **Step 3: Implement.** In each component, call `const pressable = usePressable();` once at the top (the value is static, safe to reuse across `.map`), then add `pressable.className` into the element's `cn(...)` and `onPointerDown={pressable.onPointerDown}`:
  - `mobile-bottom-nav.tsx`: the primary tab buttons (`:66-77`), the "More" button (`:81-91`), and overflow rows (`:106-121`).
  - `sidebar-menu.tsx`: the menu-row buttons.
  - `lists-view.tsx`: the list-row button (`onOpen`).
  - `recipes-view.tsx`: the recipe card/button that opens detail.

  Example (a nav tab):

```tsx
<button
  /* …existing… */
  onPointerDown={pressable.onPointerDown}
  className={cn(tabBase, pressable.className, isActive ? tabActive : tabIdle)}
>
```

- [ ] **Step 4: Run it — expect PASS**, then commit

```bash
npx vitest run src/components/shared/mobile-bottom-nav.test.tsx
git add src/components/shared/mobile-bottom-nav.tsx src/components/shared/sidebar-menu.tsx src/components/lists-view.tsx src/components/recipes-view.tsx src/components/shared/mobile-bottom-nav.test.tsx
git commit -m "feat(ui): press feedback on nav, sidebar, and rows"
```

---

## Task 8: Full check, manual device pass, PR

- [ ] **Step 1: Full suite + lint**

```bash
npm run lint && npm test -- --run
```
Expected: PASS.

- [ ] **Step 2: Manual device pass (the real gate)** — Galaxy S10 / Chrome:
  - Module switches fade-through; not instant.
  - Opening a list/recipe slides in from the right; back slides in from the left.
  - Buttons, nav tabs, and list/recipe rows visibly dip + spring on press; smooth (60fps), no jank.
  - OS "reduce motion" ON → transitions are instant cuts, but presses still tint (no dip).

- [ ] **Step 3: Contract checklist** (map each to code): differentiated transitions ✔ (Tasks 5/6); press feedback on all tappables ✔ (Tasks 3/7); reduced-motion split ✔ (CSS `motion-safe:` in Task 2 + JS short-circuit in Task 4); transform/opacity only ✔; single-file seams ✔ (`usePressable.onPointerDown`; slide keyed on existing close handlers).

- [ ] **Step 4: Open PR** with `Closes #<issue>` and the Story/Spec/Plan links.

```bash
git push -u origin feat/native-feel-interaction-polish
gh pr create --title "feat: native-feel interaction polish (transitions + press feedback)" --body "Implements docs/superpowers/specs/2026-06-16-native-feel-interaction-polish-design.md. Closes #<issue>."
```

---

## Self-review

- **Spec coverage:** fade-through (Task 5) ✔; shared-axis slide (Task 6) ✔; scale+tint press on all tappables (Tasks 3, 7) ✔; reduced-motion split (Tasks 2, 4) ✔; `usePressable` haptic seam (Task 2) ✔; back-button seam = slide keyed on existing close handlers (Task 6) ✔; no new dependency / WAAPI (Task 4) ✔; barrel exports (Tasks 1, 2, 4) ✔.
- **Type consistency:** `usePressable` → `{ className, onPointerDown }` used identically in Tasks 3 and 7. `ScreenTransition` props `{ token, mode, direction, children }` used identically in Tasks 5 and 6. `PRESSABLE` token `active:scale-[0.97]` asserted in Tasks 2, 3, 7.
- **Placeholders:** primitive tasks (1, 2, 3, 4) are fully self-contained. Tasks 5–7 extend existing test files and reuse their established render/seed helpers (named, not invented) because that infra already exists in those files — the assertions and production edits are concrete.
- **Known tradeoff:** Button `transition-all` → `transition-transform` (instant hover color on desktop), documented in Task 3 with the opt-out.
