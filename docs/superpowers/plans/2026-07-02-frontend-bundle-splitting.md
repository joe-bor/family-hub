# Frontend Bundle Splitting & CI Budget Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move Family Hub's dependencies out of the app entry chunk into long-cached vendor chunks (entry 195 kB → ~77 kB gzip), and add a CI guard so the entry chunk can't silently re-bloat.

**Architecture:** Two independent, behavior-preserving changes in `frontend/`: (1) replace the broken object-form `manualChunks` in `vite.config.ts` with a two-way function split (`react-vendor` for react/react-dom/scheduler, single `vendor` for all other node_modules); (2) a zero-dependency ESM script that reads the entry chunk from `dist/index.html`, gzips it, and fails CI over a pinned budget.

**Tech Stack:** Vite 6 / Rollup, TypeScript, Vitest, Node ESM (`package.json` has `"type": "module"`), Biome, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-07-02-frontend-bundle-splitting-design.md` (root repo `joe-bor/family-hub`)

**Repo:** All work in `frontend/` (GitHub `joe-bor/FamilyHub`). Branch `perf/bundle-splitting`. Conventional commits, no AI attribution. Merge with a **regular merge commit** (not squash) per the repo's release-please setup.

**Measured baseline (2026-07-02, for reference):**
- Broken build: entry `index` 640 kB raw / **195 kB gzip**, `react-vendor` **0.00 kB** (dead).
- After the Task 1 fix: entry ~265 kB raw / **~77 kB gzip** (77,158 bytes), `react-vendor` ~193 kB raw / ~59 kB gzip, `vendor` ~378 kB raw / ~115 kB gzip, **no circular warnings**.

---

### Task 0: Branch

- [ ] **Step 1: Create the feature branch**

```bash
cd frontend
git checkout main && git pull
git checkout -b perf/bundle-splitting
```

---

### Task 1: Fix vendor chunking in `vite.config.ts`

**Files:**
- Modify: `frontend/vite.config.ts` (the `build.rollupOptions.output.manualChunks` block, currently the object form)

Context: the current object-form `manualChunks` produces a 0 kB `react-vendor` (React leaks into the entry) and only groups 6 of 27 Radix packages. A per-library function split (react/radix/query/date + catch-all `vendor`) was tested and produces **circular chunk warnings** (`vendor -> query-vendor -> vendor`). React is the only safe library to isolate (pure leaf, imports nothing); lumping all other node_modules into one `vendor` chunk avoids cycles.

- [ ] **Step 1: Replace the manualChunks block**

Open `frontend/vite.config.ts`. Replace this exact block:

```ts
          manualChunks: {
            "react-vendor": ["react", "react-dom"],
            "query-vendor": ["@tanstack/react-query"],
            "date-vendor": ["date-fns", "react-day-picker"],
            "radix-vendor": [
              "@radix-ui/react-dialog",
              "@radix-ui/react-popover",
              "@radix-ui/react-scroll-area",
              "@radix-ui/react-tabs",
              "@radix-ui/react-slot",
              "@radix-ui/react-label",
            ],
          },
```

with the function form:

```ts
          manualChunks(id: string) {
            if (!id.includes("/node_modules/")) return undefined;
            // React is a pure leaf (imports nothing) so isolating it never
            // forms a circular chunk. Every other dependency goes into a
            // single `vendor` chunk — splitting them further (radix/query/date
            // groups) produces circular-chunk warnings because a general
            // vendor module and a specific group import each other.
            if (/\/node_modules\/(react|react-dom|scheduler)\//.test(id)) {
              return "react-vendor";
            }
            return "vendor";
          },
