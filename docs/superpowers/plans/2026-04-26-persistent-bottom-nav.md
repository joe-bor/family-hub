# Persistent Bottom Navigation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a first-class mobile bottom nav to the frontend shell so Home and the five modules are always one tap away on phones.

**Architecture:** Keep the existing Zustand-driven shell. Add a new mobile-only bottom rail component wired to `activeModule`, render it in the authenticated mobile shell, remove redundant Home buttons, and lift the calendar FAB so it clears the new rail. Do not introduce routing or desktop shell changes.

**Tech Stack:** React 19, TypeScript, Zustand, Vitest, Testing Library, Playwright, Tailwind CSS v4

**Spec:** `docs/superpowers/specs/2026-04-26-persistent-bottom-nav-design.md`

---

This is an FE-only plan. Execute it inside the `frontend/` repo on a fresh feature branch such as `feat/mobile-bottom-nav`.

## File Structure

| File | Action | Purpose |
|------|--------|---------|
| `src/components/shared/mobile-bottom-nav.tsx` | Create | Mobile-only tab rail for Home + modules |
| `src/components/shared/mobile-bottom-nav.test.tsx` | Create | Unit coverage for tab inventory and store switching |
| `src/components/shared/index.ts` | Modify | Export `MobileBottomNav` from the shared barrel |
| `src/App.tsx` | Modify | Render bottom nav in the authenticated mobile shell; add `min-h-0` shell constraints |
| `src/App.shell.test.tsx` | Create | Focused shell smoke tests for mobile vs desktop header/nav behavior |
| `src/components/shared/app-header.tsx` | Modify | Remove the redundant mobile Home button |
| `src/components/calendar/components/mobile-toolbar.tsx` | Modify | Remove the redundant Home button and prop |
| `src/components/calendar/components/mobile-toolbar.test.tsx` | Modify | Update toolbar expectations to match the new chrome |
| `src/components/calendar/calendar-module.tsx` | Modify | Stop passing `onGoHome`; pass mobile-nav FAB offset flag |
| `src/components/calendar/components/add-event-button.tsx` | Modify | Add a mobile-nav-aware bottom offset |
| `src/components/calendar/components/add-event-button.test.tsx` | Create | Lock the mobile vs desktop FAB offsets |
| `e2e/mobile-bottom-nav.spec.ts` | Create | Mobile shell E2E coverage for the new nav |

---

### Task 1: Add `MobileBottomNav`

**Files:**
- Create: `frontend/src/components/shared/mobile-bottom-nav.test.tsx`
- Create: `frontend/src/components/shared/mobile-bottom-nav.tsx`

- [ ] **Step 1: Write the failing unit test**

Create `frontend/src/components/shared/mobile-bottom-nav.test.tsx`:

```tsx
import { beforeEach, describe, expect, it } from "vitest";
import { render, renderWithUser, screen } from "@/test/test-utils";
import { useAppStore } from "@/stores";
import { MobileBottomNav } from "./mobile-bottom-nav";

describe("MobileBottomNav", () => {
  beforeEach(() => {
    useAppStore.setState({ activeModule: null, isSidebarOpen: false });
  });

  it("renders all six tabs and marks Home active when activeModule is null", () => {
    render(<MobileBottomNav />);

    expect(
      screen.getByRole("navigation", { name: /primary/i }),
    ).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /^home$/i })).toHaveAttribute(
      "aria-current",
      "page",
    );
    expect(screen.getByRole("button", { name: /^calendar$/i })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /^lists$/i })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /^chores$/i })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /^meals$/i })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /^photos$/i })).toBeInTheDocument();
  });

  it("switches modules through the app store", async () => {
    const { user } = renderWithUser(<MobileBottomNav />);

    await user.click(screen.getByRole("button", { name: /^photos$/i }));
    expect(useAppStore.getState().activeModule).toBe("photos");

    await user.click(screen.getByRole("button", { name: /^home$/i }));
    expect(useAppStore.getState().activeModule).toBeNull();
  });
});
```

- [ ] **Step 2: Run the test and confirm it fails**

Run:

```bash
cd frontend
npm run test -- --run src/components/shared/mobile-bottom-nav.test.tsx
```

Expected: FAIL because `mobile-bottom-nav.tsx` does not exist yet.

- [ ] **Step 3: Implement `MobileBottomNav`**

Create `frontend/src/components/shared/mobile-bottom-nav.tsx`:

