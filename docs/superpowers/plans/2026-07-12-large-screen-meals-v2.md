# Large-Screen Meals v2 — Full-Height Board Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the large-screen Meals week board fill the viewport height with richer banner cards, shrink the meal-type rail to ~36px of sideways text, and make image-less meals visually distinct from empty slots.

**Architecture:** The board's `<table>` becomes an ARIA-annotated CSS grid (each row is its own `grid-cols-[2.25rem_repeat(7,minmax(0,1fr))]` grid inside a flex column, so rows can stretch to share viewport height with a 170px floor). A new grid-only `MealGridCard` renders the banner anatomy; the existing `MealSlotCard` keeps serving mobile unchanged, with shared accessible-label logic extracted to `meal-slot-labels.ts`. Three meal-tint tokens power the placeholder bands and rail label colors.

**Tech Stack:** React 19, Tailwind CSS v4 (oklch tokens in `src/index.css`), Vitest + Testing Library, Playwright.

**Spec:** `docs/superpowers/specs/2026-07-12-large-screen-meals-v2-design.md` (root repo)

**Working repo/branch:** ALL code work happens in the FE repo (`frontend/`), branch `feat/large-screen-meals` (PR #287). The currently failing CI check covers v1 layout details this plan replaces — refactor those tests here, do not patch them first.

**Verification gotchas (from repo memory):**
- Type errors are caught ONLY by `npm run build` (`tsc -b`); the lint+test gate misses them. Always run build.
- E2E runs against the real backend: `BE_IMAGE_TAG=<latest released tag> docker compose up` first (compose `latest` default is stale). See `e2e_real_backend_local_run` memory / `e2e/README` if present.

---

### Task 1: Meal-tint tokens

**Files:**
- Modify: `frontend/src/index.css:35` (end of member-color block in `:root`) and `frontend/src/index.css:76` (end of color mappings in `@theme inline`)

- [ ] **Step 1: Add root variables**

In the `:root` block, directly after the `--color-orange` line, add:

```css
  /* Meal-type accents: soft banner tints + AA-contrast glyph/label foregrounds.
     Used only where a meal photo is absent (placeholder band, rail label). */
  --meal-breakfast: oklch(0.93 0.05 80);
  --meal-breakfast-foreground: oklch(0.52 0.11 70);
  --meal-lunch: oklch(0.93 0.05 150);
  --meal-lunch-foreground: oklch(0.48 0.1 150);
  --meal-dinner: oklch(0.93 0.05 285);
  --meal-dinner-foreground: oklch(0.5 0.13 285);
```

(The app currently defines a light palette only — there is no `.dark` block in `index.css` — so no dark variants are added.)

- [ ] **Step 2: Map them in `@theme inline`**

Directly after the `--color-orange: var(--color-orange);` line, add:

```css
  --color-meal-breakfast: var(--meal-breakfast);
  --color-meal-breakfast-foreground: var(--meal-breakfast-foreground);
  --color-meal-lunch: var(--meal-lunch);
  --color-meal-lunch-foreground: var(--meal-lunch-foreground);
  --color-meal-dinner: var(--meal-dinner);
  --color-meal-dinner-foreground: var(--meal-dinner-foreground);
```

This makes `bg-meal-breakfast`, `text-meal-dinner-foreground`, etc. valid utilities.

- [ ] **Step 3: Verify build**

Run: `npm run build`
Expected: exits 0.

- [ ] **Step 4: Commit**

```bash
git add src/index.css
git commit -m "feat(meals): add meal-type accent tokens"
```

---

### Task 2: Meal-type icon and tint utilities

**Files:**
- Modify: `frontend/src/components/meals/meal-type-utils.ts`
- Create: `frontend/src/components/meals/meal-type-utils.test.ts`

- [ ] **Step 1: Write the failing test**

Create `frontend/src/components/meals/meal-type-utils.test.ts`:

```ts
import { CookingPot, Croissant, Sandwich } from "lucide-react";
import { describe, expect, it } from "vitest";
import {
  formatMealType,
  mealTypeBandClasses,
  mealTypeIcon,
  mealTypeRailClasses,
} from "./meal-type-utils";

describe("formatMealType", () => {
  it("capitalizes the meal type", () => {
    expect(formatMealType("breakfast")).toBe("Breakfast");
    expect(formatMealType("lunch")).toBe("Lunch");
    expect(formatMealType("dinner")).toBe("Dinner");
  });
});

describe("mealTypeIcon", () => {
  it("maps each meal type to its glyph", () => {
    expect(mealTypeIcon("breakfast")).toBe(Croissant);
    expect(mealTypeIcon("lunch")).toBe(Sandwich);
    expect(mealTypeIcon("dinner")).toBe(CookingPot);
  });
});

describe("mealTypeBandClasses", () => {
  it("returns the tinted band background and foreground", () => {
    expect(mealTypeBandClasses("breakfast")).toBe(
      "bg-meal-breakfast text-meal-breakfast-foreground",
    );
    expect(mealTypeBandClasses("lunch")).toBe(
      "bg-meal-lunch text-meal-lunch-foreground",
    );
    expect(mealTypeBandClasses("dinner")).toBe(
      "bg-meal-dinner text-meal-dinner-foreground",
    );
  });
});

describe("mealTypeRailClasses", () => {
  it("returns the rail label foreground", () => {
    expect(mealTypeRailClasses("breakfast")).toBe(
      "text-meal-breakfast-foreground",
    );
    expect(mealTypeRailClasses("lunch")).toBe("text-meal-lunch-foreground");
    expect(mealTypeRailClasses("dinner")).toBe("text-meal-dinner-foreground");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- --run src/components/meals/meal-type-utils.test.ts`
Expected: FAIL — `mealTypeIcon` (and the class helpers) are not exported.

- [ ] **Step 3: Implement**

Replace `frontend/src/components/meals/meal-type-utils.ts` with:

```ts
import {
  CookingPot,
  Croissant,
  type LucideIcon,
  Sandwich,
} from "lucide-react";
import type { MealSlot } from "@/lib/types";

export function formatMealType(mealType: MealSlot["mealType"]): string {
  return mealType.charAt(0).toUpperCase() + mealType.slice(1);
}

const MEAL_TYPE_ICONS: Record<MealSlot["mealType"], LucideIcon> = {
  breakfast: Croissant,
  lunch: Sandwich,
  dinner: CookingPot,
};

export function mealTypeIcon(mealType: MealSlot["mealType"]): LucideIcon {
  return MEAL_TYPE_ICONS[mealType];
}

// Tailwind can't see interpolated class names, so each full class string is
// written out literally.
const MEAL_TYPE_BAND_CLASSES: Record<MealSlot["mealType"], string> = {
  breakfast: "bg-meal-breakfast text-meal-breakfast-foreground",
  lunch: "bg-meal-lunch text-meal-lunch-foreground",
  dinner: "bg-meal-dinner text-meal-dinner-foreground",
};

export function mealTypeBandClasses(mealType: MealSlot["mealType"]): string {
  return MEAL_TYPE_BAND_CLASSES[mealType];
}

const MEAL_TYPE_RAIL_CLASSES: Record<MealSlot["mealType"], string> = {
  breakfast: "text-meal-breakfast-foreground",
  lunch: "text-meal-lunch-foreground",
  dinner: "text-meal-dinner-foreground",
};

export function mealTypeRailClasses(mealType: MealSlot["mealType"]): string {
  return MEAL_TYPE_RAIL_CLASSES[mealType];
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test -- --run src/components/meals/meal-type-utils.test.ts`
Expected: PASS (4 test groups).

- [ ] **Step 5: Commit**

```bash
git add src/components/meals/meal-type-utils.ts src/components/meals/meal-type-utils.test.ts
git commit -m "feat(meals): add meal-type icon and tint class helpers"
```

---

### Task 3: Extract shared slot accessible-label helpers

**Files:**
- Create: `frontend/src/components/meals/meal-slot-labels.ts`
- Create: `frontend/src/components/meals/meal-slot-labels.test.ts`
- Modify: `frontend/src/components/meals/meal-slot-card.tsx:39-61,146-150`

The grid card (Task 4) must produce byte-identical accessible labels to the mobile card — E2E and integration tests select by these strings. Extract the label builders so both cards share one source of truth.

- [ ] **Step 1: Write the failing test**

Create `frontend/src/components/meals/meal-slot-labels.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { emptySlotLabel, filledSlotLabel } from "./meal-slot-labels";

describe("filledSlotLabel", () => {
  it("describes a draft slot", () => {
    expect(
      filledSlotLabel({
        mealType: "dinner",
        dayLabel: "Wednesday",
        draftTitle: "Adobo",
        primaryTitle: null,
        firstExtraTitle: null,
      }),
    ).toBe("Draft dinner, Wednesday: Adobo");
  });

  it("describes a primary meal", () => {
    expect(
      filledSlotLabel({
        mealType: "lunch",
        dayLabel: "Monday",
        draftTitle: null,
        primaryTitle: "Sandwich",
        firstExtraTitle: null,
      }),
    ).toBe("Open lunch, Monday: Sandwich");
  });

  it("describes extras-only slots", () => {
    expect(
      filledSlotLabel({
        mealType: "breakfast",
        dayLabel: "Sunday",
        draftTitle: null,
        primaryTitle: null,
        firstExtraTitle: "Fruit",
      }),
    ).toBe("Open breakfast, Sunday: extras - Fruit");
    expect(
      filledSlotLabel({
        mealType: "breakfast",
        dayLabel: "Sunday",
        draftTitle: null,
        primaryTitle: null,
        firstExtraTitle: null,
      }),
    ).toBe("Open breakfast, Sunday: extras");
  });

  it("omits day context when no dayLabel is given", () => {
    expect(
      filledSlotLabel({
        mealType: "lunch",
        draftTitle: null,
        primaryTitle: "Soup",
        firstExtraTitle: null,
      }),
    ).toBe("Open lunch: Soup");
  });
});

describe("emptySlotLabel", () => {
  it("describes an empty slot", () => {
    expect(
      emptySlotLabel({
        mealType: "dinner",
        dayLabel: "Friday",
        hasPendingRecipe: false,
      }),
    ).toBe("Add dinner meal, Friday");
  });

  it("describes recipe placement", () => {
    expect(
      emptySlotLabel({
        mealType: "dinner",
        dayLabel: "Friday",
        hasPendingRecipe: true,
      }),
    ).toBe("Add recipe to dinner, Friday");
  });

  it("omits day context when no dayLabel is given", () => {
    expect(
      emptySlotLabel({ mealType: "lunch", hasPendingRecipe: false }),
    ).toBe("Add lunch meal");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- --run src/components/meals/meal-slot-labels.test.ts`
Expected: FAIL — module does not exist.

- [ ] **Step 3: Implement**

Create `frontend/src/components/meals/meal-slot-labels.ts`:

```ts
import type { MealSlot } from "@/lib/types";

interface FilledSlotLabelInput {
  mealType: MealSlot["mealType"];
  dayLabel?: string;
  draftTitle: string | null;
  primaryTitle: string | null;
  firstExtraTitle: string | null;
}

interface EmptySlotLabelInput {
  mealType: MealSlot["mealType"];
  dayLabel?: string;
  hasPendingRecipe: boolean;
}

function dayContext(dayLabel?: string) {
  return dayLabel ? `, ${dayLabel}` : "";
}

export function filledSlotLabel({
  mealType,
  dayLabel,
  draftTitle,
  primaryTitle,
  firstExtraTitle,
}: FilledSlotLabelInput): string {
  const context = dayContext(dayLabel);
  if (draftTitle !== null) {
    return `Draft ${mealType}${context}: ${draftTitle}`;
  }
  if (primaryTitle !== null) {
    return `Open ${mealType}${context}: ${primaryTitle}`;
  }
  if (firstExtraTitle !== null) {
    return `Open ${mealType}${context}: extras - ${firstExtraTitle}`;
  }
  return `Open ${mealType}${context}: extras`;
}

export function emptySlotLabel({
  mealType,
  dayLabel,
  hasPendingRecipe,
}: EmptySlotLabelInput): string {
  const context = dayContext(dayLabel);
  return hasPendingRecipe
    ? `Add recipe to ${mealType}${context}`
    : `Add ${mealType} meal${context}`;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test -- --run src/components/meals/meal-slot-labels.test.ts`
Expected: PASS.

- [ ] **Step 5: Refactor `MealSlotCard` to use the helpers**

In `frontend/src/components/meals/meal-slot-card.tsx`:

Add to imports:

```ts
import { emptySlotLabel, filledSlotLabel } from "./meal-slot-labels";
```

Delete the line `const dayContext = dayLabel ? `, ${dayLabel}` : "";` (line 39).

Replace the filled-card `aria-label` expression (the nested ternary at lines 53–61) with:

```ts
        aria-label={filledSlotLabel({
          mealType: slot.mealType,
          dayLabel,
          draftTitle: draft ? draft.displayTitle : null,
          primaryTitle: !draft && slot.primary ? slot.primary.title : null,
          firstExtraTitle: firstExtraTitle ?? null,
        })}
```

Replace the empty-slot `aria-label` expression (lines 146–150) with:

```ts
      aria-label={emptySlotLabel({
        mealType: slot.mealType,
        dayLabel,
        hasPendingRecipe: pendingRecipeId !== null,
      })}
```

- [ ] **Step 6: Run the meals suites to verify no label regressed**

Run: `npm test -- --run src/components/meals src/components/meals-view.test.tsx`
Expected: PASS — labels are byte-identical, so no assertion changes.

- [ ] **Step 7: Commit**

```bash
git add src/components/meals/meal-slot-labels.ts src/components/meals/meal-slot-labels.test.ts src/components/meals/meal-slot-card.tsx
git commit -m "refactor(meals): extract shared slot accessible-label helpers"
```

---

### Task 4: `MealGridCard` — banner card for grid cells

**Files:**
- Create: `frontend/src/components/meals/meal-grid-card.tsx`
- Create: `frontend/src/components/meals/meal-grid-card.test.tsx`

- [ ] **Step 1: Write the failing tests**

Create `frontend/src/components/meals/meal-grid-card.test.tsx`:

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, expect, it, vi } from "vitest";
import type { MealSlot } from "@/lib/types";
import { MealGridCard } from "./meal-grid-card";

