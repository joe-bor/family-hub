# Large-Screen Lists Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** On large screens (1024px+), render Lists as a persistent two-pane master/detail — a ~340px lists rail plus the selected list's detail filling the remaining width — so a kitchen tablet shows every list and the active list's items at once, while mobile and the 768–1023px range keep today's drill-in navigation exactly as-is.

**Architecture:** Turn `ListsView` into a thin responsive router (the same pattern the Home hub used): below `lg` it renders `ListsMobileView` (today's `ListsView` body, moved verbatim), and at `lg+` it renders a new `ListsLargeScreen` that owns the two-pane layout. Only the mounted branch manages selection state and consumes the cross-module handoff intent, so there is no double-consume. The large-screen selection is auto-reconciled against the live lists array (first list auto-selected; a selection that disappears falls back to the first remaining), which satisfies the spec's create/delete selection AC even though the FE has **no whole-list delete UI today**.

**Tech Stack:** React 19, TypeScript, Vite, TanStack Query, Zustand (`useAppStore`), Tailwind CSS v4, Vitest + Testing Library, Playwright, Biome. No new dependencies. The existing `useIsLargeScreen()` hook (min-width 1024px, added by the Calendar story, exported from `@/hooks`) is reused unchanged — its breakpoint already matches this spec's Scope.

**Spec:** `docs/superpowers/specs/2026-07-06-large-screen-lists-design.md`
**Story:** `docs/product/backlog/large-screen-ux/large-screen-lists.md`
**Depends on (merged):** Large-screen foundations (FE PR #282), Large-screen Calendar (FE PR #285 — source of `useIsLargeScreen`).

**Repo:** All implementation work is in `frontend/` (separate git repo). This plan lives in the root docs repo for cross-repo planning. Branch: `feat/large-screen-lists`.

---

## Current Code (grounding — read before Task 1)

Everything below is verified against the current `frontend/` tree.

- **`src/components/lists-view.tsx`** — the module root. Holds `selectedListId` state (`null` = index, non-null = detail), `createOpen`, consumes `useAppStore(s => s.consumeListDetailIntent)` on mount, registers `useBackHandler(selectedListId !== null, () => setSelectedListId(null))`, and renders an index (a `mx-auto max-w-2xl` centered grid of `ListCard`) or `ListDetailView`, wrapped in `<ScreenTransition mode="slide">`. Mobile shows a `FloatingActionButton`. The create sheet's `onCreated={(id) => setSelectedListId(id)}` already selects on create.
- **`src/components/lists/list-detail-view.tsx`** — props `{ listId, preferences, preferencesStatus, onBack }`. Renders a "Back to Lists" ghost button (in both the normal and the not-found branches), then the list card + items. Uses `useIsMobile()` internally to switch between the desktop inline options and the mobile options sheet + FAB. Item content is capped at `mx-auto max-w-2xl` (spec explicitly allows this reading-width cap inside the pane). **There is no delete-whole-list action here** (only `useDeleteListItem`, item-level).
- **`src/components/lists/list-card.tsx`** — props `{ list: ListSummary, onOpen }`. Contains a local `kindMeta` map (icon + label + badge className per `ListKind`). Tall card (`min-h-32`).
- **`src/lib/types/lists.ts`** — `ListSummary = { id, name, kind, totalItems, completedItems }`. `ListKind = "grocery" | "to-do" | "general"`.
- **`src/stores/app-store.ts`** — cross-module handoff: `openListDetail(listId)` sets `{ listDetailIntent: listId, activeModule: "lists" }`; `consumeListDetailIntent()` returns and clears it. Meals/Home call `openListDetail`; Lists consumes it.
- **`src/api/hooks/use-lists.ts`** — `useLists()` (summaries), `useList(id)`, `useListPreferences()`, `useCreateList()`, `useDeleteListItem()`, etc. **No `useDeleteList`** — confirmed absent. There is no way to delete an entire list in the FE.
- **`src/hooks/use-is-large-screen.ts`** — `useIsLargeScreen()` → `useMediaQuery("(min-width: 1024px)")`; exported from the `@/hooks` barrel. `useBackHandler(enabled, handler)` no-ops when `enabled` is false.
- **Test infra:** `src/test/setup.ts` stubs `window.matchMedia` to always return `matches: false`, so the real `useIsLargeScreen()` returns **false** in tests unless mocked. `src/components/lists-view.test.tsx` already mocks `@/hooks` via a hoisted `viewport` object (`vi.mock("@/hooks", ... useIsMobile: () => viewport.isMobile)`) and seeds data with `seedMockLists([...])`, `seedMockListPreferences(...)`, `seedFamilyStore(...)` over an MSW server (`setupMswServer()`). `useBackStack` from `@/stores` is used to assert back-handler registration.

### Delete-fallback AC — how it is satisfied without a delete button

The spec AC "Deleting the selected list falls back to the first remaining list, or the empty state when none remain" cannot be wired to a delete button because none exists. Instead, `ListsLargeScreen` derives the **effective** selection from the live summaries every render (see Task 7 for the exact expression, including the in-flight-refetch guard):

- nothing selected, or the selected id vanished (delete / cross-device sync) and no refetch is in flight → first remaining list (or `null` → full-area empty state);
- a valid selected id → keep it;
- a just-created / just-selected id not yet in `summaries` while a hub refetch is in flight → keep it (its detail cache is already primed by `useCreateList`).

State is **never** overwritten by a reconcile effect — the fallback lives purely in the derivation, so a just-created id is not clobbered during the post-create hub refetch. This covers auto-select-first, delete/sync fallback, create-selects, and intent-selects in one expression. The delete-fallback test therefore simulates the summaries array losing the selected id (forcing a hub refetch that returns fewer lists), not a delete click.

## File Structure

**Create:**
- `src/components/lists/list-kind-meta.ts` — shared `kindMeta` map (moved out of `list-card.tsx`).
- `src/components/lists/lists-mobile-view.tsx` — today's `ListsView` body, moved verbatim (adjusted import paths).
- `src/components/lists/lists-empty-state.tsx` — the "No lists yet" card (extracted; shared by both layouts).
- `src/components/lists/lists-error-state.tsx` — the "Lists could not be loaded" card (extracted; shared).
- `src/components/lists/list-rail-row.tsx` (+ `.test.tsx`) — one selectable rail row.
- `src/components/lists/lists-rail.tsx` (+ `.test.tsx`) — the ~340px rail (New List + rows).
- `src/components/lists/lists-large-screen.tsx` — the two-pane layout + selection logic.

**Modify:**
- `src/components/lists/list-card.tsx` — import `kindMeta` from the shared module.
- `src/components/lists/list-detail-view.tsx` (+ `.test.tsx`) — add `showBackButton?: boolean` (default `true`).
- `src/components/lists-view.tsx` — becomes the `useIsLargeScreen()` router.
- `src/components/lists-view.test.tsx` — add `useIsLargeScreen` to the hoisted mock; add a large-screen integration `describe`.

No barrel file exists under `src/components/lists/` (imports are direct paths), so nothing to update there.

---

## Task 1: Extract shared list-kind metadata

Pure refactor so the rail row and the card share one source of truth. Covered by existing `list-card`/`lists-view` tests.

**Files:**
- Create: `src/components/lists/list-kind-meta.ts`
- Modify: `src/components/lists/list-card.tsx`

- [ ] **Step 1: Create the shared module**

`src/components/lists/list-kind-meta.ts`:

```ts
import { ClipboardList, ListTodo, ShoppingCart } from "lucide-react";
import type { ListKind } from "@/lib/types";

/** Icon + label + badge styling per list kind. Shared by list cards and rail rows. */
export const kindMeta: Record<
  ListKind,
  { label: string; icon: typeof ShoppingCart; className: string }
> = {
  grocery: {
    label: "Grocery",
    icon: ShoppingCart,
    className: "bg-emerald-100 text-emerald-700",
  },
  "to-do": {
    label: "To-do",
    icon: ListTodo,
    className: "bg-sky-100 text-sky-700",
  },
  general: {
    label: "General",
    icon: ClipboardList,
    className: "bg-amber-100 text-amber-700",
  },
};
```

- [ ] **Step 2: Point `list-card.tsx` at the shared map**

In `src/components/lists/list-card.tsx`, delete the local `kindMeta` object and its now-unused `lucide-react` icon imports, and add:

```ts
import { kindMeta } from "./list-kind-meta";
```

Also narrow the type import on line 3 to drop the now-unused `ListKind` (Biome `noUnusedImports` flags it otherwise):

```ts
import type { ListSummary } from "@/lib/types";
```

Leave the rest of the component unchanged (`const meta = kindMeta[list.kind];` still works).

- [ ] **Step 3: Run the existing tests to prove no regression**

Run: `npm test -- --run src/components/lists-view.test.tsx`
Expected: PASS (all existing cases, including "gives list cards press feedback").

- [ ] **Step 4: Lint + commit**

```bash
npm run lint
git add src/components/lists/list-kind-meta.ts src/components/lists/list-card.tsx
git commit -m "refactor(lists): extract shared list-kind metadata"
```

---

## Task 2: `ListDetailView` gains a `showBackButton` prop

The two-pane detail has no screen to go back to, so the "Back to Lists" button must be suppressible. Default `true` keeps mobile/tablet unchanged.

**Files:**
- Modify: `src/components/lists/list-detail-view.tsx`
- Test: `src/components/lists/list-detail-view.test.tsx`

- [ ] **Step 1: Write the failing test**

Add to `src/components/lists/list-detail-view.test.tsx` (match the file's existing render/seed helpers):

```tsx
it("hides the back button when showBackButton is false", async () => {
  seedMockLists([groceryList]); // reuse the file's existing fixture
  render(
    <ListDetailView
      listId={groceryList.id}
      preferences={null}
      preferencesStatus="unavailable"
      onBack={() => {}}
      showBackButton={false}
    />,
  );
  expect(
    await screen.findByRole("heading", { name: groceryList.name }),
  ).toBeInTheDocument();
  expect(
    screen.queryByRole("button", { name: /back to lists/i }),
  ).not.toBeInTheDocument();
});

it("shows the back button by default", async () => {
  seedMockLists([groceryList]);
  render(
    <ListDetailView
      listId={groceryList.id}
      preferences={null}
      preferencesStatus="unavailable"
      onBack={() => {}}
    />,
  );
  expect(
    await screen.findByRole("button", { name: /back to lists/i }),
  ).toBeInTheDocument();
});
```

> `list-detail-view.test.tsx` already defines a `groceryList` fixture (`LIST_ID` = `00000000-0000-4000-8000-000000000101`) and `setupMswServer()` — reuse them. The file currently imports only `renderWithUser` from `@/test/test-utils`; add `render` to that import (both are exported from there) before using it in the two tests above, or switch both to `renderWithUser(...)` and drop the returned `user`.

- [ ] **Step 2: Run to verify it fails**

Run: `npm test -- --run src/components/lists/list-detail-view.test.tsx`
Expected: FAIL — the `showBackButton={false}` case still finds the back button; TS error on the unknown prop.

- [ ] **Step 3: Implement the prop**

In `src/components/lists/list-detail-view.tsx`:

```ts
interface ListDetailViewProps {
  listId: string;
  preferences: ListPreferences | null;
  preferencesStatus: "ready" | "loading" | "error" | "unavailable";
  onBack: () => void;
  /** Hide the "Back to Lists" button (two-pane large-screen has no screen to back out of). */
  showBackButton?: boolean;
}

export function ListDetailView({
  listId,
  preferences,
  preferencesStatus,
  onBack,
  showBackButton = true,
}: ListDetailViewProps) {
```

Wrap **both** back buttons. Not-found branch:

```tsx
if (!list) {
  return (
    <div className="flex-1 p-4">
      {showBackButton && (
        <Button type="button" variant="ghost" onClick={onBack}>
          <ArrowLeft className="h-4 w-4" />
          Back to Lists
        </Button>
      )}
      <p className="mt-6 text-sm text-muted-foreground">
        This list could not be loaded.
      </p>
    </div>
  );
}
```

Normal branch (the block currently at the top of `mx-auto max-w-2xl space-y-4`):

```tsx
{showBackButton && (
  <Button type="button" variant="ghost" onClick={onBack} className="px-0">
    <ArrowLeft className="h-4 w-4" />
    Back to Lists
  </Button>
)}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npm test -- --run src/components/lists/list-detail-view.test.tsx`
Expected: PASS.

- [ ] **Step 5: Lint + commit**

```bash
npm run lint
git add src/components/lists/list-detail-view.tsx src/components/lists/list-detail-view.test.tsx
git commit -m "feat(lists): add showBackButton prop to ListDetailView"
```

---

## Task 3: Move today's body into `ListsMobileView`

Preserve the below-`lg` experience byte-for-byte by relocating the current `ListsView` body into its own component. `ListsView` temporarily just renders it (no behavior change yet).

**Files:**
- Create: `src/components/lists/lists-mobile-view.tsx`
- Modify: `src/components/lists-view.tsx`

- [ ] **Step 1: Create `ListsMobileView`**

Copy the **entire current body** of `ListsView` into `src/components/lists/lists-mobile-view.tsx` as `export function ListsMobileView()`. Adjust import paths for the new location (one directory deeper):

```ts
import { Plus } from "lucide-react";
import { useEffect, useState } from "react";
import { useListPreferences, useLists } from "@/api";
import {
  FloatingActionButton,
  MOBILE_FAB_SCROLL_PADDING,
  OfflineUnavailable,
  ScreenTransition,
} from "@/components/shared";
import { useBackHandler, useIsMobile } from "@/hooks";
import { useAppStore } from "@/stores";
import { Button } from "../ui/button";
import { ListCard } from "./list-card";
import { ListCreateSheet } from "./list-create-sheet";
import { ListDetailView } from "./list-detail-view";
// ...rest of the body unchanged...
```

(Only the relative imports change: `./lists/*` → `./*`, `./ui/button` → `../ui/button`. `@/`-aliased imports stay.)

- [ ] **Step 2: Reduce `ListsView` to a passthrough**

`src/components/lists-view.tsx`:

```tsx
import { ListsMobileView } from "./lists/lists-mobile-view";

export function ListsView() {
  return <ListsMobileView />;
}
```

- [ ] **Step 3: Run the existing suite unchanged**

Run: `npm test -- --run src/components/lists-view.test.tsx`
Expected: PASS — identical DOM, all cases green (empty state, error, create, detail, slide transition, back handler).

- [ ] **Step 4: Lint + commit**

```bash
npm run lint
git add src/components/lists-view.tsx src/components/lists/lists-mobile-view.tsx
git commit -m "refactor(lists): extract ListsMobileView from ListsView"
```

---

## Task 4: `ListRailRow` — a selectable rail row

**Files:**
- Create: `src/components/lists/list-rail-row.tsx`
- Test: `src/components/lists/list-rail-row.test.tsx`

- [ ] **Step 1: Write the failing test**

`src/components/lists/list-rail-row.test.tsx`:

```tsx
import { render, screen } from "@/test/test-utils";
import type { ListSummary } from "@/lib/types";
import { ListRailRow } from "./list-rail-row";

const summary: ListSummary = {
  id: "l1",
  name: "Groceries",
  kind: "grocery",
  totalItems: 6,
  completedItems: 2,
};

it("exposes a selected state with an accessible label including remaining items", () => {
  render(<ListRailRow list={summary} selected onSelect={() => {}} />);
  const row = screen.getByRole("button", {
    name: /groceries, selected, 4 items remaining/i,
  });
  expect(row).toHaveAttribute("aria-current", "true");
});

it("is not current and calls onSelect when activated", async () => {
  const onSelect = vi.fn();
  const { user } = renderWithUser(
    <ListRailRow list={summary} selected={false} onSelect={onSelect} />,
  );
  const row = screen.getByRole("button", { name: /groceries/i });
  expect(row).not.toHaveAttribute("aria-current");
  await user.click(row);
  expect(onSelect).toHaveBeenCalledTimes(1);
});

it("keeps a >=44px touch target", () => {
  render(<ListRailRow list={summary} selected={false} onSelect={() => {}} />);
  expect(screen.getByRole("button", { name: /groceries/i }).className).toContain(
    "min-h-[44px]",
  );
});
```

> Add `renderWithUser` to the import from `@/test/test-utils` (it is the helper the other list tests use).

- [ ] **Step 2: Run to verify it fails**

Run: `npm test -- --run src/components/lists/list-rail-row.test.tsx`
Expected: FAIL with "Cannot find module './list-rail-row'".

- [ ] **Step 3: Implement `ListRailRow`**

`src/components/lists/list-rail-row.tsx`:

```tsx
import { usePressable } from "@/hooks";
import type { ListSummary } from "@/lib/types";
import { cn } from "@/lib/utils";
import { kindMeta } from "./list-kind-meta";

interface ListRailRowProps {
  list: ListSummary;
  selected: boolean;
  onSelect: () => void;
}

export function ListRailRow({ list, selected, onSelect }: ListRailRowProps) {
  const pressable = usePressable();
  const meta = kindMeta[list.kind];
  const Icon = meta.icon;
  const remaining = list.totalItems - list.completedItems;
  const remainingLabel =
    list.totalItems === 0
      ? "Ready to fill"
      : `${remaining} ${remaining === 1 ? "item" : "items"} left`;
  const ariaLabel =
    list.totalItems === 0
      ? `${list.name}${selected ? ", selected" : ""}, no items yet`
      : `${list.name}${selected ? ", selected" : ""}, ${remaining} ${
          remaining === 1 ? "item" : "items"
        } remaining`;

  return (
    <button
      type="button"
      onClick={onSelect}
      onPointerDown={pressable.onPointerDown}
      aria-current={selected ? "true" : undefined}
      aria-label={ariaLabel}
      className={cn(
        "flex min-h-[44px] w-full items-center gap-3 rounded-lg border px-3 py-2.5 text-left transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring",
        selected
          ? "border-primary bg-primary/10"
          : "border-transparent hover:bg-muted",
        pressable.className,
      )}
    >
      <span
        className={cn(
          "flex h-9 w-9 shrink-0 items-center justify-center rounded-lg",
          meta.className,
        )}
      >
        <Icon className="h-5 w-5" />
      </span>
      <span className="min-w-0 flex-1">
        <span className="block truncate text-[15px] font-semibold leading-5 text-foreground">
          {list.name}
        </span>
        <span className="block text-xs leading-4 text-muted-foreground">
          {remainingLabel}
        </span>
      </span>
    </button>
  );
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npm test -- --run src/components/lists/list-rail-row.test.tsx`
Expected: PASS (3 tests).

- [ ] **Step 5: Lint + commit**

```bash
npm run lint
git add src/components/lists/list-rail-row.tsx src/components/lists/list-rail-row.test.tsx
git commit -m "feat(lists): add ListRailRow with selected state"
```

---

## Task 5: `ListsRail` — the rail container

**Files:**
- Create: `src/components/lists/lists-rail.tsx`
- Test: `src/components/lists/lists-rail.test.tsx`

- [ ] **Step 1: Write the failing test**

`src/components/lists/lists-rail.test.tsx`:

```tsx
import { render, renderWithUser, screen } from "@/test/test-utils";
import type { ListSummary } from "@/lib/types";
import { ListsRail } from "./lists-rail";

const summaries: ListSummary[] = [
  { id: "a", name: "Groceries", kind: "grocery", totalItems: 3, completedItems: 1 },
  { id: "b", name: "Chores", kind: "to-do", totalItems: 2, completedItems: 0 },
];

it("renders New List plus one row per list and marks the selected row current", () => {
  render(
    <ListsRail
      summaries={summaries}
      selectedListId="b"
      onSelect={() => {}}
      onCreate={() => {}}
    />,
  );
  expect(screen.getByRole("button", { name: /new list/i })).toBeInTheDocument();
  expect(screen.getByRole("button", { name: /chores, selected/i })).toHaveAttribute(
    "aria-current",
    "true",
  );
  expect(
    screen.getByRole("button", { name: /^Groceries,/i }),
  ).not.toHaveAttribute("aria-current");
});

it("calls onSelect with the row id and onCreate for New List", async () => {
  const onSelect = vi.fn();
  const onCreate = vi.fn();
  const { user } = renderWithUser(
    <ListsRail
      summaries={summaries}
      selectedListId="a"
      onSelect={onSelect}
      onCreate={onCreate}
    />,
  );
  await user.click(screen.getByRole("button", { name: /^Chores,/i }));
  expect(onSelect).toHaveBeenCalledWith("b");
  await user.click(screen.getByRole("button", { name: /new list/i }));
  expect(onCreate).toHaveBeenCalledTimes(1);
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npm test -- --run src/components/lists/lists-rail.test.tsx`
Expected: FAIL with "Cannot find module './lists-rail'".

- [ ] **Step 3: Implement `ListsRail`**

`src/components/lists/lists-rail.tsx`:

```tsx
import { Plus } from "lucide-react";
import type { ListSummary } from "@/lib/types";
import { Button } from "../ui/button";
import { ListRailRow } from "./list-rail-row";

interface ListsRailProps {
  summaries: ListSummary[];
  selectedListId: string | null;
  onSelect: (id: string) => void;
  onCreate: () => void;
}

export function ListsRail({
  summaries,
  selectedListId,
  onSelect,
  onCreate,
}: ListsRailProps) {
  return (
    <nav
      aria-label="Lists"
      className="flex w-[340px] shrink-0 flex-col border-r border-border"
    >
      <div className="flex items-center justify-between gap-3 border-b border-border px-4 py-3">
        <h2 className="text-[17px] font-semibold leading-6 text-foreground">
          My Lists
        </h2>
        <Button type="button" onClick={onCreate} className="min-h-[44px]">
          <Plus className="h-4 w-4" />
          New List
        </Button>
      </div>
      <ul className="flex-1 space-y-1 overflow-y-auto p-2">
        {summaries.map((list) => (
          <li key={list.id}>
            <ListRailRow
              list={list}
              selected={list.id === selectedListId}
              onSelect={() => onSelect(list.id)}
            />
          </li>
        ))}
      </ul>
    </nav>
  );
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npm test -- --run src/components/lists/lists-rail.test.tsx`
Expected: PASS (2 tests).

- [ ] **Step 5: Lint + commit**

```bash
npm run lint
git add src/components/lists/lists-rail.tsx src/components/lists/lists-rail.test.tsx
git commit -m "feat(lists): add ListsRail container"
```

---

## Task 6: Extract shared empty + error state cards

Both layouts render an identical "No lists yet" and "Lists could not be loaded" card. Extract them, then wire `ListsMobileView` to use them (identical output; existing tests prove it).

**Files:**
- Create: `src/components/lists/lists-empty-state.tsx`, `src/components/lists/lists-error-state.tsx`
- Modify: `src/components/lists/lists-mobile-view.tsx`

- [ ] **Step 1: Create `ListsEmptyState`**

`src/components/lists/lists-empty-state.tsx` (markup copied verbatim from the current "No lists yet" block):

```tsx
import { Plus } from "lucide-react";
import { Button } from "../ui/button";

export function ListsEmptyState({ onCreate }: { onCreate: () => void }) {
  return (
    <div className="rounded-lg border border-dashed border-border bg-card p-6 text-center shadow-sm">
      <h3 className="text-lg font-semibold text-foreground">No lists yet</h3>
      <p className="mx-auto mt-2 max-w-sm text-sm leading-5 text-muted-foreground">
        Create the first shared family list for groceries, errands, or anything
        else worth keeping together.
      </p>
      <Button type="button" className="mt-4" onClick={onCreate}>
        <Plus className="h-4 w-4" />
        Create first list
      </Button>
    </div>
  );
}
```

- [ ] **Step 2: Create `ListsErrorState`**

`src/components/lists/lists-error-state.tsx` (verbatim from the current error block):

```tsx
import { Button } from "../ui/button";

export function ListsErrorState({ onRetry }: { onRetry: () => void }) {
  return (
    <div className="rounded-lg border border-border bg-card p-6 shadow-sm">
      <h3 className="text-lg font-semibold text-foreground">
        Lists could not be loaded
      </h3>
      <p className="mt-2 text-sm leading-5 text-muted-foreground">
        Check your connection and try again.
      </p>
      <Button
        type="button"
        variant="outline"
        className="mt-4"
        onClick={onRetry}
      >
        Try again
      </Button>
    </div>
  );
}
```

- [ ] **Step 3: Wire `ListsMobileView` to the extracted cards**

In `src/components/lists/lists-mobile-view.tsx`, replace the inline "No lists yet" block with `<ListsEmptyState onCreate={() => setCreateOpen(true)} />` and the inline error card with `<ListsErrorState onRetry={() => lists.refetch()} />`, and import both. Leave the surrounding header/`max-w-2xl` wrappers untouched.

- [ ] **Step 4: Prove the mobile DOM is unchanged**

Run: `npm test -- --run src/components/lists-view.test.tsx`
Expected: PASS — "shows the no-lists empty state", "does not treat a list API failure as an empty family list", etc. all still green.

- [ ] **Step 5: Lint + commit**

```bash
npm run lint
git add src/components/lists/lists-empty-state.tsx src/components/lists/lists-error-state.tsx src/components/lists/lists-mobile-view.tsx
git commit -m "refactor(lists): extract shared empty and error state cards"
```

---

## Task 7: `ListsLargeScreen` — the two-pane layout

The heart of the story: rail + detail, first-list auto-select, swap-not-navigate, no back handler, intent + create wired, and the delete/sync selection fallback.

**Files:**
- Create: `src/components/lists/lists-large-screen.tsx`
- Test: covered by the integration `describe` added in Task 8 (this task ships the component; Task 8 wires the router and tests behavior end-to-end).

- [ ] **Step 1: Implement `ListsLargeScreen`**

`src/components/lists/lists-large-screen.tsx`:

```tsx
import { useEffect, useState, type ReactNode } from "react";
import { useListPreferences, useLists } from "@/api";
import { OfflineUnavailable } from "@/components/shared";
import { useAppStore } from "@/stores";
import { ListCreateSheet } from "./list-create-sheet";
import { ListDetailView } from "./list-detail-view";
import { ListsEmptyState } from "./lists-empty-state";
import { ListsErrorState } from "./lists-error-state";
import { ListsRail } from "./lists-rail";

export function ListsLargeScreen() {
  const lists = useLists();
  const preferences = useListPreferences();
  const [selectedListId, setSelectedListId] = useState<string | null>(null);
  const [createOpen, setCreateOpen] = useState(false);

  const summaries = lists.data?.data ?? [];

  // Cross-module handoff (e.g. "open this grocery list" from Meals): consume the
  // intent once list data is available and select it in the pane. Guarded on
  // `lists.data` so a pending intent is preserved (not consumed) until data loads.
  const consumeListDetailIntent = useAppStore((s) => s.consumeListDetailIntent);
  useEffect(() => {
    const data = lists.data?.data;
    if (!data) return;
    const id = consumeListDetailIntent();
    if (id && data.some((s) => s.id === id)) setSelectedListId(id);
  }, [lists.data, consumeListDetailIntent]);

  // Effective selection, derived every render so it is ALWAYS valid, without ever
  // overwriting the user's / create's / intent's chosen id in state:
  //   - nothing selected, or the selected id vanished (delete / cross-device sync)
  //     and no refetch is in flight -> first remaining list (auto-select-first +
  //     delete fallback), or null when none remain.
  //   - a valid selected id -> keep it.
  //   - a just-created / just-selected id not yet in `summaries` while a hub
  //     refetch is in flight -> keep it. `useCreateList` primes the detail cache
  //     (`setQueryData(detail(newId))`) then invalidates the hub, so the pane
  //     renders the new list immediately and it lands in `summaries` post-refetch.
  const effectiveSelectedId =
    selectedListId &&
    (summaries.some((s) => s.id === selectedListId) || lists.isFetching)
      ? selectedListId
      : (summaries[0]?.id ?? null);

  const preferencesStatus = preferences.isLoading
    ? "loading"
    : preferences.isError
      ? "error"
      : preferences.data?.data
        ? "ready"
        : "unavailable";

  const fullArea = (children: ReactNode) => (
    <div className="flex-1 overflow-y-auto p-6">
      <div className="mx-auto max-w-2xl">{children}</div>
    </div>
  );

  if (lists.isLoading) {
    return (
      <div className="flex-1 p-6 text-sm text-muted-foreground">
        Loading lists
      </div>
    );
  }
  if (lists.isError) {
    return fullArea(<ListsErrorState onRetry={() => lists.refetch()} />);
  }
  // Offline + never loaded: paused query has no data and no error.
  if (!lists.data) {
    return <OfflineUnavailable label="lists" />;
  }
  if (summaries.length === 0) {
    return (
      <>
        {fullArea(<ListsEmptyState onCreate={() => setCreateOpen(true)} />)}
        <ListCreateSheet
          open={createOpen}
          onOpenChange={setCreateOpen}
          onCreated={(id) => setSelectedListId(id)}
        />
      </>
    );
  }

  return (
    <>
      <div className="flex min-h-0 flex-1">
        <ListsRail
          summaries={summaries}
          selectedListId={effectiveSelectedId}
          onSelect={setSelectedListId}
          onCreate={() => setCreateOpen(true)}
        />
        <div className="flex min-w-0 flex-1 flex-col">
          {effectiveSelectedId && (
            <ListDetailView
              key={effectiveSelectedId}
              listId={effectiveSelectedId}
              preferences={preferences.data?.data ?? null}
              preferencesStatus={preferencesStatus}
              onBack={() => {}}
              showBackButton={false}
            />
          )}
        </div>
      </div>
      <ListCreateSheet
        open={createOpen}
        onOpenChange={setCreateOpen}
        onCreated={(id) => setSelectedListId(id)}
      />
    </>
  );
}
```

Notes for the implementer:
- **Selection is derived, never reconciled via `setState`.** State holds only what the user / create / intent explicitly chose; `effectiveSelectedId` computes the *valid* selection for rendering. This is deliberate: writing a fallback back into state would clobber a just-created id during the post-create hub refetch (the new id is not in `summaries` until the refetch lands, and `useLists().data` still holds the old summaries meanwhile). Do **not** "simplify" this into a reconcile effect.
- The `lists.isFetching` clause is what protects the just-created / just-selected id across the refetch window. `useCreateList` primes `detail(newId)` then invalidates `hub()` (`api/hooks/use-lists.ts:106-109`), so `ListDetailView` renders the new list from primed cache and it appears in the rail once the hub refetch resolves.
- Because there is no competing reconcile effect, the intent effect is also safe when the lists hub is already cached at mount (the common Meals→Lists path): it consumes the intent on the first commit with data and nothing overwrites it.
- The `key={effectiveSelectedId}` on `ListDetailView` forces a clean remount per list, so per-list local state (open sheets, options) never leaks between lists when the pane swaps.
- No `useBackHandler` here — the back handler must not intercept when both panes are visible (spec §5).
- `onBack` is a no-op because `showBackButton={false}` means it is never invoked.

- [ ] **Step 2: Typecheck**

Run: `npm run build`
Expected: exit 0 (component compiles; not yet routed).

- [ ] **Step 3: Lint + commit**

```bash
npm run lint
git add src/components/lists/lists-large-screen.tsx
git commit -m "feat(lists): add ListsLargeScreen two-pane layout"
```

---

## Task 8: Route `ListsView` by breakpoint + integration tests

**Files:**
- Modify: `src/components/lists-view.tsx`
- Test: `src/components/lists-view.test.tsx`

- [ ] **Step 1: Add `useIsLargeScreen` to the test's hoisted mock**

At the top of `src/components/lists-view.test.tsx`, extend the hoisted viewport and the `@/hooks` mock so the large-screen branch is controllable and defaults to the existing (mobile/tablet) path:

```tsx
const viewport = vi.hoisted(() => ({ isMobile: false, isLargeScreen: false }));

vi.mock("@/hooks", async (importOriginal) => {
  const actual = await importOriginal<typeof import("@/hooks")>();
  return {
    ...actual,
    useIsMobile: () => viewport.isMobile,
    useIsLargeScreen: () => viewport.isLargeScreen,
  };
});
```

In the top-level `beforeEach`, add `viewport.isLargeScreen = false;` next to `viewport.isMobile = false;` so every existing test keeps hitting `ListsMobileView`.

- [ ] **Step 2: Run the existing suite — still green through the router**

Run: `npm test -- --run src/components/lists-view.test.tsx`
Expected: PASS (unchanged — router still resolves to `ListsMobileView`).

- [ ] **Step 3: Write the failing large-screen integration tests**

Append to `src/components/lists-view.test.tsx`:

```tsx
describe("ListsView large screen (two-pane)", () => {
  beforeEach(() => {
    viewport.isLargeScreen = true;
  });

  it("renders rail and detail together and auto-selects the first list", async () => {
    seedMockLists([groceryList, todoList]);
    render(<ListsView />);
    // Rail row for both lists...
    expect(
      await screen.findByRole("button", { name: /Trader Joe's Run,/i }),
    ).toBeInTheDocument();
    expect(
      screen.getByRole("button", { name: /Weekend Reset,/i }),
    ).toBeInTheDocument();
    // ...and the first list's detail already in the pane.
    expect(
      await screen.findByRole("heading", { name: "Trader Joe's Run" }),
    ).toBeInTheDocument();
    // No "Back to Lists" in two-pane.
    expect(
      screen.queryByRole("button", { name: /back to lists/i }),
    ).not.toBeInTheDocument();
  });

  it("swaps only the detail pane when another list is selected (no back handler)", async () => {
    seedMockLists([groceryList, todoList]);
    const { user } = renderWithUser(<ListsView />);
    await screen.findByRole("heading", { name: "Trader Joe's Run" });
    await user.click(screen.getByRole("button", { name: /Weekend Reset,/i }));
    expect(
      await screen.findByRole("heading", { name: "Weekend Reset" }),
    ).toBeInTheDocument();
    // Both rails still present (no navigation happened).
    expect(
      screen.getByRole("button", { name: /Trader Joe's Run,/i }),
    ).toBeInTheDocument();
    // Back handler is NOT registered in two-pane mode.
    expect(useBackStack.getState().stack).toHaveLength(0);
  });

  it("consumes the cross-module list intent by selecting it in the pane", async () => {
    seedMockLists([groceryList, todoList]);
    act(() => {
      useAppStore.getState().openListDetail(todoList.id);
    });
    render(<ListsView />);
    expect(
      await screen.findByRole("heading", { name: "Weekend Reset" }),
    ).toBeInTheDocument();
  });

  it("selects a newly created list in the pane", async () => {
    seedMockLists([groceryList]);
    const { user } = renderWithUser(<ListsView />);
    await screen.findByRole("heading", { name: "Trader Joe's Run" });
    await user.click(screen.getByRole("button", { name: /new list/i }));
    await typeAndWait(user, screen.getByLabelText("List name"), "Target Run");
    await user.click(screen.getByRole("button", { name: "Create list" }));
    expect(
      await screen.findByRole("heading", { name: "Target Run" }),
    ).toBeInTheDocument();
  });

  it("falls back to the first remaining list when the selected list disappears", async () => {
    seedMockLists([groceryList, todoList]);
    const { user } = renderWithUser(<ListsView />);
    await user.click(screen.getByRole("button", { name: /Weekend Reset,/i }));
    await screen.findByRole("heading", { name: "Weekend Reset" });
    // Simulate the selected list being removed elsewhere (delete / cross-device
    // sync): drop it from the MSW store, then force the hub query to re-resolve.
    // getTestQueryClient() is the same client the render helpers wrap the tree in.
    act(() => {
      seedMockLists([groceryList]);
    });
    await act(async () => {
      await getTestQueryClient().refetchQueries({ queryKey: listsKeys.hub() });
    });
    expect(
      await screen.findByRole("heading", { name: "Trader Joe's Run" }),
    ).toBeInTheDocument();
  });

  it("shows the full-area no-lists empty state with no rail", async () => {
    seedMockLists([]);
    render(<ListsView />);
    expect(await screen.findByText("No lists yet")).toBeInTheDocument();
    expect(screen.queryByRole("navigation", { name: "Lists" })).not.toBeInTheDocument();
  });
});
```

> Add these imports to the test file: `useAppStore` from `@/stores` (alongside the existing `useBackStack`), `listsKeys` from `@/api`, and `getTestQueryClient` from `@/test/test-utils`. `getTestQueryClient()` returns the same client `render`/`renderWithUser` wrap the tree with (see `test-utils.tsx` `AllProviders`), so `refetchQueries({ queryKey: listsKeys.hub() })` re-resolves the summaries the component reads — `seedMockLists` alone only mutates the MSW store and does not trigger a refetch, which is why the earlier draft's "click New List" trigger did not work.

- [ ] **Step 4: Run to verify the new tests fail**

Run: `npm test -- --run src/components/lists-view.test.tsx`
Expected: FAIL — router still hardcodes `ListsMobileView`, so no rail renders.

- [ ] **Step 5: Implement the router**

`src/components/lists-view.tsx`:

```tsx
import { useIsLargeScreen } from "@/hooks";
import { ListsLargeScreen } from "./lists/lists-large-screen";
import { ListsMobileView } from "./lists/lists-mobile-view";

export function ListsView() {
  const isLargeScreen = useIsLargeScreen();
  return isLargeScreen ? <ListsLargeScreen /> : <ListsMobileView />;
}
```

- [ ] **Step 6: Run to verify all pass**

Run: `npm test -- --run src/components/lists-view.test.tsx`
Expected: PASS (existing below-`lg` cases + new two-pane cases).

- [ ] **Step 7: Lint + commit**

```bash
npm run lint
git add src/components/lists-view.tsx src/components/lists-view.test.tsx
git commit -m "feat(lists): route Lists to two-pane layout at lg+"
```

---

## Task 9: Full verification + screenshot matrix

Mirror the Calendar story's evidence discipline. Requires a real backend for screenshots (see root `MEMORY.md` → E2E against real BE).

- [ ] **Step 1: Whole-suite gate**

```bash
npm run build     # tsc + vite, exit 0
npm run lint      # Biome, exit 0
npm test -- --run # full unit/integration suite green
```

- [ ] **Step 2: Capture the spec's screenshot matrix**

Start the real backend, then capture at the spec's review sizes and cases (spec §6):
- 1024px and 1440px, each with: **several lists**, **one list**, **empty list selected** (list with no items), and the **no-lists empty state**.
- Confirm the rail is ~340px, the selected row shows the accent state, and item content caps at reading width in the pane.

```bash
# from frontend/
docker compose -f docker-compose.e2e.yml up -d --wait   # real BE on :8080
npm run dev                                              # or the project's screenshot harness
# ...capture matrix...
docker compose -f docker-compose.e2e.yml down
```

- [ ] **Step 3: Prove below-1024 is unchanged**

Capture Lists at **900px** (tablet) and **375px** (mobile) and diff against `origin/main` — the drill-in index → detail, slide transition, and back button must be visually identical.

- [ ] **Step 4: Iterate**

If any screenshot violates the spec (rail width, selected accent, touch targets ≥44px, pane reading-width cap, or a mobile regression), fix and re-capture before proceeding. Commit any fixes with `fix(lists): ...`.

> Two known behaviors to confirm as **acceptable**, not regressions: (1) the rail row shows remaining-items ("N items left") rather than the card's "X of Y done" — confirm this reads well at rail density in the screenshots, or enrich `ListRailRow` if the reviewer wants the fuller count; (2) crossing the 1024px boundary mid-session resets the selected list, because `ListsMobileView` and `ListsLargeScreen` hold separate selection state — expected on the target devices and not a spec requirement.

---

## Task 10: PR

- [ ] **Step 1: Open the PR from `feat/large-screen-lists`**

Body must include, per the root execution-issue contract:
- **Story / Spec / Plan** links (this file).
- **Verification:** build/lint/test results + the screenshot matrix summary + the below-1024 unchanged evidence.
- **Execution contract checklist** mapping each spec AC to code + tests:
  - Two-pane at 1024px+, no full-screen swap on select → `ListsView` router + `ListsLargeScreen`; test "renders rail and detail together", "swaps only the detail pane".
  - First-list auto-select + create/delete fallbacks → `effectiveSelectedId` pure derivation; tests "auto-selects the first list", "selects a newly created list", "falls back to the first remaining list".
  - Cross-module handoff selects in pane → intent effect; test "consumes the cross-module list intent".
  - Existing item flows unchanged in the pane → reused `ListDetailView`; existing detail tests.
  - Below 1024px + mobile unchanged → `ListsMobileView` moved verbatim; full existing `lists-view.test.tsx` suite + 900/375px screenshots.
  - Screenshot review at 1024/1440 across the four cases → Task 9 matrix.

- [ ] **Step 2: Merge, then update root docs (orchestrator, post-merge)**

After merge, in the **root** repo set `docs/product/backlog/large-screen-ux/large-screen-lists.md` `status: done`, bump `updated:`, record the PR under `prs:`, and move its `roadmap.md` line from "Next up" into "Shipped" (opened → merged) — exactly as the Calendar story was closed out.

---

## Self-Review (completed during planning)

**1. Spec coverage:**
- §3.1 Layout (rail ~340px, right pane, selected accent) → Tasks 4, 5, 7 (`ListsRail` `w-[340px]`, `ListRailRow` selected border/bg).
- §3.2 Behavior — first-list auto-select, swap-only, intent-selects, create-selects, delete-fallback, empty states, below-1024 unchanged → Task 7 (`effectiveSelectedId` pure derivation + intent effect, `ListCreateSheet` wiring, `ListsEmptyState`), Tasks 3/8 (mobile path preserved). All covered by Task 8 tests.
- §5 Accessibility — rail as a nav list, selected announced, activation-not-focus selection, back handler doesn't intercept, ≥44px targets → `ListsRail` `<nav aria-label="Lists">` + `<ul>`, `ListRailRow` `aria-current` + composed `aria-label` + `min-h-[44px]`, no `useBackHandler` in `ListsLargeScreen` (asserted by "stack has length 0").
- §6 AC screenshot matrix → Task 9.
- §7 Out of Scope (no delete/reorder/sharing, no add-item/category flow changes, no mobile changes, no BE) → respected; `ListDetailView` reused unchanged apart from the additive `showBackButton` prop.

**2. Placeholder scan:** No TBD/"handle edge cases"/"similar to Task N" — every code step carries full code. The former soft spot (Task 8's "disappears" refetch trigger) now has a concrete, runnable solution using `getTestQueryClient().refetchQueries({ queryKey: listsKeys.hub() })`.

**3. Type consistency:** `ListSummary` fields (`id/name/kind/totalItems/completedItems`) used consistently across `ListRailRow`, `ListsRail`, `ListsLargeScreen`. `ListDetailView` prop names (`listId`, `preferences`, `preferencesStatus`, `onBack`, `showBackButton`) match Task 2's signature. `kindMeta` shape identical across Tasks 1 and 4. Router branch names (`ListsMobileView`, `ListsLargeScreen`) consistent Tasks 3/7/8.

**Key risk flagged:** there is **no whole-list delete** in the FE, so the delete-fallback AC is satisfied by pure selection derivation against the live summaries (Task 7), and its test simulates the array losing the selected id (via a forced hub refetch) rather than clicking a delete control.
