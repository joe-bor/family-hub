# Polished PWA Installability + Honest Offline UX — Design

> **Story:** `docs/product/backlog/mobile-ux/pwa-installability.md`
> **Scope:** UI + config only. No offline *data* (that is Option C / offline reads).

## Problem

Family Hub ships as a PWA but it is configured, not polished. Four concrete problems:

1. **Surprise reloads.** `registerType: "autoUpdate"` makes the service worker `skipWaiting()` + `clientsClaim()` silently; a deploy can hard-reload the app mid-use and drop an in-progress event form.
2. **No install affordance.** Nothing invites the user to install, even though Android Chrome (the verified Galaxy S10 device) fires `beforeinstallprompt` and a one-tap install is one row of UI. The roadmap flags this as the highest-leverage daily-use win.
3. **Dishonest offline.** When the network drops, TanStack Query (default `networkMode: "online"`) silently pauses queries/mutations and the app looks broken with no explanation.
4. **Stale/leaky config.** A 1.2 MB bundle-stats file is publicly served and precached in prod; two dead Google-Fonts CDN cache rules linger after Nunito self-hosting; the Lighthouse `pwa` category was removed upstream but is still requested/asserted; the manifest and meta descriptions disagree; the manifest hard-locks portrait.

## Current FE state (verified 2026-06-13, paths relative to `frontend/`)

- `vite.config.ts:19-81` — `VitePWA({ registerType: "autoUpdate", manifest{…}, workbox{…} })`. Manifest: `name`/`short_name` "FamilyHub", `description` "Your family's command center", `display` "standalone", `orientation` "portrait" (`:30`), theme/bg `#faf8f5`, icons 192/512/maskable-512. Workbox precaches the shell, `navigateFallback: "/index.html"` with `/api` denylisted, and has two CacheFirst Google-Fonts rules (`:56-79`).
- `vite.config.ts:82-87` — `visualizer({ filename: "dist/stats.html" })` runs on **every** build, unconditionally, writing into `dist/`.
- `index.html:7` — `<meta name="description" content="Family calendar and organizer">`. `:16` has `apple-mobile-web-app-capable`; the standard `mobile-web-app-capable` is **absent**.
- `tsconfig.app.json:8` — `"types": ["vite/client"]` (no `vite-plugin-pwa/react`).
- `lighthouserc.cjs:9-15,35` — `onlyCategories` includes `"pwa"`; assertions include `"categories:pwa": ["warn", { minScore: 0.5 }]`. `@lhci/cli ^0.15.1` bundles Lighthouse 12, which removed the PWA category.
- `package.json:20` — `"analyze": "npm run build && echo '…dist/stats.html'"`.
- **Toast** (`src/components/ui/toaster.tsx`): module-level `toast({ title, description, action, variant })`; `action` is a `ToastActionElement` (`<ToastAction>` from `toast.tsx`). Auto-dismisses at `TOAST_REMOVE_DELAY = 5000` ms; `<Toaster />` is mounted in every branch of `App.tsx`.
- **Sidebar** (`src/components/shared/sidebar-menu.tsx`): a `menuItems` array (Family Settings, Preferences, Sign Out) rendered as `min-h-11` buttons inside a `SideSheet`; manages its own modal/sheet open state.
- **MobileSheet** (`src/components/ui/mobile-sheet.tsx`): `MobileSheet({ isOpen, onClose, title, initialHeight })`, vaul-based, full Radix dialog semantics.
- **App root** (`src/App.tsx`): authenticated shell is `<AppHeader/>` → content → `{isMobile && <MobileBottomNav/>}` → `<SidebarMenu/>`, then `<Toaster/>`. `main.tsx` wraps `<App/>` in `<QueryProvider>`.
- **Query client** (`src/providers/query-provider.tsx`): `staleTime 5m`, `gcTime 30m`, `refetchOnWindowFocus false`, **no `networkMode` set** ⇒ default `"online"`.
- **Hooks barrel** (`src/hooks/index.ts`): re-exports `useDebounce`, `useGoogleAuthReturn`, `useIsMobile`, `useMediaQuery`. New hooks go here.

## Goals

- New SW versions prompt instead of forcing a reload; the user chooses when to reload.
- An install row that does the right thing on each platform and disappears once installed.
- An honest, accessible, non-blocking offline indicator that explains the paused-write behavior.
- A build with no leaked stats file, no dead cache rules, a working Lighthouse run, consistent metadata, and an unlocked orientation.

## Non-goals

- No API/response caching, Query persistence, IndexedDB, or offline data reads (Option C).
- No offline write queue, Background Sync, Web Push, or notifications.
- No change to TanStack Query `networkMode`.
- No change to manifest `name`/`short_name`, `<title>`, icons, or theme colors.
- No router or architectural change; the install/instructions UI reuses existing primitives only.

