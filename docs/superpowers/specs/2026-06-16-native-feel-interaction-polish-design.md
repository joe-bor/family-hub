# Native-Feel Interaction Polish — Design

> **Story:** `docs/product/backlog/mobile-ux/native-feel-interaction-polish.md`
> **Scope:** FE only. CSS/motion + two small primitives. No haptics, no back-button logic, no new gestures, no new dependency.

## Problem

Family Hub's mobile shell is structurally complete but the *interaction layer* still reads as "a web app inside a shell." Two concrete gaps, both surfaced during phone dogfooding (Galaxy S10 / Chrome, 2026-06-16):

1. **Abrupt screen changes.** Switching modules via the bottom nav and opening/closing list/recipe detail views are instant content swaps with no motion continuity. Back especially feels like a hard cut. (The home-dashboard spec deliberately deferred page transitions — this story revisits exactly that boundary.)
2. **Taps don't feel acknowledged.** Buttons, nav items, and list rows have hover/active *color* states but no immediate *tactile* press feedback, so on a touch screen a tap doesn't feel registered until the data/navigation resolves.

This story adds one cohesive "acknowledgment layer": every tap is *felt* (press state) and every screen change has *continuity* (transition) — while leaving clean seams for the two sibling stories ([optional-haptics](../../product/backlog/mobile-ux/optional-haptics.md), [native-back-button](../../product/backlog/mobile-ux/native-back-button.md)).

## Current FE state (verified 2026-06-16, paths relative to `frontend/`)

- **Module switching** — `src/App.tsx:50-91` `renderModule(activeModule)` is a `switch` returning each module; the active one renders at `src/App.tsx:168-170` inside `<main className="flex-1 min-h-0 flex flex-col overflow-hidden">`. Non-primary modules are `lazy()` + wrapped in `<LazyModule>`/`<Suspense>` (`:24-45`); Home and Calendar are eager. `activeModule` is Zustand (`useAppStore`, `src/stores/app-store.ts`); `null` = Home (mobile only).
- **Bottom nav** — `src/components/shared/mobile-bottom-nav.tsx`: raw `<button>`s (`:66-91`) styled by `tabBase` (`:33-34`, includes `transition-colors`) + `tabActive`/`tabIdle`; tap calls `setActiveModule`. Overflow modules render as buttons inside a `MobileSheet` (`:106-121`).
- **Button primitive** — `src/components/ui/button.tsx`: `cva` `buttonVariants` (base at `:7-8` already has `transition-all`), 6 variants / 6 sizes, `asChild` via Radix `Slot`. The shared tappable.
- **Detail views (slide targets)** — `src/components/lists-view.tsx`: `selectedListId` state (`:14`); when non-null, early-returns `<ListDetailView … onBack={() => setSelectedListId(null)} />` (`:19-33`); rows call `onOpen={() => setSelectedListId(list.id)}` (`:146`). `src/components/recipes-view.tsx`: same shape with `selectedRecipeId` (`:87`) gating `<RecipeDetailView>` vs the grid (`:168-177`), back via `setSelectedRecipeId(null)`. Event "detail" is a **modal** (`event-detail-modal`), not an in-view swap.
- **Deps** — `tailwindcss-animate ^1.0.7`, `vaul ^1.1.2`, `lucide-react ^0.576.0`, `clsx`/`tailwind-merge`/`cva`. **No** `framer-motion`/`motion`. Tailwind v4; `cn()` from `@/lib/utils`.
- **Hooks barrel** (`src/hooks/index.ts`) exports `useDebounce`, `useGoogleAuthReturn`, `useIsMobile`, `useMediaQuery`. New hooks go here.
- **Sheets/dialogs/sidebar already animate** (vaul / `motion-safe:animate-in`) and were **not** reported as harsh — out of scope.

## Goals

- Module/tab switches animate with a **fade-through**; list/recipe detail open/close animate with a **shared-axis slide** (forward in-from-right, back in-from-left).
- Every tap-responsive control gives an immediate **scale + tint** press response on `pointerdown`, springing back on release.
- One reusable press primitive and one reusable transition primitive, so behavior is consistent and the haptics/back-button stories plug into a single seam each.
- No new dependency; reuse `tailwindcss-animate` + the Web Animations API.
- Fully `prefers-reduced-motion`-aware, 60fps on the S10 (transform/opacity only).

## Non-goals

- No haptics (Vibration API) — that is [optional-haptics](../../product/backlog/mobile-ux/optional-haptics.md). This story only exposes the seam.
- No hardware back-button interception — that is [native-back-button](../../product/backlog/mobile-ux/native-back-button.md). This story only makes "back" animate when the existing close handlers fire.
- No restyle of the already-animated sheet/dialog/sidebar motion.
- No drag-to-create / pinch-to-zoom (separate stories).
- No router and no change to how `activeModule`/detail state is stored.
- No simultaneous exit+enter "cross-dissolve" of outgoing content (see D2 — enter-transition only; YAGNI without `framer-motion`).