```

- [ ] **Step 2: Build and verify chunks + no circular warnings**

Run:

```bash
npm run build 2>&1 | tee /tmp/fh-build.log
grep -i circular /tmp/fh-build.log && echo "FAIL: circular warnings present" || echo "OK: no circular warnings"
grep -E "react-vendor|vendor-|index-.*\.js" /tmp/fh-build.log | grep -v css
```

Expected: `react-vendor` chunk is ~193 kB raw / ~59 kB gzip (NOT 0.00 kB), a `vendor` chunk ~378 kB raw / ~115 kB gzip appears, the entry `index-*.js` is ~265 kB raw / ~77 kB gzip, and **no** circular-chunk lines.

- [ ] **Step 3: Measure the entry gzip (records the number the CI budget is based on)**

Run:

```bash
ENTRY=$(grep -oE '/assets/index-[^"]*\.js' dist/index.html | head -1)
echo "entry: $ENTRY"
gzip -c "dist${ENTRY}" | wc -c | awk '{printf "entry gzip: %.1f kB (%d bytes)\n",$1/1024,$1}'
```

Expected: ~77 kB (~77,158 bytes). If it is materially higher than ~80 kB, note it — Task 2's budget constant (90 kB) may need to be raised to `measured * 1.15`.

- [ ] **Step 4: Sanity-check the app still runs (behavior unchanged)**

Run `npm run preview`, open the served URL, confirm the app loads and you can navigate between modules (Home, Calendar, Lists). This only moves modules between files; there should be zero visible change. Stop the preview server when done.

- [ ] **Step 5: Commit**

```bash
git add vite.config.ts
git commit -m "perf(build): split react and vendor deps out of the entry chunk"
```

---

### Task 2: Bundle-size check script + unit test

**Files:**
- Create: `frontend/scripts/check-bundle-size.js`
- Create: `frontend/src/test/check-bundle-size.test.ts`
- Modify: `frontend/package.json` (add `check:bundle` script)

Design: the script exposes two pure, unit-testable helpers plus a `main()` that wires them to the filesystem and `process.exit`. Helpers are pure so the test never needs a real build. `package.json` has `"type": "module"`, so `.js` is ESM and `export` works. Vitest only includes `src/**/*.{test,spec}.{ts,tsx}`, so the test lives under `src/test/` and imports the helpers from the script via a relative path.

Build-safety note (verified against the current tsconfig): `tsconfig.app.json` excludes `src/**/*.test.ts` and the whole `src/test` dir, and the new `scripts/*.js` is in no tsconfig `include`. So `npm run build` (`tsc -b && vite build`) never type-checks the test or the script — the cross-directory `.ts`→`.js` import cannot break the production build. Vitest (esbuild) resolves it fine at test time.

- [ ] **Step 1: Write the failing test**

Create `frontend/src/test/check-bundle-size.test.ts`:

```ts
import { gzipSync } from "node:zlib";
import { describe, expect, it } from "vitest";
import {
  resolveEntryChunk,
  checkGzipBudget,
} from "../../scripts/check-bundle-size.js";

describe("resolveEntryChunk", () => {
  it("returns the single module-script src", () => {
    const html = `<!doctype html><html><head>
      <script type="module" crossorigin src="/assets/index-ABC123.js"></script>
      <link rel="modulepreload" crossorigin href="/assets/vendor-XYZ.js">
      </head><body></body></html>`;
    expect(resolveEntryChunk(html)).toBe("/assets/index-ABC123.js");
  });

  it("is filename-agnostic (does not rely on the index- prefix)", () => {
    const html = `<script type="module" src="/assets/main-DEADBEEF.js"></script>`;
    expect(resolveEntryChunk(html)).toBe("/assets/main-DEADBEEF.js");
  });

  it("throws when there is no module-script tag (fail closed)", () => {
    const html = `<script src="/assets/legacy.js"></script>`;
    expect(() => resolveEntryChunk(html)).toThrow(/no module-script/i);
  });

  it("throws when there are multiple module-script tags (fail closed)", () => {
    const html = `
      <script type="module" src="/assets/a.js"></script>
      <script type="module" src="/assets/b.js"></script>`;
    expect(() => resolveEntryChunk(html)).toThrow(/multiple module-script/i);
  });
});

describe("checkGzipBudget", () => {
  const budget = 100; // bytes, for the test

  it("passes when gzipped size is under budget", () => {
    const small = gzipSync(Buffer.alloc(10, "a"));
    const result = checkGzipBudget(small, budget);
    expect(result.ok).toBe(true);
    expect(result.gzipBytes).toBe(small.length);
  });

  it("fails when gzipped size exceeds budget", () => {
    // Incompressible random bytes so the gzip output reliably exceeds 100 bytes.
    const big = gzipSync(Buffer.from(Array.from({ length: 5000 }, () => Math.floor(Math.random() * 256))));
    const result = checkGzipBudget(big, budget);
    expect(result.ok).toBe(false);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npx vitest run src/test/check-bundle-size.test.ts
```

Expected: FAIL — cannot resolve `../../scripts/check-bundle-size.js` (module does not exist).

- [ ] **Step 3: Create the script**

Create `frontend/scripts/check-bundle-size.js`:

```js
// @ts-check
import { gzipSync } from "node:zlib";
import { readFileSync } from "node:fs";
import { join, dirname } from "node:path";
import { fileURLToPath } from "node:url";

// Budget for the entry chunk, gzipped. Baseline measured 2026-07-02: the fixed
// build's entry is ~77 kB gzip (77,158 bytes). 90 kB gives ~15% headroom while
// still catching a regression such as React leaking back into the entry (which
// would push it toward ~135 kB). See the spec: gate the ENTRY chunk, not total
// initial JS — a React leak moves bytes between chunks without changing the total.
export const MAX_ENTRY_GZIP_BYTES = 92160; // 90 * 1024

/**
 * Extract the single ES-module entry chunk path from built index.html.
 * Fails closed if there is not exactly one module-script tag.
 * @param {string} html
 * @returns {string}
 */