function buildSlot(overrides: Partial<MealSlot> = {}): MealSlot {
  return {
    id: null,
    weekStartDate: "2026-07-05",
    dayIndex: 3,
    mealType: "dinner",
    primary: null,
    extras: [],
    note: null,
    ...overrides,
  } as MealSlot;
}

describe("MealGridCard", () => {
  it("renders a photo banner and title for a recipe-backed meal", () => {
    const slot = buildSlot({
      primary: {
        id: "p1",
        role: "primary",
        sourceType: "recipe",
        recipeId: "r1",
        title: "Chicken Adobo",
        imageUrl: "https://example.com/adobo.jpg",
        note: "rice night",
      },
    });
    render(
      <MealGridCard
        slot={slot}
        readOnly={false}
        dayLabel="Wednesday"
        onSelectSlot={vi.fn()}
      />,
    );

    const card = screen.getByRole("button", {
      name: "Open dinner, Wednesday: Chicken Adobo",
    });
    expect(card).toBeVisible();
    expect(card.querySelector("img")).toHaveAttribute(
      "src",
      "https://example.com/adobo.jpg",
    );
    expect(screen.getByText("rice night")).toBeVisible();
    // Grid cards carry no meal-type eyebrow; the rail labels the row.
    expect(screen.queryByText("Dinner")).not.toBeInTheDocument();
  });

  it("renders a tinted placeholder band for image-less meals", () => {
    const slot = buildSlot({
      primary: {
        id: "p1",
        role: "primary",
        sourceType: "quick",
        recipeId: null,
        title: "Leftovers",
        imageUrl: null,
        note: null,
      },
    });
    render(
      <MealGridCard
        slot={slot}
        readOnly={false}
        dayLabel="Wednesday"
        onSelectSlot={vi.fn()}
      />,
    );

    const card = screen.getByRole("button", {
      name: "Open dinner, Wednesday: Leftovers",
    });
    const band = card.querySelector('[data-slot="meal-band"]');
    expect(band).not.toBeNull();
    expect(band).toHaveClass("bg-meal-dinner");
    expect(card.querySelector("img")).toBeNull();
    // Filled card, not an add target: solid border, no dashed styling.
    expect(card.className).not.toContain("border-dashed");
  });

  it("shows extras chips with overflow count", () => {
    const slot = buildSlot({
      primary: {
        id: "p1",
        role: "primary",
        sourceType: "quick",
        recipeId: null,
        title: "Pasta",
        imageUrl: null,
        note: null,
      },
      extras: [
        { id: "e1", title: "Garlic bread" },
        { id: "e2", title: "Salad" },
        { id: "e3", title: "Juice" },
      ] as MealSlot["extras"],
    });
    render(
      <MealGridCard
        slot={slot}
        readOnly={false}
        dayLabel="Monday"
        onSelectSlot={vi.fn()}
      />,
    );

    expect(screen.getByText("Garlic bread")).toBeVisible();
    expect(screen.getByText("Salad")).toBeVisible();
    expect(screen.getByText("+1 more")).toBeVisible();
    expect(screen.queryByText("Juice")).not.toBeInTheDocument();
  });

  it("renders a draft badge and hides extras while drafting", () => {
    const slot = buildSlot({
      extras: [{ id: "e1", title: "Rice" }] as MealSlot["extras"],
    });
    render(
      <MealGridCard
        slot={slot}
        readOnly={false}
        dayLabel="Friday"
        draft={{
          target: { dayIndex: 3, mealType: "dinner" },
          displayTitle: "Sinigang",
          displayImageUrl: null,
          displayNote: null,
          primary: {
            sourceType: "quick",
            recipeId: null,
            title: "Sinigang",
            imageUrl: null,
            note: null,
          },
          note: null,
        }}
        onSelectSlot={vi.fn()}
      />,
    );

    expect(
      screen.getByRole("button", { name: "Draft dinner, Friday: Sinigang" }),
    ).toBeVisible();
    expect(screen.getByText("Draft")).toBeVisible();
    expect(screen.queryByText("Rice")).not.toBeInTheDocument();
  });

  it("renders a full-cell add target for empty editable slots", async () => {
    const user = userEvent.setup();
    const onSelectSlot = vi.fn();
    const slot = buildSlot();
    render(
      <MealGridCard
        slot={slot}
        readOnly={false}
        dayLabel="Wednesday"
        onSelectSlot={onSelectSlot}
      />,
    );

    const target = screen.getByRole("button", {
      name: "Add dinner meal, Wednesday",
    });
    expect(target).toBeVisible();
    expect(target.className).toContain("border-dashed");
    expect(screen.getByText("Add dinner")).toBeVisible();

    await user.click(target);
    expect(onSelectSlot).toHaveBeenCalledWith(slot);
  });

  it("labels recipe placement on empty slots", () => {
    const slot = buildSlot();
    render(
      <MealGridCard
        slot={slot}
        readOnly={false}
        dayLabel="Wednesday"
        pendingRecipeId="r9"
        onSelectSlot={vi.fn()}
      />,
    );

    expect(
      screen.getByRole("button", { name: "Add recipe to dinner, Wednesday" }),
    ).toBeVisible();
  });

  it("renders a quiet non-interactive cell for empty read-only slots", () => {
    const slot = buildSlot();
    render(
      <MealGridCard
        slot={slot}
        readOnly
        dayLabel="Wednesday"
        onSelectSlot={vi.fn()}
      />,
    );

    expect(screen.queryByRole("button")).not.toBeInTheDocument();
    expect(screen.getByText("Dinner")).toBeVisible();
  });

  it("marks the active planning target", () => {
    const slot = buildSlot();
    render(
      <MealGridCard
        slot={slot}
        readOnly={false}
        dayLabel="Wednesday"
        isPlanningTarget
        onSelectSlot={vi.fn()}
      />,
    );

    expect(
      screen.getByRole("button", { name: "Add dinner meal, Wednesday" }),
    ).toHaveAttribute("aria-current", "true");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- --run src/components/meals/meal-grid-card.test.tsx`
Expected: FAIL — `./meal-grid-card` does not exist.

- [ ] **Step 3: Implement `MealGridCard`**

Create `frontend/src/components/meals/meal-grid-card.tsx`:

```tsx
import { Plus } from "lucide-react";
import { useState } from "react";
import type { MealSlot } from "@/lib/types";
import { cn } from "@/lib/utils";
import type { MealPlanningDraft } from "./meal-planning-session";
import { emptySlotLabel, filledSlotLabel } from "./meal-slot-labels";
import {
  formatMealType,
  mealTypeBandClasses,
  mealTypeIcon,
} from "./meal-type-utils";

interface MealGridCardProps {
  slot: MealSlot;
  readOnly: boolean;
  pendingRecipeId?: string | null;
  draft?: MealPlanningDraft | null;
  isPlanningTarget?: boolean;
  dayLabel: string;
  onSelectSlot: (slot: MealSlot) => void;
}

// Grid-only banner card. The mobile day-card layout keeps using MealSlotCard;
// the two share label logic via meal-slot-labels.
export function MealGridCard({
  slot,
  readOnly,
  pendingRecipeId = null,
  draft = null,
  isPlanningTarget = false,
  dayLabel,
  onSelectSlot,
}: MealGridCardProps) {
  const [imgFailed, setImgFailed] = useState(false);
  const primary = draft
    ? {
        title: draft.displayTitle,
        imageUrl: draft.displayImageUrl,
        note: draft.displayNote,
      }
    : slot.primary;
  const hasExtras = !draft && slot.extras.length > 0;
  const BandIcon = mealTypeIcon(slot.mealType);

  if (primary || hasExtras) {
    return (
      <button
        type="button"
        className={cn(
          "flex h-full w-full flex-col overflow-hidden rounded-lg border border-border bg-card text-left shadow-sm transition-colors",
          readOnly ? "cursor-default" : "hover:bg-muted/50",
          isPlanningTarget
            ? "border-primary/70 bg-primary/5 ring-2 ring-primary/20"
            : null,
        )}
        aria-current={isPlanningTarget ? "true" : undefined}
        aria-label={filledSlotLabel({
          mealType: slot.mealType,
          dayLabel,
          draftTitle: draft ? draft.displayTitle : null,
          primaryTitle: !draft && slot.primary ? slot.primary.title : null,
          firstExtraTitle: slot.extras[0]?.title ?? null,
        })}
        onClick={() => onSelectSlot(slot)}
      >
        <div className="relative min-h-14 w-full shrink-0 basis-[42%]">
          {primary?.imageUrl && !imgFailed ? (
            <img
              src={primary.imageUrl}
              alt=""
              className="absolute inset-0 h-full w-full object-cover"
              onError={() => setImgFailed(true)}
            />
          ) : (
            <div
              data-slot="meal-band"
              className={cn(
                "flex h-full w-full items-center justify-center",
                mealTypeBandClasses(slot.mealType),
              )}
            >
              <BandIcon aria-hidden className="h-6 w-6 opacity-80" />
            </div>
          )}
        </div>
        <div className="flex min-h-0 flex-1 flex-col gap-1 p-3">
          {draft ? (
            <span className="inline-flex self-start rounded-full bg-primary/10 px-2 py-0.5 text-xs font-semibold text-primary">
              Draft
            </span>
          ) : null}
          <p className="line-clamp-2 text-sm font-semibold text-foreground">
            {primary?.title ?? "Extras"}
          </p>
          {primary?.note ? (
            <p className="line-clamp-2 text-xs text-muted-foreground">
              {primary.note}
            </p>
          ) : null}
          {slot.note ? (
            <p className="line-clamp-2 text-xs italic text-muted-foreground">
              {slot.note}
            </p>
          ) : null}
          {hasExtras ? (
            <div className="mt-auto flex flex-wrap gap-1 pt-1">
              {slot.extras.slice(0, 2).map((extra) => (
                <span
                  key={extra.id}
                  className="rounded-full bg-secondary px-2 py-1 text-xs font-medium text-secondary-foreground"
                >
                  {extra.title}
                </span>
              ))}
              {slot.extras.length > 2 ? (
                <span className="rounded-full bg-secondary px-2 py-1 text-xs font-medium text-secondary-foreground">
                  +{slot.extras.length - 2} more
                </span>
              ) : null}
            </div>
          ) : null}
        </div>
      </button>
    );
  }

  if (readOnly) {
    return (
      <div
        className={cn(
          "flex h-full w-full items-center justify-center rounded-lg border border-dashed border-border bg-muted/30 p-3 text-sm text-muted-foreground",
          isPlanningTarget ? "border-primary/70 ring-2 ring-primary/20" : null,
        )}
        aria-current={isPlanningTarget ? "true" : undefined}
      >
        <span className="font-medium">{formatMealType(slot.mealType)}</span>
      </div>
    );
  }

  return (
    <button
      type="button"
      className={cn(
        "flex h-full min-h-20 w-full flex-col items-center justify-center gap-2 rounded-lg border border-dashed border-border bg-background p-3 transition-colors hover:border-primary/50 hover:bg-primary/5",
        isPlanningTarget
          ? "border-primary/70 bg-primary/5 ring-2 ring-primary/20"
          : null,
      )}
      aria-current={isPlanningTarget ? "true" : undefined}
      aria-label={emptySlotLabel({
        mealType: slot.mealType,
        dayLabel,
        hasPendingRecipe: pendingRecipeId !== null,
      })}
      onClick={() => onSelectSlot(slot)}
    >
      <span className="flex h-8 w-8 items-center justify-center rounded-full bg-primary/10">
        <Plus aria-hidden className="h-4 w-4 text-primary" />
      </span>
      <span className="text-sm font-medium text-muted-foreground">
        Add {slot.mealType}
      </span>
    </button>
  );
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test -- --run src/components/meals/meal-grid-card.test.tsx`
Expected: PASS (8 tests).

- [ ] **Step 5: Commit**

```bash
git add src/components/meals/meal-grid-card.tsx src/components/meals/meal-grid-card.test.tsx
git commit -m "feat(meals): add grid banner slot card with tinted placeholder band"
```

---

### Task 5: Rewrite `MealGrid` as a full-height ARIA grid with a sideways rail

**Files:**
- Modify: `frontend/src/components/meals/meal-grid.tsx` (full rewrite)
- Modify: `frontend/src/components/meals-view.test.tsx` (only if assertions fail — see Step 3)

- [ ] **Step 1: Rewrite the component**

Replace the contents of `frontend/src/components/meals/meal-grid.tsx` with:

```tsx
import { formatLocalDate, parseLocalDate } from "@/lib/time-utils";
import type { MealBoard, MealSlot } from "@/lib/types";
import { cn } from "@/lib/utils";
import { MealGridCard } from "./meal-grid-card";
import type {
  MealPlanningDraft,
  MealPlanningTarget,
} from "./meal-planning-session";
import { formatMealType, mealTypeRailClasses } from "./meal-type-utils";

function shortWeekday(date: string) {
  return parseLocalDate(date).toLocaleDateString("en-US", {
    weekday: "short",
  });
}

function fullWeekday(date: string) {
  return parseLocalDate(date).toLocaleDateString("en-US", {
    weekday: "long",
  });
}

const MEAL_ROWS: Array<MealSlot["mealType"]> = ["breakfast", "lunch", "dinner"];

// One shared column template so header and body rows always align.
const GRID_COLS = "grid grid-cols-[2.25rem_repeat(7,minmax(0,1fr))]";

interface MealGridProps {
  board: MealBoard;
  readOnly: boolean;
  pendingRecipeId?: string | null;
  planningDrafts?: MealPlanningDraft[];
  planningTarget?: MealPlanningTarget | null;
  onSelectSlot: (slot: MealSlot) => void;
}

// ARIA table semantics are attached explicitly: real <table> rows cannot
// stretch to share viewport height, a flex column of CSS grids can.
export function MealGrid({
  board,
  readOnly,
  pendingRecipeId = null,
  planningDrafts = [],
  planningTarget = null,
  onSelectSlot,
}: MealGridProps) {
  const todayIso = formatLocalDate(new Date());

  return (
    <div
      role="table"
      aria-label="Weekly meals"
      className="flex flex-1 flex-col overflow-hidden rounded-lg border border-border"
    >
      <div role="rowgroup">
        <div role="row" className={GRID_COLS}>
          <div
            role="columnheader"
            className="border-b border-r border-border bg-muted/40"
          />
          {board.days.map((day) => {
            const isToday = day.date === todayIso;

            return (
              <div
                key={day.date}
                role="columnheader"
                aria-label={fullWeekday(day.date)}
                aria-current={isToday ? "date" : undefined}
                className={cn(
                  "border-b border-r border-border p-2 text-center text-sm font-semibold text-foreground last:border-r-0",
                  isToday ? "bg-primary/10" : "bg-muted/40",
                )}
              >
                <span className="flex items-center justify-center gap-1.5">
                  <span
                    className={cn(
                      "text-xs font-semibold uppercase",
                      isToday ? "text-primary" : "text-muted-foreground",
                    )}
                  >
                    {shortWeekday(day.date)}
                  </span>
                  <span
                    className={cn(
                      "flex h-6 min-w-6 items-center justify-center rounded-full px-1 text-sm",
                      isToday
                        ? "bg-primary text-primary-foreground"
                        : "text-foreground",
                    )}
                  >
                    {parseLocalDate(day.date).getDate()}
                  </span>
                </span>
              </div>
            );
          })}
        </div>
      </div>
      <div role="rowgroup" className="flex flex-1 flex-col">
        {MEAL_ROWS.map((mealType) => (
          <div
            key={mealType}
            role="row"
            className={cn(
              GRID_COLS,
              "min-h-[170px] flex-1 [&:last-child>*]:border-b-0",
            )}
          >
            <div
              role="rowheader"
              className="flex items-center justify-center border-b border-r border-border bg-muted/30"
            >
              <span
                className={cn(
                  "rotate-180 text-xs font-semibold uppercase tracking-widest [writing-mode:vertical-rl]",
                  mealTypeRailClasses(mealType),
                )}
              >
                {formatMealType(mealType)}
              </span>
            </div>
            {board.days.map((day) => {
              const slot = day.slots.find(
                (candidate) => candidate.mealType === mealType,
              );
              if (!slot) return null;

              return (
                <div
                  key={`${day.date}-${mealType}`}
                  role="cell"
                  className={cn(
                    "border-b border-r border-border p-2 last:border-r-0",
                    day.date === todayIso ? "bg-primary/5" : null,
                  )}
                >
                  <MealGridCard
                    slot={slot}
                    readOnly={readOnly}
                    pendingRecipeId={pendingRecipeId}
                    draft={
                      planningDrafts.find(
                        (draft) =>
                          draft.target.dayIndex === slot.dayIndex &&
                          draft.target.mealType === slot.mealType,
                      ) ?? null
                    }
                    isPlanningTarget={
                      planningTarget?.dayIndex === slot.dayIndex &&
                      planningTarget.mealType === slot.mealType
                    }
                    dayLabel={fullWeekday(day.date)}
                    onSelectSlot={onSelectSlot}
                  />
                </div>
              );
            })}
          </div>
        ))}
      </div>
    </div>
  );
}
```

Notes for the implementer:
- `[&:last-child>*]:border-b-0` removes the bottom border from every cell of the last body row. The explicit arbitrary-selector form is used (not `last:*:border-b-0`) to rule out the `& > *:last-child` misreading, which would instead strip the Saturday cell's border on every row.
- Deliberately NO `min-h-0` on the table root or body rowgroup, deviating from the spec's `flex-1 min-h-0` hint: `min-h-0` would zero out the content's minimum height, so on short viewports the root's `overflow-hidden` would clip the dinner row instead of letting the section scroll. With min-height auto, the 170px row floors propagate up and force the section's `overflow-y-auto` to scroll — which is what the spec's own acceptance criteria require.
- The old `overflow-x-auto` wrapper is gone on purpose: columns are `minmax(0,1fr)`, so the grid can never exceed its container width at lg+.

- [ ] **Step 2: Run the full unit suite**

Run: `npm test -- --run`
Expected: The `meal-grid` / `meals-view` grid tests should pass unchanged — they select via `getByRole("table"/"columnheader"/"rowheader")` and `aria-current`, all of which the ARIA attributes preserve. Note any failures for Step 3.

- [ ] **Step 3: Fix assertions that referenced table internals (only if Step 2 failed)**

Expected possible failures in `frontend/src/components/meals-view.test.tsx`:
- Queries using `within(table).getByText("Breakfast")` may now resolve to the rail rowheader instead of card eyebrows — grid cards no longer render eyebrows. Update such assertions to target the rail (`getByRole("rowheader", { name: "Breakfast" })`) or the specific card by accessible name.
- Any assertion that grid cards show a truncated single-line title (`p.truncate`) should be updated to the new two-line clamp class `line-clamp-2`.

Do NOT weaken accessible-name assertions — labels must remain identical.

- [ ] **Step 4: Visual smoke check**

Run the dev server and view Meals at ≥1024px width: rail labels read bottom-to-top, cards show banners, today column tinted.
Expected: board renders; height still content-driven (full-height chain lands in Task 6).

- [ ] **Step 5: Commit**

```bash
git add src/components/meals/meal-grid.tsx src/components/meals-view.test.tsx
git commit -m "feat(meals): rebuild week board as ARIA grid with sideways rail"
```

---

### Task 6: Full-height layout chain in `MealsView`

**Files:**
- Modify: `frontend/src/components/meals-view.tsx:546-547`

- [ ] **Step 1: Stretch the section and content column**

Replace (line 546):

```tsx
    <section className="flex-1 overflow-y-auto p-4 sm:p-6">
      <div className="mx-auto flex max-w-[1600px] flex-col gap-4">
```

with:

```tsx
    <section className="flex flex-1 flex-col overflow-y-auto p-4 sm:p-6">
      <div className="mx-auto flex w-full max-w-[1600px] flex-1 flex-col gap-4">
```

Why: the section becomes a flex column so the inner content div can take `flex-1`; the inner div needs `w-full` because a flex item with `mx-auto` otherwise shrinks to content width. `MealGrid`'s root already carries `flex-1` (deliberately without `min-h-0` — see Task 5 notes), so it absorbs leftover height on tall viewports, and when the three 170px-minimum rows don't fit, its content minimum pushes past the section height and the existing `overflow-y-auto` scrolls — no extra code.

Note: this wrapper is shared with the mobile path, so mobile's DOM classes do change (flex column + stretched inner div). The effect is a transparent stretched column with identical visual output; Task 7 Step 5's mobile E2E run is the regression proof for the spec's "mobile unchanged" criterion.

- [ ] **Step 2: Run the unit suite**

Run: `npm test -- --run`
Expected: PASS. (Mobile path renders the same children in a taller flex column; no assertions target section classes.)

- [ ] **Step 3: Visual verification at target sizes**

Dev server, editable current week:
- 1280×800, 1440×900, and 1920×1080: board bottom edge reaches the section's bottom padding; three rows share the height; no whitespace pool below the board.
- Window squeezed to ~600px tall: rows stop shrinking at 170px and the module scrolls vertically.
- Below 1024px wide: stacked day cards unchanged.

- [ ] **Step 4: Commit**

```bash
git add src/components/meals-view.tsx
git commit -m "feat(meals): stretch week board to fill large-screen viewport"
```

---

### Task 7: E2E — selectors, vertical-fill assertions, visual matrix

**Files:**
- Modify: `frontend/e2e/large-screen-meals.spec.ts`

Pre-req: backend running per `e2e_real_backend_local_run` memory (`BE_IMAGE_TAG=<latest released tag> docker compose up -d`).

- [ ] **Step 1: Update selectors that assumed a real `<table>`**

In `frontend/e2e/large-screen-meals.spec.ts`, replace every occurrence of:

```ts
table.locator('th[scope="col"][aria-current="date"]')
```

with:

```ts
table.locator('[role="columnheader"][aria-current="date"]')
```

(5 occurrences total: 1 in the geometry test at line 175, 2 as `table.locator(...)` in the visual-matrix test at lines 309 and 338, and 2 chained off `page.getByRole("table"…)` in the long-names and past-week sections at lines 406 and 427.)

In the geometry test, the scroller math read `element.parentElement` when the board sat in an `overflow-x-auto` wrapper. Replace the `geometry` evaluation block (lines 178–201) with:

```ts
      const geometry = await table.evaluate((element) => {
        const box = element.getBoundingClientRect();
        const root = document.documentElement;

        return {
          tableRight: box.right,
          viewportWidth: window.innerWidth,
          tableScrollWidth: element.scrollWidth,
          tableClientWidth: element.clientWidth,
          documentClientWidth: root.clientWidth,
          documentScrollWidth: root.scrollWidth,
        };
      });

      expect(geometry.tableScrollWidth).toBeLessThanOrEqual(
        geometry.tableClientWidth + 1,
      );
      expect(geometry.documentScrollWidth).toBeLessThanOrEqual(
        geometry.documentClientWidth + 1,
      );
      expect(geometry.tableRight).toBeLessThanOrEqual(
        geometry.viewportWidth + 1,
      );
```

- [ ] **Step 2: Replace the eyebrow/truncation metrics in the long-names section**

Delete the `longEyebrow` / `longEyebrowMetrics` block and its three `expect`s (lines 360–379) — grid cards no longer have eyebrows. In their place assert the rail carries the only meal-type text:

```ts
    const boardTable = page.getByRole("table", { name: "Weekly meals" });
    await expect(
      boardTable.getByText("Breakfast", { exact: true }),
    ).toHaveCount(1); // rail only — cards carry no eyebrow
```

Replace the `longTitle` single-line truncation block (lines 380–396) with a two-line clamp assertion:

```ts
    const longTitle = boardTable.locator('[role="cell"] p.line-clamp-2').first();
    const longTitleMetrics = await longTitle.evaluate((element) => {
      const style = window.getComputedStyle(element);
      return {
        clientHeight: element.clientHeight,
        scrollHeight: element.scrollHeight,
        lineHeight: Number.parseFloat(style.lineHeight),
      };
    });
    // Clamped: content overflows, and the visible box is exactly two lines.
    expect(longTitleMetrics.scrollHeight).toBeGreaterThan(
      longTitleMetrics.clientHeight,
    );
    expect(longTitleMetrics.clientHeight).toBeLessThanOrEqual(
      longTitleMetrics.lineHeight * 2 + 2,
    );
```

- [ ] **Step 3: Add the vertical-fill and short-viewport test**

Add this test inside the `test.describe` block, after the geometry test:

```ts
  test("fills the viewport height and floors row height on short screens", async ({
    page,
    request,
  }) => {
    test.setTimeout(60000);
    const registration = await registerFamily(request, {
      familyName: "Meals Vertical Fill",
      members: [{ name: "Sam", color: "teal" }],
    });
    await authenticateAndOpenMeals(page, registration);
    const table = page.getByRole("table", { name: "Weekly meals" });

    // Tall viewports: the board's bottom edge lands within the section's
    // bottom padding — no pooled whitespace below the board.
    for (const [width, height] of [
      [1280, 800],
      [1440, 900],
      [1920, 1080],
    ] as const) {
      await page.setViewportSize({ width, height });
      await expect(table).toBeVisible();
      const box = await table.boundingBox();
      expect(box).not.toBeNull();
      expect(box!.y + box!.height).toBeGreaterThanOrEqual(height - 40);
      expect(box!.y + box!.height).toBeLessThanOrEqual(height + 1);
    }

    // Rail: ~36px wide, vertical labels.
    const rail = table.locator('[role="rowheader"]').first();
    const railBox = await rail.boundingBox();
    expect(railBox).not.toBeNull();
    expect(railBox!.width).toBeLessThanOrEqual(40);

    // Short viewport: rows floor at 170px and the module scrolls instead.
    await page.setViewportSize({ width: 1280, height: 560 });
    const bodyRows = table.locator('[role="rowgroup"]').nth(1).locator('[role="row"]');
    await expect(bodyRows).toHaveCount(3);
    for (let index = 0; index < 3; index += 1) {
      const rowBox = await bodyRows.nth(index).boundingBox();
      expect(rowBox).not.toBeNull();
      expect(rowBox!.height).toBeGreaterThanOrEqual(169);
    }
    const scrolls = await table.evaluate((element) => {
      const section = element.closest("section");
      return section
        ? section.scrollHeight > section.clientHeight + 1
        : false;
    });
    expect(scrolls).toBe(true);
  });
```

- [ ] **Step 4: Run the large-screen spec**

Run: `npm run test:e2e -- e2e/large-screen-meals.spec.ts --project=chromium`
Expected: all tests pass (geometry, breakpoint, vertical fill, visual matrix).

- [ ] **Step 5: Run the mobile spec to prove mobile is untouched**

Run: `npm run test:e2e -- e2e/mobile-meals.spec.ts --project="Mobile Chrome"`
Expected: PASS, no changes needed.

- [ ] **Step 6: Review the visual matrix screenshots**

Open the Playwright report (`npx playwright show-report`) and review all 12 attachments (full / empty / long-names / past × 1024 / 1440 / 1920) against the spec's acceptance criteria: banners render, tinted bands distinct from dashed empties, rail readable, today column highlighted, no clipping.

- [ ] **Step 7: Commit**

```bash
git add e2e/large-screen-meals.spec.ts
git commit -m "test(meals): assert full-height board geometry and refresh visual matrix"
```

---

### Task 8: Final verification and ship

**Files:** none new.

- [ ] **Step 1: Full gate**

```bash
npm run build && npm run lint && npm test -- --run
```

Expected: all pass (build is the only type-check gate — do not skip it).

- [ ] **Step 2: Full E2E (both projects)**

```bash
npm run test:e2e -- e2e/large-screen-meals.spec.ts --project=chromium
npm run test:e2e -- e2e/mobile-meals.spec.ts --project="Mobile Chrome"
```

Expected: PASS.

- [ ] **Step 3: Push and update PR #287**

```bash
git push origin feat/large-screen-meals
```

Update the PR body: add a "v2 revision" section linking the v2 spec (`docs/superpowers/specs/2026-07-12-large-screen-meals-v2-design.md`) and this plan, with the requirement checklist mapped to commits (full-height board, sideways rail, banner cards, tint tokens, distinct placeholder band, refactored E2E). Note that the previously failing CI check was superseded by the refactored tests.

- [ ] **Step 4: Verify CI on the PR**

Run: `gh pr checks 287 --repo joe-bor/FamilyHub --watch`
Expected: all checks green.