---

## Decision Summary

### D1. Update model: `registerType: "prompt"` + a `PWAUpdater` (B1)

- `vite.config.ts:20`: `registerType: "autoUpdate"` → `"prompt"`. The waiting SW no longer activates itself.
- `tsconfig.app.json:8`: `"types": ["vite/client", "vite-plugin-pwa/react"]` so `virtual:pwa-register/react` types resolve.
- New `src/components/pwa/pwa-updater.tsx`: a render-light component using `useRegisterSW()` from `virtual:pwa-register/react`.
  - `onNeedRefresh` → show a **sticky** toast (see [Toast extension](#toast-extension-sticky-duration)): title "Update available", description "A new version of Family Hub is ready.", `action` = a `<ToastAction>` "Reload" that calls `updateServiceWorker(true)`.
  - `onOfflineReady` → optional one-time default-duration toast "Ready to work offline" (honest: this only means the *app shell* is cached — it does **not** imply data offline; copy must not over-promise).
  - Renders `null`; its only job is the effect + toast.
- **Mount:** render `<PWAUpdater />` once at the top of `App.tsx`'s returned tree (it must coexist with a mounted `<Toaster />`; mounting it inside the authenticated shell next to `<Toaster/>` is sufficient — updates surface while the app is in use).
- **Why this ships atomically with the `registerType` change:** with `"prompt"` and no updater, a new SW would wait forever and updates would silently never apply. The `registerType` flip and `PWAUpdater` therefore live in the **same PR** (group b).

```tsx
// src/components/pwa/pwa-updater.tsx (sketch)
import { useRegisterSW } from "virtual:pwa-register/react";
import { ToastAction } from "@/components/ui/toast";
import { toast } from "@/components/ui/toaster";

export function PWAUpdater() {
  const { updateServiceWorker } = useRegisterSW({
    onNeedRefresh() {
      toast({
        title: "Update available",
        description: "A new version of Family Hub is ready.",
        duration: Number.POSITIVE_INFINITY, // sticky until Reload/dismiss
        action: (
          <ToastAction altText="Reload to update" onClick={() => updateServiceWorker(true)}>
            Reload
          </ToastAction>
        ),
      });
    },
  });
  return null;
}
```

### D2. Install affordance: `use-install-prompt` + standalone detection + sidebar row (B2)

**Standalone/platform detection** — `src/lib/pwa.ts` (pure, testable helpers, no React):

```ts
export function isStandalone(): boolean {
  return (
    window.matchMedia?.("(display-mode: standalone)").matches === true ||
    (navigator as Navigator & { standalone?: boolean }).standalone === true
  );
}
export function isIOS(): boolean {
  // iPadOS 13+ reports as Mac; include the touch check.
  const ua = navigator.userAgent;
  return /iphone|ipad|ipod/i.test(ua) ||
    (/macintosh/i.test(ua) && navigator.maxTouchPoints > 1);
}
```

**Hook** — `src/hooks/use-install-prompt.ts`:

```ts
// Returns { canInstall, promptInstall }. Only Chromium fires beforeinstallprompt.
export function useInstallPrompt() {
  const [deferred, setDeferred] = useState<BeforeInstallPromptEvent | null>(null);
  useEffect(() => {
    const onBIP = (e: Event) => { e.preventDefault(); setDeferred(e as BeforeInstallPromptEvent); };
    const onInstalled = () => setDeferred(null);
    window.addEventListener("beforeinstallprompt", onBIP);
    window.addEventListener("appinstalled", onInstalled);
    return () => {
      window.removeEventListener("beforeinstallprompt", onBIP);
      window.removeEventListener("appinstalled", onInstalled);
    };
  }, []);
  const promptInstall = useCallback(async () => {
    if (!deferred) return;
    await deferred.prompt();
    await deferred.userChoice;     // resolve regardless of accept/dismiss
    setDeferred(null);             // a prompt can only be used once
  }, [deferred]);
  return { canInstall: deferred !== null, promptInstall };
}
```

`BeforeInstallPromptEvent` is not in the DOM lib; declare a minimal local type in the hook file (or `src/lib/pwa.ts`).

**Sidebar row** — `src/components/shared/sidebar-menu.tsx`. Add an **"Install app"** row (lucide `Download` icon, `min-h-11` to match) with three states, computed once per render:

| Condition | Row behavior |
|---|---|
| `isStandalone()` | **Not rendered** (already installed). |
| `canInstall` (Chromium) | Tap → `promptInstall()`. |
| otherwise (iOS Safari, Firefox, etc.) | Tap → open an install-instructions `MobileSheet`. |

Placement: above **Sign Out** (the install action belongs with account/app actions, not destructive ones). Implement as a conditional entry — not a static `menuItems` member — because visibility and action both depend on runtime state. `isStandalone()` is read once at mount via `useState(() => isStandalone())` (it does not change within a session).

**Instructions sheet** — `src/components/shared/install-instructions-sheet.tsx`, a `MobileSheet` (`initialHeight="half"`) whose copy is chosen by platform:

- **iOS Safari:** "Tap the Share button, then **Add to Home Screen**." (with the share-glyph hint)
- **Other (Firefox / unsupported):** "Open your browser menu, then **Install** / **Add to Home Screen**." (generic)

Copy is intentionally minimal and refinable; the sheet is the manual fallback for browsers that never fire `beforeinstallprompt`.

### D3. Online/offline: `use-online-status` + `OfflineBanner` (B3)

**Hook** — `src/hooks/use-online-status.ts`:

```ts
export function useOnlineStatus(): boolean {
  const [online, setOnline] = useState(() => navigator.onLine);
  useEffect(() => {
    const on = () => setOnline(true);
    const off = () => setOnline(false);
    window.addEventListener("online", on);
    window.addEventListener("offline", off);
    return () => { window.removeEventListener("online", on); window.removeEventListener("offline", off); };
  }, []);
  return online;
}
```

**Banner** — `src/components/shared/offline-banner.tsx`: renders `null` when online; when offline, a slim, full-width, non-blocking bar (lucide `WifiOff` + text), `role="status"`, `aria-live="polite"`. Copy:

> **You're offline — changes won't save until you reconnect.**

This is the honest explanation of the default `networkMode: "online"` pause behavior. **Placement:** mounted in `App.tsx`'s authenticated shell as a flex child directly **above** `<MobileBottomNav/>` (so it sits above the bottom nav on mobile, and at the column bottom on desktop). It appears within ~1s because `offline`/`online` events fire synchronously on connectivity change. It must not overlay or block content (slim bar, part of the flex column, not a fixed overlay over the nav).

### D4. Config cleanups (B4)

**D4a — stats.html out of production.** Gate the visualizer behind an env flag and move its output out of `dist/`:

```ts
// vite.config.ts
plugins: [
  react({…}),
  VitePWA({…}),
  ...(process.env.ANALYZE === "true"
    ? [visualizer({ filename: ".analyze/stats.html", open: false, gzipSize: true, brotliSize: true })]
    : []),
],
```

Add belt-and-suspenders to workbox: `globIgnores: ["**/stats.html"]`. Update `package.json:20`:

```json
"analyze": "ANALYZE=true npm run build && echo 'Bundle analysis at .analyze/stats.html'",
```

Add `.analyze/` to `.gitignore`. **Acceptance:** a normal `npm run build` yields a `dist/` with no `stats.html`, and `dist/sw.js`'s precache list does not reference it.

**D4b — delete dead font rules.** Remove both Google-Fonts CDN `runtimeCaching` rules (`vite.config.ts:56-79`). Verify first that nothing references the CDN: grep `index.html` + `src/` for `fonts.googleapis` / `gstatic` (expected: zero hits; Nunito is `@fontsource/nunito`, imported in `main.tsx` and precached via the `woff2` glob). After removal `runtimeCaching` is empty and can be dropped entirely.

**D4c — Lighthouse PWA category.** Remove `"pwa"` from `onlyCategories` (`lighthouserc.cjs:14`) and delete the `"categories:pwa"` assertion (`:35`). The other four categories and warn-only thresholds stay.

**D4d — reconcile description (PD2).** Set both to one string:

> "Your family's command center for calendar, lists, chores, and meals."

- `vite.config.ts:25` manifest `description`
- `index.html:7` meta `description`

**D4e — modern meta.** Add `<meta name="mobile-web-app-capable" content="yes">` to `index.html` alongside the existing `apple-mobile-web-app-capable` (keep the apple one for older iOS).

**D4f — orientation (PD1).** `vite.config.ts:30`: `orientation: "portrait"` → `"any"` (confirmed with Joe 2026-06-13).

---

## Toast extension (sticky duration)

The update prompt must persist until the user acts; the current toast hard-codes a 5 s auto-dismiss. Extend `toast()` minimally and backward-compatibly:

- Add optional `duration?: number` to the `toast()` argument (default `TOAST_REMOVE_DELAY`).
- Only schedule the auto-dismiss `setTimeout` when `Number.isFinite(duration) && duration > 0`. `Infinity`/`0` ⇒ sticky (dismissed only via `ToastAction` flow or `ToastClose`).

This is the only change to shared toast code, ships in PR (b), and existing callers are unaffected (they pass no `duration`).

## File map

| File | Action | Group | Purpose |
|---|---|---|---|
| `src/lib/pwa.ts` | Create | a | `isStandalone()`, `isIOS()` + `BeforeInstallPromptEvent` type |
| `src/hooks/use-install-prompt.ts` | Create | a | `beforeinstallprompt`/`appinstalled` hook |
| `src/components/shared/install-instructions-sheet.tsx` | Create | a | Manual-install `MobileSheet` |
| `src/components/shared/sidebar-menu.tsx` | Modify | a | Add 3-state "Install app" row |
| `src/hooks/index.ts` | Modify | a, b | Export new hooks |
| `vite.config.ts` | Modify | b, c | `registerType` (b); D4a/b/d/f (c) |
| `tsconfig.app.json` | Modify | b | Add `vite-plugin-pwa/react` type |
| `src/components/pwa/pwa-updater.tsx` | Create | b | `useRegisterSW` → sticky update toast |
| `src/components/ui/toaster.tsx` | Modify | b | Optional `duration` (backward-compatible) |
| `src/hooks/use-online-status.ts` | Create | b | `online`/`offline` hook |
| `src/components/shared/offline-banner.tsx` | Create | b | Accessible offline bar |
| `src/App.tsx` | Modify | b | Mount `<PWAUpdater/>` + `<OfflineBanner/>` |
| `index.html` | Modify | c | Meta description (D4d) + `mobile-web-app-capable` (D4e) |
| `lighthouserc.cjs` | Modify | c | Remove PWA category (D4c) |
| `package.json` | Modify | c | `analyze` script (D4a) |
| `.gitignore` | Modify | c | Ignore `.analyze/` |
| `frontend/docs/TECHNICAL-DEBT.md` | Modify | c | Update PWA item §5; record cleanups |

## Testing strategy

- **Unit (Vitest):** `use-online-status` (mock `navigator.onLine`, dispatch `online`/`offline`); `use-install-prompt` (dispatch `beforeinstallprompt` then `appinstalled`; assert `canInstall` flips, `promptInstall` calls `prompt()` and clears); `isStandalone`/`isIOS` (matchMedia + UA mocks); toast `duration` (sticky toast not auto-removed; default still removes).
- **Component (Vitest + Testing Library):** sidebar install row across the three states (standalone ⇒ absent; canInstall ⇒ tap calls `promptInstall`; else ⇒ tap opens instructions sheet); `PWAUpdater` update toast renders with a Reload action whose click calls a mocked `updateServiceWorker` (mock `virtual:pwa-register/react`); `OfflineBanner` shows only when offline with `role="status"`.
- **E2E (Playwright):** offline banner shows on `context.setOffline(true)` and clears on reconnect. **SW behaviors (update prompt, precache) run only on a production build** — exercise them against `npm run preview`/built output, never `npm run dev`.
- **Build verification:** after `npm run build`, assert no `dist/stats.html` and that `dist/sw.js` precache list excludes `stats.html`; grep for zero `fonts.googleapis`/`gstatic` references.
- **Manual device pass:** Android Chrome (Galaxy S10) install via the row; iOS Safari instructions sheet → Add to Home Screen → launches standalone (row then hidden); desktop Chrome/Edge install + update prompt.
- Follow `frontend/AGENTS.md` E2E gotchas (`safeClick`, `waitForDialogReady`, async-form timing, dedicated `QueryClient` with `gcTime: Infinity` for any cache assertions).

## Out of scope

API/response caching, Query persistence, IndexedDB, offline data, write queue, Background Sync, Web Push, notifications; `networkMode` change; manifest name/icons/title; manifest `screenshots`/`shortcuts`/`categories` and iOS startup images (deferred follow-up).

## Acceptance criteria

(Mirror of the story.) Update prompt is user-dismissible with a working Reload and no silent reloads; sidebar install row one-taps on Chromium / instructs elsewhere / hides when installed; offline banner appears <~1s and is accessible; `npm run build` has no `stats.html` (not precached/served); no Google-Fonts CDN rules remain and fonts still load; `npm run lighthouse` runs without the PWA category and passes warn-only thresholds; descriptions match, `mobile-web-app-capable` present, orientation unlocked; no offline-data/persistence/push introduced.

## Self-review notes

- **Placeholders:** none — every decision names exact files/lines and gives copy.
- **Consistency:** `registerType: "prompt"` and `PWAUpdater` are bound to the same PR (D1) to avoid stranded updates; the toast `duration` extension is the only shared-code change and is backward-compatible.
- **Ambiguity resolved:** offline banner is a flex child above the bottom nav (not a fixed overlay); standalone is read once at mount; the visualizer is removed from the plugin array entirely unless `ANALYZE=true` (not merely repointed) so it can't sweep into precache.
- **Honesty:** `onOfflineReady` copy is constrained to "app shell cached," never "works offline," to stay truthful pending Option C.
