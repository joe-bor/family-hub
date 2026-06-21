# Home Organizer Summary Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Evolve the mobile Home surface from calendar-only "The Now" into an organizer summary by adding a quiet state line (chores left + tonight's dinner) and a "Since you last opened" activity feed for calendar + lists — all derived on-device, no backend.

**Architecture:** A set of **pure functions** (`src/lib/home-activity/`) do the work — normalize calendar events + list summaries into a comparable snapshot, diff against the previous snapshot into coalesced `ActivityItem`s, and group them for display. A thin **orchestration hook** (`use-activity-feed`) wires those pure functions to the existing TanStack queries, an `idb-keyval` store (mirroring the FE #222 offline persister), and `lastSeen`/`hiddenAt` localStorage markers. Two small presentational components (`StateLine`, `ActivityFeed`) render inside the unchanged `HomeDashboard`.

**Tech Stack:** React 19, TanStack Query v5, `idb-keyval` (already a dep), Vitest + @testing-library/react, Tailwind v4. Reuses `getEventKey` (`time-utils.ts`), `usePressable` (`hooks/use-pressable.ts`), the persister DI pattern (`lib/offline/persister.ts`), and `useDashboardNow` (`components/home/hooks/use-hero-state.ts`).

**Spec:** `docs/superpowers/specs/2026-06-20-home-organizer-summary-design.md` (read it before starting; §4 mechanics and §4.4 edge cases are load-bearing).

**Verified codebase facts (do not re-litigate):**
- No entity exposes `updatedAt` → ordering uses on-device `detectedAt`. The snapshot stores all watched fields so "edited" is a field compare, not a timestamp compare.
- Calendar returns **expanded instances with `id: string | null`**; key them with `getEventKey` (`event.id ?? ` `` `${recurringEventId}_${formatLocalDate(date)}` `` ). Coalesce series in the calendar group by `recurringEventId`.
- `useLists()` returns `ListSummary[]` = `{id,name,kind,totalItems,completedItems}` — **no items, no timestamps**. Lists detection = created/renamed/removed + count deltas only.
- The offline persister (FE #222) is a single-key TanStack persister — **not** a reusable KV store. Build a **new** `idb-keyval` store; wire its clear into `useLogout()` and `handleUnauthorized()`.
- The FE is **device-local** (no family-tz helper). Use device-local dates.
- Feed = **calendar + lists** only. Chores + meals appear in the state line, never the feed.

---

## File Structure

**Create:**
- `frontend/src/lib/home-activity/types.ts` — `ActivityModule`, `ActivityKind`, `SnapshotEntry`, `Snapshot`, `ActivityItem`, `ActivityState`, `FeedRow`, `FeedGroup`, `Feed`.
- `frontend/src/lib/home-activity/normalize.ts` — `buildSnapshot(events, lists)` (pure).
- `frontend/src/lib/home-activity/diff.ts` — `diffSnapshots`, `mergeLog`, `reconcileLog`, `pruneLog` (pure).
- `frontend/src/lib/home-activity/group.ts` — `buildFeed` (pure).
- `frontend/src/lib/home-activity/markers.ts` — `isMeaningfulOpen` (pure) + localStorage `get/setLastSeen`, `get/setHiddenAt`.
- `frontend/src/lib/home-activity/store.ts` — `createHomeActivityStore(idb)`, module singleton, `clearHomeActivity()`, constants.
- `frontend/src/lib/home-activity/constants.ts` — windows/gaps/cap/key names.
- `frontend/src/components/home/hooks/use-activity-feed.ts` — orchestration hook.
- `frontend/src/components/home/hooks/use-state-line.ts` — state-line derivation hook.
- `frontend/src/components/home/components/state-line.tsx` — state line UI.
- `frontend/src/components/home/components/activity-feed.tsx` — feed UI (group + rows + divider + empty).
- `*.test.ts(x)` siblings for each module above.

**Modify:**
- `frontend/src/components/home/home-dashboard.tsx` — render `<StateLine/>` after `<HeroCard/>`, `<ActivityFeed/>` after `<ComingUp/>`.
- `frontend/src/api/hooks/use-auth.ts:204` — `await clearHomeActivity()` in `useLogout`.
- `frontend/src/api/client/http-client.ts:48` — `await clearHomeActivity()` in `handleUnauthorized`.
- `frontend/src/test/setup.ts` — reset activity markers/store between tests.

**Conventions:** Vitest. Pure-function tests need no providers. Hook tests use `renderHook` with a `QueryClientProvider` wrapper (fresh `QueryClient` with `gcTime: Infinity`, `retry: false`). idb is dependency-injected (see `persister.test.ts` `fakeIdb()`). Run a single test file with `npm test -- --run src/lib/home-activity/diff.test.ts`.

---

## Task 1: Types + snapshot normalization

**Files:**
- Create: `frontend/src/lib/home-activity/types.ts`
- Create: `frontend/src/lib/home-activity/normalize.ts`
- Test: `frontend/src/lib/home-activity/normalize.test.ts`

- [ ] **Step 1: Write `types.ts`** (no test; pure type declarations consumed by later tasks)

```ts
export type ActivityModule = "calendar" | "lists";
export type ActivityKind = "added" | "edited" | "removed";

/** A comparable snapshot of one tracked entity. Stores every watched field so
 * "edited" is a field compare (the backend exposes no updatedAt). */
export interface SnapshotEntry {
  storeKey: string; // `${module}:${entityKey}` — unique across modules
  module: ActivityModule;
  title: string;
  // calendar fields (undefined for lists)
  date?: string; // formatLocalDate(event.date)
  startTime?: string;
  endTime?: string;
  endDate?: string;
  isAllDay?: boolean;
  location?: string;
  memberId?: string;
  recurringEventId?: string;
  entityId?: string | null; // raw calendar id (may be null) / list id
  // lists fields (undefined for calendar)
  totalItems?: number;
  completedItems?: number;
}

export type Snapshot = Record<string, SnapshotEntry>;

export interface ActivityItem {
  storeKey: string;
  module: ActivityModule;
  kind: ActivityKind;
  title: string;
  detail?: string;
  memberId?: string;
  recurringEventId?: string;
  date?: string;
  entityId?: string | null;
  detectedAt: number;
}

export interface ActivityState {
  snapshot: Snapshot;
  log: ActivityItem[];
  snapshotSavedAt: number;
}

export interface FeedRow {
  storeKey: string;
  kind: ActivityKind;
  label: string; // "Dentist", "Groceries"
  detail?: string; // "Tue 9:00 AM", "+3 items"
  memberId?: string;
  module: ActivityModule;
  date?: string;
  entityId?: string | null;
}

export interface FeedGroup {
  id: string; // "calendar" or `lists:${listId}`
  module: ActivityModule;
  summary: string; // "Calendar · 2 added, 1 changed" | "Groceries · +3 items"
  rows: FeedRow[]; // length 1 → render directly (no expander); >1 → expandable
  newest: number; // max detectedAt in group (for ordering + divider placement)
}

export interface Feed {
  groups: FeedGroup[];
  dividerAfter: number; // index after which the "earlier" divider renders; -1 = no divider
  overflow: number; // count beyond the cap (for "and N more")
}
```

- [ ] **Step 2: Write the failing test** (`normalize.test.ts`)

```ts
import { describe, expect, it } from "vitest";
import type { CalendarEvent, ListSummary } from "@/lib/types";
import { buildSnapshot } from "./normalize";

const ev = (over: Partial<CalendarEvent>): CalendarEvent => ({
  id: "e1", title: "Dentist", startTime: "9:00 AM", endTime: "10:00 AM",
  date: new Date(2026, 5, 23), memberId: "m1", isAllDay: false, ...over,
});
const list = (over: Partial<ListSummary>): ListSummary => ({
  id: "l1", name: "Groceries", kind: "grocery", totalItems: 3, completedItems: 1, ...over,
});

describe("buildSnapshot", () => {
  it("keys a regular event under calendar:<id> with watched fields", () => {
    const snap = buildSnapshot([ev({})], []);
    expect(snap["calendar:e1"]).toMatchObject({
      module: "calendar", title: "Dentist", date: "2026-06-23", memberId: "m1",
    });
  });

  it("keys an expanded instance (id:null) by recurringEventId + date", () => {
    const snap = buildSnapshot(
      [ev({ id: null, recurringEventId: "r9", date: new Date(2026, 5, 24) })], [],
    );
    expect(snap["calendar:r9_2026-06-24"]).toBeDefined();
  });

  it("keys a list under lists:<id> with counts", () => {
    const snap = buildSnapshot([], [list({})]);
    expect(snap["lists:l1"]).toMatchObject({
      module: "lists", title: "Groceries", totalItems: 3, completedItems: 1,
    });
  });
});
```

- [ ] **Step 3: Run it, expect FAIL**

Run: `npm test -- --run src/lib/home-activity/normalize.test.ts`
Expected: FAIL — "buildSnapshot is not a function".

- [ ] **Step 4: Implement `normalize.ts`**

```ts
import { formatLocalDate, getEventKey } from "@/lib/time-utils";
import type { CalendarEvent, ListSummary } from "@/lib/types";
import type { Snapshot, SnapshotEntry } from "./types";

export function buildSnapshot(
  events: CalendarEvent[],
  lists: ListSummary[],
): Snapshot {
  const snap: Snapshot = {};

  for (const e of events) {
    const storeKey = `calendar:${getEventKey(e)}`;
    snap[storeKey] = {
      storeKey,
      module: "calendar",
      title: e.title,
      date: formatLocalDate(e.date),
      startTime: e.startTime,
      endTime: e.endTime,
      endDate: e.endDate ? formatLocalDate(e.endDate) : undefined,
      isAllDay: e.isAllDay,
      location: e.location,
      memberId: e.memberId,
      recurringEventId: e.recurringEventId,
      entityId: e.id,
    };
  }

  for (const l of lists) {
    const storeKey = `lists:${l.id}`;
    snap[storeKey] = {
      storeKey,
      module: "lists",
      title: l.name,
      totalItems: l.totalItems,
      completedItems: l.completedItems,
      entityId: l.id,
    };
  }

  return snap;
}
```

- [ ] **Step 5: Run it, expect PASS**

Run: `npm test -- --run src/lib/home-activity/normalize.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 6: Commit**

```bash
git add frontend/src/lib/home-activity/types.ts frontend/src/lib/home-activity/normalize.ts frontend/src/lib/home-activity/normalize.test.ts
git commit -m "feat(home): normalize calendar+list data into activity snapshot"
```

---

## Task 2: Diff snapshots into ActivityItems

**Files:**
- Create: `frontend/src/lib/home-activity/diff.ts`
- Test: `frontend/src/lib/home-activity/diff.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, expect, it } from "vitest";
import { buildSnapshot } from "./normalize";
import { diffSnapshots } from "./diff";
import type { CalendarEvent, ListSummary } from "@/lib/types";

const ev = (over: Partial<CalendarEvent>): CalendarEvent => ({
  id: "e1", title: "Dentist", startTime: "9:00 AM", endTime: "10:00 AM",
  date: new Date(2026, 5, 23), memberId: "m1", isAllDay: false, ...over,
});
const list = (over: Partial<ListSummary>): ListSummary => ({
  id: "l1", name: "Groceries", kind: "grocery", totalItems: 3, completedItems: 1, ...over,
});
const T = 1_000;

describe("diffSnapshots", () => {
  it("reports an added calendar event with a date/time detail", () => {
    const fresh = buildSnapshot([ev({})], []);
    const items = diffSnapshots(fresh, {}, T);
    expect(items).toHaveLength(1);
    expect(items[0]).toMatchObject({ kind: "added", module: "calendar", title: "Dentist", detectedAt: T });
    expect(items[0].detail).toContain("9:00 AM");
  });

  it("reports an edited event when a watched field changes", () => {
    const prev = buildSnapshot([ev({})], []);
    const fresh = buildSnapshot([ev({ startTime: "5:00 PM" })], []);
    const items = diffSnapshots(fresh, prev, T);
    expect(items[0]).toMatchObject({ kind: "edited" });
  });

  it("reports a removed event using the previous title", () => {
    const prev = buildSnapshot([ev({})], []);
    const items = diffSnapshots({}, prev, T);
    expect(items[0]).toMatchObject({ kind: "removed", title: "Dentist" });
  });

  it("ignores unchanged entities", () => {
    const prev = buildSnapshot([ev({})], []);
    const fresh = buildSnapshot([ev({})], []);
    expect(diffSnapshots(fresh, prev, T)).toHaveLength(0);
  });

  it("summarizes list item additions and completions", () => {
    const prev = buildSnapshot([], [list({ totalItems: 3, completedItems: 1 })]);
    const added = diffSnapshots(buildSnapshot([], [list({ totalItems: 6, completedItems: 1 })]), prev, T);
    expect(added[0].detail).toBe("+3 items");
    const done = diffSnapshots(buildSnapshot([], [list({ totalItems: 3, completedItems: 3 })]), prev, T);
    expect(done[0].detail).toBe("2 checked off");
  });
});
```

- [ ] **Step 2: Run it, expect FAIL** — `npm test -- --run src/lib/home-activity/diff.test.ts` → "diffSnapshots is not a function".

- [ ] **Step 3: Implement `diff.ts` (diffSnapshots)**

```ts
import { parseLocalDate } from "@/lib/time-utils";
import type { ActivityItem, ActivityKind, Snapshot, SnapshotEntry } from "./types";

const CALENDAR_FIELDS: (keyof SnapshotEntry)[] = [
  "title", "date", "startTime", "endTime", "endDate", "isAllDay", "location", "memberId",
];

function calendarChanged(a: SnapshotEntry, b: SnapshotEntry): boolean {
  return CALENDAR_FIELDS.some((f) => a[f] !== b[f]);
}

function listChanged(a: SnapshotEntry, b: SnapshotEntry): boolean {
  return a.title !== b.title || a.totalItems !== b.totalItems || a.completedItems !== b.completedItems;
}

function calendarDetail(e: SnapshotEntry): string {
  if (e.isAllDay) return "all day";
  return e.startTime ? `${affixDay(e.date)} ${e.startTime}`.trim() : (e.date ?? "");
}

function affixDay(date?: string): string {
  if (!date) return "";
  // parseLocalDate (not raw `new Date("yyyy-MM-dd")`) per the repo's TZ convention.
  return parseLocalDate(date).toLocaleDateString(undefined, { weekday: "short" }); // "Tue"
}

function listDetail(fresh: SnapshotEntry, prev?: SnapshotEntry): string {
  if (!prev) return fresh.title; // created
  if (fresh.title !== prev.title) return "renamed";
  const totalDelta = (fresh.totalItems ?? 0) - (prev.totalItems ?? 0);
  const doneDelta = (fresh.completedItems ?? 0) - (prev.completedItems ?? 0);
  if (totalDelta > 0) return `+${totalDelta} items`;
  if (totalDelta < 0) return `${-totalDelta} removed`;
  if (doneDelta > 0) return `${doneDelta} checked off`;
  return "updated";
}

function item(entry: SnapshotEntry, kind: ActivityKind, detail: string, detectedAt: number): ActivityItem {
  return {
    storeKey: entry.storeKey,
    module: entry.module,
    kind,
    title: entry.title,
    detail,
    memberId: entry.memberId,
    recurringEventId: entry.recurringEventId,
    date: entry.date,
    entityId: entry.entityId,
    detectedAt,
  };
}

export function diffSnapshots(fresh: Snapshot, prev: Snapshot, detectedAt: number): ActivityItem[] {
  const out: ActivityItem[] = [];

  for (const key of Object.keys(fresh)) {
    const f = fresh[key];
    const p = prev[key];
    if (!p) {
      out.push(item(f, "added", f.module === "calendar" ? calendarDetail(f) : listDetail(f), detectedAt));
      continue;
    }
    const changed = f.module === "calendar" ? calendarChanged(f, p) : listChanged(f, p);
    if (changed) {
      out.push(item(f, "edited", f.module === "calendar" ? calendarDetail(f) : listDetail(f, p), detectedAt));
    }
  }

  for (const key of Object.keys(prev)) {
    if (!fresh[key]) out.push(item(prev[key], "removed", "", detectedAt));
  }

  return out;
}
```

- [ ] **Step 4: Run it, expect PASS** — `npm test -- --run src/lib/home-activity/diff.test.ts`.

- [ ] **Step 5: Commit**

```bash
git add frontend/src/lib/home-activity/diff.ts frontend/src/lib/home-activity/diff.test.ts
git commit -m "feat(home): diff activity snapshots into added/edited/removed items"
```

---

## Task 3: Merge, reconcile, prune the change log

**Files:**
- Modify: `frontend/src/lib/home-activity/diff.ts` (append functions)
- Test: `frontend/src/lib/home-activity/log.test.ts`

- [ ] **Step 1: Write the failing test** (`log.test.ts`)

```ts
import { describe, expect, it } from "vitest";
import { mergeLog, pruneLog, reconcileLog } from "./diff";
import type { ActivityItem } from "./types";

const mk = (over: Partial<ActivityItem>): ActivityItem => ({
  storeKey: "calendar:e1", module: "calendar", kind: "edited",
  title: "Soccer", detail: "x", detectedAt: 1, ...over,
});

describe("mergeLog", () => {
  it("coalesces repeated deltas for one key into the latest entry", () => {
    const log = [mk({ kind: "edited", detail: "old", detectedAt: 1 })];
    const next = mergeLog(log, [mk({ kind: "edited", detail: "new", detectedAt: 5 })]);
    expect(next).toHaveLength(1);
    expect(next[0]).toMatchObject({ detail: "new", detectedAt: 5 });
  });

  it("keeps 'added' when an add is later edited", () => {
    const log = [mk({ kind: "added", detectedAt: 1 })];
    const next = mergeLog(log, [mk({ kind: "edited", detectedAt: 2 })]);
    expect(next[0].kind).toBe("added");
  });

  it("drops an unseen add that is later removed (net no-op)", () => {
    const log = [mk({ kind: "added", detectedAt: 1 })];
    expect(mergeLog(log, [mk({ kind: "removed", detectedAt: 2 })])).toHaveLength(0);
  });

  it("keeps 'removed' when a prior edit is later removed", () => {
    const log = [mk({ kind: "edited", detectedAt: 1 })];
    const next = mergeLog(log, [mk({ kind: "removed", detectedAt: 2 })]);
    expect(next[0].kind).toBe("removed");
  });
});

describe("reconcileLog", () => {
  it("drops a logged 'added' whose key is gone from fresh (phantom add)", () => {
    const log = [mk({ kind: "added", storeKey: "calendar:e1" })];
    expect(reconcileLog(log, new Set())).toHaveLength(0);
  });

  it("demotes a logged 'edited' to 'removed' when the key vanished", () => {
    const log = [mk({ kind: "edited", storeKey: "calendar:e1" })];
    const next = reconcileLog(log, new Set());
    expect(next[0].kind).toBe("removed");
  });

  it("keeps entries whose key still exists", () => {
    const log = [mk({ storeKey: "calendar:e1" })];
    expect(reconcileLog(log, new Set(["calendar:e1"]))).toHaveLength(1);
  });
});

describe("pruneLog", () => {
  it("drops entries older than maxAge", () => {
    const log = [mk({ detectedAt: 0 }), mk({ storeKey: "calendar:e2", detectedAt: 100 })];
    expect(pruneLog(log, 100, 50)).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run it, expect FAIL** — `npm test -- --run src/lib/home-activity/log.test.ts`.

- [ ] **Step 3: Append to `diff.ts`**

```ts
export function mergeLog(log: ActivityItem[], deltas: ActivityItem[]): ActivityItem[] {
  const byKey = new Map(log.map((i) => [i.storeKey, i]));
  for (const d of deltas) {
    const existing = byKey.get(d.storeKey);
    if (!existing) {
      byKey.set(d.storeKey, d);
      continue;
    }
    if (d.kind === "removed") {
      // An add then a remove within the window net-cancels — no lingering row
      // (spec §4.4 / §9 AC). Otherwise the removal wins.
      if (existing.kind === "added") {
        byKey.delete(d.storeKey);
        continue;
      }
      byKey.set(d.storeKey, { ...d, kind: "removed", detectedAt: Math.max(existing.detectedAt, d.detectedAt) });
      continue;
    }
    // d is added/edited: a prior "added" stays "added"; otherwise take the delta's kind.
    const kind = existing.kind === "added" ? "added" : d.kind;
    byKey.set(d.storeKey, { ...d, kind, detectedAt: Math.max(existing.detectedAt, d.detectedAt) });
  }
  return [...byKey.values()];
}

export function reconcileLog(log: ActivityItem[], freshKeys: Set<string>): ActivityItem[] {
  const out: ActivityItem[] = [];
  for (const i of log) {
    if (freshKeys.has(i.storeKey)) {
      out.push(i);
    } else if (i.kind === "added") {
      // never surfaced as real; drop the phantom
    } else if (i.kind === "edited") {
      out.push({ ...i, kind: "removed" });
    } else {
      out.push(i); // already removed
    }
  }
  return out;
}

export function pruneLog(log: ActivityItem[], now: number, maxAgeMs: number): ActivityItem[] {
  return log.filter((i) => now - i.detectedAt <= maxAgeMs);
}
```

- [ ] **Step 4: Run it, expect PASS.**

- [ ] **Step 5: Commit**

```bash
git add frontend/src/lib/home-activity/diff.ts frontend/src/lib/home-activity/log.test.ts
git commit -m "feat(home): coalesce, reconcile, and prune the activity change log"
```

---

## Task 4: Build the display feed (grouping, divider, cap)

**Files:**
- Create: `frontend/src/lib/home-activity/group.ts`
- Create: `frontend/src/lib/home-activity/constants.ts`
- Test: `frontend/src/lib/home-activity/group.test.ts`

- [ ] **Step 1: Write `constants.ts`**

```ts
export const ACTIVITY_WINDOW_MS = 1000 * 60 * 60 * 48; // 48h rolling window
export const MEANINGFUL_GAP_MS = 1000 * 60 * 60 * 4; // 4h hidden → next open is "meaningful"
export const STALE_RESEED_MS = ACTIVITY_WINDOW_MS; // away longer than the window → silent reseed
export const FEED_ROW_CAP = 20; // max groups shown (post-coalesce)
export const FEED_EVENT_WINDOW_DAYS = 28; // calendar detection window
export const LAST_SEEN_KEY = "family-hub-activity-last-seen";
export const HIDDEN_AT_KEY = "family-hub-activity-hidden-at";
```

- [ ] **Step 2: Write the failing test** (`group.test.ts`)

```ts
import { describe, expect, it } from "vitest";
import { buildFeed } from "./group";
import type { ActivityItem } from "./types";

const cal = (over: Partial<ActivityItem>): ActivityItem => ({
  storeKey: "calendar:e1", module: "calendar", kind: "added", title: "Dentist",
  detail: "Tue 9:00 AM", detectedAt: 10, ...over,
});
const lst = (over: Partial<ActivityItem>): ActivityItem => ({
  storeKey: "lists:l1", module: "lists", kind: "edited", title: "Groceries",
  detail: "+3 items", detectedAt: 20, ...over,
});

describe("buildFeed", () => {
  it("coalesces 2+ calendar changes into one expandable group", () => {
    const feed = buildFeed([cal({ storeKey: "calendar:a" }), cal({ storeKey: "calendar:b", kind: "edited" })], 0, 20);
    const calGroup = feed.groups.find((g) => g.module === "calendar");
    expect(calGroup?.summary).toMatch(/Calendar/);
    expect(calGroup?.rows).toHaveLength(2);
  });

  it("renders a single calendar change directly (one row, no count summary)", () => {
    const feed = buildFeed([cal({})], 0, 20);
    const g = feed.groups[0];
    expect(g.rows).toHaveLength(1);
    expect(g.summary).toContain("Dentist");
  });

  it("collapses a recurring series to one row by recurringEventId", () => {
    const feed = buildFeed(
      [cal({ storeKey: "calendar:r_1", recurringEventId: "r", title: "Soccer", kind: "edited" }),
       cal({ storeKey: "calendar:r_2", recurringEventId: "r", title: "Soccer", kind: "edited" })],
      0, 20,
    );
    const g = feed.groups.find((x) => x.module === "calendar");
    expect(g?.rows).toHaveLength(1);
  });

  it("gives each changed list its own group", () => {
    const feed = buildFeed([lst({ storeKey: "lists:a" }), lst({ storeKey: "lists:b", title: "Camping" })], 0, 20);
    expect(feed.groups.filter((g) => g.module === "lists")).toHaveLength(2);
  });

  it("orders newest-first and places the divider after the last 'new' group", () => {
    const feed = buildFeed([cal({ detectedAt: 30 }), lst({ detectedAt: 10 })], 20, 20);
    expect(feed.groups[0].newest).toBe(30); // calendar (new) first
    expect(feed.dividerAfter).toBe(0); // divider after the one new group
  });

  it("omits the divider when everything is new or everything is earlier", () => {
    expect(buildFeed([cal({ detectedAt: 30 })], 0, 20).dividerAfter).toBe(-1);
    expect(buildFeed([cal({ detectedAt: 5 })], 20, 20).dividerAfter).toBe(-1);
  });

  it("caps groups and reports overflow", () => {
    const many = Array.from({ length: 25 }, (_, i) => lst({ storeKey: `lists:${i}` }));
    const feed = buildFeed(many, 0, 20);
    expect(feed.groups).toHaveLength(20);
    expect(feed.overflow).toBe(5);
  });
});
```

- [ ] **Step 3: Run it, expect FAIL.**

- [ ] **Step 4: Implement `group.ts`**

```ts
import type { ActivityItem, Feed, FeedGroup, FeedRow } from "./types";

const MODULE_RANK: Record<string, number> = { calendar: 0, lists: 1 };

function rowOf(i: ActivityItem): FeedRow {
  return {
    storeKey: i.storeKey, kind: i.kind, label: i.title, detail: i.detail,
    memberId: i.memberId, module: i.module, date: i.date, entityId: i.entityId,
  };
}

function calendarSummary(rows: FeedRow[], items: ActivityItem[]): string {
  const added = items.filter((i) => i.kind === "added").length;
  const changed = items.filter((i) => i.kind === "edited").length;
  const removed = items.filter((i) => i.kind === "removed").length;
  const parts: string[] = [];
  if (added) parts.push(`${added} added`);
  if (changed) parts.push(`${changed} changed`);
  if (removed) parts.push(`${removed} removed`);
  return `Calendar · ${parts.join(", ")}`;
}

export function buildFeed(log: ActivityItem[], lastSeen: number, cap: number): Feed {
  // Calendar: collapse series by recurringEventId, then group all calendar items together.
  const calItems = log.filter((i) => i.module === "calendar");
  const seenSeries = new Set<string>();
  const calRows: FeedRow[] = [];
  const calForSummary: ActivityItem[] = [];
  for (const i of calItems.sort((a, b) => b.detectedAt - a.detectedAt)) {
    if (i.recurringEventId) {
      if (seenSeries.has(i.recurringEventId)) continue;
      seenSeries.add(i.recurringEventId);
    }
    calRows.push(rowOf(i));
    calForSummary.push(i);
  }

  const groups: FeedGroup[] = [];
  if (calRows.length === 1) {
    const i = calForSummary[0];
    groups.push({
      id: "calendar", module: "calendar",
      summary: `${i.title}${i.detail ? ` · ${i.detail}` : ""}`,
      rows: calRows, newest: i.detectedAt,
    });
  } else if (calRows.length > 1) {
    groups.push({
      id: "calendar", module: "calendar",
      summary: calendarSummary(calRows, calForSummary),
      rows: calRows, newest: Math.max(...calForSummary.map((i) => i.detectedAt)),
    });
  }

  // Lists: one group per list.
  for (const i of log.filter((x) => x.module === "lists")) {
    groups.push({
      id: i.storeKey, module: "lists",
      summary: `${i.title}${i.detail ? ` · ${i.detail}` : ""}`,
      rows: [rowOf(i)], newest: i.detectedAt,
    });
  }

  groups.sort((a, b) => b.newest - a.newest || MODULE_RANK[a.module] - MODULE_RANK[b.module] || a.summary.localeCompare(b.summary));

  const overflow = Math.max(0, groups.length - cap);
  const capped = groups.slice(0, cap);

  const newCount = capped.filter((g) => g.newest > lastSeen).length;
  const dividerAfter = newCount > 0 && newCount < capped.length ? newCount - 1 : -1;

  return { groups: capped, dividerAfter, overflow };
}
```

- [ ] **Step 5: Run it, expect PASS.**

- [ ] **Step 6: Commit**

```bash
git add frontend/src/lib/home-activity/group.ts frontend/src/lib/home-activity/constants.ts frontend/src/lib/home-activity/group.test.ts
git commit -m "feat(home): build coalesced activity feed with divider and cap"
```

---

## Task 5: Meaningful-open + markers

**Files:**
- Create: `frontend/src/lib/home-activity/markers.ts`
- Test: `frontend/src/lib/home-activity/markers.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, expect, it } from "vitest";
import { isMeaningfulOpen } from "./markers";
import { MEANINGFUL_GAP_MS } from "./constants";

const day = (d: number) => new Date(2026, 5, d, 12, 0, 0).getTime();

describe("isMeaningfulOpen", () => {
  it("is true on cold start", () => {
    expect(isMeaningfulOpen({ coldStart: true, now: day(1), hiddenAt: day(1), lastSeen: day(1) })).toBe(true);
  });
  it("is true when hidden longer than the gap", () => {
    const now = day(1);
    expect(isMeaningfulOpen({ coldStart: false, now, hiddenAt: now - MEANINGFUL_GAP_MS - 1, lastSeen: now - 10 })).toBe(true);
  });
  it("is true when the local day changed since lastSeen", () => {
    expect(isMeaningfulOpen({ coldStart: false, now: day(2), hiddenAt: day(2) - 1000, lastSeen: day(1) })).toBe(true);
  });
  it("is false for a short same-day peek", () => {
    const now = day(1);
    expect(isMeaningfulOpen({ coldStart: false, now, hiddenAt: now - 5000, lastSeen: now - 6000 })).toBe(false);
  });
});
```

- [ ] **Step 2: Run it, expect FAIL.**

- [ ] **Step 3: Implement `markers.ts`**

```ts
import { formatLocalDate } from "@/lib/time-utils";
import { HIDDEN_AT_KEY, LAST_SEEN_KEY, MEANINGFUL_GAP_MS } from "./constants";

export function isMeaningfulOpen(o: {
  coldStart: boolean; now: number; hiddenAt: number; lastSeen: number;
}): boolean {
  if (o.coldStart) return true;
  if (o.now - o.hiddenAt > MEANINGFUL_GAP_MS) return true;
  return formatLocalDate(new Date(o.now)) !== formatLocalDate(new Date(o.lastSeen));
}

function readNum(key: string): number {
  try {
    const raw = localStorage.getItem(key);
    return raw ? Number(raw) : 0;
  } catch {
    return 0;
  }
}
function writeNum(key: string, value: number): void {
  try {
    localStorage.setItem(key, String(value));
  } catch {
    // storage unavailable (private mode / webview) — markers degrade to 0
  }
}

export const getLastSeen = () => readNum(LAST_SEEN_KEY);
export const setLastSeen = (v: number) => writeNum(LAST_SEEN_KEY, v);
export const getHiddenAt = () => readNum(HIDDEN_AT_KEY);
export const setHiddenAt = (v: number) => writeNum(HIDDEN_AT_KEY, v);
```

- [ ] **Step 4: Run it, expect PASS.**

- [ ] **Step 5: Commit**

```bash
git add frontend/src/lib/home-activity/markers.ts frontend/src/lib/home-activity/markers.test.ts
git commit -m "feat(home): meaningful-open detection + lastSeen/hiddenAt markers"
```

---

## Task 6: Persisted activity store (idb-keyval) + clear

**Files:**
- Modify: `frontend/src/lib/home-activity/constants.ts` (add store names)
- Create: `frontend/src/lib/home-activity/store.ts`
- Test: `frontend/src/lib/home-activity/store.test.ts`

- [ ] **Step 1: Append store constants to `constants.ts`**

```ts
// Reuse the existing offline IndexedDB database; isolate in a new object store.
export const ACTIVITY_DB_NAME = "family-hub-offline";
export const ACTIVITY_STORE_NAME = "home-activity";
export const ACTIVITY_STATE_KEY = "activity-state";
```

- [ ] **Step 2: Write the failing test** (mirror `persister.test.ts` DI fake)

```ts
import { describe, expect, it, vi } from "vitest";
import { createHomeActivityStore, type IdbKeyvalLike } from "./store";
import type { ActivityState } from "./types";

function fakeIdb(): { idb: IdbKeyvalLike; store: Map<string, unknown> } {
  const store = new Map<string, unknown>();
  const idb: IdbKeyvalLike = {
    createStore: vi.fn(() => ({}) as never),
    get: vi.fn(async (k: IDBValidKey) => store.get(String(k)) as never),
    set: vi.fn(async (k: IDBValidKey, v: unknown) => void store.set(String(k), v)),
    del: vi.fn(async (k: IDBValidKey) => void store.delete(String(k))),
    clear: vi.fn(async () => store.clear()),
  };
  return { idb, store };
}

const state: ActivityState = { snapshot: { "lists:l1": { storeKey: "lists:l1", module: "lists", title: "Groceries" } }, log: [], snapshotSavedAt: 5 };

describe("createHomeActivityStore", () => {
  it("round-trips state through idb", async () => {
    const { idb } = fakeIdb();
    const s = createHomeActivityStore(idb);
    await s.saveState(state);
    expect(await s.loadState()).toEqual(state);
  });

  it("returns null when nothing is stored", async () => {
    const s = createHomeActivityStore(fakeIdb().idb);
    expect(await s.loadState()).toBeNull();
  });

  it("clear wipes the store", async () => {
    const { idb } = fakeIdb();
    const s = createHomeActivityStore(idb);
    await s.saveState(state);
    await s.clear();
    expect(await s.loadState()).toBeNull();
  });
});
```

- [ ] **Step 3: Run it, expect FAIL.**

- [ ] **Step 4: Implement `store.ts`** (same guard/DI shape as `persister.ts`)

```ts
import { clear, createStore, del, get, set, type UseStore } from "idb-keyval";
import { ACTIVITY_DB_NAME, ACTIVITY_STATE_KEY, ACTIVITY_STORE_NAME } from "./constants";
import type { ActivityState } from "./types";

export interface IdbKeyvalLike {
  createStore: typeof createStore;
  get: typeof get;
  set: typeof set;
  del: typeof del;
  clear: typeof clear;
}

const defaultIdb: IdbKeyvalLike = { createStore, get, set, del, clear };

export interface HomeActivityStore {
  loadState: () => Promise<ActivityState | null>;
  saveState: (state: ActivityState) => Promise<void>;
  clear: () => Promise<void>;
}

export function createHomeActivityStore(idb: IdbKeyvalLike = defaultIdb): HomeActivityStore {
  let cached: UseStore | null | undefined;
  function store(): UseStore | null {
    if (cached !== undefined) return cached;
    try {
      cached = idb.createStore(ACTIVITY_DB_NAME, ACTIVITY_STORE_NAME);
    } catch {
      cached = null;
    }
    return cached;
  }

  return {
    async loadState() {
      const s = store();
      if (!s) return null;
      try {
        return (await idb.get<ActivityState>(ACTIVITY_STATE_KEY, s)) ?? null;
      } catch {
        return null;
      }
    },
    async saveState(state) {
      const s = store();
      if (!s) return;
      try {
        await idb.set(ACTIVITY_STATE_KEY, state, s);
      } catch {
        // quota/blocked — best-effort
      }
    },
    async clear() {
      const s = store();
      if (!s) return;
      try {
        await idb.clear(s);
      } catch {
        // nothing to leak if unavailable
      }
    },
  };
}

const appStore = createHomeActivityStore();
export const loadActivityState = () => appStore.loadState();
export const saveActivityState = (state: ActivityState) => appStore.saveState(state);

/** Wipe persisted activity. Called from useLogout() and handleUnauthorized();
 * never rejects — prevents one account's activity leaking to the next on a shared device. */
export const clearHomeActivity = () => appStore.clear();
```

- [ ] **Step 5: Run it, expect PASS.**

- [ ] **Step 6: Commit**

```bash
git add frontend/src/lib/home-activity/store.ts frontend/src/lib/home-activity/constants.ts frontend/src/lib/home-activity/store.test.ts
git commit -m "feat(home): persist activity snapshot+log to a dedicated idb store"
```

---

## Task 7: Orchestration hook `use-activity-feed`

**Files:**
- Create: `frontend/src/components/home/hooks/use-activity-feed.ts`
- Test: `frontend/src/components/home/hooks/use-activity-feed.test.tsx`

This hook owns: the wide events query + lists query, the load/visibility effects, the diff→merge→reconcile→prune→persist cycle, meaningful-open + divider, and returns the display `Feed`. Keep IO injectable so it is testable without real IndexedDB/localStorage.

- [ ] **Step 1: Write the failing test** (focused on observable behavior; deps injected)

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { renderHook, waitFor } from "@testing-library/react";
import type { ReactNode } from "react";
import { describe, expect, it, vi } from "vitest";
import type { CalendarEvent, ListSummary } from "@/lib/types";
import { useActivityFeed } from "./use-activity-feed";

vi.mock("@/api", async (orig) => {
  const actual = await orig<typeof import("@/api")>();
  return {
    ...actual,
    useCalendarEvents: () => ({ data: { data: events } }),
    useLists: () => ({ data: { data: lists } }),
  };
});

let events: CalendarEvent[] = [];
let lists: ListSummary[] = [];

function wrapper({ children }: { children: ReactNode }) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false, gcTime: Infinity } } });
  return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
}

