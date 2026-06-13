# Polished PWA Installability + Honest Offline UX — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make Family Hub a polished, honest installable PWA — controlled update prompt, sidebar install affordance, accessible offline banner, and a cleaned-up/de-leaked PWA config — without adding any offline *data* caching.

**Architecture:** Three independent FE PRs. (c) config-only cleanups in `vite.config.ts`/`index.html`/`lighthouserc.cjs`/`package.json`; (b) `registerType: "prompt"` + a `PWAUpdater` (sticky update toast via `useRegisterSW`) + a `useOnlineStatus` hook and `OfflineBanner`; (a) a `useInstallPrompt` hook + standalone/iOS detection + a 3-state sidebar "Install app" row with a manual-instructions sheet. Reuses existing toast, MobileSheet, and sidebar primitives only.

**Tech Stack:** React 19 (React Compiler), TypeScript, Vite 7, `vite-plugin-pwa` ^1.3.0 (Workbox), Vitest + Testing Library (jsdom), Playwright, Biome.

**Spec:** `docs/superpowers/specs/2026-06-13-pwa-installability-design.md`
**Story:** `docs/product/backlog/mobile-ux/pwa-installability.md`

---

All paths are relative to `frontend/`. Use **regular merge commits** (release-please) and conventional commit messages. Execute each Part on its own branch off the latest `frontend/main`. Recommended merge order: **Part 1 (c) → Part 2 (b) → Part 3 (a)** — the three touch mostly disjoint files/hunks, but (b) and (c) both touch `vite.config.ts` (different lines), so landing (c) first avoids a trivial rebase.

> **TDD note:** the hooks/components below are unit-tested test-first. Pure config tasks (Part 1) are not unit-testable; each is verified by an explicit build/grep/lighthouse command with expected output instead of a fake test.

---

## Part 1 — Issue (c): PWA config cleanups

**Branch:** `chore/pwa-config-cleanup`
**Commit-type note:** `chore`/`build`/`docs` ⇒ no release bump (intended; this PR ships no user-facing behavior).

### Task C1: Keep `stats.html` out of the production build

**Files:**
- Modify: `vite.config.ts` (plugin array `:13-88`, workbox `:52-80`)
- Modify: `package.json:20` (`analyze` script)
- Modify: `.gitignore`

- [ ] **Step 1 — Gate the visualizer behind `ANALYZE` and move it out of `dist/`.** In `vite.config.ts`, delete the unconditional `visualizer({ filename: "dist/stats.html", … })` entry (`:82-87`) and instead spread it conditionally at the end of the `plugins` array:

  ```ts
  plugins: [
    react({ babel: { plugins: ["babel-plugin-react-compiler"] } }),
    VitePWA({ /* unchanged */ }),
    ...(process.env.ANALYZE === "true"
      ? [
          visualizer({
            filename: ".analyze/stats.html",
            open: false,
            gzipSize: true,
            brotliSize: true,
          }),
        ]
      : []),
  ],
  ```

- [ ] **Step 2 — Belt-and-suspenders in Workbox.** In the `workbox` block, add `globIgnores`:

  ```ts
  workbox: {
    globPatterns: ["**/*.{js,css,html,ico,png,svg,woff,woff2}"],
    globIgnores: ["**/stats.html"],
    navigateFallback: "/index.html",
    navigateFallbackDenylist: [/^\/api/],
    // runtimeCaching removed in Task C2
  },
  ```

- [ ] **Step 3 — Update the `analyze` script** (`package.json:20`):

  ```json
  "analyze": "ANALYZE=true npm run build && echo 'Bundle analysis at .analyze/stats.html'",
  ```

- [ ] **Step 4 — Ignore the analyze output.** Append to `.gitignore`:

  ```
  .analyze/
  ```

- [ ] **Step 5 — Verify a normal build is clean.**

  ```bash
  npm run build
  test ! -e dist/stats.html && echo "OK: no dist/stats.html"
  grep -c "stats.html" dist/sw.js || echo "OK: stats.html not in precache"
  ```

  Expected: "OK: no dist/stats.html" and the precache grep returns 0 / "not in precache".

- [ ] **Step 6 — Verify the analyze path still works and stays out of dist/.**

  ```bash
  npm run analyze
  test -e .analyze/stats.html && test ! -e dist/stats.html && echo "OK: stats in .analyze only"
  ```

- [ ] **Step 7 — Commit.**

  ```bash
  git add vite.config.ts package.json .gitignore
  git commit -m "build(pwa): keep bundle stats.html out of the deployed dist/ and precache"
  ```

### Task C2: Remove dead Google-Fonts CDN runtime-cache rules

**Files:** Modify `vite.config.ts` (workbox `runtimeCaching` `:56-79`)

- [ ] **Step 1 — Confirm nothing references the CDN.**

  ```bash
  grep -rn "fonts.googleapis\|gstatic" index.html src
  ```

  Expected: **no matches** (Nunito is `@fontsource/nunito`, imported in `src/main.tsx`, precached via the `woff2` glob). If there ARE matches, stop and reassess — the rules are not dead.

- [ ] **Step 2 — Delete both `runtimeCaching` rules.** Remove the entire `runtimeCaching: [ … ]` array from the `workbox` block (the two `fonts.googleapis` + `fonts.gstatic` CacheFirst entries). After this the `workbox` block is `globPatterns` + `globIgnores` + `navigateFallback` + `navigateFallbackDenylist` only.

- [ ] **Step 3 — Verify fonts still precache and build is green.**

  ```bash
  npm run build
  grep -c "nunito" dist/sw.js && echo "OK: nunito fonts precached"
  ```

  Expected: build succeeds; precache references the self-hosted Nunito `woff2` files.

- [ ] **Step 4 — Commit.**

  ```bash
  git add vite.config.ts
  git commit -m "build(pwa): drop dead Google Fonts CDN runtime cache rules (Nunito is self-hosted)"
  ```

### Task C3: Remove the dead Lighthouse PWA category

