# Frontend Bundle Splitting & Cold-Load Reduction

**Date:** 2026-07-02
**Status:** Approved design, ready for implementation plan
**Owner:** Family Hub frontend
**Origin:** 2026-07-01 whole-product review follow-up. A production build revealed the main entry chunk is 640 kB (195 kB gzip) and the `react-vendor` split silently produces a 0 kB chunk — React/React-DOM leaked into the entry.

## Summary

Shrink the JavaScript that Family Hub loads on every cold start / PWA launch by fixing the failed vendor chunking in `vite.config.ts`, and add a lightweight CI budget check so the main chunk cannot silently re-bloat. The change only moves modules between output files — no runtime behavior changes.

This matters because Family Hub is an installed mobile PWA used daily; the entry chunk is paid on every cold launch and is precached in full by the service worker.

## Problem

A production build (`npx vite build`) shows:

```
dist/assets/react-vendor-*.js     0.00 kB   <- split silently failed
dist/assets/index-*.js          640.29 kB   gzip: 195.22 kB   <- everything landed here
dist/assets/radix-vendor-*.js    78.47 kB   (only 6 of 27 @radix packages)
```

Root causes:

1. **Object-form `manualChunks` failed for React.** `"react-vendor": ["react", "react-dom"]` produced a 0 kB chunk; React and React-DOM (~130 kB raw) were bundled into the main `index` chunk instead of a long-cached vendor chunk.
2. **Incomplete Radix grouping.** Only 6 of 27 `@radix-ui/*` packages are listed in `radix-vendor`; the other 21 scatter into the entry and feature chunks.
3. **No regression guard.** Nothing in CI would catch the entry chunk growing again (this is exactly how the React leak went unnoticed).

Not problems (confirmed during exploration, out of scope): lucide-react and date-fns are already imported as tree-shakeable named imports; the React Compiler (`babel-plugin-react-compiler`) already handles runtime memoization; route-level `lazy()` code splitting already exists for non-primary modules.

## Design

Two independent, behavior-preserving changes in `frontend/`, each verifiable by rebuilding.

### Component 1 — Vendor chunking fix

**File:** `frontend/vite.config.ts` (the `build.rollupOptions.output.manualChunks` block)

Replace the object form with a function that inspects each module id and routes `node_modules` packages into stable vendor chunks:

- `node_modules/react`, `react-dom`, `scheduler`, and React's jsx-runtime → `react-vendor`
- any `node_modules/@radix-ui/*` → `radix-vendor` (all 27, not just 6)
- `@tanstack/react-query` → `query-vendor`
- `date-fns` and `react-day-picker` → `date-vendor`
- any other `node_modules` module → a general `vendor` chunk, so future dependencies land in a cacheable chunk rather than the entry

Matching must be robust to path separators and nested `node_modules` (match on `/node_modules/<pkg>/` segments, not a bare `includes`, so e.g. a package merely depending on react is not miscategorized). The function returns `undefined` for application source, leaving Vite/Rollup's existing behavior (entry chunk + `lazy()` feature chunks) untouched.

**Success criteria:**

- After rebuild, `react-vendor` is non-zero (~130 kB raw) and contains react + react-dom.
- The main `index` chunk drops materially from its current 640 kB raw / 195 kB gzip (target: entry gzip at or below ~140 kB; exact post-fix number is measured and pinned in the plan).
- No new behavior: the app renders and all existing lint/unit/E2E/Lighthouse CI steps pass unchanged.

Application-level lazy-loading is explicitly *not* expanded in this change. If, after the vendor split, the entry chunk is still heavier than the budget, that is handled by follow-up work — this spec's scope is the vendor split plus the guard.

### Component 2 — CI bundle-size budget

**Files:** `frontend/scripts/check-bundle-size.mjs` (new), a `check:bundle` script in `frontend/package.json`, one new step in `frontend/.github/workflows/ci.yml`, and a colocated unit test.

The script:

1. Reads `dist/index.html` and extracts the entry chunk path from the `<script type="module" ... src="/assets/index-*.js">` tag. This avoids enabling Vite's build manifest and is robust to content-hashed filenames. If no such tag is found, exit non-zero with a clear message (fail closed).
2. Reads that file from `dist/`, gzips it, and compares the gzipped byte size to a budget constant (`MAX_ENTRY_GZIP_BYTES`).
3. Prints the measured size vs. budget and exits 0 (under) or 1 (over).

**Budget value:** set from the measured post-fix entry gzip size plus ~15% headroom, pinned as a named constant with a comment recording the baseline and date. Chosen so the current (fixed) build passes comfortably while a regression on the scale of the React leak fails the build.

**CI wiring:** a new step in the existing `check` job of `ci.yml`, placed immediately after the existing `- run: npm run build`, invoking `npm run check:bundle`. It runs on every push and PR to `main`, alongside the existing Lighthouse assertions. This is complementary to Lighthouse CI (which gates runtime page metrics, not the specific entry-chunk gzip size), not a replacement.

**Testing:** a colocated Vitest unit test drives the size-checking logic against fixtures — one payload under budget (expect pass/exit 0) and one over budget (expect fail/exit 1) — plus the missing-entry-tag case (expect fail closed). The script is structured so its core (given an entry file path + budget, return pass/fail) is unit-testable without invoking a real build.

## Non-goals

- Expanding application-level `lazy()` code splitting (possible follow-up if the entry chunk stays above budget after the vendor split).
- Swapping or removing dependencies, or changing how lucide/date-fns are imported (already tree-shakeable).
- Per-package "one chunk per npm package" granularity (more HTTP requests, over-engineered for this user base).
- Any runtime behavior, styling, or feature change.
- Adding the `size-limit` package or other new tooling — the zero-dependency script covers the need.

## Rollout

Standard FE flow: feature branch `perf/bundle-splitting` in `joe-bor/FamilyHub`, PR to `main`, regular merge commit. `perf:` commits → patch release via release-please. No env vars, no migrations, no droplet changes. Users see only a faster cold load; the service worker precache manifest updates automatically on the next deploy.

## Verification checklist (for the PR)

- [ ] Rebuild shows `react-vendor` non-zero and entry `index` chunk materially smaller (before/after numbers in the PR body).
- [ ] `npm run check:bundle` passes locally against the fixed build.
- [ ] The bundle-check unit test covers under-budget, over-budget, and missing-entry cases.
- [ ] `npm run lint`, `npm run test:coverage`, E2E, and Lighthouse CI all still pass.
- [ ] No application source behavior change — only `vite.config.ts`, the new script/test, `package.json`, and `ci.yml`.