describe("useActivityFeed", () => {
  it("surfaces a newly-seen list change as a feed group", async () => {
    events = [];
    lists = [{ id: "l1", name: "Groceries", kind: "grocery", totalItems: 0, completedItems: 0 }];
    const io = makeMemoryIo(); // seeds an empty prior snapshot so the list reads as 'added'
    const { result, rerender } = renderHook(() => useActivityFeed({ io, nowProvider: () => 1000 }), { wrapper });
    await waitFor(() => expect(result.current.feed.groups.length).toBeGreaterThan(0));
    expect(result.current.feed.groups[0].summary).toContain("Groceries");
  });
});

// minimal in-memory IO double for the hook's injected dependencies
function makeMemoryIo() {
  let state: import("@/lib/home-activity/types").ActivityState | null = {
    snapshot: {}, log: [], snapshotSavedAt: 0,
  };
  let lastSeen = 0;
  let hiddenAt = 0;
  return {
    loadState: vi.fn(async () => state),
    saveState: vi.fn(async (s: typeof state) => { state = s; }),
    getLastSeen: () => lastSeen,
    setLastSeen: (v: number) => { lastSeen = v; },
    getHiddenAt: () => hiddenAt,
    setHiddenAt: (v: number) => { hiddenAt = v; },
  };
}
```

- [ ] **Step 2: Run it, expect FAIL.**

- [ ] **Step 3: Implement `use-activity-feed.ts`**

```ts
import { addDays, startOfDay } from "date-fns";
import { useEffect, useMemo, useRef, useState } from "react";
import { useCalendarEvents, useLists } from "@/api";
import {
  ACTIVITY_WINDOW_MS, FEED_EVENT_WINDOW_DAYS, FEED_ROW_CAP, STALE_RESEED_MS,
} from "@/lib/home-activity/constants";
import { diffSnapshots, mergeLog, pruneLog, reconcileLog } from "@/lib/home-activity/diff";
import { buildFeed } from "@/lib/home-activity/group";
import {
  getHiddenAt, getLastSeen, isMeaningfulOpen, setHiddenAt, setLastSeen,
} from "@/lib/home-activity/markers";
import { buildSnapshot } from "@/lib/home-activity/normalize";
import { loadActivityState, saveActivityState } from "@/lib/home-activity/store";
import type { ActivityState, Feed } from "@/lib/home-activity/types";
import { formatLocalDate } from "@/lib/time-utils";