**Files:** Modify `lighthouserc.cjs` (`:9-15`, `:35`)

- [ ] **Step 1 — Remove `"pwa"` from `onlyCategories`** (`:14`) so the array is `["performance", "accessibility", "best-practices", "seo"]`.

- [ ] **Step 2 — Delete the `"categories:pwa"` assertion** line (`:35`).

- [ ] **Step 3 — Verify Lighthouse runs without erroring on the removed category.**

  ```bash
  npm run lighthouse
  ```

  Expected: lhci builds, collects, and asserts the four remaining categories with no "Unknown category pwa" error and existing warn-only thresholds reported (warnings allowed; the run must not error out).

- [ ] **Step 4 — Commit.**

  ```bash
  git add lighthouserc.cjs
  git commit -m "ci(lighthouse): drop the removed PWA category (Lighthouse 12 no longer defines it)"
  ```

### Task C4: Reconcile description, add modern meta, unlock orientation

**Files:** Modify `vite.config.ts` (`:25`, `:30`), `index.html` (`:7`, after `:16`)

- [ ] **Step 1 — Manifest description** (`vite.config.ts:25`):

  ```ts
  description: "Your family's command center for calendar, lists, chores, and meals.",
  ```

- [ ] **Step 2 — Manifest orientation** (`vite.config.ts:30`):

  ```ts
  orientation: "any",
  ```

- [ ] **Step 3 — Meta description** (`index.html:7`) — match the manifest exactly:

  ```html
  <meta name="description" content="Your family's command center for calendar, lists, chores, and meals." />
  ```

- [ ] **Step 4 — Add the modern capability meta** in `index.html` directly after the existing `apple-mobile-web-app-capable` line (`:16`), keeping the apple one for older iOS:

  ```html
  <meta name="mobile-web-app-capable" content="yes" />
  ```

- [ ] **Step 5 — Verify.**

  ```bash
  npm run build
  grep -c "command center for calendar" dist/index.html && echo "OK: meta reconciled"
  grep -c '"orientation":"any"' dist/manifest.webmanifest && echo "OK: orientation any"
  grep -c 'name="mobile-web-app-capable"' dist/index.html && echo "OK: modern meta present"
  ```

  Expected: all three "OK" lines.

- [ ] **Step 6 — Commit.**

  ```bash
  git add vite.config.ts index.html
  git commit -m "feat(pwa): reconcile app description, add mobile-web-app-capable, unlock orientation"
  ```

  > Note: this is the one `feat` in Part 1 (manifest is user-visible at install). If you prefer Part 1 to ship no release bump, use `build(pwa): …` instead — product call; either is acceptable.

### Task C5: Record the cleanups in TECHNICAL-DEBT.md, open PR

**Files:** Modify `frontend/docs/TECHNICAL-DEBT.md` (PWA item §5 `:90-107`; Completed table `:186`)

- [ ] **Step 1 — Update PWA item §5** to reflect what Option B did and what remains (Option C). Replace the "Optimization Opportunities"/"Lighthouse PWA Checklist" bullets so they read: installability + controlled updates + honest offline UX shipped (link the story); **offline data/API caching and background sync remain deferred to Option C / offline reads**. Mark the stats.html-leak, dead-font-rules, and Lighthouse-PWA-category items as resolved here, dated 2026-06-13.

- [ ] **Step 2 — Add a Completed-items row** to the table (`:186`):

  ```
  | PWA config cleanup (stats.html leak, dead font cache rules, Lighthouse PWA category) | - | (c) | Jun 13, 2026 |
  ```

- [ ] **Step 3 — Full gate + commit.**

  ```bash
  npm run lint
  npm run test -- --run
  git add docs/TECHNICAL-DEBT.md
  git commit -m "docs(pwa): record config cleanups; scope offline-data to Option C"
  ```

- [ ] **Step 4 — Push + PR.**

  ```bash
  git push -u origin chore/pwa-config-cleanup
  gh pr create --repo joe-bor/FamilyHub --fill
  ```

  PR body: `Closes #<c>`, Story/Spec/Plan links, and a checklist mapping each acceptance item (no dist/stats.html; no CDN font rules; lighthouse runs; description match; modern meta; orientation any) to its commit. Note the description-wording choice (PD2).

---

## Part 2 — Issue (b): Controlled update prompt + offline banner

**Branch:** `feat/pwa-update-and-offline-banner`

### Task B1: Extend the toast with an optional sticky `duration`

**Files:**
- Modify: `src/components/ui/toaster.tsx` (`toast()` `:40-72`)
- Test: `src/components/ui/toaster.test.tsx` (create)

- [ ] **Step 1 — Write the failing test.** Create `src/components/ui/toaster.test.tsx`:

  ```tsx
  import { act, render, screen } from "@/test/test-utils";
  import { Toaster, toast } from "./toaster";

  describe("toast duration", () => {
    it("auto-dismisses with the default duration", () => {
      vi.useFakeTimers();
      render(<Toaster />);
      act(() => {
        toast({ title: "Bye soon" });
      });
      expect(screen.getByText("Bye soon")).toBeInTheDocument();
      act(() => {
        vi.advanceTimersByTime(5000);
      });
      // open flips to false → Radix begins close; assert it is no longer the live title
      expect(screen.queryByText("Bye soon")).not.toBeInTheDocument();
    });

    it("stays mounted when duration is Infinity", () => {
      vi.useFakeTimers();
      render(<Toaster />);
      act(() => {
        toast({ title: "Sticky", duration: Number.POSITIVE_INFINITY });
      });
      act(() => {
        vi.advanceTimersByTime(60000);
      });
      expect(screen.getByText("Sticky")).toBeInTheDocument();
    });
  });
  ```

- [ ] **Step 2 — Run, expect FAIL** (`duration` not accepted; sticky test fails because everything auto-dismisses):

  ```bash
  npm run test -- --run src/components/ui/toaster.test.tsx
  ```

