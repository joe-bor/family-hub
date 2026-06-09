# Sidebar + Settings + Onboarding Mobile Pass — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make onboarding/login, the sidebar/menu, and the settings modals comfortable and correct for daily phone use by two adults.

**Architecture:** FE-only ergonomics pass. Swap `100vh` for `dvh` + safe-area on auth/onboarding/shell; make auth forms keyboard-safe; remove the dead header Settings button; rebuild the sidebar on a new Radix-Dialog-based `SideSheet` primitive (focus-trap/Esc/scrim/swipe) with a confirmed Sign Out; bound the member-profile modal's height; and collapse the 7-tab bottom nav to ≤5 slots with a scalable `MobileSheet`-based "More" overflow (Photos dropped). No backend, routing, or desktop changes.

**Tech Stack:** React 19, TypeScript, Zustand, `@radix-ui/react-dialog`, Tailwind CSS v4 + tailwindcss-animate, Vitest, Testing Library, Playwright.

**Spec:** `docs/superpowers/specs/2026-06-08-mobile-sidebar-settings-onboarding-design.md`
**Story:** `docs/product/backlog/mobile-ux/sidebar-settings-onboarding-mobile.md`

---

This is an **FE-only** plan. Execute it inside the `frontend/` repo on a fresh feature branch such as `feat/mobile-sidebar-settings-onboarding`. All paths below are relative to `frontend/`. Use **regular merge commits** (release-please) and conventional commit messages.

## File Structure

| File | Action | Purpose |
|------|--------|---------|
| `src/components/onboarding/onboarding-welcome.tsx` | Modify | `dvh` + safe-area |
| `src/components/onboarding/onboarding-family-name.tsx` | Modify | `dvh` + safe-area + keyboard-safe |
| `src/components/onboarding/onboarding-members.tsx` | Modify | `dvh` + safe-area |
| `src/components/onboarding/onboarding-credentials.tsx` | Modify | `dvh` + safe-area + keyboard-safe + gated autofocus |
| `src/components/auth/login-form.tsx` | Modify | `dvh` + safe-area + keyboard-safe + gated autofocus |
| `src/App.tsx` | Modify | `h-screen` → `h-dvh` shell |
| `src/components/shared/app-header.tsx` | Modify | Remove dead Settings button |
| `src/components/shared/app-header.test.tsx` | Create | Lock single header entry |
| `src/components/settings/member-profile-modal.tsx` | Modify | `max-h` + scroll parity |
| `src/components/ui/side-sheet.tsx` | Create | Left-anchored Radix Dialog primitive + swipe-to-close |
| `src/components/shared/sidebar-menu.tsx` | Modify | Rebuild on `SideSheet`; label fix; ≥44px targets; Sign Out confirm |
| `src/components/shared/sidebar-menu.test.tsx` | Create | Esc-close + Sign Out confirm |
| `src/components/shared/mobile-bottom-nav.tsx` | Modify | ≤5 slots + "More" overflow; drop Photos |
| `src/components/shared/mobile-bottom-nav.test.tsx` | Modify | Slot inventory + More flow |
| `e2e/mobile-settings-sidebar.spec.ts` | Create | Sidebar + More E2E |

---

### Task 1: `dvh` + safe-area on auth / onboarding / shell

Style-only refactor — no new behavior, so verification is "existing suites stay green + visual smoke." Do **not** invent a unit test for class names.

**Files:**
- Modify: `src/App.tsx`
- Modify: `src/components/onboarding/onboarding-welcome.tsx`
- Modify: `src/components/onboarding/onboarding-family-name.tsx`
- Modify: `src/components/onboarding/onboarding-members.tsx`
- Modify: `src/components/onboarding/onboarding-credentials.tsx`
- Modify: `src/components/auth/login-form.tsx`

- [ ] **Step 1: Shell** — in `src/App.tsx`, change both shell wrappers from `h-screen` to `h-dvh`:
  - `LoadingScreen`'s `<div className="h-screen ...">` → `h-dvh`
  - the authenticated shell `<div className="h-screen flex flex-col bg-background">` → `h-dvh`

- [ ] **Step 2: Onboarding + login containers** — in each of the five screens, replace the outer container class.
  - `onboarding-welcome.tsx:10`: replace `min-h-screen` with `min-h-dvh` and append safe-area padding. New outer class:

    ```tsx
    <div className="flex flex-col items-center justify-center min-h-dvh p-4 md:p-6 bg-gradient-to-b from-primary/10 to-background [padding-top:max(1rem,env(safe-area-inset-top))] [padding-bottom:max(1rem,env(safe-area-inset-bottom))]">
    ```

  - `onboarding-members.tsx:61` and `onboarding-family-name.tsx:37`: replace `min-h-screen` with `min-h-dvh` and add the same safe-area padding utilities:

    ```tsx
    <div className="flex flex-col min-h-dvh p-4 md:p-6 bg-background [padding-top:max(1rem,env(safe-area-inset-top))] [padding-bottom:max(1rem,env(safe-area-inset-bottom))]">
    ```

  (`onboarding-credentials.tsx` and `login-form.tsx` get the same outer change, but their inner layout is handled in Task 2 — do those there to keep one diff per file.)