---

## Decision Summary

### D1. Press feedback: `usePressable()` seam + `PRESSABLE` class

One hook is the single integration point for the visual **and** the future haptic pulse.

`src/hooks/use-pressable.ts`:

```ts
// Visual: scale gated by motion-safe (reduced-motion drops it); tint overlay is
// always on (it is feedback, not decoration) → implements the reduced-motion split in CSS alone.
export const PRESSABLE =
  "relative transition-transform duration-100 ease-out motion-safe:active:scale-[0.97] " +
  "before:pointer-events-none before:absolute before:inset-0 before:rounded-[inherit] " +
  "before:bg-foreground before:opacity-0 before:transition-opacity before:duration-100 active:before:opacity-[0.06]";

export function usePressable() {
  // Seam: the optional-haptics story adds `useHaptics().tap()` here, touching ONE file.
  const onPointerDown = useCallback(() => {}, []);
  return { className: PRESSABLE, onPointerDown };
}
```

- **Button** (`src/components/ui/button.tsx`): call `usePressable()` internally and merge — `cn(buttonVariants({ variant, size }), pressable.className, className)` plus `onPointerDown` composed with any caller-supplied handler. Every `<Button>` is covered for free.
- **Raw tappables** — bottom-nav buttons (`mobile-bottom-nav.tsx`), the "More"/overflow rows, sidebar rows, and the list/recipe row buttons + tappable cards: spread `usePressable()` (`className` into their `cn(...)`, `onPointerDown`).
- **Excluded:** text inputs, selects, non-interactive surfaces, and disabled controls (`:active` won't fire; fine).
- Alternative considered: a `<Pressable asChild>` Slot wrapper. Rejected for v1 — the hook avoids wrapping every element and keeps one seam. (Plan may revisit.)

### D2. `ScreenTransition` primitive (Web Animations API, enter-transition)

`src/components/shared/screen-transition.tsx`:

```tsx
type Mode = "fade" | "slide";
export function ScreenTransition({
  token, mode, direction = "forward", children,
}: { token: string | number | null; mode: Mode; direction?: "forward" | "back"; children: React.ReactNode }) {
  const ref = useRef<HTMLDivElement>(null);
  const prev = useRef(token);
  const reduce = usePrefersReducedMotion(); // useMediaQuery("(prefers-reduced-motion: reduce)")
  useLayoutEffect(() => {
    if (prev.current === token) { return; }
    prev.current = token;
    const el = ref.current;
    if (!el || reduce) { return; } // reduced-motion → instant cut
    const kf = mode === "slide"
      ? [{ opacity: 0, transform: `translateX(${direction === "back" ? "-22%" : "22%"})` }, { opacity: 1, transform: "none" }]
      : [{ opacity: 0, transform: "scale(1.012)" }, { opacity: 1, transform: "none" }];
    el.animate(kf, { duration: mode === "slide" ? 280 : 200, easing: "cubic-bezier(0.2, 0, 0, 1)" });
  }, [token, mode, direction, reduce]);
  return <div ref={ref} className="min-h-0 flex-1 flex flex-col">{children}</div>;
}
```

**Enter-transition only** (the *incoming* screen animates in; the outgoing is replaced): this removes the harsh instant pop, is robust against the existing `<Suspense>` (no need to hold outgoing content while the incoming lazy module resolves), and needs no `framer-motion`/`AnimatePresence`. For shared-axis the incoming-from-the-correct-side is the dominant perceived cue, so a slide-in conveys forward/back direction without a simultaneous exit. A full exit+enter cross-dissolve is explicitly deferred (non-goal).

Add `src/hooks/use-prefers-reduced-motion.ts` (thin wrapper over the existing `useMediaQuery`) and export from the barrel, unless an equivalent already exists.

### D3. Module/tab switch → fade-through

Wrap the module render in `App.tsx:168-170`:

```tsx
<main className="flex-1 min-h-0 flex flex-col overflow-hidden">
  <ScreenTransition token={activeModule} mode="fade">
    {renderModule(activeModule)}
  </ScreenTransition>
</main>
```

`token={activeModule}` re-animates on every nav change (incl. `null`/Home). Peers → fade-through, never a directional slide. Lazy modules still show their `<LazyModule>` fallback inside the faded-in container on first visit — acceptable.

### D4. Detail open/close → shared-axis slide

In `lists-view.tsx` and `recipes-view.tsx`, wrap the list-vs-detail output in `ScreenTransition mode="slide"`, keyed on the selection id, with direction derived from the transition:

```tsx
// recipes-view.tsx (shape; lists-view mirrors with selectedListId)
<ScreenTransition token={selectedRecipeId ?? "__list__"} mode="slide"
  direction={selectedRecipeId ? "forward" : "back"}>
  {selectedRecipeId ? <RecipeDetailView … /> : <RecipeGrid … />}
</ScreenTransition>
```

Forward (open) slides in from the right; clearing the selection (the existing `onBack`/`setSelected…(null)`) slides the list back in from the left. `meals-view` is included only if it has an equivalent in-view detail; otherwise skipped.

### D5. Seams for the sibling stories

- **Haptics:** the only edit needed later is inside `usePressable().onPointerDown` (add `haptics.tap()`), and optionally a stronger pulse on destructive `<Button variant="destructive">`. No call-site churn.
- **Back-button:** `native-back-button` will intercept hardware back and call the *same* close handlers (`setSelectedListId(null)`, `setMoreOpen(false)`, etc.). Because D4 keys the slide on selection state, the reverse animation comes for free — that story adds no transition code.

---

## File map

| File | Action | Purpose |
|---|---|---|
| `src/hooks/use-pressable.ts` | Create | `PRESSABLE` class + `usePressable()` (haptic seam) |
| `src/hooks/use-prefers-reduced-motion.ts` | Create | `useMediaQuery("(prefers-reduced-motion: reduce)")` wrapper (if absent) |
| `src/hooks/index.ts` | Modify | Export the two hooks |
| `src/components/shared/screen-transition.tsx` | Create | `ScreenTransition` (fade \| slide, direction, reduced-motion) |
| `src/components/shared/index.ts` | Modify | Export `ScreenTransition` |
| `src/components/ui/button.tsx` | Modify | Apply `usePressable()` to all buttons |
| `src/components/shared/mobile-bottom-nav.tsx` | Modify | `usePressable()` on tab + overflow buttons |
| `src/components/shared/sidebar-menu.tsx` | Modify | `usePressable()` on menu rows |
| `src/App.tsx` | Modify | Wrap `renderModule` in `ScreenTransition mode="fade"` |
| `src/components/lists-view.tsx` | Modify | Slide-wrap list↔detail |
| `src/components/recipes-view.tsx` | Modify | Slide-wrap grid↔detail |
| `src/components/**` (rows/cards) | Modify | Apply `usePressable()` to remaining tappable rows/cards |

## Testing strategy

- **Unit (Vitest):** `usePressable` returns `PRESSABLE` + a callable `onPointerDown` (the seam — assert it is wired so haptics can hook it). `usePrefersReducedMotion` flips on `matchMedia` change.
- **Component (Vitest + Testing Library):** `ScreenTransition` — on `token` change it calls `el.animate` with slide-vs-fade keyframes and the right direction; under mocked `prefers-reduced-motion: reduce` it does **not** animate (instant cut), and content still updates. (Mock `Element.prototype.animate`; jsdom has no real WAAPI.) A regression test that `<Button>` and a nav button carry the press class and fire `onPointerDown`.
- **Reduced-motion split:** assert the scale utility is `motion-safe:`-gated (class present) while the tint (`active:before:opacity-…`) is not — i.e., feedback survives reduced-motion.
- **Manual device pass (the real gate):** Galaxy S10 / Chrome — module switches fade, list/recipe open slides right and back slides left, presses dip + spring at 60fps; then OS "reduce motion" on → cuts are instant but presses still tint. Motion *feel* cannot be asserted in jsdom (same as the bottom-sheet story).
- Follow `frontend/AGENTS.md` test gotchas (store auto-reset, async-form timing).

## Out of scope

Haptics/Vibration API; hardware back-button interception; drag-to-create; pinch-to-zoom; simultaneous exit+enter cross-dissolve; any change to sheet/dialog/sidebar motion, the router (none), or state storage; new animation dependencies.

## Acceptance criteria

(Mirror of the story.)

- Module/tab switches fade-through; list/recipe detail open/close slide on a shared axis (forward right, back left), built on the existing duration/easing tokens — no new motion system, no `framer-motion`.
- Every interactive control (buttons, nav items, navigating rows, tappable cards, icon buttons, FAB) shows an immediate pressed state on `pointerdown`, not hover-only.
- Under `prefers-reduced-motion: reduce`, transitions are instant cuts and press feedback keeps the tint but drops the scale.
- 60fps on the Galaxy S10, transform/opacity only, no layout thrash.
- `usePressable` exposes a single haptic seam and the slide keys on existing close handlers, so haptics and back-button land without per-call-site changes.

## Self-review notes

- **Placeholders:** none — every change names a real file/symbol verified 2026-06-16; primitives have sketches.
- **No new dependency:** confirmed `framer-motion` absent; `ScreenTransition` uses WAAPI, press visual uses Tailwind + a CSS overlay.
- **Reduced-motion** is implemented two ways and both are correct: CSS `motion-safe:` for the press scale, and a JS `reduce` short-circuit in `ScreenTransition`.
- **Ambiguity resolved:** enter-transition only (not exit+enter) — chosen for Suspense-safety and simplicity, stated as a non-goal so it is not mistaken for full shared-axis. Tint always-on, scale motion-gated. `token` uses a `"__list__"` sentinel so list→detail→list animates on both legs.
- **Seam honesty:** this story ships the seams but no haptics and no back-button behavior; both siblings edit exactly one location each.