- [ ] **Step 3 — Implement.** In `toaster.tsx`, add `duration` to the `toast()` parameter and guard the timeout:

  ```ts
  export function toast({
    title,
    description,
    action,
    variant,
    duration = TOAST_REMOVE_DELAY,
  }: Omit<ToastData, "id" | "open"> & { duration?: number }) {
    const id = genId();
    const newToast: ToastData = { id, title, description, action, variant, open: true };
    dispatch([newToast, ...memoryToasts.filter((t) => t.open)].slice(0, TOAST_LIMIT));

    if (Number.isFinite(duration) && duration > 0) {
      setTimeout(() => {
        dispatch(memoryToasts.map((t) => (t.id === id ? { ...t, open: false } : t)));
      }, duration);
    }
    return id;
  }
  ```

  (Leave `ToastData` itself unchanged — `duration` is a call-time concern, not stored.)

- [ ] **Step 4 — Run, expect PASS.**

  ```bash
  npm run test -- --run src/components/ui/toaster.test.tsx
  ```

- [ ] **Step 5 — Commit.**

  ```bash
  git add src/components/ui/toaster.tsx src/components/ui/toaster.test.tsx
  git commit -m "feat(ui): support optional sticky duration on toast"
  ```

### Task B2: Switch to prompted updates + virtual-module types + test alias

**Files:**
- Modify: `vite.config.ts:20` (`registerType`)
- Modify: `tsconfig.app.json:8` (`types`)
- Modify: `vitest.config.ts` (`alias` block `:14`)
- Create: `src/test/mocks/virtual-pwa-register.ts`

- [ ] **Step 1 — `registerType`** (`vite.config.ts:20`): `"autoUpdate"` → `"prompt"`.

- [ ] **Step 2 — Types** (`tsconfig.app.json:8`):

  ```json
  "types": ["vite/client", "vite-plugin-pwa/react"],
  ```

- [ ] **Step 3 — Make the virtual module resolvable in Vitest.** Create `src/test/mocks/virtual-pwa-register.ts`:

  ```ts
  // Test stub for vite-plugin-pwa's virtual module (the PWA plugin isn't loaded
  // in vitest). Individual tests override behavior with vi.mock(...).
  export function useRegisterSW(_opts?: {
    onNeedRefresh?: () => void;
    onOfflineReady?: () => void;
    onRegisteredSW?: (url: string, r?: ServiceWorkerRegistration) => void;
    onRegisterError?: (err: unknown) => void;
  }) {
    return {
      needRefresh: [false, () => {}] as const,
      offlineReady: [false, () => {}] as const,
      updateServiceWorker: async (_reload?: boolean) => {},
    };
  }
  ```

  In `vitest.config.ts`, add to the `alias` block (`:14`):

  ```ts
  "virtual:pwa-register/react": new URL(
    "./src/test/mocks/virtual-pwa-register.ts",
    import.meta.url,
  ).pathname,
  ```

  (Match the existing alias syntax in that file — if it uses `path.resolve(__dirname, …)`, use that form instead.)

- [ ] **Step 4 — Verify build + typecheck still pass** (the `PWAUpdater` consumer lands in B3; this step only confirms config is valid):

  ```bash
  npm run build
  ```

  Expected: build succeeds; `dist/sw.js` exists and no longer self-activates (registerType prompt). No commit yet — bundle with B3 so updates aren't stranded.

### Task B3: `PWAUpdater` — sticky update toast via `useRegisterSW`

**Files:**
- Create: `src/components/pwa/pwa-updater.tsx`
- Create: `src/components/pwa/pwa-updater.test.tsx`
- Modify: `src/main.tsx` (mount once at root)

- [ ] **Step 1 — Write the failing test.** Create `src/components/pwa/pwa-updater.test.tsx`:

  ```tsx
  import { act, render, screen } from "@/test/test-utils";
  import userEvent from "@testing-library/user-event";
  import { Toaster } from "@/components/ui/toaster";

  const updateServiceWorker = vi.fn();
  let captured: { onNeedRefresh?: () => void; onOfflineReady?: () => void } = {};

  vi.mock("virtual:pwa-register/react", () => ({
    useRegisterSW: (opts: { onNeedRefresh?: () => void; onOfflineReady?: () => void }) => {
      captured = opts;
      return { updateServiceWorker };
    },
  }));

  import { PWAUpdater } from "./pwa-updater";

  describe("PWAUpdater", () => {
    it("shows an update toast with a Reload action that updates the SW", async () => {
      const user = userEvent.setup();
      render(
        <>
          <PWAUpdater />
          <Toaster />
        </>,
      );
      // simulate the plugin signaling a waiting SW
      act(() => {
        captured.onNeedRefresh?.();
      });
      expect(screen.getByText("Update available")).toBeInTheDocument();
      await user.click(screen.getByRole("button", { name: /reload/i }));
      expect(updateServiceWorker).toHaveBeenCalledWith(true);
    });
  });
  ```

- [ ] **Step 2 — Run, expect FAIL** (module not found):

  ```bash
  npm run test -- --run src/components/pwa/pwa-updater.test.tsx
  ```