- [ ] **Step 3: Verify existing suites stay green**

  ```bash
  cd frontend
  npm run test -- --run src/components/onboarding src/App.shell.test.tsx
  npm run lint
  ```

  Expected: PASS / no new lint errors.

- [ ] **Step 4: Visual smoke** — `npm run dev`, Chrome DevTools at `390x844` with device-toolbar on. Confirm the "Get Started" / "Continue" CTAs sit above the browser chrome on welcome and members, and nothing hugs the notch.

- [ ] **Step 5: Commit**

  ```bash
  git add src/App.tsx src/components/onboarding/onboarding-welcome.tsx src/components/onboarding/onboarding-family-name.tsx src/components/onboarding/onboarding-members.tsx src/components/onboarding/onboarding-credentials.tsx src/components/auth/login-form.tsx
  git commit -m "fix(mobile): use dvh + safe-area on auth, onboarding, and shell"
  ```

---

### Task 2: Keyboard-safe auth forms (credentials, family-name, login)

Stop force-centering on mobile and disable mobile autofocus so the keyboard doesn't cover inputs on first paint.

**Files:**
- Modify: `src/components/auth/login-form.tsx`
- Modify: `src/components/onboarding/onboarding-credentials.tsx`
- Test: `src/components/auth/login-form.test.tsx` (create if absent; otherwise extend)

- [ ] **Step 1: Write the failing test** — create/extend `src/components/auth/login-form.test.tsx`:

  ```tsx
  import { beforeEach, describe, expect, it, vi } from "vitest";
  import { render, screen } from "@/test/test-utils";
  import { LoginForm } from "./login-form";

  let mockIsMobile = true;
  vi.mock("@/hooks", async (importOriginal) => {
    const actual = await importOriginal<typeof import("@/hooks")>();
    return { ...actual, useIsMobile: () => mockIsMobile };
  });

  describe("LoginForm", () => {
    beforeEach(() => {
      mockIsMobile = true;
    });

    it("does not autofocus the username field on mobile", () => {
      render(<LoginForm onSwitchToOnboarding={vi.fn()} />);
      expect(screen.getByLabelText(/username/i)).not.toBe(document.activeElement);
    });

    it("autofocuses the username field on desktop", () => {
      mockIsMobile = false;
      render(<LoginForm onSwitchToOnboarding={vi.fn()} />);
      expect(screen.getByLabelText(/username/i)).toBe(document.activeElement);
    });
  });
  ```

- [ ] **Step 2: Run and confirm it fails**

  ```bash
  cd frontend
  npm run test -- --run src/components/auth/login-form.test.tsx
  ```

  Expected: FAIL — username is currently always autofocused.

- [ ] **Step 3: Implement in `login-form.tsx`**
  - Add the hook import: `import { useIsMobile } from "@/hooks";` and `const isMobile = useIsMobile();` in the component body.
  - Outer container (line 45): `min-h-screen` → `min-h-dvh` + safe-area, and only center vertically off-mobile:

    ```tsx
    <div className="flex flex-col min-h-dvh p-4 md:p-6 bg-background overflow-y-auto [padding-top:max(1rem,env(safe-area-inset-top))] [padding-bottom:max(1rem,env(safe-area-inset-bottom))]">
      <div className={cn("flex-1 flex flex-col max-w-md mx-auto w-full", !isMobile && "justify-center")}>
    ```

    (add `import { cn } from "@/lib/utils";`)
  - Username input: change `autoFocus` → `autoFocus={!isMobile}`.

- [ ] **Step 4: Implement in `onboarding-credentials.tsx`**
  - Add `import { useIsMobile } from "@/hooks";`, `import { cn } from "@/lib/utils";`, and `const isMobile = useIsMobile();`.
  - Outer container (line 69): `min-h-screen` → `min-h-dvh` + safe-area + `overflow-y-auto`.
  - Inner wrapper (line 85): `className="flex-1 flex flex-col justify-center max-w-md mx-auto w-full"` → gate centering: `className={cn("flex-1 flex flex-col max-w-md mx-auto w-full", !isMobile && "justify-center")}`.
  - Username input (line 110): `autoFocus` → `autoFocus={!isMobile}`.