interface ActivityIo {
  loadState: () => Promise<ActivityState | null>;
  saveState: (s: ActivityState) => Promise<void>;
  getLastSeen: () => number;
  setLastSeen: (v: number) => void;
  getHiddenAt: () => number;
  setHiddenAt: (v: number) => void;
}

const defaultIo: ActivityIo = {
  loadState: loadActivityState, saveState: saveActivityState,
  getLastSeen, setLastSeen, getHiddenAt, setHiddenAt,
};

const EMPTY_FEED: Feed = { groups: [], dividerAfter: -1, overflow: 0 };

export function useActivityFeed({
  io = defaultIo,
  nowProvider = () => Date.now(),
}: { io?: ActivityIo; nowProvider?: () => number } = {}) {
  const today = startOfDay(new Date(nowProvider()));
  const range = useMemo(
    () => ({ startDate: formatLocalDate(today), endDate: formatLocalDate(addDays(today, FEED_EVENT_WINDOW_DAYS)) }),
    [today],
  );
  const { data: eventsData } = useCalendarEvents(range);
  const { data: listsData } = useLists();
  const events = eventsData?.data ?? [];
  const lists = listsData?.data ?? [];

  const [feed, setFeed] = useState<Feed>(EMPTY_FEED);
  const [tick, setTick] = useState(0); // bumped on return-to-visible to re-run detection
  const coldStartRef = useRef(true);

  // Detection cycle: runs on data change AND on each return-to-visible.
  useEffect(() => {
    let cancelled = false;
    async function detect() {
      const now = nowProvider();
      const fresh = buildSnapshot(events, lists);
      const prior = await io.loadState();

      // First run (no prior) OR long absence → reseed silently. Without this a
      // brand-new user (or one returning after >48h) would see a full "added"
      // dump of everything currently scheduled instead of "all caught up".
      if (!prior || now - prior.snapshotSavedAt > STALE_RESEED_MS) {
        await io.saveState({ snapshot: fresh, log: [], snapshotSavedAt: now });
        io.setLastSeen(now); // nothing to mark as "new" yet
        coldStartRef.current = false;
        if (!cancelled) setFeed(EMPTY_FEED);
        return;
      }

      const deltas = diffSnapshots(fresh, prior.snapshot, now);
      const freshKeys = new Set(Object.keys(fresh));
      // Merge BEFORE reconcile so an add+remove pair cancels (reconcile-first
      // would drop the add before merge could net it against the remove).
      let log = mergeLog(prior.log, deltas);
      log = reconcileLog(log, freshKeys);
      log = pruneLog(log, now, ACTIVITY_WINDOW_MS);
      await io.saveState({ snapshot: fresh, log, snapshotSavedAt: now });

      const meaningful = isMeaningfulOpen({
        coldStart: coldStartRef.current, now,
        hiddenAt: io.getHiddenAt(), lastSeen: io.getLastSeen(),
      });
      const lastSeen = io.getLastSeen();
      if (!cancelled) setFeed(buildFeed(log, lastSeen, FEED_ROW_CAP));
      if (meaningful) io.setLastSeen(now); // advance divider AFTER computing the feed
      coldStartRef.current = false;
    }
    void detect();
    return () => { cancelled = true; };
    // eslint-disable-next-line react-hooks/exhaustive-deps -- events/lists/tick changes drive re-detection
  }, [events, lists, tick]);

  // Record hide time + re-detect on return-to-visible (one listener, feed-owned).
  useEffect(() => {
    function onVisibility() {
      if (document.visibilityState === "hidden") {
        io.setHiddenAt(nowProvider());
      } else {
        // returning to visible: bump the tick so the detection effect reruns
        setTick((t) => t + 1);
      }
    }
    document.addEventListener("visibilitychange", onVisibility);
    return () => document.removeEventListener("visibilitychange", onVisibility);
  }, [io, nowProvider]);

  return { feed };
}
```

> Note on the visibility listener: the hero's `useDashboardNow` listens to drive `now`; this feed listener is a *separate single-purpose concern* (record `hiddenAt`, re-detect on return). Spec §4.5/§9 were relaxed (2026-06-21 plan review) to allow this rather than mandate one shared listener — two passive, single-purpose listeners are harmless and clearer than coupling the hero hook to the feed. The hidden→`hiddenAt` write is unique to this listener (TanStack's `refetchOnWindowFocus` covers re-detection on *visible*, but not the *hidden* transition).

- [ ] **Step 4: Run it, expect PASS** (the seeded list surfaces as a group).

- [ ] **Step 5: Commit**

```bash
git add frontend/src/components/home/hooks/use-activity-feed.ts frontend/src/components/home/hooks/use-activity-feed.test.tsx
git commit -m "feat(home): orchestration hook wiring diff cycle to queries + storage"
```

---

## Task 8: State-line hook

**Files:**
- Create: `frontend/src/components/home/hooks/use-state-line.ts`
- Test: `frontend/src/components/home/hooks/use-state-line.test.tsx`

- [ ] **Step 1: Write the failing test**

```tsx
import { describe, expect, it, vi } from "vitest";
import { renderHook } from "@testing-library/react";
import { useStateLine } from "./use-state-line";