- [ ] **Step 3 — Implement** `src/components/pwa/pwa-updater.tsx`:

  ```tsx
  import { useRegisterSW } from "virtual:pwa-register/react";
  import { ToastAction } from "@/components/ui/toast";
  import { toast } from "@/components/ui/toaster";

  /**
   * Surfaces a user-controlled "update available" prompt instead of the old
   * autoUpdate silent reload. Renders nothing; mounted once near the root.
   */
  export function PWAUpdater() {
    useRegisterSW({
      onNeedRefresh() {
        toast({
          title: "Update available",
          description: "A new version of Family Hub is ready.",
          duration: Number.POSITIVE_INFINITY,
          action: (
            <ToastAction
              altText="Reload to update"
              onClick={() => updateServiceWorker(true)}
            >
              Reload
            </ToastAction>
          ),
        });
      },
      onOfflineReady() {
        toast({
          title: "Ready to use",
          description: "Family Hub is installed and the app shell is cached.",
        });
      },
    });

    // updateServiceWorker is captured from the hook return; hoist it so the
    // action closure can call it.
    const { updateServiceWorker } = useRegisterSW();
    return null;
  }
  ```

  > **Correctness fix:** calling `useRegisterSW` twice is wrong. Implement with a single call and a ref so the toast action can reach the latest `updateServiceWorker`:

  ```tsx
  import { useRef } from "react";
  import { useRegisterSW } from "virtual:pwa-register/react";
  import { ToastAction } from "@/components/ui/toast";
  import { toast } from "@/components/ui/toaster";

  export function PWAUpdater() {
    const updateRef = useRef<(reload?: boolean) => void>(() => {});
    const { updateServiceWorker } = useRegisterSW({
      onNeedRefresh() {
        toast({
          title: "Update available",
          description: "A new version of Family Hub is ready.",
          duration: Number.POSITIVE_INFINITY,
          action: (
            <ToastAction altText="Reload to update" onClick={() => updateRef.current(true)}>
              Reload
            </ToastAction>
          ),
        });
      },
      onOfflineReady() {
        toast({
          title: "Ready to use",
          description: "Family Hub is installed and the app shell is cached.",
        });
      },
    });
    updateRef.current = updateServiceWorker;
    return null;
  }
  ```

- [ ] **Step 4 — Run, expect PASS.**

  ```bash
  npm run test -- --run src/components/pwa/pwa-updater.test.tsx
  ```

- [ ] **Step 5 — Mount once at root.** In `src/main.tsx`, render `<PWAUpdater />` as a sibling of `<App />` inside `<QueryProvider>`:

  ```tsx
  import { PWAUpdater } from "@/components/pwa/pwa-updater";
  // …
  <QueryProvider>
    <App />
    <PWAUpdater />
  </QueryProvider>
  ```

  (`toast()` dispatches to the module-global store, so the `<Toaster />` already mounted inside `App` renders the prompt regardless of which auth branch is active.)

- [ ] **Step 6 — Commit B2 + B3 together** (registerType + updater ship atomically):

  ```bash
  npm run lint && npm run test -- --run src/components/pwa src/components/ui/toaster.test.tsx
  git add vite.config.ts tsconfig.app.json vitest.config.ts src/test/mocks/virtual-pwa-register.ts src/components/pwa src/main.tsx
  git commit -m "feat(pwa): prompt before applying service worker updates instead of silent reload"
  ```

### Task B4: `useOnlineStatus` hook

**Files:**
- Create: `src/hooks/use-online-status.ts`
- Create: `src/hooks/use-online-status.test.ts`
- Modify: `src/hooks/index.ts`

- [ ] **Step 1 — Write the failing test.** Create `src/hooks/use-online-status.test.ts`:

  ```ts
  import { act, renderHook } from "@/test/test-utils";
  import { useOnlineStatus } from "./use-online-status";

  function setOnLine(value: boolean) {
    Object.defineProperty(navigator, "onLine", { configurable: true, value });
  }

  describe("useOnlineStatus", () => {
    it("initializes from navigator.onLine", () => {
      setOnLine(false);
      const { result } = renderHook(() => useOnlineStatus());
      expect(result.current).toBe(false);
    });

    it("reacts to online/offline events", () => {
      setOnLine(true);
      const { result } = renderHook(() => useOnlineStatus());
      expect(result.current).toBe(true);
      act(() => {
        setOnLine(false);
        window.dispatchEvent(new Event("offline"));
      });
      expect(result.current).toBe(false);
      act(() => {
        setOnLine(true);
        window.dispatchEvent(new Event("online"));
      });
      expect(result.current).toBe(true);
    });
  });
  ```

- [ ] **Step 2 — Run, expect FAIL.**

  ```bash
  npm run test -- --run src/hooks/use-online-status.test.ts
  ```

- [ ] **Step 3 — Implement** `src/hooks/use-online-status.ts`:

  ```ts
  import { useEffect, useState } from "react";

  /** Tracks browser connectivity via navigator.onLine + online/offline events. */
  export function useOnlineStatus(): boolean {
    const [online, setOnline] = useState(() => navigator.onLine);
    useEffect(() => {
      const handleOnline = () => setOnline(true);
      const handleOffline = () => setOnline(false);
      window.addEventListener("online", handleOnline);
      window.addEventListener("offline", handleOffline);
      return () => {
        window.removeEventListener("online", handleOnline);
        window.removeEventListener("offline", handleOffline);
      };
    }, []);
    return online;
  }
  ```

- [ ] **Step 4 — Export** from `src/hooks/index.ts`:

  ```ts
  export { useOnlineStatus } from "./use-online-status";
  ```

- [ ] **Step 5 — Run, expect PASS, then commit.**

  ```bash
  npm run test -- --run src/hooks/use-online-status.test.ts
  git add src/hooks/use-online-status.ts src/hooks/use-online-status.test.ts src/hooks/index.ts
  git commit -m "feat(hooks): add useOnlineStatus"
  ```

### Task B5: `OfflineBanner`

**Files:**
- Create: `src/components/shared/offline-banner.tsx`
- Create: `src/components/shared/offline-banner.test.tsx`
- Modify: `src/components/shared/index.ts` (barrel — confirm path; the shared barrel exports `AppHeader`, `SidebarMenu`, etc.)

- [ ] **Step 1 — Write the failing test.** Create `src/components/shared/offline-banner.test.tsx`:

  ```tsx
  import { render, screen } from "@/test/test-utils";
  import { OfflineBanner } from "./offline-banner";

  let online = true;
  vi.mock("@/hooks", async (importOriginal) => {
    const actual = await importOriginal<typeof import("@/hooks")>();
    return { ...actual, useOnlineStatus: () => online };
  });

  describe("OfflineBanner", () => {
    it("renders nothing when online", () => {
      online = true;
      const { container } = render(<OfflineBanner />);
      expect(container).toBeEmptyDOMElement();
    });

    it("shows an accessible status when offline", () => {
      online = false;
      render(<OfflineBanner />);
      const status = screen.getByRole("status");
      expect(status).toHaveTextContent(/offline/i);
      expect(status).toHaveTextContent(/won't save/i);
      expect(status).toHaveAttribute("aria-live", "polite");
    });
  });
  ```