- [ ] **Step 5: Apply the same outer/centering change to `onboarding-family-name.tsx`** (its inner wrapper at line 52 uses `justify-center`). It has no autofocus to gate — just gate the `justify-center` the same way and ensure the outer container already got `min-h-dvh` + safe-area in Task 1.

- [ ] **Step 6: Run the test and the onboarding suite**

  ```bash
  cd frontend
  npm run test -- --run src/components/auth/login-form.test.tsx src/components/onboarding
  ```

  Expected: PASS.

- [ ] **Step 7: Commit**

  ```bash
  git add src/components/auth/login-form.tsx src/components/auth/login-form.test.tsx src/components/onboarding/onboarding-credentials.tsx src/components/onboarding/onboarding-family-name.tsx
  git commit -m "fix(mobile): keep auth form inputs above the keyboard"
  ```

---

### Task 3: Single settings entry in the header

Remove the dead `<Settings>` button; the `Menu` button is the one entry.

**Files:**
- Modify: `src/components/shared/app-header.tsx`
- Create: `src/components/shared/app-header.test.tsx`

- [ ] **Step 1: Write the failing test** — create `src/components/shared/app-header.test.tsx`:

  ```tsx
  import { beforeEach, describe, expect, it, vi } from "vitest";
  import { render, screen, seedFamilyStore } from "@/test/test-utils";
  import { AppHeader } from "./app-header";

  vi.mock("@/hooks", async (importOriginal) => {
    const actual = await importOriginal<typeof import("@/hooks")>();
    return { ...actual, useIsMobile: () => true };
  });

  describe("AppHeader (mobile)", () => {
    beforeEach(() => {
      seedFamilyStore({
        name: "Test Family",
        members: [{ id: "m1", name: "Alice", color: "coral" }],
      });
    });

    it("exposes exactly one header control: Menu", () => {
      render(<AppHeader />);
      expect(screen.getByRole("button", { name: /menu/i })).toBeInTheDocument();
      expect(screen.getAllByRole("button")).toHaveLength(1);
    });
  });
  ```

- [ ] **Step 2: Run and confirm it fails**

  ```bash
  cd frontend
  npm run test -- --run src/components/shared/app-header.test.tsx
  ```

  Expected: FAIL — there are currently two buttons (Menu + dead Settings).

- [ ] **Step 3: Remove the dead button** in `src/components/shared/app-header.tsx`
  - Drop `Settings` from the `lucide-react` import (line 1).
  - Delete the entire trailing Settings `<Button>` block (lines ~97-105):

    ```tsx
    {/* Settings */}
    <Button variant="ghost" size="icon" className="text-muted-foreground hover:text-foreground">
      <Settings className="h-5 w-5" />
    </Button>
    ```

  - The right-hand `<div className={cn("flex items-center", ...)}>` now holds only the desktop-only weather + member indicators; leave those untouched.

