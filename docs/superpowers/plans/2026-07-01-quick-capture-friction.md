# Quick-Capture Friction Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the repeat-tax from the three highest-frequency phone capture flows: list item entry (keep-open multi-add), event time selection (5-minute wheel + duration preservation + nudges), and event location (add the missing input).

**Architecture:** Frontend-only changes in the FamilyHub repo (`frontend/` in the workspace, `joe-bor/FamilyHub` on GitHub). No backend, API-contract, or schema changes — `location` already exists end-to-end except for the form input. Three independent slices: `time-utils` helpers → `TimePicker` → `EventForm` → `ListItemSheet`/`MobileSheet`.

**Tech Stack:** React 19, react-hook-form + Zod, TanStack Query, Vitest + Testing Library, Playwright (real-BE E2E), Biome.

**Spec:** `docs/superpowers/specs/2026-07-01-quick-capture-friction-design.md` (root workspace repo `joe-bor/family-hub`)

**Branch:** `feat/quick-capture-friction` off `main` in the FE repo. Commit after every task. Do not deploy.

---

### Task 1: `time-utils` helpers — minutes→time string and clamped add

**Files:**
- Modify: `src/lib/time-utils.ts`
- Test: `src/lib/time-utils.test.ts` (exists; append)

The event form needs to do arithmetic on `"HH:mm"` 24h strings. `getTimeInMinutes` already exists; add the inverse and a clamped add.

- [ ] **Step 1: Write the failing tests**

Append to `src/lib/time-utils.test.ts`:

```typescript
import { addMinutesToTime, minutesToTime24 } from "./time-utils";

describe("minutesToTime24", () => {
  it("formats minutes since midnight as HH:mm", () => {
    expect(minutesToTime24(0)).toBe("00:00");
    expect(minutesToTime24(9 * 60 + 5)).toBe("09:05");
    expect(minutesToTime24(23 * 60 + 59)).toBe("23:59");
  });

  it("clamps out-of-range input to the same day", () => {
    expect(minutesToTime24(-10)).toBe("00:00");
    expect(minutesToTime24(24 * 60 + 30)).toBe("23:59");
  });
});

describe("addMinutesToTime", () => {
  it("adds minutes to an HH:mm string", () => {
    expect(addMinutesToTime("09:30", 15)).toBe("09:45");
    expect(addMinutesToTime("09:30", -15)).toBe("09:15");
  });

  it("clamps at day boundaries instead of wrapping", () => {
    expect(addMinutesToTime("23:50", 15)).toBe("23:59");
    expect(addMinutesToTime("00:05", -15)).toBe("00:00");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- --run src/lib/time-utils.test.ts`
Expected: FAIL — `minutesToTime24` / `addMinutesToTime` are not exported.

- [ ] **Step 3: Implement the helpers**

Append to `src/lib/time-utils.ts`:

```typescript
/**
 * Format minutes-since-midnight as a 24h "HH:mm" string.
 * Input is clamped to [0, 1439] — callers doing time arithmetic near
 * midnight get a same-day clamp, never a wrap to the next day.
 */
export function minutesToTime24(totalMinutes: number): string {
  const clamped = Math.max(0, Math.min(totalMinutes, 23 * 60 + 59));
  const hours = Math.floor(clamped / 60);
  const minutes = clamped % 60;
  return `${hours.toString().padStart(2, "0")}:${minutes.toString().padStart(2, "0")}`;
}

/**
 * Add (or subtract) minutes to a 24h "HH:mm" string, clamped to the day.
 */
export function addMinutesToTime(time24: string, delta: number): string {
  return minutesToTime24(getTimeInMinutes(time24) + delta);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test -- --run src/lib/time-utils.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/lib/time-utils.ts src/lib/time-utils.test.ts
git commit -m "feat(calendar): add clamped time-arithmetic helpers to time-utils"
```

---

### Task 2: TimePicker — 5-minute wheel steps with off-grid preservation

**Files:**
- Modify: `src/components/ui/time-picker.tsx`
- Test: `src/components/ui/time-picker.test.tsx` (create if absent; check for an existing test file first and append if present)

Today `MINUTES = Array.from({ length: 60 }, (_, i) => i)` (line ~21) renders 60 wheel positions. Replace with 5-minute steps; if the picker opens on an off-grid value (e.g. 9:47 from a Google-synced event), include that exact value as an extra selected position so saving without touching the wheel never silently rounds.