- [ ] **Step 2 — Run, expect FAIL.**

  ```bash
  npm run test -- --run src/components/shared/offline-banner.test.tsx
  ```

- [ ] **Step 3 — Implement** `src/components/shared/offline-banner.tsx`:

  ```tsx
  import { WifiOff } from "lucide-react";
  import { useOnlineStatus } from "@/hooks";

  /**
   * Slim, non-blocking, honest offline indicator. Explains TanStack Query's
   * default networkMode pause (queries/mutations wait while offline). It does
   * NOT imply offline data — that is Option C.
   */
  export function OfflineBanner() {
    const online = useOnlineStatus();
    if (online) return null;
    return (
      <div
        role="status"
        aria-live="polite"
        className="flex items-center justify-center gap-2 border-t border-amber-500/30 bg-amber-500/15 px-3 py-1.5 text-xs font-medium text-amber-900"
      >
        <WifiOff className="h-3.5 w-3.5" aria-hidden="true" />
        You're offline — changes won't save until you reconnect.
      </div>
    );
  }
  ```

- [ ] **Step 4 — Export** from the shared barrel `src/components/shared/index.ts`:

  ```ts
  export { OfflineBanner } from "./offline-banner";
  ```

- [ ] **Step 5 — Run, expect PASS, then commit.**

  ```bash
  npm run test -- --run src/components/shared/offline-banner.test.tsx && npm run lint
  git add src/components/shared/offline-banner.tsx src/components/shared/offline-banner.test.tsx src/components/shared/index.ts
  git commit -m "feat(shared): add accessible offline banner"
  ```

### Task B6: Mount the banner + offline E2E + PR

**Files:**
- Modify: `src/App.tsx` (authenticated shell return; import from `@/components/shared`)
- Create: `e2e/pwa-offline-banner.spec.ts`

- [ ] **Step 1 — Mount the banner above the bottom nav.** In `App.tsx`, add `OfflineBanner` to the `@/components/shared` import and render it directly before the `MobileBottomNav` line in the authenticated return:

  ```tsx
  <OfflineBanner />
  {isMobile && isAuthenticated && setupComplete && <MobileBottomNav />}
  ```

- [ ] **Step 2 — Write the E2E** `e2e/pwa-offline-banner.spec.ts` (run on the mobile project; seed auth per `e2e/helpers/api-helpers.ts`):

  ```ts
  import { expect, test } from "@playwright/test";
  import { registerFamily, seedBrowserAuth, waitForHydration } from "./helpers/api-helpers";

  test("offline banner appears offline and clears on reconnect", async ({ page, context }) => {
    const family = await registerFamily();
    await seedBrowserAuth(page, family);
    await page.goto("/");
    await waitForHydration(page);

    await context.setOffline(true);
    await expect(page.getByRole("status").filter({ hasText: /offline/i })).toBeVisible({ timeout: 2000 });

    await context.setOffline(false);
    await expect(page.getByRole("status").filter({ hasText: /offline/i })).toBeHidden({ timeout: 2000 });
  });
  ```

  (Adjust the helper imports/signatures to match `e2e/helpers/api-helpers.ts`. `context.setOffline` fires the same `offline`/`online` events the hook listens to — no production SW build is required for the banner, since the banner does not depend on the SW.)

- [ ] **Step 3 — Full gate.**

  ```bash
  npm run lint
  npm run test -- --run
  npm run test:e2e -- pwa-offline-banner
  ```

  Expected: all green.

- [ ] **Step 4 — Device/preview smoke (required before merge).** SW update behavior only runs on a production build:

  ```bash
  npm run build && npm run preview
  ```

  - Toggle the OS/network offline → banner appears < ~1s, clears on reconnect.
  - In DevTools → Application → Service Workers, with two builds, confirm the **"Update available"** toast appears (no auto-reload) and **Reload** applies the new SW. (Or verify on a real deploy later — never on `npm run dev`.)

- [ ] **Step 5 — Push + PR.**

  ```bash
  git add src/App.tsx e2e/pwa-offline-banner.spec.ts
  git commit -m "feat(pwa): show offline banner above the bottom nav"
  git push -u origin feat/pwa-update-and-offline-banner
  gh pr create --repo joe-bor/FamilyHub --fill
  ```

  PR body: `Closes #<b>`, Story/Spec/Plan links, checklist (update prompt dismissible + working Reload + no silent reload; offline banner <1s + accessible; no networkMode change; no offline data), and the preview/device smoke results.

---

## Part 3 — Issue (a): Install affordance + standalone detection

**Branch:** `feat/pwa-install-row`

### Task A1: `src/lib/pwa.ts` — standalone + iOS detection

**Files:**
- Create: `src/lib/pwa.ts`
- Create: `src/lib/pwa.test.ts`

- [ ] **Step 1 — Write the failing test.** Create `src/lib/pwa.test.ts`:

  ```ts
  import { isIOS, isStandalone } from "./pwa";

  function mockMatchMedia(matches: boolean) {
    vi.mocked(window.matchMedia).mockImplementation((query: string) => ({
      matches,
      media: query,
      onchange: null,
      addListener: vi.fn(),
      removeListener: vi.fn(),
      addEventListener: vi.fn(),
      removeEventListener: vi.fn(),
      dispatchEvent: vi.fn(),
    }));
  }
  function setUA(ua: string, maxTouchPoints = 0) {
    Object.defineProperty(navigator, "userAgent", { configurable: true, value: ua });
    Object.defineProperty(navigator, "maxTouchPoints", { configurable: true, value: maxTouchPoints });
  }

  describe("isStandalone", () => {
    it("true when display-mode standalone matches", () => {
      mockMatchMedia(true);
      expect(isStandalone()).toBe(true);
    });
    it("false in a normal browser tab", () => {
      mockMatchMedia(false);
      Object.defineProperty(navigator, "standalone", { configurable: true, value: undefined });
      expect(isStandalone()).toBe(false);
    });
  });

  describe("isIOS", () => {
    it("detects iPhone UA", () => {
      setUA("Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X)");
      expect(isIOS()).toBe(true);
    });
    it("detects iPadOS reporting as Mac with touch", () => {
      setUA("Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)", 5);
      expect(isIOS()).toBe(true);
    });
    it("is false for desktop Chrome", () => {
      setUA("Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)", 0);
      expect(isIOS()).toBe(false);
    });
  });
  ```