- [ ] **Step 4: Run the test**

  ```bash
  cd frontend
  npm run test -- --run src/components/shared/app-header.test.tsx
  ```

  Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add src/components/shared/app-header.tsx src/components/shared/app-header.test.tsx
  git commit -m "fix(mobile): remove dead settings button from app header"
  ```

---

### Task 4: Bound the member-profile modal height

Give `MemberProfileModal` scroll parity with `FamilySettingsModal`.

**Files:**
- Modify: `src/components/settings/member-profile-modal.tsx`

- [ ] **Step 1: Apply the change** — in `member-profile-modal.tsx`, change the `DialogContent` class (line 141) from:

  ```tsx
  <DialogContent className="max-w-sm">
  ```

  to:

  ```tsx
  <DialogContent className="max-w-sm max-h-[90vh] overflow-y-auto">
  ```

- [ ] **Step 2: Verify the existing modal test still passes**

  ```bash
  cd frontend
  npm run test -- --run src/components/settings/member-profile-modal.test.tsx
  ```

  Expected: PASS.

- [ ] **Step 3: Visual check** — `npm run dev` at `390x844`, open a member profile, expand the Google Calendar section, confirm Save/Cancel stay reachable via internal scroll.

- [ ] **Step 4: Commit**

  ```bash
  git add src/components/settings/member-profile-modal.tsx
  git commit -m "fix(mobile): bound member profile modal height with scroll"
  ```

---

### Task 5: Rebuild the sidebar on a Radix `SideSheet`

Create a reusable left-anchored dialog primitive and rebuild `SidebarMenu` on it: focus-trap + Esc + scrim + scroll-lock for free, swipe-to-close, fixed label, ≥44px targets, confirmed Sign Out.

**Files:**
- Create: `src/components/ui/side-sheet.tsx`
- Modify: `src/components/shared/sidebar-menu.tsx`
- Create: `src/components/shared/sidebar-menu.test.tsx`

- [ ] **Step 1: Create the `SideSheet` primitive** — `src/components/ui/side-sheet.tsx`:

  ```tsx
  import * as DialogPrimitive from "@radix-ui/react-dialog";
  import { type ReactNode, useRef, useState } from "react";
  import { cn } from "@/lib/utils";

  const SWIPE_CLOSE_THRESHOLD = 60; // px of leftward drag before dismiss

  interface SideSheetProps {
    open: boolean;
    onOpenChange: (open: boolean) => void;
    /** Accessible name for the dialog (visually hidden). */
    title: string;
    children: ReactNode;
    className?: string;
  }

  export function SideSheet({
    open,
    onOpenChange,
    title,
    children,
    className,
  }: SideSheetProps) {
    const startX = useRef<number | null>(null);
    const [dragX, setDragX] = useState(0);

    const handleTouchStart = (e: React.TouchEvent) => {
      startX.current = e.touches[0]?.clientX ?? null;
    };
    const handleTouchMove = (e: React.TouchEvent) => {
      if (startX.current === null) return;
      const delta = (e.touches[0]?.clientX ?? startX.current) - startX.current;
      if (delta < 0) setDragX(delta); // track leftward drag only
    };
    const handleTouchEnd = () => {
      if (dragX < -SWIPE_CLOSE_THRESHOLD) onOpenChange(false);
      startX.current = null;
      setDragX(0);
    };

    return (
      <DialogPrimitive.Root open={open} onOpenChange={onOpenChange}>
        <DialogPrimitive.Portal>
          <DialogPrimitive.Overlay
            className={cn(
              "fixed inset-0 z-40 bg-foreground/20 backdrop-blur-sm",
              "data-[state=open]:animate-in data-[state=closed]:animate-out",
              "data-[state=open]:fade-in-0 data-[state=closed]:fade-out-0",
            )}
          />
          <DialogPrimitive.Content
            onTouchStart={handleTouchStart}
            onTouchMove={handleTouchMove}
            onTouchEnd={handleTouchEnd}
            style={dragX ? { transform: `translateX(${dragX}px)` } : undefined}
            className={cn(
              "fixed inset-y-0 left-0 z-50 flex w-[min(20rem,85vw)] flex-col bg-card shadow-2xl",
              "[padding-top:env(safe-area-inset-top)] [padding-bottom:env(safe-area-inset-bottom)]",
              "data-[state=open]:animate-in data-[state=closed]:animate-out duration-200",
              "data-[state=open]:slide-in-from-left data-[state=closed]:slide-out-to-left",
              className,
            )}
          >
            <DialogPrimitive.Title className="sr-only">
              {title}
            </DialogPrimitive.Title>
            {children}
          </DialogPrimitive.Content>
        </DialogPrimitive.Portal>
      </DialogPrimitive.Root>
    );
  }
  ```

- [ ] **Step 2: Write the failing sidebar test** — `src/components/shared/sidebar-menu.test.tsx`:

  ```tsx
  import { beforeEach, describe, expect, it, vi } from "vitest";
  import {
    renderWithUser,
    screen,
    seedFamilyStore,
  } from "@/test/test-utils";
  import { useAppStore } from "@/stores";
  import { SidebarMenu } from "./sidebar-menu";

  const logoutSpy = vi.fn();
  vi.mock("@/api", async (importOriginal) => {
    const actual = await importOriginal<typeof import("@/api")>();
    return { ...actual, useLogout: () => logoutSpy };
  });

  describe("SidebarMenu", () => {
    beforeEach(() => {
      logoutSpy.mockClear();
      seedFamilyStore({
        name: "Test Family",
        members: [{ id: "m1", name: "Alice", color: "coral" }],
      });
      useAppStore.setState({ isSidebarOpen: true });
    });

    it("closes on Escape", async () => {
      const { user } = renderWithUser(<SidebarMenu />);
      expect(screen.getByText("Test Family")).toBeInTheDocument();
      await user.keyboard("{Escape}");
      expect(useAppStore.getState().isSidebarOpen).toBe(false);
    });

    it("requires confirmation before signing out", async () => {
      const { user } = renderWithUser(<SidebarMenu />);
      await user.click(screen.getByRole("button", { name: /sign out/i }));
      expect(logoutSpy).not.toHaveBeenCalled();

      // confirmation dialog appears
      await user.click(
        screen.getByRole("button", { name: /^sign out$/i, hidden: false }),
      );
      expect(logoutSpy).toHaveBeenCalledTimes(1);
    });
  });
  ```

  > Note: the confirm step targets the destructive "Sign Out" button inside the confirm dialog. If the menu item and the confirm button collide by accessible name in your Testing Library version, label the menu item "Sign Out" and the confirm action "Yes, sign out" and update this assertion accordingly.

- [ ] **Step 3: Run and confirm it fails**

  ```bash
  cd frontend
  npm run test -- --run src/components/shared/sidebar-menu.test.tsx
  ```

  Expected: FAIL — no confirm dialog yet; Esc may not close the custom aside.

- [ ] **Step 4: Rebuild `sidebar-menu.tsx`** on `SideSheet` with the confirm + label + tap-target changes:

  ```tsx
  import { LogOut, Users, X } from "lucide-react";
  import { useEffect, useState } from "react";
  import { useFamilyMembers, useFamilyName, useLogout } from "@/api";
  import { FamilySettingsModal, MemberProfileModal } from "@/components/settings";
  import { Button } from "@/components/ui/button";
  import {
    Dialog,
    DialogContent,
    DialogDescription,
    DialogHeader,
    DialogTitle,
  } from "@/components/ui/dialog";
  import { SideSheet } from "@/components/ui/side-sheet";
  import { colorMap } from "@/lib/types";
  import { cn } from "@/lib/utils";
  import { useAppStore } from "@/stores";

  export function SidebarMenu() {
    const isOpen = useAppStore((state) => state.isSidebarOpen);
    const closeSidebar = useAppStore((state) => state.closeSidebar);

    const familyName = useFamilyName();
    const familyMembers = useFamilyMembers();
    const logout = useLogout();

    const [isSettingsOpen, setIsSettingsOpen] = useState(false);
    const [selectedMemberId, setSelectedMemberId] = useState<string | null>(null);
    const [showSignOutConfirm, setShowSignOutConfirm] = useState(false);

    useEffect(() => {
      if (isOpen) {
        const autoOpenMemberId = sessionStorage.getItem("open-member-profile");
        if (autoOpenMemberId) {
          sessionStorage.removeItem("open-member-profile");
          setSelectedMemberId(autoOpenMemberId);
        }
      }
    }, [isOpen]);

    const menuItems = [
      {
        icon: Users,
        label: "Family Settings",
        action: () => setIsSettingsOpen(true),
      },
      {
        icon: LogOut,
        label: "Sign Out",
        action: () => setShowSignOutConfirm(true),
      },
    ];

    return (
      <>
        <SideSheet
          open={isOpen}
          onOpenChange={(open) => !open && closeSidebar()}
          title="Menu"
        >
          {/* Header */}
          <div className="flex items-center justify-between p-6 border-b border-border">
            <div>
              <h2 className="text-lg font-bold text-foreground">
                {familyName || "Family Hub"}
              </h2>
              <p className="text-sm text-muted-foreground">Menu</p>
            </div>
            <Button
              variant="ghost"
              size="icon"
              aria-label="Close menu"
              onClick={closeSidebar}
              className="h-11 w-11"
            >
              <X className="h-5 w-5" />
            </Button>
          </div>

          {/* Family Members */}
          <div className="p-4 border-b border-border">
            <h3 className="text-xs font-semibold text-muted-foreground uppercase tracking-wider mb-3">
              Family Members
            </h3>
            <div className="space-y-2">
              {familyMembers.length > 0 ? (
                familyMembers.map((member) => {
                  const colors = colorMap[member.color];
                  return (
                    <button
                      key={member.id}
                      type="button"
                      onClick={() => setSelectedMemberId(member.id)}
                      className="w-full min-h-11 flex items-center gap-3 px-3 py-2.5 rounded-lg hover:bg-muted transition-colors"
                    >
                      {member.avatarUrl ? (
                        <img
                          src={member.avatarUrl}
                          alt={member.name}
                          className="w-8 h-8 rounded-full object-cover"
                        />
                      ) : (
                        <div
                          className={cn(
                            "w-8 h-8 rounded-full flex items-center justify-center text-card text-sm font-bold",
                            colors?.bg,
                          )}
                        >
                          {member.name.charAt(0)}
                        </div>
                      )}
                      <span className="text-sm font-medium text-foreground">
                        {member.name}
                      </span>
                    </button>
                  );
                })
              ) : (
                <p className="text-sm text-muted-foreground px-3">
                  No family members yet
                </p>
              )}
            </div>
          </div>

          {/* Menu Items */}
          <nav className="flex-1 p-4">
            <div className="space-y-1">
              {menuItems.map((item) => {
                const Icon = item.icon;
                return (
                  <button
                    key={item.label}
                    type="button"
                    onClick={item.action}
                    className="w-full min-h-11 flex items-center gap-3 px-3 py-3 rounded-lg text-sm font-medium transition-colors text-muted-foreground hover:bg-muted hover:text-foreground"
                  >
                    <Icon className="h-5 w-5" />
                    {item.label}
                  </button>
                );
              })}
            </div>
          </nav>

          {/* Version */}
          <div className="border-t border-border px-6 py-4">
            <p className="text-xs text-muted-foreground">v{__APP_VERSION__}</p>
          </div>
        </SideSheet>

        {/* Family Settings Modal */}
        <FamilySettingsModal
          open={isSettingsOpen}
          onOpenChange={setIsSettingsOpen}
        />

        {/* Member Profile Modal */}
        {selectedMemberId && (
          <MemberProfileModal
            open={!!selectedMemberId}
            onOpenChange={(open) => !open && setSelectedMemberId(null)}
            memberId={selectedMemberId}
          />
        )}

        {/* Sign Out Confirmation */}
        <Dialog open={showSignOutConfirm} onOpenChange={setShowSignOutConfirm}>
          <DialogContent className="max-w-sm">
            <DialogHeader>
              <DialogTitle>Sign out?</DialogTitle>
              <DialogDescription>
                You'll need your family username and password to sign back in.
              </DialogDescription>
            </DialogHeader>
            <div className="flex justify-end gap-3 pt-4">
              <Button
                variant="outline"
                onClick={() => setShowSignOutConfirm(false)}
              >
                Cancel
              </Button>
              <Button variant="destructive" onClick={() => logout()}>
                Sign Out
              </Button>
            </div>
          </DialogContent>
        </Dialog>
      </>
    );
  }
  ```

  Key changes from the old component: `SideSheet` replaces the hand-rolled `aside`+backdrop (so Esc/scrim/focus-trap come from Radix); subtitle "Calendar Settings" → "Menu"; member rows and menu items get `min-h-11` and larger padding (≥44px); close button `h-11 w-11`; Sign Out routes through a confirm dialog instead of calling `logout` directly.

- [ ] **Step 5: Run the sidebar test**

  ```bash
  cd frontend
  npm run test -- --run src/components/shared/sidebar-menu.test.tsx
  ```

  Expected: PASS. (If the two "Sign Out" names collide, apply the rename noted in Step 2.)

- [ ] **Step 6: Manual device check for swipe** — `npm run dev` at `390x844` with touch emulation: open the menu, drag left past ~60px, confirm it dismisses; confirm scrim tap and Esc also close and that focus returns to the Menu button.

- [ ] **Step 7: Commit**

  ```bash
  git add src/components/ui/side-sheet.tsx src/components/shared/sidebar-menu.tsx src/components/shared/sidebar-menu.test.tsx
  git commit -m "feat(mobile): rebuild sidebar as accessible side sheet with sign-out confirm"
  ```

---

### Task 6: Bottom nav ≤5 slots + "More" overflow

Collapse 7 tabs to 4 primary + a `MobileSheet`-based "More"; drop Photos.

**Files:**
- Modify: `src/components/shared/mobile-bottom-nav.tsx`
- Modify: `src/components/shared/mobile-bottom-nav.test.tsx`

- [ ] **Step 1: Rewrite the test** — replace `src/components/shared/mobile-bottom-nav.test.tsx`:

  ```tsx
  import { beforeEach, describe, expect, it } from "vitest";
  import { render, renderWithUser, screen } from "@/test/test-utils";
  import { useAppStore } from "@/stores";
  import { MobileBottomNav } from "./mobile-bottom-nav";

  describe("MobileBottomNav", () => {
    beforeEach(() => {
      useAppStore.setState({ activeModule: null });
    });

    it("shows four primary tabs plus More, and no Photos", () => {
      render(<MobileBottomNav />);
      const nav = screen.getByRole("navigation", { name: /primary/i });
      expect(nav).toBeInTheDocument();

      for (const label of ["Home", "Calendar", "Lists", "Chores", "More"]) {
        expect(
          screen.getByRole("button", { name: new RegExp(`^${label}$`, "i") }),
        ).toBeInTheDocument();
      }
      // Overflow + deferred modules are not in the bar.
      expect(
        screen.queryByRole("button", { name: /^meals$/i }),
      ).not.toBeInTheDocument();
      expect(
        screen.queryByRole("button", { name: /^photos$/i }),
      ).not.toBeInTheDocument();
    });

    it("opens the More sheet and switches to an overflow module", async () => {
      const { user } = renderWithUser(<MobileBottomNav />);
      await user.click(screen.getByRole("button", { name: /^more$/i }));

      const meals = await screen.findByRole("button", { name: /^meals$/i });
      await user.click(meals);

      expect(useAppStore.getState().activeModule).toBe("meals");
      // sheet closes
      expect(
        screen.queryByRole("button", { name: /^recipes$/i }),
      ).not.toBeInTheDocument();
    });

    it("marks More active when an overflow module is active", () => {
      useAppStore.setState({ activeModule: "recipes" });
      render(<MobileBottomNav />);
      expect(screen.getByRole("button", { name: /^more$/i })).toHaveAttribute(
        "aria-current",
        "page",
      );
    });
  });
  ```

- [ ] **Step 2: Run and confirm it fails**

  ```bash
  cd frontend
  npm run test -- --run src/components/shared/mobile-bottom-nav.test.tsx
  ```

  Expected: FAIL — current nav has 7 tabs and no More.

- [ ] **Step 3: Rewrite `mobile-bottom-nav.tsx`**:

  ```tsx
  import {
    BookOpenText,
    Calendar,
    CheckSquare,
    Home,
    ListTodo,
    type LucideIcon,
    MoreHorizontal,
    UtensilsCrossed,
  } from "lucide-react";
  import { useState } from "react";
  import { MobileSheet } from "@/components/ui/mobile-sheet";
  import { cn } from "@/lib/utils";
  import { type ModuleType, useAppStore } from "@/stores";

  type NavModule = { id: ModuleType | null; label: string; icon: LucideIcon };

  // Ordered navigable modules. First PRIMARY_COUNT render in the bar; the rest
  // overflow into the "More" sheet. Photos is intentionally omitted (deferred).
  // Appending a future module here makes it overflow automatically.
  const MODULES: NavModule[] = [
    { id: null, label: "Home", icon: Home },
    { id: "calendar", label: "Calendar", icon: Calendar },
    { id: "lists", label: "Lists", icon: ListTodo },
    { id: "chores", label: "Chores", icon: CheckSquare },
    { id: "meals", label: "Meals", icon: UtensilsCrossed },
    { id: "recipes", label: "Recipes", icon: BookOpenText },
  ];
  const PRIMARY_COUNT = 4;
  const primaryModules = MODULES.slice(0, PRIMARY_COUNT);
  const overflowModules = MODULES.slice(PRIMARY_COUNT);

  const tabBase =
    "flex min-h-14 min-w-16 flex-1 flex-col items-center justify-center gap-0.5 rounded-lg px-2 py-2 text-[11px] leading-none font-semibold transition-colors";
  const tabActive = "bg-primary text-primary-foreground shadow-sm shadow-primary/20";
  const tabIdle = "text-muted-foreground hover:bg-muted hover:text-foreground";

  export function MobileBottomNav() {
    const activeModule = useAppStore((state) => state.activeModule);
    const setActiveModule = useAppStore((state) => state.setActiveModule);
    const [moreOpen, setMoreOpen] = useState(false);

    const overflowActive = overflowModules.some((m) => m.id === activeModule);

    const selectOverflow = (id: ModuleType | null) => {
      setActiveModule(id);
      setMoreOpen(false);
    };

    return (
      <>
        <nav
          aria-label="Primary"
          className="z-30 shrink-0 border-t border-border bg-card/95 backdrop-blur supports-[backdrop-filter]:bg-card/85"
        >
          <div
            className="flex gap-1 px-2 pt-2"
            style={{ paddingBottom: "max(0.5rem, env(safe-area-inset-bottom))" }}
          >
            {primaryModules.map((tab) => {
              const Icon = tab.icon;
              const isActive = activeModule === tab.id;
              return (
                <button
                  key={tab.label}
                  type="button"
                  onClick={() => setActiveModule(tab.id)}
                  aria-label={tab.label}
                  aria-current={isActive ? "page" : undefined}
                  className={cn(tabBase, isActive ? tabActive : tabIdle)}
                >
                  <Icon className="h-4 w-4" />
                  <span className="whitespace-nowrap leading-3">{tab.label}</span>
                </button>
              );
            })}

            {overflowModules.length > 0 && (
              <button
                type="button"
                onClick={() => setMoreOpen(true)}
                aria-label="More"
                aria-current={overflowActive ? "page" : undefined}
                className={cn(tabBase, overflowActive ? tabActive : tabIdle)}
              >
                <MoreHorizontal className="h-4 w-4" />
                <span className="whitespace-nowrap leading-3">More</span>
              </button>
            )}
          </div>
        </nav>

        <MobileSheet
          isOpen={moreOpen}
          onClose={() => setMoreOpen(false)}
          title="More"
        >
          <div className="space-y-1">
            {overflowModules.map((tab) => {
              const Icon = tab.icon;
              const isActive = activeModule === tab.id;
              return (
                <button
                  key={tab.label}
                  type="button"
                  onClick={() => selectOverflow(tab.id)}
                  aria-current={isActive ? "page" : undefined}
                  className={cn(
                    "flex w-full min-h-11 items-center gap-3 rounded-lg px-3 py-3 text-base font-medium transition-colors",
                    isActive
                      ? "bg-primary/10 text-primary"
                      : "text-foreground hover:bg-muted",
                  )}
                >
                  <Icon className="h-5 w-5" />
                  <span>{tab.label}</span>
                </button>
              );
            })}
          </div>
        </MobileSheet>
      </>
    );
  }
  ```

- [ ] **Step 4: Run the test**

  ```bash
  cd frontend
  npm run test -- --run src/components/shared/mobile-bottom-nav.test.tsx
  ```

  Expected: PASS.

- [ ] **Step 5: Check the shell + nav-related suites** (the bar inventory changed)

  ```bash
  cd frontend
  npm run test -- --run src/App.shell.test.tsx src/components/shared
  ```

  Expected: PASS. If `App.shell.test.tsx` or `navigation-tabs.test.tsx` asserted a Photos/Meals tab in the mobile bar, update those expectations to the new inventory.

- [ ] **Step 6: Commit**

  ```bash
  git add src/components/shared/mobile-bottom-nav.tsx src/components/shared/mobile-bottom-nav.test.tsx
  git commit -m "feat(mobile): collapse bottom nav to five slots with More overflow"
  ```

---

### Task 7: E2E + full verification + PR

**Files:**
- Create: `e2e/mobile-settings-sidebar.spec.ts`

- [ ] **Step 1: Write the E2E spec** — `e2e/mobile-settings-sidebar.spec.ts`:

  ```ts
  import { expect, test } from "@playwright/test";
  import { registerFamily, seedBrowserAuth } from "./helpers/api-helpers";
  import { clearStorage, safeClick, waitForHydration } from "./helpers/test-helpers";

  test.describe("Mobile menu + More overflow", () => {
    test.beforeEach(async ({ page, request, isMobile }) => {
      test.skip(!isMobile, "Mobile-only tests");
      await page.goto("/");
      await clearStorage(page);
      const reg = await registerFamily(request, {
        familyName: "Menu Test Family",
        members: [{ name: "Alice", color: "coral" }],
      });
      await seedBrowserAuth(page, reg);
      await page.reload();
      await waitForHydration(page);
    });

    test("More overflow reaches Meals and Recipes", async ({ page }) => {
      await expect(
        page.getByRole("button", { name: /^meals$/i }),
      ).toHaveCount(0); // not in the bar
      await safeClick(page.getByRole("button", { name: /^more$/i }));
      await expect(
        page.getByRole("dialog", { name: /more/i }),
      ).toBeVisible();
      await safeClick(page.getByRole("button", { name: /^recipes$/i }));
      await expect(page.getByRole("dialog", { name: /more/i })).toHaveCount(0);
    });

    test("menu opens and closes via Escape", async ({ page }) => {
      await safeClick(page.getByRole("button", { name: /^menu$/i }));
      await expect(page.getByText("Menu Test Family")).toBeVisible();
      await page.keyboard.press("Escape");
      await expect(page.getByText("Menu Test Family")).toHaveCount(0);
    });
  });
  ```

  > If `registerFamily`/`seedBrowserAuth` signatures differ from the bottom-nav spec's usage, mirror the exact call shape used in `e2e/mobile-bottom-nav.spec.ts`.

- [ ] **Step 2: Run the E2E spec**

  ```bash
  cd frontend
  npm run test:e2e -- mobile-settings-sidebar.spec.ts
  ```

  Expected: PASS.

- [ ] **Step 3: Full verification gate**

  ```bash
  cd frontend
  npm run lint
  npm run test -- --run
  npm run test:e2e
  ```

  Expected: all green.

- [ ] **Step 4: Open the PR**

  ```bash
  cd frontend
  git push -u origin feat/mobile-sidebar-settings-onboarding
  gh pr create --fill
  ```

  PR body must include `Closes #<N>` for the FE issue and a checklist mapping each spec acceptance criterion to its task/commit.

---

## Self-Review

**Spec coverage:** D1→Task 1; D2→Task 2; D3→Task 3; D4→Task 5 (+ `side-sheet.tsx`); D5→Task 4; D6→Task 6. E2E + verification→Task 7. All six decisions and the acceptance criteria are covered.

**Open verification risks (flagged, not placeholders):**
- Swipe-to-close (Task 5) and keyboard-above-input behavior (Task 2) cannot be asserted in jsdom — both have explicit on-device smoke steps.
- The "Sign Out" accessible-name collision (menu item vs confirm button) has a concrete fallback rename in Task 5, Step 2.
- Task 6, Step 5 calls out updating any pre-existing shell/nav test that hard-codes the old 7-tab inventory.

**Type consistency:** `ModuleType | null` ids, `setActiveModule`, `useAppStore`, `MobileSheet`'s `{ isOpen, onClose, title, children }` contract, and `SideSheet`'s `{ open, onOpenChange, title, children }` contract are used consistently across tasks.
</content>