vi.mock("@/api", () => ({
  useChoresBoard: () => ({ data: { data: { today: { summary: { remaining: 3 } } } } }),
  useMealsBoard: () => ({ data: { data: { days: [{ date: "2026-06-21", slots: [{ mealType: "dinner", primary: { title: "Tacos" } }] }] } } }),
}));

describe("useStateLine", () => {
  it("reports chores remaining and tonight's dinner for the local day", () => {
    const { result } = renderHook(() => useStateLine({ now: new Date(2026, 5, 21) }));
    expect(result.current.choresRemaining).toBe(3);
    expect(result.current.dinnerTitle).toBe("Tacos");
    expect(result.current.isEmpty).toBe(false);
  });
});
```

- [ ] **Step 2: Run it, expect FAIL.**

- [ ] **Step 3: Implement `use-state-line.ts`**

```ts
import { useChoresBoard, useMealsBoard } from "@/api";
import { formatLocalDate, getWeekStartSunday } from "@/lib/time-utils";

export function useStateLine({ now = new Date() }: { now?: Date } = {}) {
  const weekStart = formatLocalDate(getWeekStartSunday(now));
  const todayStr = formatLocalDate(now);
  const { data: chores } = useChoresBoard();
  const { data: meals } = useMealsBoard(weekStart);

  const choresRemaining = chores?.data?.today.summary.remaining ?? 0;

  const day = meals?.data?.days.find((d) => d.date === todayStr);
  const dinnerTitle = day?.slots.find((s) => s.mealType === "dinner")?.primary?.title ?? null;

  const isEmpty = choresRemaining === 0 && !dinnerTitle;
  return { choresRemaining, dinnerTitle, isEmpty };
}
```

- [ ] **Step 4: Run it, expect PASS.**

- [ ] **Step 5: Commit**

```bash
git add frontend/src/components/home/hooks/use-state-line.ts frontend/src/components/home/hooks/use-state-line.test.tsx
git commit -m "feat(home): derive state line (chores remaining + tonight's dinner)"
```

---

## Task 9: `StateLine` component

**Files:**
- Create: `frontend/src/components/home/components/state-line.tsx`
- Test: `frontend/src/components/home/components/state-line.test.tsx`

- [ ] **Step 1: Write the failing test**

```tsx
import { render, screen } from "@testing-library/react";
import { describe, expect, it } from "vitest";
import { StateLine } from "./state-line";