- [ ] **Step 2 — Run, expect FAIL.**

  ```bash
  npm run test -- --run src/lib/pwa.test.ts
  ```

- [ ] **Step 3 — Implement** `src/lib/pwa.ts`:

  ```ts
  /** A new version is available event from Chromium browsers. */
  export interface BeforeInstallPromptEvent extends Event {
    readonly platforms: string[];
    readonly userChoice: Promise<{ outcome: "accepted" | "dismissed"; platform: string }>;
    prompt(): Promise<void>;
  }

  /** True when the app is running as an installed/standalone PWA. */
  export function isStandalone(): boolean {
    if (typeof window === "undefined") return false;
    const mql = window.matchMedia?.("(display-mode: standalone)");
    return (
      mql?.matches === true ||
      (navigator as Navigator & { standalone?: boolean }).standalone === true
    );
  }

  /** True for iOS/iPadOS Safari (which never fires beforeinstallprompt). */
  export function isIOS(): boolean {
    const ua = navigator.userAgent;
    return (
      /iphone|ipad|ipod/i.test(ua) ||
      (/macintosh/i.test(ua) && navigator.maxTouchPoints > 1)
    );
  }
  ```

- [ ] **Step 4 — Run, expect PASS, commit.**

  ```bash
  npm run test -- --run src/lib/pwa.test.ts
  git add src/lib/pwa.ts src/lib/pwa.test.ts
  git commit -m "feat(pwa): add standalone + iOS detection helpers"
  ```

### Task A2: `useInstallPrompt` hook

**Files:**
- Create: `src/hooks/use-install-prompt.ts`
- Create: `src/hooks/use-install-prompt.test.ts`
- Modify: `src/hooks/index.ts`

- [ ] **Step 1 — Write the failing test.** Create `src/hooks/use-install-prompt.test.ts`:

  ```ts
  import { act, renderHook, waitFor } from "@/test/test-utils";
  import type { BeforeInstallPromptEvent } from "@/lib/pwa";
  import { useInstallPrompt } from "./use-install-prompt";

  function makeBIP() {
    const event = new Event("beforeinstallprompt") as BeforeInstallPromptEvent & {
      prompt: ReturnType<typeof vi.fn>;
    };
    (event as { prompt: unknown }).prompt = vi.fn().mockResolvedValue(undefined);
    Object.defineProperty(event, "userChoice", {
      value: Promise.resolve({ outcome: "accepted", platform: "web" }),
    });
    return event;
  }

  describe("useInstallPrompt", () => {
    it("becomes installable after beforeinstallprompt and clears after appinstalled", async () => {
      const { result } = renderHook(() => useInstallPrompt());
      expect(result.current.canInstall).toBe(false);

      const event = makeBIP();
      act(() => {
        window.dispatchEvent(event);
      });
      await waitFor(() => expect(result.current.canInstall).toBe(true));

      act(() => {
        window.dispatchEvent(new Event("appinstalled"));
      });
      await waitFor(() => expect(result.current.canInstall).toBe(false));
    });

    it("promptInstall calls prompt() and consumes the event", async () => {
      const { result } = renderHook(() => useInstallPrompt());
      const event = makeBIP();
      act(() => {
        window.dispatchEvent(event);
      });
      await waitFor(() => expect(result.current.canInstall).toBe(true));

      await act(async () => {
        await result.current.promptInstall();
      });
      expect(event.prompt).toHaveBeenCalledOnce();
      expect(result.current.canInstall).toBe(false);
    });
  });
  ```

- [ ] **Step 2 — Run, expect FAIL.**

  ```bash
  npm run test -- --run src/hooks/use-install-prompt.test.ts
  ```

- [ ] **Step 3 — Implement** `src/hooks/use-install-prompt.ts`:

  ```ts
  import { useEffect, useState } from "react";
  import type { BeforeInstallPromptEvent } from "@/lib/pwa";

  /**
   * Captures the Chromium `beforeinstallprompt` event so the UI can offer a
   * one-tap install. canInstall is false on browsers that never fire it
   * (iOS Safari, Firefox) and after the app is installed.
   */
  export function useInstallPrompt() {
    const [deferred, setDeferred] = useState<BeforeInstallPromptEvent | null>(null);

    useEffect(() => {
      const handleBeforeInstall = (event: Event) => {
        event.preventDefault();
        setDeferred(event as BeforeInstallPromptEvent);
      };
      const handleInstalled = () => setDeferred(null);
      window.addEventListener("beforeinstallprompt", handleBeforeInstall);
      window.addEventListener("appinstalled", handleInstalled);
      return () => {
        window.removeEventListener("beforeinstallprompt", handleBeforeInstall);
        window.removeEventListener("appinstalled", handleInstalled);
      };
    }, []);

    async function promptInstall() {
      if (!deferred) return;
      await deferred.prompt();
      await deferred.userChoice;
      setDeferred(null); // a captured prompt can only be used once
    }

    return { canInstall: deferred !== null, promptInstall };
  }
  ```

- [ ] **Step 4 — Export** from `src/hooks/index.ts`:

  ```ts
  export { useInstallPrompt } from "./use-install-prompt";
  ```