```tsx
import {
  Calendar,
  CheckSquare,
  Home,
  ImageIcon,
  ListTodo,
  UtensilsCrossed,
  type LucideIcon,
} from "lucide-react";
import { cn } from "@/lib/utils";
import { type ModuleType, useAppStore } from "@/stores";

const tabs: Array<{
  id: ModuleType | null;
  label: string;
  icon: LucideIcon;
}> = [
  { id: null, label: "Home", icon: Home },
  { id: "calendar", label: "Calendar", icon: Calendar },
  { id: "lists", label: "Lists", icon: ListTodo },
  { id: "chores", label: "Chores", icon: CheckSquare },
  { id: "meals", label: "Meals", icon: UtensilsCrossed },
  { id: "photos", label: "Photos", icon: ImageIcon },
];

export function MobileBottomNav() {
  const activeModule = useAppStore((state) => state.activeModule);
  const setActiveModule = useAppStore((state) => state.setActiveModule);

  return (
    <nav
      aria-label="Primary"
      className="sm:hidden shrink-0 border-t border-border bg-card/95 backdrop-blur supports-[backdrop-filter]:bg-card/85 z-30"
    >
      <div
        className="grid grid-cols-6 gap-1 px-2 pt-2"
        style={{ paddingBottom: "max(0.5rem, env(safe-area-inset-bottom))" }}
      >
        {tabs.map((tab) => {
          const Icon = tab.icon;
          const isActive = activeModule === tab.id;

          return (
            <button
              key={tab.label}
              type="button"
              aria-current={isActive ? "page" : undefined}
              onClick={() => setActiveModule(tab.id)}
              className={cn(
                "flex min-w-0 flex-col items-center gap-1 rounded-xl px-1 py-2 text-[10px] font-medium transition-colors",
                isActive
                  ? "bg-primary text-primary-foreground"
                  : "text-muted-foreground hover:bg-muted hover:text-foreground",
              )}
            >
              <Icon className="h-5 w-5 shrink-0" />
              <span className="truncate">{tab.label}</span>
            </button>
          );
        })}
      </div>
    </nav>
  );
}
```

- [ ] **Step 4: Re-run the unit test**

Run:

```bash
cd frontend
npm run test -- --run src/components/shared/mobile-bottom-nav.test.tsx
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/shared/mobile-bottom-nav.tsx src/components/shared/mobile-bottom-nav.test.tsx
git commit -m "feat(shell): add mobile bottom nav component"
```

---

### Task 2: Wire the nav into the app shell

**Files:**
- Modify: `frontend/src/components/shared/index.ts`
- Modify: `frontend/src/App.tsx`
- Create: `frontend/src/App.shell.test.tsx`

- [ ] **Step 1: Write the shell smoke test**

Create `frontend/src/App.shell.test.tsx`:

```tsx
import { beforeEach, describe, expect, it, vi } from "vitest";
import FamilyHub from "./App";
import { render, screen, seedAuthStore, seedFamilyStore } from "./test/test-utils";
import { useAppStore } from "./stores";

let mockIsMobile = true;

vi.mock("@/api", async (importOriginal) => {
  const actual = await importOriginal<typeof import("@/api")>();
  return {
    ...actual,
    useSetupComplete: () => true,
  };
});

vi.mock("@/hooks", async (importOriginal) => {
  const actual = await importOriginal<typeof import("@/hooks")>();
  return {
    ...actual,
    useGoogleAuthReturn: () => {},
    useIsMobile: () => mockIsMobile,
  };
});

vi.mock("@/components/calendar", () => ({
  CalendarModule: () => <div>Calendar Surface</div>,
}));

vi.mock("@/components/home", () => ({
  HomeDashboard: () => <div>Home Surface</div>,
}));

describe("FamilyHub shell", () => {
  beforeEach(() => {
    seedAuthStore({ isAuthenticated: true });
    seedFamilyStore({
      name: "Test Family",
      members: [{ id: "member-1", name: "Alice", color: "coral" }],
    });
    mockIsMobile = true;
  });

  it("renders AppHeader and bottom nav on mobile Home", async () => {
    useAppStore.setState({ activeModule: null, isSidebarOpen: false });
    render(<FamilyHub />);

    expect(await screen.findByText("Home Surface")).toBeInTheDocument();
    expect(screen.getByRole("navigation", { name: /primary/i })).toBeInTheDocument();
    expect(screen.getByText("Test Family")).toBeInTheDocument();
  });

  it("suppresses AppHeader on mobile Calendar but keeps the bottom nav", async () => {
    useAppStore.setState({ activeModule: "calendar", isSidebarOpen: false });
    render(<FamilyHub />);

    expect(await screen.findByText("Calendar Surface")).toBeInTheDocument();
    expect(screen.getByRole("navigation", { name: /primary/i })).toBeInTheDocument();
    expect(screen.queryByText("Test Family")).not.toBeInTheDocument();
  });

  it("does not render the mobile bottom nav on desktop", async () => {
    mockIsMobile = false;
    useAppStore.setState({ activeModule: "calendar", isSidebarOpen: false });
    render(<FamilyHub />);

    expect(await screen.findByText("Calendar Surface")).toBeInTheDocument();
    expect(
      screen.queryByRole("navigation", { name: /primary/i }),
    ).not.toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run the shell test and confirm it fails**

Run:

```bash
cd frontend
npm run test -- --run src/App.shell.test.tsx
```

Expected: FAIL because `MobileBottomNav` is not exported or rendered yet.

- [ ] **Step 3: Export and render `MobileBottomNav`**

Update `frontend/src/components/shared/index.ts`:

```ts
export { AppHeader } from "./app-header";
export { MobileBottomNav } from "./mobile-bottom-nav";
export { NavigationTabs, type TabType } from "./navigation-tabs";
export { SidebarMenu } from "./sidebar-menu";
export { ThemeProvider } from "./theme-provider";
```

Update `frontend/src/App.tsx`:

```tsx
import { AppHeader, MobileBottomNav, NavigationTabs, SidebarMenu } from "@/components/shared";
```

Replace the authenticated shell with:

```tsx
return (
  <>
    <div className="h-screen flex flex-col bg-background">
      {!(isMobile && activeModule === "calendar") && <AppHeader />}

      <div className="flex-1 flex min-h-0 overflow-hidden">
        <NavigationTabs />
        <main className="flex-1 min-h-0 flex flex-col overflow-hidden">
          {renderModule(activeModule)}
        </main>
      </div>

      {isMobile && isAuthenticated && setupComplete && <MobileBottomNav />}
      <SidebarMenu />
    </div>
    <Toaster />
  </>
);
```

- [ ] **Step 4: Re-run the shell tests**

Run:

```bash
cd frontend
npm run test -- --run src/App.shell.test.tsx src/components/shared/mobile-bottom-nav.test.tsx
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/shared/index.ts src/App.tsx src/App.shell.test.tsx
git commit -m "feat(shell): render mobile bottom nav in the app shell"
```

---

### Task 3: Remove redundant Home buttons from mobile chrome

**Files:**
- Modify: `frontend/src/components/shared/app-header.tsx`
- Modify: `frontend/src/components/calendar/components/mobile-toolbar.tsx`
- Modify: `frontend/src/components/calendar/components/mobile-toolbar.test.tsx`
- Modify: `frontend/src/components/calendar/calendar-module.tsx`
- Modify: `frontend/src/App.shell.test.tsx`

- [ ] **Step 1: Add failing expectations for the new chrome rules**

Update `frontend/src/components/calendar/components/mobile-toolbar.test.tsx` to remove `onGoHome` from every render call and add:

```tsx
it("does not render a home button once bottom nav exists", () => {
  render(<MobileToolbar members={mockMembers} onOpenSidebar={vi.fn()} />);
  expect(
    screen.queryByRole("button", { name: /home/i }),
  ).not.toBeInTheDocument();
});
```

Update `frontend/src/App.shell.test.tsx` by adding:

```tsx
it("renders only one Home button on mobile module screens", async () => {
  useAppStore.setState({ activeModule: "lists", isSidebarOpen: false });
  render(<FamilyHub />);

  expect(await screen.findByRole("button", { name: /^lists$/i })).toBeInTheDocument();
  expect(screen.getAllByRole("button", { name: /^home$/i })).toHaveLength(1);
});
```

- [ ] **Step 2: Run the targeted tests and confirm they fail**

Run:

```bash
cd frontend
npm run test -- --run src/App.shell.test.tsx src/components/calendar/components/mobile-toolbar.test.tsx
```

Expected: FAIL because `AppHeader` and `MobileToolbar` still expose extra Home buttons.

- [ ] **Step 3: Remove the extra Home affordances**

In `frontend/src/components/shared/app-header.tsx`:

- remove `Home` from the Lucide import
- remove `activeModule`, `setActiveModule`, and `showHomeButton`
- delete the mobile Home `Button`

The left side of the header should become:

```tsx
<div className="flex items-center gap-4">
  <Button
    variant="ghost"
    size="icon"
    aria-label="Menu"
    className="text-muted-foreground hover:text-foreground"
    onClick={openSidebar}
  >
    <Menu className="h-6 w-6" />
  </Button>
  <div>
    <h1 className="text-xl font-bold text-foreground">
      {familyName || "Family Hub"}
    </h1>
    {!isMobile && (
      <div className="flex items-center gap-2 text-sm text-muted-foreground">
        <span>{formatDate(currentDate)}</span>
        <span>•</span>
        <span>{formatTime(new Date())}</span>
      </div>
    )}
  </div>