describe("StateLine", () => {
  it("renders both segments", () => {
    render(<StateLine choresRemaining={3} dinnerTitle="Tacos" />);
    expect(screen.getByText(/3 chores left today/i)).toBeInTheDocument();
    expect(screen.getByText(/Tacos/)).toBeInTheDocument();
  });
  it("renders nothing when empty", () => {
    const { container } = render(<StateLine choresRemaining={0} dinnerTitle={null} />);
    expect(container).toBeEmptyDOMElement();
  });
});
```

- [ ] **Step 2: Run it, expect FAIL.**

- [ ] **Step 3: Implement `state-line.tsx`**

```tsx
interface StateLineProps {
  choresRemaining: number;
  dinnerTitle: string | null;
}

export function StateLine({ choresRemaining, dinnerTitle }: StateLineProps) {
  const segments: string[] = [];
  if (choresRemaining > 0) {
    segments.push(`${choresRemaining} chore${choresRemaining === 1 ? "" : "s"} left today`);
  }
  if (dinnerTitle) segments.push(`${dinnerTitle} tonight`);
  if (segments.length === 0) return null;

  return (
    <p className="px-4 pt-2 text-sm text-muted-foreground">{segments.join(" · ")}</p>
  );
}
```

- [ ] **Step 4: Run it, expect PASS.**

- [ ] **Step 5: Commit**

```bash
git add frontend/src/components/home/components/state-line.tsx frontend/src/components/home/components/state-line.test.tsx
git commit -m "feat(home): StateLine component"
```

---

## Task 10: `ActivityFeed` component

**Files:**
- Create: `frontend/src/components/home/components/activity-feed.tsx`
- Test: `frontend/src/components/home/components/activity-feed.test.tsx`

- [ ] **Step 1: Write the failing test**

```tsx
import { fireEvent, render, screen } from "@testing-library/react";
import { describe, expect, it, vi } from "vitest";
import type { Feed } from "@/lib/home-activity/types";
import { ActivityFeed } from "./activity-feed";