- [ ] **Step 5 — Run, expect PASS, commit.**

  ```bash
  npm run test -- --run src/hooks/use-install-prompt.test.ts
  git add src/hooks/use-install-prompt.ts src/hooks/use-install-prompt.test.ts src/hooks/index.ts
  git commit -m "feat(hooks): add useInstallPrompt"
  ```

### Task A3: `InstallInstructionsSheet`

**Files:**
- Create: `src/components/shared/install-instructions-sheet.tsx`
- Create: `src/components/shared/install-instructions-sheet.test.tsx`

- [ ] **Step 1 — Write the failing test.** Create `src/components/shared/install-instructions-sheet.test.tsx`:

  ```tsx
  import { render, screen } from "@/test/test-utils";
  import { InstallInstructionsSheet } from "./install-instructions-sheet";

  let ios = false;
  vi.mock("@/lib/pwa", async (importOriginal) => {
    const actual = await importOriginal<typeof import("@/lib/pwa")>();
    return { ...actual, isIOS: () => ios };
  });

  describe("InstallInstructionsSheet", () => {
    it("shows iOS Share/Add to Home Screen copy on iOS", () => {
      ios = true;
      render(<InstallInstructionsSheet isOpen onClose={vi.fn()} />);
      expect(screen.getByText(/add to home screen/i)).toBeInTheDocument();
      expect(screen.getByText(/share/i)).toBeInTheDocument();
    });

    it("shows generic browser-menu copy elsewhere", () => {
      ios = false;
      render(<InstallInstructionsSheet isOpen onClose={vi.fn()} />);
      expect(screen.getByText(/browser menu/i)).toBeInTheDocument();
    });
  });
  ```

- [ ] **Step 2 — Run, expect FAIL.**

  ```bash
  npm run test -- --run src/components/shared/install-instructions-sheet.test.tsx
  ```

- [ ] **Step 3 — Implement** `src/components/shared/install-instructions-sheet.tsx`:

  ```tsx
  import { MobileSheet } from "@/components/ui/mobile-sheet";
  import { isIOS } from "@/lib/pwa";

  interface InstallInstructionsSheetProps {
    isOpen: boolean;
    onClose: () => void;
  }

  export function InstallInstructionsSheet({ isOpen, onClose }: InstallInstructionsSheetProps) {
    const ios = isIOS();
    return (
      <MobileSheet isOpen={isOpen} onClose={onClose} title="Install Family Hub" initialHeight="half">
        <div className="space-y-4 px-4 pb-6 text-sm text-foreground">
          {ios ? (
            <ol className="list-decimal space-y-2 pl-5">
              <li>
                Tap the <span className="font-semibold">Share</span> button in Safari's toolbar.
              </li>
              <li>
                Choose <span className="font-semibold">Add to Home Screen</span>.
              </li>
              <li>
                Tap <span className="font-semibold">Add</span> — Family Hub appears on your home screen.
              </li>
            </ol>
          ) : (
            <ol className="list-decimal space-y-2 pl-5">
              <li>Open your browser menu (⋮ or ☰).</li>
              <li>
                Choose <span className="font-semibold">Install</span> or{" "}
                <span className="font-semibold">Add to Home Screen</span>.
              </li>
              <li>Confirm to add Family Hub to your home screen.</li>
            </ol>
          )}
        </div>
      </MobileSheet>
    );
  }
  ```

- [ ] **Step 4 — Run, expect PASS, commit.**

  ```bash
  npm run test -- --run src/components/shared/install-instructions-sheet.test.tsx
  git add src/components/shared/install-instructions-sheet.tsx src/components/shared/install-instructions-sheet.test.tsx
  git commit -m "feat(shared): add manual install-instructions sheet"
  ```

### Task A4: `InstallAppRow` (3-state) + wire into the sidebar

**Files:**
- Create: `src/components/shared/install-app-row.tsx`
- Create: `src/components/shared/install-app-row.test.tsx`
- Modify: `src/components/shared/sidebar-menu.tsx`

> **Design refinement (spec D2):** the row's visibility + action depend on runtime state, so it is realized as a dedicated, isolated component rather than a static `menuItems` entry — easier to test and keeps `sidebar-menu.tsx` thin.

- [ ] **Step 1 — Write the failing test.** Create `src/components/shared/install-app-row.test.tsx`:

  ```tsx
  import { render, screen } from "@/test/test-utils";
  import userEvent from "@testing-library/user-event";
  import { InstallAppRow } from "./install-app-row";

  let standalone = false;
  let canInstall = false;
  const promptInstall = vi.fn();

  vi.mock("@/lib/pwa", async (importOriginal) => {
    const actual = await importOriginal<typeof import("@/lib/pwa")>();
    return { ...actual, isStandalone: () => standalone };
  });
  vi.mock("@/hooks", async (importOriginal) => {
    const actual = await importOriginal<typeof import("@/hooks")>();
    return { ...actual, useInstallPrompt: () => ({ canInstall, promptInstall }) };
  });

  describe("InstallAppRow", () => {
    it("renders nothing when already installed", () => {
      standalone = true;
      const { container } = render(<InstallAppRow />);
      expect(container).toBeEmptyDOMElement();
    });

    it("one-taps install on Chromium", async () => {
      standalone = false;
      canInstall = true;
      const user = userEvent.setup();
      render(<InstallAppRow />);
      await user.click(screen.getByRole("button", { name: /install app/i }));
      expect(promptInstall).toHaveBeenCalledOnce();
    });

    it("opens the instructions sheet when prompt is unavailable", async () => {
      standalone = false;
      canInstall = false;
      const user = userEvent.setup();
      render(<InstallAppRow />);
      await user.click(screen.getByRole("button", { name: /install app/i }));
      expect(await screen.findByText(/install family hub/i)).toBeInTheDocument();
    });
  });
  ```

- [ ] **Step 2 — Run, expect FAIL.**

  ```bash
  npm run test -- --run src/components/shared/install-app-row.test.tsx
  ```