</div>
```

In `frontend/src/components/calendar/components/mobile-toolbar.tsx`:

- remove `Home` from the Lucide import
- remove the `onGoHome` prop from `MobileToolbarProps`
- remove the Home button from the header row

The toolbar prop shape becomes:

```tsx
interface MobileToolbarProps {
  members: FamilyMember[];
  onOpenSidebar: () => void;
}
```

In `frontend/src/components/calendar/calendar-module.tsx`:

- remove `setActiveModule` from the app-store selector block
- stop passing `onGoHome`

The mobile toolbar call becomes:

```tsx
<MobileToolbar members={members} onOpenSidebar={openSidebar} />
```

- [ ] **Step 4: Re-run the shell and toolbar tests**

Run:

```bash
cd frontend
npm run test -- --run src/App.shell.test.tsx src/components/calendar/components/mobile-toolbar.test.tsx
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/shared/app-header.tsx src/components/calendar/components/mobile-toolbar.tsx src/components/calendar/components/mobile-toolbar.test.tsx src/components/calendar/calendar-module.tsx src/App.shell.test.tsx
git commit -m "refactor(shell): remove redundant mobile home buttons"
```

---

### Task 4: Lift the Calendar FAB above the new nav

**Files:**
- Modify: `frontend/src/components/calendar/components/add-event-button.tsx`
- Modify: `frontend/src/components/calendar/calendar-module.tsx`
- Create: `frontend/src/components/calendar/components/add-event-button.test.tsx`

- [ ] **Step 1: Write the failing FAB offset test**

Create `frontend/src/components/calendar/components/add-event-button.test.tsx`:

```tsx
import { describe, expect, it, vi } from "vitest";
import { render, screen } from "@/test/test-utils";
import { AddEventButton } from "./add-event-button";

describe("AddEventButton", () => {
  it("uses the elevated offset when mobile nav is present", () => {
    render(<AddEventButton onClick={vi.fn()} offsetForMobileNav />);

    expect(screen.getByRole("button", { name: /add event/i })).toHaveStyle(
      "bottom: max(4.5rem, calc(env(safe-area-inset-bottom) + 4.5rem))",
    );
  });

  it("keeps the existing offset when mobile nav is not present", () => {
    render(<AddEventButton onClick={vi.fn()} />);

    expect(screen.getByRole("button", { name: /add event/i })).toHaveStyle(
      "bottom: max(2rem, calc(env(safe-area-inset-bottom) + 1rem))",
    );
  });
});
```

- [ ] **Step 2: Run the FAB test and confirm it fails**

Run:

```bash
cd frontend
npm run test -- --run src/components/calendar/components/add-event-button.test.tsx
```

Expected: FAIL because `offsetForMobileNav` does not exist yet.

- [ ] **Step 3: Implement the mobile-nav offset**

Update `frontend/src/components/calendar/components/add-event-button.tsx`:

```tsx
import { Plus } from "lucide-react";
import { Button } from "@/components/ui/button";

interface AddEventButtonProps {
  onClick: () => void;
  offsetForMobileNav?: boolean;
}

export function AddEventButton({
  onClick,
  offsetForMobileNav = false,
}: AddEventButtonProps) {
  const bottom = offsetForMobileNav
    ? "max(4.5rem, calc(env(safe-area-inset-bottom) + 4.5rem))"
    : "max(2rem, calc(env(safe-area-inset-bottom) + 1rem))";

  return (
    <Button
      onClick={onClick}
      className="fixed right-8 z-40 h-14 w-14 rounded-full bg-primary shadow-lg transition-all hover:bg-primary/90 hover:shadow-xl"
      style={{ bottom }}
      size="icon"
      aria-label="Add event"
    >
      <Plus className="h-7 w-7 text-primary-foreground" />
    </Button>
  );
}
```

Update `frontend/src/components/calendar/calendar-module.tsx`:

```tsx
<AddEventButton
  onClick={openAddEventModal}
  offsetForMobileNav={isMobile}
