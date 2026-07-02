# Frontend Bundle Splitting & Cold-Load Reduction

**Date:** 2026-07-02
**Status:** Approved design, ready for implementation plan
**Owner:** Family Hub frontend
**Origin:** 2026-07-01 whole-product review follow-up. A production build revealed the main entry chunk is 640 kB (195 kB gzip) and the `react-vendor` split silently produces a 0 kB chunk — React/React-DOM leaked into the entry.

## Summary

Fix the failed vendor chunking in `vite.config.ts` so Family Hub's dependencies land in long-cached vendor chunks instead of the application entry chunk, and add a lightweight CI budget check so the entry chunk cannot silently re-bloat. The change only moves modules between output files — no runtime behavior changes.

**What this does and does not buy us (measured 2026-07-02):** total first-visit JS is roughly unchanged (~250 kB gzip either way — the bytes are reorganized, not removed). The wins are: (1) the entry chunk drops from **195 kB → 77 kB gzip (~60%)**, so every app deploy re-downloads far less — ~174 kB of React + other vendor code now sits in chunks that only change when dependencies change, not on every FE release; (2) it repairs a genuine misconfiguration where the intended `react-vendor` optimization was silently dead (a 0 kB chunk); (3) cold start downloads three parallelizable chunks instead of one monolith. This matters because Family Hub is an installed mobile PWA that updates fairly often; the caching win is felt on every post-update launch.

## Problem

A production build (`npx vite build`) shows:

```
dist/assets/react-vendor-*.js     0.00 kB   <- split silently failed
dist/assets/index-*.js          640.29 kB   gzip: 195.22 kB   <- everything landed here
dist/assets/radix-vendor-*.js    78.47 kB   (only 6 of 27 @radix packages)
```

Root causes:

1. **Object-form `manualChunks` failed for React.** `"react-vendor": ["react", "react-dom"]` produced a 0 kB chunk; React and React-DOM (~130 kB raw) were bundled into the main `index` chunk instead of a long-cached vendor chunk.
2. **Incomplete Radix grouping.** Only 6 of 27 `@radix-ui/*` packages are listed in `radix-vendor`; the other 21 scatter into the entry and feature chunks. (The fix folds all of Radix into the single `vendor` chunk, so the count no longer matters.)
3. **No regression guard.** Nothing in CI would catch the entry chunk growing again (this is exactly how the React leak went unnoticed).

Not problems (confirmed during exploration, out of scope): lucide-react and date-fns are already imported as tree-shakeable named imports; the React Compiler (`babel-plugin-react-compiler`) already handles runtime memoization; route-level `lazy()` code splitting already exists for non-primary modules.

## Design

Two independent, behavior-preserving changes in `frontend/`, each verifiable by rebuilding.

### Component 1 — Vendor chunking fix

**File:** `frontend/vite.config.ts` (the `build.rollupOptions.output.manualChunks` block)

Replace the object form with a two-way function split:

- `node_modules/react`, `react-dom`, `scheduler` → `react-vendor`
- every other `node_modules` module → a single `vendor` chunk
- application source → `undefined` (leaves Vite/Rollup's entry chunk + existing `lazy()` feature chunks untouched)

**Why two chunks and not per-library groups (react/radix/query/date + catch-all):** the per-library approach was tested during the spec-to-plan review and produces **circular chunk warnings** (`vendor -> query-vendor -> vendor`, `vendor -> radix-vendor -> vendor`) because a general `vendor` module and a specific vendor group import each other. React is the only dependency safe to isolate — it is a pure leaf that imports nothing, so `react-vendor` never forms a cycle. Lumping all other `node_modules` (Radix, TanStack Query, date-fns, zod, react-hook-form, etc.) into one `vendor` chunk avoids cycles entirely and still pulls everything out of the entry. Measured result: no warnings, entry 77 kB gzip, `react-vendor` 59 kB gzip, `vendor` 115 kB gzip.

Matching must be robust to path separators and nested `node_modules` (match on `/node_modules/<pkg>/` segments via regex, not a bare `includes`, so a package merely depending on react is not miscategorized).

**Success criteria (measured 2026-07-02):**

- After rebuild, `react-vendor` is non-zero (~193 kB raw / ~59 kB gzip) and contains react + react-dom.
- The entry `index` chunk drops from 640 kB raw / 195 kB gzip to ~265 kB raw / ~77 kB gzip.
- Build emits **no circular chunk warnings**.
- No new behavior: the app renders and all existing lint/unit/E2E/Lighthouse CI steps pass unchanged.

Application-level lazy-loading is explicitly *not* expanded in this change. If, after the vendor split, the entry chunk is still heavier than the budget, that is handled by follow-up work — this spec's scope is the vendor split plus the guard.

### Component 2 — CI bundle-size budget

**Files:** `frontend/scripts/check-bundle-size.js` (new — plain ESM, since `package.json` has `"type": "module"` and Biome lints `.js`), a `check:bundle` script in `frontend/package.json`, one new step in `frontend/.github/workflows/ci.yml`, and a colocated unit test under `src/` (Vitest's `include` is `src/**/*.{test,spec}.{ts,tsx}`, so the test importing the script's pure helpers lives at `src/test/check-bundle-size.test.ts`).

The script:

1. Reads `dist/index.html` and extracts the entry chunk path from the single `<script type="module" ... src>` tag — whatever the `src` points to is the entry, regardless of its filename (do not hard-code the `index-` prefix, which is incidental and could change). Attribute order is not assumed (`crossorigin` appears between `type` and `src` in the current output). This avoids enabling Vite's build manifest and is robust to content-hashed filenames. If zero or more than one module-script tag is found, exit non-zero with a clear message (fail closed).
2. Reads that file from `dist/`, gzips it, and compares the gzipped byte size to a budget constant (`MAX_ENTRY_GZIP_BYTES`).
3. Prints the measured size vs. budget and exits 0 (under) or 1 (over).

**Budget value:** `MAX_ENTRY_GZIP_BYTES = 92160` (90 kB), set from the measured post-fix entry gzip of ~77 kB (77,158 bytes on 2026-07-02) plus ~15% headroom, pinned as a named constant with a comment recording the baseline and date.

**Why gate the entry chunk specifically (not total initial JS):** the historical regression was React leaking *into the entry* — which would spike the entry from 77 kB back toward ~135 kB while leaving total initial JS unchanged (React just moves between chunks). An entry-chunk budget catches exactly that class of regression, plus application-code bloat. A total-initial-JS budget would miss the React leak, so entry-chunk gating is the right guard for the problem we actually hit.

**CI wiring:** a new step in the existing `check` job of `ci.yml`, placed immediately after the existing `- run: npm run build`, invoking `npm run check:bundle`. It runs on every push and PR to `main`, alongside the existing Lighthouse assertions. This is complementary to Lighthouse CI (which gates runtime page metrics, not the specific entry-chunk gzip size), not a replacement.

**Testing:** a colocated Vitest unit test drives the size-checking logic against fixtures — one payload under budget (expect pass/exit 0) and one over budget (expect fail/exit 1) — plus the fail-closed cases where `index.html` has zero or multiple module-script tags (expect non-zero exit). The script is structured so its core (given an entry file path + budget, return pass/fail; and given HTML, resolve the single entry src) is unit-testable without invoking a real build.

## Non-goals

- Expanding application-level `lazy()` code splitting (possible follow-up if the entry chunk stays above budget after the vendor split).
- Swapping or removing dependencies, or changing how lucide/date-fns are imported (already tree-shakeable).
- Per-package or per-library-group chunk granularity (causes circular chunk warnings, as measured; more HTTP requests; over-engineered for this user base).
- Any runtime behavior, styling, or feature change.
- Adding the `size-limit` package or other new tooling — the zero-dependency script covers the need.

## Rollout

Standard FE flow: feature branch `perf/bundle-splitting` in `joe-bor/FamilyHub`, PR to `main`, regular merge commit. `perf:` commits → patch release via release-please. No env vars, no migrations, no droplet changes. Users see only a faster cold load; the service worker precache manifest updates automatically on the next deploy.

## Verification checklist (for the PR)

- [ ] Rebuild shows `react-vendor` non-zero (~59 kB gzip), entry `index` chunk ~77 kB gzip, and **no circular chunk warnings** (before/after numbers in the PR body).
- [ ] `npm run check:bundle` passes locally against the fixed build.
- [ ] The bundle-check unit test covers under-budget, over-budget, and the zero/multiple-module-tag fail-closed cases.
- [ ] `npm run lint`, `npm run test:coverage`, E2E, and Lighthouse CI all still pass.
- [ ] No application source behavior change — only `vite.config.ts`, the new script/test, `package.json`, and `ci.yml`.