- [ ] **Step 3 — Implement** `src/components/shared/install-app-row.tsx`:

  ```tsx
  import { Download } from "lucide-react";
  import { useState } from "react";
  import { InstallInstructionsSheet } from "@/components/shared/install-instructions-sheet";
  import { useInstallPrompt } from "@/hooks";
  import { isStandalone } from "@/lib/pwa";

  export function InstallAppRow() {
    // display-mode does not change within a session; read once.
    const [installed] = useState(() => isStandalone());
    const { canInstall, promptInstall } = useInstallPrompt();
    const [showInstructions, setShowInstructions] = useState(false);

    if (installed) return null;

    const handleClick = () => {
      if (canInstall) {
        void promptInstall();
      } else {
        setShowInstructions(true);
      }
    };

    return (
      <>
        <button
          type="button"
          onClick={handleClick}
          className="w-full min-h-11 flex items-center gap-3 px-3 py-3 rounded-lg text-sm font-medium transition-colors text-muted-foreground hover:bg-muted hover:text-foreground"
        >
          <Download className="h-5 w-5" />
          Install app
        </button>
        <InstallInstructionsSheet
          isOpen={showInstructions}
          onClose={() => setShowInstructions(false)}
        />
      </>
    );
  }
  ```

- [ ] **Step 4 — Wire into `sidebar-menu.tsx`.** Import `InstallAppRow` and render it immediately above the **Sign Out** row inside the nav `.space-y-1` div. Replace the `menuItems.map(...)` body so the install row precedes Sign Out:

  ```tsx
  import { Fragment } from "react";
  import { InstallAppRow } from "@/components/shared/install-app-row";
  // …inside the <nav> .space-y-1 div:
  {menuItems.map((item) => {
    const Icon = item.icon;
    const button = (
      <button
        type="button"
        onClick={item.action}
        className="w-full min-h-11 flex items-center gap-3 px-3 py-3 rounded-lg text-sm font-medium transition-colors text-muted-foreground hover:bg-muted hover:text-foreground"
      >
        <Icon className="h-5 w-5" />
        {item.label}
      </button>
    );
    return item.label === "Sign Out" ? (
      <Fragment key={item.label}>
        <InstallAppRow />
        {button}
      </Fragment>
    ) : (
      <Fragment key={item.label}>{button}</Fragment>
    );
  })}
  ```

  (Keep the existing `key` semantics; the `button` previously carried `key={item.label}` — move the key to the `Fragment` as shown.)

- [ ] **Step 5 — Run + lint.**

  ```bash
  npm run test -- --run src/components/shared/install-app-row.test.tsx && npm run lint
  ```

- [ ] **Step 6 — Commit.**

  ```bash
  git add src/components/shared/install-app-row.tsx src/components/shared/install-app-row.test.tsx src/components/shared/sidebar-menu.tsx
  git commit -m "feat(pwa): add sidebar Install app row (one-tap install / manual instructions / hidden when installed)"
  ```

### Task A5: Full gate + PR

- [ ] **Step 1 — Full gate.**

  ```bash
  npm run lint
  npm run test -- --run
  npm run test:e2e
  ```

  Expected: all green. (No new E2E required for the install row — `beforeinstallprompt` is not dispatchable in Playwright reliably; the three states are covered by the component test. Optionally add an E2E asserting the row is **absent** in a standalone context if convenient.)

- [ ] **Step 2 — Device smoke (required before merge).**
  - Android Chrome (Galaxy S10): sidebar shows **Install app**; tap → native install dialog; after install the row is gone (relaunch standalone).
  - iOS Safari: tap **Install app** → instructions sheet with Share → Add to Home Screen copy; complete it; relaunched standalone hides the row.
  - Desktop Chrome/Edge: install via the row.

- [ ] **Step 3 — Push + PR.**

  ```bash
  git push -u origin feat/pwa-install-row
  gh pr create --repo joe-bor/FamilyHub --fill
  ```

  PR body: `Closes #<a>`, Story/Spec/Plan links, checklist (one-tap Chromium; instructions elsewhere; hidden when installed), and device-smoke results.

---

## Self-Review

**Spec coverage:**
- D1 (update model) → B2 + B3 (registerType + tsconfig types + PWAUpdater + sticky toast B1).
- D2 (install affordance) → A1 (detection) + A2 (hook) + A3 (instructions) + A4 (row + sidebar).
- D3 (online/offline) → B4 (hook) + B5 (banner) + B6 (mount + E2E).
- D4a stats.html → C1; D4b font rules → C2; D4c lighthouse → C3; D4d description → C4; D4e meta → C4; D4f orientation → C4.
- Toast extension → B1. TECHNICAL-DEBT + PRD/roadmap doc updates → C5 (tech-debt) + root docs (handled separately by the orchestrator).

**Placeholder scan:** none — every code/test step shows full content; config steps show exact lines + verify commands.

**Type/name consistency:** `BeforeInstallPromptEvent` defined in A1, consumed in A2. `useInstallPrompt` returns `{ canInstall, promptInstall }` — used identically in A4 + tests. `useOnlineStatus(): boolean` — B4/B5 agree. `InstallInstructionsSheet({ isOpen, onClose })` matches `MobileSheet`'s API and A4's usage. `toast({ …, duration })` (B1) matches PWAUpdater's call (B3).

**Flagged execution risks (not placeholders):**
- `useRegisterSW` must be called exactly once in `PWAUpdater` — B3 Step 3 shows the corrected single-call + ref pattern (the first sketch's double-call is explicitly wrong; use the second).
- `virtual:pwa-register/react` is not resolvable in vitest without the alias — B2 Step 3 adds it; confirm the existing `vitest.config.ts` alias syntax and match it.
- SW update + precache behaviors do not run on `npm run dev` — exercise via `npm run preview`/build (B6 Step 4, A5 Step 2).
- `context.setOffline` drives the banner E2E without a prod SW build because the banner depends only on `navigator.onLine` events, not the SW (B6 Step 2).