- [ ] **Step 1: Write the failing tests**

In `src/components/ui/time-picker.test.tsx`:

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, expect, it, vi } from "vitest";
import { TimePicker } from "./time-picker";

describe("TimePicker minute granularity", () => {
  it("shows 5-minute steps when the current value is on the grid", async () => {
    const user = userEvent.setup();
    render(<TimePicker value="09:30" onChange={vi.fn()} />);
    await user.click(screen.getByRole("button", { name: /9:30 AM/i }));

    // On-grid minutes present (buttons repeat for infinite scroll — use getAllByRole)
    expect(screen.getAllByRole("button", { name: "35" }).length).toBeGreaterThan(0);
    // 1-minute positions absent
    expect(screen.queryByRole("button", { name: "31" })).not.toBeInTheDocument();
  });

  it("includes an off-grid current value as an extra position", async () => {
    const user = userEvent.setup();
    render(<TimePicker value="09:47" onChange={vi.fn()} />);
    await user.click(screen.getByRole("button", { name: /9:47 AM/i }));

    expect(screen.getAllByRole("button", { name: "47" }).length).toBeGreaterThan(0);
    expect(screen.getAllByRole("button", { name: "45" }).length).toBeGreaterThan(0);
    expect(screen.queryByRole("button", { name: "46" })).not.toBeInTheDocument();
  });

  it("preserves an off-grid value when confirmed untouched", async () => {
    const user = userEvent.setup();
    const onChange = vi.fn();
    render(<TimePicker value="09:47" onChange={onChange} />);
    await user.click(screen.getByRole("button", { name: /9:47 AM/i }));
    await user.click(screen.getByRole("button", { name: "OK" }));

    expect(onChange).toHaveBeenCalledWith("09:47");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- --run src/components/ui/time-picker.test.tsx`
Expected: FAIL — the "31"/"46" queries find buttons (60 positions currently rendered).

- [ ] **Step 3: Implement 5-minute steps**

In `src/components/ui/time-picker.tsx`:

Replace the `MINUTES` constant:

```typescript
const MINUTES = Array.from({ length: 12 }, (_, i) => i * 5); // 0, 5, … 55
```

Inside the `TimePicker` component, derive the minute wheel items when the popover opens (state, not render-time memo, so items do NOT recompute mid-scroll as `minute` changes):

```tsx
const [minuteItems, setMinuteItems] = useState<readonly number[]>(MINUTES);

// Force re-mount of wheel columns when popover opens (existing effect — extend it)
useEffect(() => {
  if (open) {
    setMountKey((k) => k + 1);
    // Snapshot the minute wheel for this open cycle: pure 5-minute grid,
    // plus the current off-grid value (if any) so existing data is never
    // silently rounded. `minute` is read once per open on purpose.
    setMinuteItems(
      MINUTES.includes(minute)
        ? MINUTES
        : [...MINUTES, minute].sort((a, b) => a - b),
    );
  }
  // biome-ignore lint/correctness/useExhaustiveDependencies: `minute` is intentionally sampled only at open
}, [open]);
```

Pass `minuteItems` to the minute `WheelColumn` instead of `MINUTES`:

```tsx
<WheelColumn
  key={`minute-${mountKey}`}
  items={minuteItems}
  value={minute}
  onChange={setMinute}
  formatItem={(m) => m.toString().padStart(2, "0")}
/>
```

No `WheelColumn` changes are needed — it already renders whatever `items` it receives.

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test -- --run src/components/ui/time-picker.test.tsx`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/components/ui/time-picker.tsx src/components/ui/time-picker.test.tsx
git commit -m "feat(calendar): step time-picker minutes by 5, preserving off-grid values"
```

---

### Task 3: EventForm — duration preservation on start-time change

**Files:**
- Modify: `src/components/calendar/components/event-form.tsx`
- Test: `src/components/calendar/components/event-form.test.tsx` (exists; append)

Today `Start Time`'s `TimePicker` does `onChange={(time) => setValue("startTime", time)}` (line ~243) and end time never moves. Change: any start-time change shifts end time by the same delta (clamped to 23:59); end-time changes only affect duration.

Heads-up from `frontend/CLAUDE.md`: event-form tests must pass explicit `defaultValues` (incl. `memberId`) and use `waitForMemberSelected` from `@/test/test-utils` before interacting, or CI races.

- [ ] **Step 1: Write the failing tests**

Append to `src/components/calendar/components/event-form.test.tsx` (reuse the file's existing render helpers/fixtures — `testMembers`, `waitForMemberSelected` — matching its current conventions):

```tsx
describe("duration preservation", () => {
  it("shifts end time by the same delta when start time changes", async () => {
    const user = userEvent.setup();
    render(
      <EventForm
        mode="add"
        defaultValues={{
          memberId: testMembers[0].id,
          startTime: "09:00",
          endTime: "10:00",
        }}
        onSubmit={vi.fn()}
        onCancel={vi.fn()}
      />,
    );
    await waitForMemberSelected(testMembers[0].name);

    // Open start-time picker (trigger shows formatted current value)
    await user.click(screen.getByRole("button", { name: /9:00 AM/i }));
    // Pick hour 11 (first visible instance) and confirm
    await user.click(screen.getAllByRole("button", { name: "11" })[0]);
    await user.click(screen.getByRole("button", { name: "OK" }));

    // Start 09:00 → 11:00 (+2h) ⇒ end 10:00 → 12:00
    expect(screen.getByRole("button", { name: /11:00 AM/i })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /12:00 PM/i })).toBeInTheDocument();
  });

  it("clamps the shifted end time to 11:59 PM", async () => {
    const user = userEvent.setup();
    render(
      <EventForm
        mode="add"
        defaultValues={{
          memberId: testMembers[0].id,
          startTime: "21:00",
          endTime: "22:30",
        }}
        onSubmit={vi.fn()}
        onCancel={vi.fn()}
      />,
    );
    await waitForMemberSelected(testMembers[0].name);

    await user.click(screen.getByRole("button", { name: /9:00 PM/i }));
    await user.click(screen.getAllByRole("button", { name: "11" })[0]);
    await user.click(screen.getByRole("button", { name: "OK" }));

    // Start 21:00 → 23:00; end 22:30 + 2h = 24:30 ⇒ clamped 23:59
    expect(screen.getByRole("button", { name: /11:59 PM/i })).toBeInTheDocument();
  });

  it("leaves start time alone when only end time changes", async () => {
    const user = userEvent.setup();
    render(
      <EventForm
        mode="add"
        defaultValues={{
          memberId: testMembers[0].id,
          startTime: "09:00",
          endTime: "10:00",
        }}
        onSubmit={vi.fn()}
        onCancel={vi.fn()}
      />,
    );
    await waitForMemberSelected(testMembers[0].name);

    await user.click(screen.getByRole("button", { name: /10:00 AM/i }));
    await user.click(screen.getAllByRole("button", { name: "11" })[0]);
    await user.click(screen.getByRole("button", { name: "OK" }));

    expect(screen.getByRole("button", { name: /9:00 AM/i })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /11:00 AM/i })).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- --run src/components/calendar/components/event-form.test.tsx`
Expected: FAIL — end time does not shift (first two tests).

- [ ] **Step 3: Implement duration preservation**

In `src/components/calendar/components/event-form.tsx`:

Add to the imports from `@/lib/time-utils`:

```typescript
import {
  addMinutesToTime,
  getSmartDefaultTimes,
  getTimeInMinutes,
  minutesToTime24,
  parseLocalDate,
} from "@/lib/time-utils";
```

Add a handler above the JSX (near `toggleAllDay`):

```tsx
/**
 * Changing start time preserves the event's duration by shifting end time
 * by the same delta, clamped to 23:59 (single-day form model). If the
 * current times are inverted/equal (mid-edit), fall back to 1 hour.
 */
const handleStartTimeChange = (time: string) => {
  const prevStart = getTimeInMinutes(startTimeValue ?? time);
  const prevEnd = getTimeInMinutes(endTimeValue ?? time);
  const duration = prevEnd > prevStart ? prevEnd - prevStart : 60;
  setValue("startTime", time);
  setValue("endTime", minutesToTime24(getTimeInMinutes(time) + duration), {
    shouldValidate: true,
  });
};
```

Wire it into the start `TimePicker` (replacing the inline `setValue`):

```tsx
<TimePicker
  value={startTimeValue}
  onChange={handleStartTimeChange}
  placeholder="Start time"
  error={!!errors.startTime}
/>
```

End `TimePicker` stays `onChange={(time) => setValue("endTime", time, { shouldValidate: true })}`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test -- --run src/components/calendar/components/event-form.test.tsx`
Expected: PASS (all pre-existing tests too — the smart-defaults tests must not regress).

- [ ] **Step 5: Commit**

```bash
git add src/components/calendar/components/event-form.tsx src/components/calendar/components/event-form.test.tsx
git commit -m "feat(calendar): preserve event duration when start time changes"
```

---

### Task 4: EventForm — ±15 minute nudge buttons

**Files:**
- Modify: `src/components/calendar/components/event-form.tsx`
- Test: `src/components/calendar/components/event-form.test.tsx` (append)

Small `−15` / `+15` buttons under each time field so common adjustments never open the wheel. Nudging start uses `handleStartTimeChange` (so duration preservation applies); nudging end only changes duration.

- [ ] **Step 1: Write the failing tests**

Append to `event-form.test.tsx` inside a new `describe("time nudges")`:

```tsx
describe("time nudges", () => {
  it("nudges start time and shifts end time with it", async () => {
    const user = userEvent.setup();
    render(
      <EventForm
        mode="add"
        defaultValues={{
          memberId: testMembers[0].id,
          startTime: "09:00",
          endTime: "10:00",
        }}
        onSubmit={vi.fn()}
        onCancel={vi.fn()}
      />,
    );
    await waitForMemberSelected(testMembers[0].name);

    await user.click(
      screen.getByRole("button", { name: "Start time later by 15 minutes" }),
    );

    expect(screen.getByRole("button", { name: /9:15 AM/i })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /10:15 AM/i })).toBeInTheDocument();
  });

  it("nudges end time without moving start time", async () => {
    const user = userEvent.setup();
    render(
      <EventForm
        mode="add"
        defaultValues={{
          memberId: testMembers[0].id,
          startTime: "09:00",
          endTime: "10:00",
        }}
        onSubmit={vi.fn()}
        onCancel={vi.fn()}
      />,
    );
    await waitForMemberSelected(testMembers[0].name);

    await user.click(
      screen.getByRole("button", { name: "End time earlier by 15 minutes" }),
    );

    expect(screen.getByRole("button", { name: /9:00 AM/i })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /9:45 AM/i })).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- --run src/components/calendar/components/event-form.test.tsx`
Expected: FAIL — nudge buttons not found.

- [ ] **Step 3: Implement the nudge row**

In `event-form.tsx`, add a handler next to `handleStartTimeChange`:

```tsx
const nudgeTime = (field: "startTime" | "endTime", delta: number) => {
  const current = field === "startTime" ? startTimeValue : endTimeValue;
  if (!current) return;
  const next = addMinutesToTime(current, delta);
  if (field === "startTime") {
    handleStartTimeChange(next);
  } else {
    setValue("endTime", next, { shouldValidate: true });
  }
};
```

Add a small presentational helper in the same file (above `EventForm`):

```tsx
function TimeNudgeRow({
  label,
  onNudge,
}: {
  label: "Start time" | "End time";
  onNudge: (delta: number) => void;
}) {
  return (
    <div className="flex gap-2">
      <Button
        type="button"
        variant="outline"
        size="sm"
        className="h-11 min-w-11 flex-1 bg-transparent px-2 text-xs text-muted-foreground"
        aria-label={`${label} earlier by 15 minutes`}
        onClick={() => onNudge(-15)}
      >
        −15
      </Button>
      <Button
        type="button"
        variant="outline"
        size="sm"
        className="h-11 min-w-11 flex-1 bg-transparent px-2 text-xs text-muted-foreground"
        aria-label={`${label} later by 15 minutes`}
        onClick={() => onNudge(15)}
      >
        +15
      </Button>
    </div>
  );
}
```

Render one row under each picker inside the existing Start/End grid cells:

```tsx
<div className="space-y-2">
  <Label>Start Time</Label>
  <TimePicker
    value={startTimeValue}
    onChange={handleStartTimeChange}
    placeholder="Start time"
    error={!!errors.startTime}
  />
  <TimeNudgeRow label="Start time" onNudge={(d) => nudgeTime("startTime", d)} />
  <FormError message={errors.startTime?.message} />
</div>
<div className="space-y-2">
  <Label>End Time</Label>
  <TimePicker
    value={endTimeValue}
    onChange={(time) => setValue("endTime", time, { shouldValidate: true })}
    placeholder="End time"
    error={!!errors.endTime}
  />
  <TimeNudgeRow label="End time" onNudge={(d) => nudgeTime("endTime", d)} />
  <FormError message={errors.endTime?.message} />
</div>
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test -- --run src/components/calendar/components/event-form.test.tsx`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/components/calendar/components/event-form.tsx src/components/calendar/components/event-form.test.tsx
git commit -m "feat(calendar): add ±15 minute nudge buttons to event time fields"
```

---

### Task 5: EventForm — Location field inside an "Add details" expander

**Files:**
- Modify: `src/components/calendar/components/event-form.tsx`
- Test: `src/components/calendar/components/event-form.test.tsx` (append)

`location` already exists in `eventFormSchema` (max 255), `CreateEventRequest`/`UpdateEventRequest`, the modal's `eventToFormData` (edit prefill), and `calendar-module.tsx` payload mapping (lines ~321/~380). Only the input is missing. Rename the `Add description` expander to `Add details`, reveal Location above Description, and auto-expand when either has content.

- [ ] **Step 1: Write the failing tests**

Append to `event-form.test.tsx`:

```tsx
describe("location field", () => {
  it("reveals Location and Description behind Add details", async () => {
    const user = userEvent.setup();
    render(
      <EventForm
        mode="add"
        defaultValues={{ memberId: testMembers[0].id }}
        onSubmit={vi.fn()}
        onCancel={vi.fn()}
      />,
    );
    await waitForMemberSelected(testMembers[0].name);

    expect(screen.queryByLabelText("Location")).not.toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: /add details/i }));
    expect(screen.getByLabelText("Location")).toBeInTheDocument();
    expect(screen.getByLabelText("Description")).toBeInTheDocument();
  });

  it("submits the entered location", async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();
    render(
      <EventForm
        mode="add"
        defaultValues={{
          memberId: testMembers[0].id,
          title: "Swim class",
        }}
        onSubmit={onSubmit}
        onCancel={vi.fn()}
      />,
    );
    await waitForMemberSelected(testMembers[0].name);

    await user.click(screen.getByRole("button", { name: /add details/i }));
    await user.type(screen.getByLabelText("Location"), "YMCA pool");
    await user.click(screen.getByRole("button", { name: /add event/i }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith(
        expect.objectContaining({ location: "YMCA pool" }),
      );
    });
  });

  it("starts expanded in edit mode when location is present", async () => {
    render(
      <EventForm
        mode="edit"
        defaultValues={{
          memberId: testMembers[0].id,
          title: "Dentist",
          date: "2026-07-01",
          startTime: "09:30",
          endTime: "10:30",
          location: "Smile Dental",
        }}
        onSubmit={vi.fn()}
        onCancel={vi.fn()}
      />,
    );
    await waitForMemberSelected(testMembers[0].name);

    expect(screen.getByLabelText("Location")).toHaveValue("Smile Dental");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- --run src/components/calendar/components/event-form.test.tsx`
Expected: FAIL — no "Add details" button, no Location input.

- [ ] **Step 3: Implement**

In `event-form.tsx`:

Rename the state to match its widened meaning and extend the auto-expand effect:

```tsx
const [showDetails, setShowDetails] = useState(false);

// Auto-expand details if initial values have content (edit mode / re-render)
useEffect(() => {
  if (initialValues.description || initialValues.location) {
    setShowDetails(true);
  }
}, [initialValues.description, initialValues.location]);
```

Replace the collapsible Description block (the `{!showDescription ? … : …}` JSX) with:

```tsx
{/* Details: Location + Description (collapsible) */}
<div className="space-y-2">
  {!showDetails ? (
    <button
      type="button"
      onClick={() => setShowDetails(true)}
      className="flex items-center gap-1 text-sm text-muted-foreground hover:text-foreground transition-colors"
    >
      <ChevronDown className="w-3.5 h-3.5" />
      Add details
    </button>
  ) : (
    <>
      <Label htmlFor="location">Location</Label>
      <Input
        id="location"
        {...register("location")}
        placeholder="Where?"
        maxLength={255}
        className={cn("bg-input", errors.location && "border-destructive")}
        aria-invalid={!!errors.location}
      />
      <FormError message={errors.location?.message} />

      <Label htmlFor="description">Description</Label>
      <textarea
        id="description"
        {...register("description")}
        placeholder="Add notes or details..."
        className={cn(
          "flex min-h-[88px] w-full resize-y rounded-lg border border-input bg-input px-3 py-2 text-[15px] leading-5 placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50",
          errors.description && "border-destructive",
        )}
        maxLength={2000}
        aria-invalid={!!errors.description}
      />
      {descriptionValue && descriptionValue.length > 1900 && (
        <p className="text-xs text-muted-foreground text-right">
          {descriptionValue.length}/2000
        </p>
      )}
      <FormError message={errors.description?.message} />
    </>
  )}
</div>
```

Also update any existing tests that reference "Add description" copy.

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test -- --run src/components/calendar/components/event-form.test.tsx`
Expected: PASS (fix any pre-existing "Add description" copy assertions).

- [ ] **Step 5: Commit**

```bash
git add src/components/calendar/components/event-form.tsx src/components/calendar/components/event-form.test.tsx
git commit -m "feat(calendar): add location input behind Add details expander"
```

---

### Task 6: MobileSheet — configurable cancel label

**Files:**
- Modify: `src/components/ui/mobile-sheet.tsx`
- Test: `src/components/ui/mobile-sheet.test.tsx` (exists; append)

`MobileSheet` hardcodes the header dismiss label to `Cancel` (line ~241). The multi-add session needs it to read `Done` after the first successful add. Additive prop, default unchanged.

- [ ] **Step 1: Write the failing test**

Append to `src/components/ui/mobile-sheet.test.tsx` (follow the file's existing render/setup helpers):

```tsx
it("renders a custom cancel label when provided", () => {
  render(
    <MobileSheet isOpen onClose={vi.fn()} title="Add Item" cancelLabel="Done">
      <p>content</p>
    </MobileSheet>,
  );
  expect(screen.getByRole("button", { name: "Done" })).toBeInTheDocument();
  expect(screen.queryByRole("button", { name: "Cancel" })).not.toBeInTheDocument();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- --run src/components/ui/mobile-sheet.test.tsx`
Expected: FAIL — `cancelLabel` prop does not exist / label still "Cancel".

- [ ] **Step 3: Implement**

In `mobile-sheet.tsx`:

```typescript
export interface MobileSheetProps {
  // …existing props…
  /** Header dismiss label. Defaults to "Cancel". */
  cancelLabel?: string;
}
```

Destructure `cancelLabel = "Cancel"` in the component signature and replace the hardcoded text at the dismiss button (line ~241):

```tsx
{cancelLabel}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test -- --run src/components/ui/mobile-sheet.test.tsx`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/components/ui/mobile-sheet.tsx src/components/ui/mobile-sheet.test.tsx
git commit -m "feat(ui): support custom cancel label on MobileSheet"
```

---

### Task 7: ListItemSheet — keep-open multi-add session

**Files:**
- Modify: `src/components/lists/list-item-sheet.tsx`
- Test: `src/components/lists/list-item-sheet.test.tsx` (exists; append)

In **create mode**, a successful save keeps the sheet open: text clears, input refocuses, category persists, header label flips to `Done` after the first add. Edit mode unchanged. Failed saves keep the typed text (already the case — `onError` never resets the form).

- [ ] **Step 1: Write the failing tests**

Append to `src/components/lists/list-item-sheet.test.tsx`, following the file's existing setup (query-client wrapper, `list` fixture, mocked services). Sketch:

```tsx
describe("multi-add session (create mode)", () => {
  it("keeps the sheet open and clears the text after a successful add", async () => {
    const user = userEvent.setup();
    const onOpenChange = vi.fn();
    renderSheet({ mode: "create", onOpenChange }); // file's existing helper

    await user.type(screen.getByLabelText("Item text"), "Milk");
    await user.click(screen.getByRole("button", { name: "Save item" }));

    await waitFor(() => {
      expect(screen.getByLabelText("Item text")).toHaveValue("");
    });
    expect(onOpenChange).not.toHaveBeenCalledWith(false);
    expect(screen.getByLabelText("Item text")).toHaveFocus();
  });

  it("persists the category selection across adds", async () => {
    const user = userEvent.setup();
    renderSheet({ mode: "create" });

    await user.selectOptions(
      screen.getByLabelText("Category"),
      testCategories[0].id, // file's existing category fixture
    );
    await user.type(screen.getByLabelText("Item text"), "Milk");
    await user.click(screen.getByRole("button", { name: "Save item" }));

    await waitFor(() => {
      expect(screen.getByLabelText("Item text")).toHaveValue("");
    });
    expect(screen.getByLabelText("Category")).toHaveValue(testCategories[0].id);
  });

  it("switches the header dismiss label to Done after the first add", async () => {
    const user = userEvent.setup();
    renderSheet({ mode: "create" });

    expect(screen.getByRole("button", { name: "Cancel" })).toBeInTheDocument();
    await user.type(screen.getByLabelText("Item text"), "Milk");
    await user.click(screen.getByRole("button", { name: "Save item" }));

    await waitFor(() => {
      expect(screen.getByRole("button", { name: "Done" })).toBeInTheDocument();
    });
    expect(screen.queryByRole("button", { name: "Cancel" })).not.toBeInTheDocument();
  });

  it("still closes on save in edit mode", async () => {
    const user = userEvent.setup();
    const onOpenChange = vi.fn();
    renderSheet({ mode: "edit", item: testItem, onOpenChange });

    await user.click(screen.getByRole("button", { name: "Save item" }));

    await waitFor(() => {
      expect(onOpenChange).toHaveBeenCalledWith(false);
    });
  });
});
```

Adapt selectors/fixtures to the file's actual helpers — the behaviors asserted are the contract.

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- --run src/components/lists/list-item-sheet.test.tsx`
Expected: FAIL — sheet closes on create success; no Done label.

- [ ] **Step 3: Implement the session behavior**

In `list-item-sheet.tsx`:

Add session state (near the other `useState` calls):

```tsx
// Multi-add session: count of successful adds since this open cycle began.
// > 0 flips the header dismiss label to "Done" — the added items are already
// saved, so there is nothing left to cancel.
const [addedCount, setAddedCount] = useState(0);
```

Reset it on fresh open — extend the existing fresh-open effect (the `if (!wasOpen && open)` block):

```tsx
setAddedCount(0);
```

Change the create branch of `submit` (currently `onSuccess: () => onOpenChange(false)`):

```tsx
createItem.mutate(
  {
    text: values.text,
    categoryId: values.categoryId ?? null,
  },
  {
    onSuccess: () => {
      setAddedCount((c) => c + 1);
      // Keep the sheet open for the next item: clear text, keep category.
      form.reset({ text: "", categoryId: values.categoryId ?? null });
      form.setFocus("text");
    },
    onError: handleItemError,
  },
);
```

Note: the existing `useEffect` that calls `form.reset` on `[form, item, open]` only fires when those change — it will not clobber the mid-session reset.

Pass the label to the sheet:

```tsx
<MobileSheet
  isOpen={open}
  onClose={() => onOpenChange(false)}
  title={mode === "edit" ? "Edit Item" : "Add Item"}
  cancelLabel={mode === "create" && addedCount > 0 ? "Done" : "Cancel"}
  initialHeight="half"
  headerRight={/* unchanged */}
>
```

The just-added row appearing in the list behind the sheet (existing optimistic mutation) plus the cleared input is the confirmation — add no toast or flash.

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test -- --run src/components/lists/list-item-sheet.test.tsx`
Expected: PASS (including all pre-existing tests — especially the 404-recovery ones).

- [ ] **Step 5: Commit**

```bash
git add src/components/lists/list-item-sheet.tsx src/components/lists/list-item-sheet.test.tsx
git commit -m "feat(lists): keep add-item sheet open for rapid multi-add"
```

---

### Task 8: E2E — grocery multi-add and event-with-location

**Files:**
- Create: `e2e/quick-capture.spec.ts`

Requires the real BE on :8080 (`docker compose -f docker-compose.e2e.yml up -d --wait`). Use the existing helpers in `e2e/helpers/` (`registerFamily`, `seedBrowserAuth`, `safeClick`, `waitForDialogReady`, `waitForHydration`) and mirror the structure of an existing spec (e.g. `e2e/lists.spec.ts`) for setup.

- [ ] **Step 1: Write the two specs**

```typescript
import { expect, test } from "@playwright/test";
import { registerFamily, seedBrowserAuth } from "./helpers/api-helpers";

test.describe("quick capture", () => {
  test("adds three grocery items in one sheet session", async ({
    page,
    request,
  }) => {
    const registration = await registerFamily(request, {
      familyName: "QC Family",
      members: [{ name: "Joe", color: "teal" }],
    });
    await page.goto("/");
    await seedBrowserAuth(page, registration);
    await page.goto("/");

    // Create a grocery list through the UI
    await page.getByRole("button", { name: "Lists" }).click();
    await page.getByRole("button", { name: /create first list|create list/i }).click();
    await page.getByLabel(/list name|name/i).fill("Groceries");
    await page.getByRole("button", { name: /^create/i }).click();
    await page.getByRole("button", { name: /groceries/i }).click();

    // One sheet session, three items
    await page.getByRole("button", { name: "Add item" }).click();
    for (const item of ["Milk", "Eggs", "Bananas"]) {
      await page.getByLabel("Item text").fill(item);
      await page.getByRole("button", { name: "Save item" }).click();
      await expect(page.getByLabel("Item text")).toHaveValue("");
    }
    await page.getByRole("button", { name: "Done" }).click();

    for (const item of ["Milk", "Eggs", "Bananas"]) {
      await expect(page.getByRole("button", { name: item })).toBeVisible();
    }
  });

  test("creates an event with a location and shows it in the detail view", async ({
    page,
    request,
  }) => {
    const registration = await registerFamily(request, {
      familyName: "QC Family 2",
      members: [{ name: "Joe", color: "teal" }],
    });
    await page.goto("/");
    await seedBrowserAuth(page, registration);
    await page.goto("/");

    await page.getByRole("button", { name: "Add event" }).click();
    await page.getByLabel("Event Name").fill("Swim class");
    await page.getByRole("button", { name: /add details/i }).click();
    await page.getByLabel("Location").fill("YMCA pool");
    await page.getByRole("button", { name: /^add event$/i }).click();

    // Open the event from today's view and assert the location renders
    await page.getByRole("button", { name: /swim class/i }).first().click();
    await expect(page.getByText("YMCA pool")).toBeVisible();
  });
});
```

Adjust selectors against the real DOM while writing (existing specs are the source of truth for stable selectors) — the flows asserted are the contract.

- [ ] **Step 2: Run the E2E specs**

```bash
docker compose -f docker-compose.e2e.yml up -d --wait
E2E_REAL_API=true npx playwright test e2e/quick-capture.spec.ts
```

Expected: 2 passed.

- [ ] **Step 3: Commit**

```bash
git add e2e/quick-capture.spec.ts
git commit -m "test(e2e): cover grocery multi-add and event location quick-capture flows"
```

---

### Task 9: Full verification and PR

- [ ] **Step 1: Run the full gate**

```bash
npm run lint && npm test -- --run && npm run build
```

Expected: lint clean, all unit tests pass, build succeeds.

- [ ] **Step 2: Manual smoke on mobile viewport**

Run `npm run dev`, open http://localhost:5173 at a ~390px viewport with the local BE running, and verify against the spec's success criteria:
- Add 3+ list items in one sheet session; Done closes it.
- Create an event: start-time change drags end time along; ±15 nudges work; a 9:47 edit shows 47 on the wheel and saves unchanged if untouched.
- Add a location through Add details; confirm it renders in the event detail.

- [ ] **Step 3: Open the PR**

```bash
git push -u origin feat/quick-capture-friction
gh pr create --repo joe-bor/FamilyHub \
  --title "feat: quick-capture friction fixes (lists multi-add, time picker, location)" \
  --body "Closes #<issue>. See issue for the execution contract and acceptance checklist."
```

Include in the PR body the final checklist mapping each non-negotiable requirement (from the Issue) to code and tests. No AI attribution.