export function resolveEntryChunk(html) {
  const matches = [
    ...html.matchAll(/<script\b[^>]*\btype=["']module["'][^>]*>/gi),
  ]
    .map((m) => /\bsrc=["']([^"']+)["']/i.exec(m[0])?.[1])
    .filter((src) => typeof src === "string");

  if (matches.length === 0) {
    throw new Error("check-bundle-size: no module-script tag found in index.html");
  }
  if (matches.length > 1) {
    throw new Error(
      `check-bundle-size: expected one module-script tag, found multiple module-script tags: ${matches.join(", ")}`,
    );
  }
  return /** @type {string} */ (matches[0]);
}

/**
 * Compare a chunk's gzipped size to a byte budget.
 * @param {Buffer} gzipped
 * @param {number} budgetBytes
 */
export function checkGzipBudget(gzipped, budgetBytes) {
  return { ok: gzipped.length <= budgetBytes, gzipBytes: gzipped.length, budgetBytes };
}

function main() {
  const distDir = join(dirname(fileURLToPath(import.meta.url)), "..", "dist");
  const html = readFileSync(join(distDir, "index.html"), "utf8");
  const entry = resolveEntryChunk(html);
  const file = join(distDir, entry.replace(/^\//, ""));
  const gzipped = gzipSync(readFileSync(file));
  const { ok, gzipBytes, budgetBytes } = checkGzipBudget(gzipped, MAX_ENTRY_GZIP_BYTES);

  const kb = (n) => (n / 1024).toFixed(1);
  console.log(`Entry chunk: ${entry}`);
  console.log(`Gzip size:   ${kb(gzipBytes)} kB (${gzipBytes} bytes)`);
  console.log(`Budget:      ${kb(budgetBytes)} kB (${budgetBytes} bytes)`);

  if (!ok) {
    console.error(
      `\n❌ Entry chunk exceeds budget by ${kb(gzipBytes - budgetBytes)} kB. ` +
        `Investigate what landed in the entry (npm run analyze) before raising MAX_ENTRY_GZIP_BYTES.`,
    );
    process.exit(1);
  }
  console.log("\n✅ Entry chunk within budget.");
}

// Run only when invoked directly (not when imported by the test).
if (process.argv[1] === fileURLToPath(import.meta.url)) {
  main();
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
npx vitest run src/test/check-bundle-size.test.ts
```

Expected: PASS (6 tests).

- [ ] **Step 5: Add the npm script**

In `frontend/package.json`, add to `"scripts"` (next to `analyze`):

```json
    "check:bundle": "node scripts/check-bundle-size.js",
```

- [ ] **Step 6: Run the check against the real build (from Task 1)**

```bash
npm run build && npm run check:bundle
```

Expected: prints the entry chunk (~77 kB), budget 90 kB, and `✅ Entry chunk within budget.` with exit 0.

- [ ] **Step 7: Lint + full unit suite stay green**

```bash
npm run lint
npx vitest run
```

Expected: Biome passes on the new `.js` and `.ts` files; all unit tests pass.

- [ ] **Step 8: Commit**

```bash
git add scripts/check-bundle-size.js src/test/check-bundle-size.test.ts package.json
git commit -m "test(build): add zero-dep entry-chunk size budget check"
```

---

### Task 3: Wire the budget check into CI

**Files:**
- Modify: `frontend/.github/workflows/ci.yml` (add one step after the existing `- run: npm run build`)

- [ ] **Step 1: Add the CI step**

In `frontend/.github/workflows/ci.yml`, find the existing build step:

```yaml
      - run: npm run build
```

Insert immediately after it (before `- name: Collect Lighthouse reports`):

```yaml
      - name: Check bundle size budget
        run: npm run check:bundle
```

- [ ] **Step 2: Validate the workflow YAML locally**

```bash
node -e "const y=require('fs').readFileSync('.github/workflows/ci.yml','utf8'); const m=y.match(/npm run build[\s\S]*?check:bundle/); if(!m){throw new Error('check:bundle step not placed after build')} console.log('OK: check:bundle step follows build')"
```

Expected: `OK: check:bundle step follows build`.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci(build): fail the build when the entry chunk exceeds its size budget"
```

---

### Task 4: Verify and open the PR

- [ ] **Step 1: Full local pre-flight**

```bash
npm run lint
npx vitest run
npm run build && npm run check:bundle
```

Expected: all green; `check:bundle` reports the entry ~77 kB within the 90 kB budget. (E2E is exercised by CI against the real backend; it does not need to run locally for this change.)

- [ ] **Step 2: Push and open the PR**

```bash
git push -u origin perf/bundle-splitting
gh pr create --repo joe-bor/FamilyHub \
  --title "perf(build): split vendor deps out of the entry chunk + CI size budget" \
  --body "Closes #<ISSUE_NUMBER>

Fixes the silently-dead \`react-vendor\` split (was a 0 kB chunk — React had leaked into the entry) and adds a CI guard so the entry chunk can't re-bloat unnoticed.

Measured (gzip):
- entry index: 195 kB -> ~77 kB (-60%)
- react-vendor: 0 kB (dead) -> ~59 kB (now a real, long-cached chunk)
- vendor: new ~115 kB chunk (all other node_modules)
- no circular-chunk warnings

Note: total first-visit JS is ~unchanged (~250 kB) — the win is caching (React + vendor no longer re-download on every FE deploy) and repairing the dead split, not fewer first-load bytes.

Spec: joe-bor/family-hub docs/superpowers/specs/2026-07-02-frontend-bundle-splitting-design.md
Plan: joe-bor/family-hub docs/superpowers/plans/2026-07-02-frontend-bundle-splitting.md

Contract:
- [ ] react-vendor non-zero (~59 kB gzip); entry ~77 kB gzip; no circular warnings
- [ ] check:bundle gates the entry chunk (90 kB budget) and runs in CI after build
- [ ] bundle-check unit test covers under/over budget + zero/multiple module-tag fail-closed
- [ ] lint, unit, E2E, Lighthouse all green; no application-source behavior change"
```

Replace `<ISSUE_NUMBER>` with the FE implementation Issue number.

---

## Execution contract (non-negotiables)

1. `manualChunks` is the two-way function split: `react-vendor` for react/react-dom/scheduler, single `vendor` for all other node_modules, `undefined` for app source. No per-library groups (they cause circular-chunk warnings).
2. Build emits **no circular chunk warnings**; `react-vendor` is non-zero; entry ~77 kB gzip.
3. The size check reads the single `<script type="module">` src from `dist/index.html` (filename-agnostic) and **fails closed** on zero or multiple module tags.
4. Budget gates the **entry chunk** gzip at 90 kB (`MAX_ENTRY_GZIP_BYTES = 92160`), baseline ~77 kB.
5. The check is a zero-dependency ESM `.js` script; no new packages (no `size-limit`).
6. CI runs `check:bundle` after `npm run build`; existing lint/unit/E2E/Lighthouse steps stay unchanged and green.
7. No application-source or runtime behavior change — only `vite.config.ts`, the new script + test, `package.json`, and `ci.yml`.