/>
```

- [ ] **Step 4: Re-run the FAB test**

Run:

```bash
cd frontend
npm run test -- --run src/components/calendar/components/add-event-button.test.tsx
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/calendar/components/add-event-button.tsx src/components/calendar/components/add-event-button.test.tsx src/components/calendar/calendar-module.tsx
git commit -m "fix(calendar): lift mobile add-event button above bottom nav"
```

---

### Task 5: Add Playwright coverage for mobile nav

**Files:**
- Create: `frontend/e2e/mobile-bottom-nav.spec.ts`

- [ ] **Step 1: Write the failing E2E spec**

Create `frontend/e2e/mobile-bottom-nav.spec.ts`:

```ts
import { expect, test } from "@playwright/test";
import { registerFamily, seedBrowserAuth } from "./helpers/api-helpers";
import { clearStorage, safeClick, waitForHydration } from "./helpers/test-helpers";

test.describe("Mobile bottom nav", () => {
  test.beforeEach(async ({ page, request, isMobile }) => {
    test.skip(!isMobile, "Mobile-only tests");

    await page.goto("/");
    await clearStorage(page);

    const reg = await registerFamily(request, {
      familyName: "Nav Test Family",
      members: [
        { name: "Alice", color: "coral" },
        { name: "Bob", color: "teal" },
      ],
    });

    await seedBrowserAuth(page, reg);
    await page.reload();
    await waitForHydration(page);
  });

  test("switches between Home and all five modules", async ({ page }) => {
    const nav = page.getByRole("navigation", { name: /primary/i });
    await expect(nav).toBeVisible();

    await expect(page.getByRole("heading", { name: "Home", level: 1 })).toBeVisible();

    await safeClick(page.getByRole("button", { name: /^calendar$/i }));
    await expect(page.getByRole("button", { name: "Add event" })).toBeVisible();

    await safeClick(page.getByRole("button", { name: /^lists$/i }));
    await expect(page.getByText("My Lists")).toBeVisible();

    await safeClick(page.getByRole("button", { name: /^chores$/i }));
    await expect(page.getByText("Today's Chores")).toBeVisible();

    await safeClick(page.getByRole("button", { name: /^meals$/i }));
    await expect(page.getByText("Meal Planning")).toBeVisible();

    await safeClick(page.getByRole("button", { name: /^photos$/i }));
    await expect(page.getByText("Family Photos")).toBeVisible();

    await safeClick(page.getByRole("button", { name: /^home$/i }));
    await expect(page.getByRole("heading", { name: "Home", level: 1 })).toBeVisible();
  });

  test("is absent on login and onboarding", async ({ page }) => {
    await clearStorage(page);
    await page.goto("/");
    await page.reload();
    await waitForHydration(page);

    await expect(
      page.getByRole("navigation", { name: /primary/i }),
    ).toHaveCount(0);

    await page.getByRole("button", { name: /create an account/i }).click();

    await expect(
      page.getByRole("navigation", { name: /primary/i }),
    ).toHaveCount(0);
  });
});
```

- [ ] **Step 2: Run the E2E spec and confirm it fails**

Run:

```bash
cd frontend
npm run test:e2e -- mobile-bottom-nav.spec.ts
```

Expected: FAIL before the implementation is fully wired.

- [ ] **Step 3: Re-run the E2E spec after Tasks 1–4 are complete**

Run:

```bash
cd frontend
npm run test:e2e -- mobile-bottom-nav.spec.ts
```

Expected: PASS.

- [ ] **Step 4: Commit**

```bash
cd frontend
git add e2e/mobile-bottom-nav.spec.ts
git commit -m "test(shell): cover mobile bottom nav flow"
```

---

### Task 6: Visual verification and PR

**Files:**
- No new files. Verification against the completed branch.

- [ ] **Step 1: Run the focused FE test suite**

Run:

```bash
cd frontend
npm run test -- --run src/components/shared/mobile-bottom-nav.test.tsx src/App.shell.test.tsx src/components/calendar/components/mobile-toolbar.test.tsx src/components/calendar/components/add-event-button.test.tsx
```

Expected: PASS.

- [ ] **Step 2: Run the mobile E2E spec**

Run:

```bash
cd frontend
npm run test:e2e -- mobile-bottom-nav.spec.ts
```

Expected: PASS.

- [ ] **Step 3: Manual visual smoke in the mobile viewport**

Run:

```bash
cd frontend
npm run dev
```

In Chrome DevTools at `390x844`, verify:

- Home shows `AppHeader` plus the new bottom nav
- Calendar hides `AppHeader`, keeps `MobileToolbar`, and the FAB clears the nav
- Lists / Chores / Meals / Photos show `AppHeader` plus the new bottom nav
- sidebar backdrop covers the nav
- opening a dialog or sheet covers the nav
- there is no horizontal overflow introduced by the nav rail

- [ ] **Step 4: Open the PR**

```bash
cd frontend
git status
git push -u origin feat/mobile-bottom-nav
gh pr create --fill
```

Expected: clean branch, pushed feature branch, PR opened against `main`.