const feed: Feed = {
  groups: [
    { id: "calendar", module: "calendar", summary: "Calendar · 2 added", newest: 30, rows: [
      { storeKey: "calendar:a", kind: "added", label: "Dentist", detail: "Tue 9:00 AM", module: "calendar" },
      { storeKey: "calendar:b", kind: "edited", label: "Soccer", detail: "5:00 PM", module: "calendar" },
    ] },
    { id: "lists:l1", module: "lists", summary: "Groceries · +3 items", newest: 10, rows: [
      { storeKey: "lists:l1", kind: "edited", label: "Groceries", detail: "+3 items", module: "lists", entityId: "l1" },
    ] },
  ],
  dividerAfter: 0,
  overflow: 0,
};

describe("ActivityFeed", () => {
  it("renders groups, an expander for multi-row groups, and the divider", () => {
    render(<ActivityFeed feed={feed} onSelectRow={vi.fn()} />);
    expect(screen.getByText("Calendar · 2 added")).toBeInTheDocument();
    expect(screen.getByText(/earlier/i)).toBeInTheDocument();
    expect(screen.queryByText("Dentist")).not.toBeInTheDocument(); // collapsed
    fireEvent.click(screen.getByRole("button", { name: /Calendar/ }));
    expect(screen.getByText("Dentist")).toBeInTheDocument(); // expanded
  });

  it("renders the empty state when there are no groups", () => {
    render(<ActivityFeed feed={{ groups: [], dividerAfter: -1, overflow: 0 }} onSelectRow={vi.fn()} />);
    expect(screen.getByText(/all caught up/i)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run it, expect FAIL.**

- [ ] **Step 3: Implement `activity-feed.tsx`** (reuse `usePressable` on interactive rows)

```tsx
import { useState } from "react";
import { usePressable } from "@/hooks/use-pressable";
import type { Feed, FeedGroup, FeedRow } from "@/lib/home-activity/types";
import { cn } from "@/lib/utils";

interface ActivityFeedProps {
  feed: Feed;
  onSelectRow: (row: FeedRow) => void;
}

export function ActivityFeed({ feed, onSelectRow }: ActivityFeedProps) {
  if (feed.groups.length === 0) {
    return (
      <section className="px-4 pt-6" aria-label="Recent changes">
        <h2 className="text-sm font-medium text-muted-foreground">Since you last opened</h2>
        <p className="pt-2 text-sm text-muted-foreground">You're all caught up.</p>
      </section>
    );
  }

  return (
    <section className="px-4 pt-6" aria-label="Recent changes">
      <h2 className="text-sm font-medium text-muted-foreground">Since you last opened</h2>
      <ul className="pt-2">
        {feed.groups.map((group, index) => (
          <li key={group.id}>
            <ActivityGroup group={group} onSelectRow={onSelectRow} />
            {index === feed.dividerAfter && (
              <div className="my-2 flex items-center gap-2 text-xs text-muted-foreground/70">
                <span className="h-px flex-1 bg-border" /> earlier <span className="h-px flex-1 bg-border" />
              </div>
            )}
          </li>
        ))}
      </ul>
      {feed.overflow > 0 && (
        <p className="pt-1 text-xs text-muted-foreground/70">and {feed.overflow} more</p>
      )}
    </section>
  );
}

function ActivityGroup({ group, onSelectRow }: { group: FeedGroup; onSelectRow: (r: FeedRow) => void }) {
  const [open, setOpen] = useState(false);
  const press = usePressable();
  const expandable = group.rows.length > 1;

  if (!expandable) {
    const row = group.rows[0];
    return (
      <button
        type="button"
        className={cn(press.className, "flex w-full items-center justify-between rounded-2xl py-3 text-left")}
        onPointerDown={press.onPointerDown}
        onClick={() => onSelectRow(row)}
      >
        <span className="text-sm">{group.summary}</span>
      </button>
    );
  }

  return (
    <div>
      <button
        type="button"
        aria-expanded={open}
        className={cn(press.className, "flex w-full items-center justify-between rounded-2xl py-3 text-left")}
        onPointerDown={press.onPointerDown}
        onClick={() => setOpen((v) => !v)}
      >
        <span className="text-sm">{group.summary}</span>
        <span aria-hidden className="text-muted-foreground">{open ? "▾" : "▸"}</span>
      </button>
      {open && (
        <ul className="pl-3">
          {group.rows.map((row) => (
            <li key={row.storeKey}>
              <button
                type="button"
                className="flex w-full items-center justify-between rounded-xl py-2 text-left text-sm"
                onClick={() => onSelectRow(row)}
              >
                <span>{markerFor(row.kind)} {row.label}</span>
                {row.detail && <span className="text-muted-foreground">{row.detail}</span>}
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

function markerFor(kind: FeedRow["kind"]): string {
  return kind === "added" ? "+" : kind === "removed" ? "−" : "~";
}
```

- [ ] **Step 4: Run it, expect PASS.**

- [ ] **Step 5: Commit**

```bash
git add frontend/src/components/home/components/activity-feed.tsx frontend/src/components/home/components/activity-feed.test.tsx
git commit -m "feat(home): ActivityFeed component with expand/collapse + divider"
```

---

## Task 11: Integrate into `HomeDashboard`

**Files:**
- Modify: `frontend/src/components/home/home-dashboard.tsx`
- Test: `frontend/src/components/home/home-dashboard.test.tsx` (extend existing)

- [ ] **Step 1: Add a deep-link handler test** (extend existing dashboard test)

```tsx
// In home-dashboard.test.tsx, add to the existing describe block:
it("renders the state line and activity feed regions", async () => {
  render(<HomeDashboard nowOverride={new Date(2026, 5, 21, 12)} />);
  expect(await screen.findByText(/Since you last opened/i)).toBeInTheDocument();
});
```

- [ ] **Step 2: Run it, expect FAIL** — `npm test -- --run src/components/home/home-dashboard.test.tsx`.

- [ ] **Step 3: Wire the components into `home-dashboard.tsx`**

Add imports:
```tsx
import { useActivityFeed } from "./hooks/use-activity-feed";
import { useStateLine } from "./hooks/use-state-line";
import { StateLine } from "./components/state-line";
import { ActivityFeed } from "./components/activity-feed";
import type { FeedRow } from "@/lib/home-activity/types";
```

Inside the component body (after `heroState`):
```tsx
const stateLine = useStateLine({ now });
const { feed } = useActivityFeed({ nowProvider: () => now.getTime() });

const handleFeedSelect = (row: FeedRow) => {
  if (row.module === "calendar") {
    const match = [...today, ...comingUp].find((e) => getEventKey(e) === row.storeKey.replace("calendar:", ""));
    if (match) {
      handleEventClick(match);
    } else {
      // Event is outside the dashboard's 3-day window → open the Calendar module.
      useAppStore.getState().setActiveModule("calendar");
    }
    return;
  }
  // lists row → switch to the Lists module (list-detail open is owned by ListsView)
  useAppStore.getState().setActiveModule("lists");
};
```
> Import `useAppStore` from `@/stores` (already the module-switch store used by the bottom nav).

In the JSX, after `<HeroCard ... />` add:
```tsx
<StateLine choresRemaining={stateLine.choresRemaining} dinnerTitle={stateLine.dinnerTitle} />
```
After `<ComingUp ... />` add:
```tsx
<ActivityFeed feed={feed} onSelectRow={handleFeedSelect} />
```

> Note: the calendar deep-link only matches events inside the dashboard's 3-day window; rows outside it open the Calendar module (the `setActiveModule("calendar")` fallback above).
>
> **Mobile gating (spec §7/§9):** no new gating code is needed — `App.tsx:123` redirects non-mobile `activeModule === null` → Calendar, so `HomeDashboard` (and therefore the state line + feed) mount **only** on mobile. Add a dashboard test at a desktop width (or assert the `App.tsx` redirect) to lock this acceptance criterion.

- [ ] **Step 4: Run it, expect PASS** + run the full home suite: `npm test -- --run src/components/home`.

- [ ] **Step 5: Commit**

```bash
git add frontend/src/components/home/home-dashboard.tsx frontend/src/components/home/home-dashboard.test.tsx
git commit -m "feat(home): render state line + activity feed in the dashboard"
```

---

## Task 12: Clear activity on logout + 401

**Files:**
- Modify: `frontend/src/api/hooks/use-auth.ts` (`useLogout`, ~line 218)
- Modify: `frontend/src/api/client/http-client.ts` (`handleUnauthorized`, ~line 71)
- Test: `frontend/src/api/hooks/use-auth.test.tsx` (extend), `frontend/src/api/client/http-client.test.ts` (extend if present)

- [ ] **Step 1: Write the failing test** (logout clears activity)

```tsx
// In use-auth.test.tsx:
import * as activityStore from "@/lib/home-activity/store";
it("clears persisted home activity on logout", async () => {
  const spy = vi.spyOn(activityStore, "clearHomeActivity").mockResolvedValue();
  const { result } = renderHook(() => useLogout(), { wrapper });
  await result.current();
  expect(spy).toHaveBeenCalled();
});
```

- [ ] **Step 2: Run it, expect FAIL.**

- [ ] **Step 3: Wire the clears**

In `use-auth.ts`, add the import and the call beside `clearOfflineReadCache()`:
```ts
import { clearHomeActivity } from "@/lib/home-activity/store";
// ...inside useLogout(), after `await clearOfflineReadCache();`
await clearHomeActivity();
```
In `http-client.ts`, beside the existing clear in `handleUnauthorized()`:
```ts
import { clearHomeActivity } from "@/lib/home-activity/store";
// ...after `await clearOfflineReadCache();`
await clearHomeActivity();
```

- [ ] **Step 4: Run it, expect PASS.**

- [ ] **Step 5: Commit**

```bash
git add frontend/src/api/hooks/use-auth.ts frontend/src/api/client/http-client.ts frontend/src/api/hooks/use-auth.test.tsx
git commit -m "fix(home): clear persisted activity on logout and 401"
```

---

## Task 13: Test isolation + full verification

**Files:**
- Modify: `frontend/src/test/setup.ts`

- [ ] **Step 1: Reset activity markers between tests** (markers are localStorage; see the FE store-reset gotcha in CLAUDE.md — new persisted state leaks across tests unless reset)

```ts
// In src/test/setup.ts afterEach (alongside resetAllStores / resetTestQueryClient):
import { HIDDEN_AT_KEY, LAST_SEEN_KEY } from "@/lib/home-activity/constants";
// ...in afterEach:
try {
  localStorage.removeItem(LAST_SEEN_KEY);
  localStorage.removeItem(HIDDEN_AT_KEY);
} catch {
  // jsdom localStorage always present; guard for safety
}
```

- [ ] **Step 2: Run the full unit suite**

Run: `npm test -- --run`
Expected: all green, including the new `src/lib/home-activity/*` and `src/components/home/*` tests.

- [ ] **Step 3: Lint + typecheck + build**

Run: `npm run lint && npm run build`
Expected: no Biome errors; `tsc` clean; Vite build succeeds.

- [ ] **Step 4: Commit**

```bash
git add frontend/src/test/setup.ts
git commit -m "test(home): reset activity markers between tests"
```

- [ ] **Step 5: Manual smoke (optional, recommended before PR)**

Run the app (`npm run dev`), open the mobile home surface, and verify: the state line shows chores/dinner when present; making a list change in another tab then returning to Home surfaces it under "Since you last opened"; a quick background→foreground does not clear the "new" divider; logging out and back in shows an empty feed.

---

## Notes for the implementer

- **Reuse, don't reinvent:** `getEventKey` (composite calendar key), `formatLocalDate`/`getWeekStartSunday` (device-local dates), `usePressable` (press + haptics), and the `createOfflineReadCache` DI shape (mirrored by `createHomeActivityStore`) all already exist. Do not add new date libraries or a second persister abstraction.
- **No backend.** Everything is derived on-device. If you find yourself wanting an `/api/activity` endpoint, stop — that is the deferred AI-phase work (spec §12).
- **Device-local only.** Do not introduce family-timezone math; the FE has no helper for it (spec §10.4).
- **Feed = calendar + lists.** Chores and meals belong to the state line, never the feed (spec anchor 7).
- **Member-color dot (spec §6) is deferred in this plan.** `ActivityItem`/`FeedRow` carry `memberId`, so adding a small leading color dot later is a pure UI change (pass `members` into `ActivityFeed`, map `memberId`→color). Left out of v1 to keep the feed minimal; call it out in the PR so it is a conscious omission, not a silent drop.
