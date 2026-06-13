---
id: mobile-pwa-installability
title: Polished PWA installability + honest offline UX
epic: mobile-ux
status: in-progress
priority: P2
created: 2026-06-13
updated: 2026-06-13
issues:
  - FE #213 (a) install affordance + standalone detection
  - FE #214 (b) controlled update prompt + offline banner
  - FE #215 (c) PWA config cleanups
prs: []
---

## Context

Family Hub already ships as a PWA (`vite-plugin-pwa`), and the roadmap flagged a **sidebar "Install app" row** as "probably the single highest-leverage daily-use improvement available for one row of UI" (seed in [sidebar-settings-story.md](sidebar-settings-story.md#new-ideas-surfaced-backlog-seeds-not-in-scope)). But the current PWA is configured but not *polished*: it can hard-reload mid-use without warning, has no install affordance, looks broken when the network drops, and ships stale/leaky config (a publicly-served bundle-stats file, dead font-cache rules, a Lighthouse category that no longer exists).

This story makes the PWA **polished and honest**: get it onto home screens, give a controlled update prompt instead of a surprise reload, tell the truth when offline, and clean up the config. It is **UI + config only**.

It **supersedes** the loose "PWA install row in the sidebar" seed in `sidebar-settings-story.md` and the "Device-member association" seed remains separate.

### The honesty line (what this is NOT)

This story deliberately does **not** add offline *data*. That is a separate, larger effort tracked as **Option C / offline reads** (API/response caching, TanStack Query persistence, IndexedDB). Conflating the two is how PWAs end up claiming "works offline" while showing stale or empty data. Here, when the device is offline we tell the user plainly that changes won't save — we do not pretend the data is available. See PRD §7.5.2.

### Pre-implementation state (verified 2026-06-13, paths relative to `frontend/`)

- `vite.config.ts:20` — `registerType: "autoUpdate"` → the service worker silently `skipWaiting()` + `clientsClaim()`; a new version can hard-reload mid-use with **no prompt** (can drop an in-progress event form).
- `vite.config.ts:30` — manifest `orientation: "portrait"` is a **hard lock**.
- `vite.config.ts:25` vs `index.html:7` — manifest description ("Your family's command center") and meta description ("Family calendar and organizer") **disagree**.
- `vite.config.ts:56-79` — two Google-Fonts CDN runtime-cache rules that are **dead**: Nunito is self-hosted via `@fontsource/nunito` and precached (tech-debt #109 "self-host Nunito" already shipped).
- `vite.config.ts:82-87` — `rollup-plugin-visualizer` writes `dist/stats.html` (~1.2 MB), which is swept into the precache manifest **and** `deploy.sh` rsyncs `dist/` wholesale, so `/stats.html` is **publicly served in production** and cached into every visitor's SW. Minor info disclosure + wasted cache.
- `index.html:14-18` — iOS meta present (`apple-mobile-web-app-capable`); the modern standard `mobile-web-app-capable` is **missing**.
- `lighthouserc.cjs:14,35` — requests/asserts the **`pwa` category**, which Lighthouse 12 (bundled by `@lhci/cli ^0.15.1`) **removed** (May 2024). Dead config that can error.
- No SW-registration code, no `beforeinstallprompt`/`appinstalled`/`navigator.onLine` usage, no online/offline UI, no update prompt exist yet.
- TanStack Query (`src/providers/query-provider.tsx`) uses default `networkMode: "online"` → queries/mutations **pause** when offline. This story explains that behavior to the user; it does **not** change `networkMode`.

## Product decisions

### PD1. Orientation: unlock to `any` (confirmed with Joe, 2026-06-13)

Change manifest `orientation` from `"portrait"` to `"any"`. Phones keep their natural orientation; the future landscape touchscreen/wall hub (still the long-term product direction per the roadmap) is not forced into portrait. This is strictly more permissive than today and reverts to `"portrait"` trivially if a portrait-only need appears.

### PD2. Description wording: one reconciled line (product call, noted here)

Both the manifest `description` and the `index.html` meta `description` become:

> **"Your family's command center for calendar, lists, chores, and meals."**

Rationale: keeps the established "command center" brand phrasing (PRD vision language and the current manifest), but makes it honest and descriptive of the **shipped** modules — better for the Android install dialog subtitle, search snippets, and link previews than either of the two current strings. Manifest `name`/`short_name` ("FamilyHub") and the `<title>` are unchanged (out of scope).

## Scope

**In (UI + config only):**

1. **Controlled updates (B1)** — `registerType: "prompt"`; a `PWAUpdater` mounted near root that shows a dismissible toast with a working **Reload** action when a new SW is ready. No silent forced reloads.
2. **Install affordance (B2)** — a sidebar **"Install app"** row: one-tap install on Chromium (`beforeinstallprompt`), a manual-instructions bottom sheet elsewhere (iOS Safari / Firefox), **hidden when already installed/standalone**.
3. **Online/offline (B3)** — a `use-online-status` hook and a slim, accessible, non-blocking offline banner that explains changes won't save while offline.
4. **Config cleanups (B4)** — remove `stats.html` from production output; delete dead Google-Fonts CDN cache rules; fix the Lighthouse PWA category; reconcile the description (PD2); add modern `mobile-web-app-capable`; unlock orientation (PD1).

**Out (explicitly — these are Option C or later):**

- No API/response caching, no TanStack Query persistence, no IndexedDB, no offline *data* reads.
- No offline write queue, Background Sync, Web Push, or notifications.
- No change to TanStack Query `networkMode`.
- No manifest `name`/`short_name`/`<title>` change; no router/architecture changes.
- Optional richer-install metadata (manifest `screenshots`/`shortcuts`/`categories`, iOS `apple-touch-startup-image` splash screens) is **deferred to a follow-up** (see Follow-ups), not built here.

## Acceptance criteria

- [ ] A new SW version surfaces a user-dismissible "update available" prompt with a working **Reload** action; no silent forced reloads.
- [ ] The sidebar shows an "Install app" row that one-taps install on Chromium, opens manual instructions elsewhere, and is hidden when already installed.
- [ ] An offline banner appears within ~1s of going offline and disappears on reconnect; it is accessible (`role="status"`, `aria-live="polite"`).
- [ ] `npm run build` produces a `dist/` with **no `stats.html`**; `/stats.html` is neither deployed nor precached (verified in `dist/sw.js`).
- [ ] No Google-Fonts CDN runtime-cache rules remain; app fonts still load (self-hosted).
- [ ] `npm run lighthouse` runs without the removed PWA category and passes the existing warn-only thresholds.
- [ ] Manifest description and `index.html` meta description match (PD2); modern `mobile-web-app-capable` present; orientation no longer hard-locked to portrait (PD1).
- [ ] No offline data caching / query persistence / push was introduced.

## Implementation grouping (FE issues)

Three logically-grouped FE issues, each branch + PR, each linking Story/Spec/Plan with an execution contract:

- **(a) Install affordance + standalone detection** — `use-install-prompt`, standalone/iOS detection, sidebar Install row + instructions sheet.
- **(b) Controlled update prompt + offline banner** — `registerType: "prompt"`, tsconfig types, `PWAUpdater`/`useRegisterSW`, `use-online-status`, offline banner.
- **(c) PWA config cleanups** — stats.html gating + `globIgnores`, dead font rules, Lighthouse PWA category, description reconcile, `mobile-web-app-capable`, orientation, plus a tech-debt sweep.

The three are largely independent (different files/hunks); recommended merge order is (c) → (b) → (a). See the plan for the dependency notes.

## Spec / Plan

- Spec: [`docs/superpowers/specs/2026-06-13-pwa-installability-design.md`](../../../superpowers/specs/2026-06-13-pwa-installability-design.md)
- Plan: [`docs/superpowers/plans/2026-06-13-pwa-installability.md`](../../../superpowers/plans/2026-06-13-pwa-installability.md)

## Risks

- **SW E2E only runs on a production build.** Service-worker update + offline behaviors do not run on the dev server; SW/offline E2E must run against `npm run preview` / built output, not `npm run dev`.
- **`beforeinstallprompt` is Chromium-only.** The install row must degrade to manual instructions on iOS Safari / Firefox and hide entirely when already installed — three states, all tested.
- **Switching to `registerType: "prompt"` without the updater would strand updates.** The `PWAUpdater` component and the `registerType` change must ship in the **same** PR (b), or new versions would wait forever.
- **Honesty regression risk.** If a later story adds offline data, the banner copy must be revisited so it doesn't contradict actual offline capability.

## Follow-ups (out of scope, capture as separate issues if pursued)

- Richer install metadata: manifest `screenshots` + `shortcuts` + `categories` (richer Android install dialog and home-screen shortcuts); iOS `apple-touch-startup-image` splash screens (removes the white launch flash).
- **Option C / offline reads** — the real offline-data effort (API caching, Query persistence). This story's offline banner is the honest placeholder until then.

## Non-goals

- Building any offline-data, sync, push, or notification capability.
- Redesigning the sidebar, bottom nav, or any existing surface beyond adding one row.
- Changing the app name, icons, or theme colors.
