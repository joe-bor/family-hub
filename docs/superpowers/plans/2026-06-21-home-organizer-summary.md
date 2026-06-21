# Home Organizer Summary Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Evolve the mobile Home surface from calendar-only "The Now" into an organizer summary by adding a quiet state line (chores left + tonight's dinner) and a "Since you last opened" activity feed for calendar + lists — all derived on-device, no backend.

**Architecture:** A set of **pure functions** (`src/lib/home-activity/`) do the work — normalize calendar events + list summaries into a comparable snapshot, diff against the previous snapshot (window-bounds-aware) into coalesced `ActivityItem`s, and group them for display. A thin **orchestration hook** (`use-activity-feed`) wires those pure functions to the existing TanStack queries, an `idb-keyval` store in its **own database** (DI pattern mirroring the FE #222 offline persister — but a separate DB, see Task 6), and `lastSeen`/`hiddenAt` localStorage markers. It gates detection until both queries settle, serializes cycles through a promise-queue mutex, classifies a meaningful open only on cold-start/visible-transition, and refetches on return-to-visible. Two small presentational components (`StateLine`, `ActivityFeed`) render inside the unchanged `HomeDashboard`, which is mounted **only on mobile** by a synchronous `useIsMobile` gate.

**Tech Stack:** React 19, TanStack Query v5, `idb-keyval` (already a dep), Vitest + @testing-library/react, Tailwind v4. Reuses `getEventKey` (`time-utils.ts`), `usePressable` (`hooks/use-pressable.ts`), the persister DI pattern (`lib/offline/persister.ts`), `useFamilyMemberMap` (`api/hooks/use-family.ts`, member-color dot), and the app-store navigation-intent pattern (`stores/app-store.ts`, deep-links). The feed owns a **new** single-purpose `visibilitychange` listener (records `hiddenAt`, refetches + re-detects on visible) — this is *not* a reuse of `useDashboardNow`, which has its own separate visibility listener; the two passive listeners coexist by design (spec §4.5).

**Spec:** `docs/superpowers/specs/2026-06-20-home-organizer-summary-design.md` (read it before starting; §4 mechanics and §4.4 edge cases are load-bearing).

**Verified codebase facts (do not re-litigate):**
- No entity exposes `updatedAt` → ordering uses on-device `detectedAt`. The snapshot stores all watched fields so "edited" is a field compare, not a timestamp compare.
- Calendar returns **expanded instances with `id: string | null`**; key them with `getEventKey` (`event.id ?? ` `` `${recurringEventId}_${formatLocalDate(date)}` `` ). Coalesce series in the calendar group by `(recurringEventId, kind, detail)` — **not `recurringEventId` alone**, which would hide a second, different change to the same series.
- `useLists()` returns `ListSummary[]` = `{id,name,kind,totalItems,completedItems}` — **no items, no timestamps**. Lists detection = created/renamed/removed + count deltas only. A bare `completedItems` *decrease* (un-check) is **not** surfaced (no generic "updated" signal).
- The offline persister (FE #222) is a single-key TanStack persister — **not** a reusable KV store. Build a **new** `idb-keyval` store in its **own database** (`family-hub-home-activity`, *not* the shared `family-hub-offline` DB — `idb-keyval.createStore` opens with no version, so it cannot add a store to an existing DB). Wire its clear (IDB **+ both localStorage markers**) into `useLogout()` and `handleUnauthorized()`.
- `refetchOnWindowFocus` is **off** (`providers/query-client.ts`) → return-to-visible must `refetch()` explicitly before diffing.
- `App.tsx` renders `HomeDashboard` for `activeModule === null` **before** its desktop-redirect `useEffect` → gate the organizer subtree **synchronously** with `useIsMobile`, not via the redirect.
- The calendar detection window `[today, +W]` **slides daily** → the diff is **window-bounds-aware** (snapshot stores its window; diff only over the overlap) so rollover does not fabricate added/removed at the edges.
- `useFamilyMemberMap()` (`api/hooks/use-family.ts`) already resolves `memberId → color`; the member-color dot is in v1, not deferred.
- The app-store already has a navigation-intent pattern (`start*Draft`/`consume*Draft`); reuse it for entity deep-links.
- The FE is **device-local** (no family-tz helper). Use device-local dates.
- Feed = **calendar + lists** only. Chores + meals appear in the state line, never the feed.

---

## File Structure

**Create:**
- `frontend/src/lib/home-activity/types.ts` — `ActivityModule`, `ActivityKind`, `SnapshotEntry`, `Snapshot`, `ActivityItem`, `EventWindow`, `ActivityState`, `FeedRow`, `FeedGroup`, `Feed`.
- `frontend/src/lib/home-activity/normalize.ts` — `buildSnapshot(events, lists)` (pure).
- `frontend/src/lib/home-activity/diff.ts` — `diffSnapshots` (window-aware), `mergeLog`, `reconcileLog`, `pruneLog` (pure).
- `frontend/src/lib/home-activity/group.ts` — `buildFeed` (pure; signature-collapse, sub-row cap).
- `frontend/src/lib/home-activity/navigation.ts` — `resolveFeedSelection` (pure deep-link resolver).
- `frontend/src/lib/home-activity/markers.ts` — `isMeaningfulOpen` (pure) + localStorage `get/setLastSeen`, `get/setHiddenAt`.
- `frontend/src/lib/home-activity/store.ts` — `createHomeActivityStore(idb)`, module singleton, `clearHomeActivity()` (store + markers).
- `frontend/src/lib/home-activity/constants.ts` — windows/gaps/caps/key names + **dedicated DB name**.
- `frontend/src/components/home/hooks/use-activity-feed.ts` — orchestration hook (gated, serialized, refetch-on-visible).
- `frontend/src/components/home/hooks/use-state-line.ts` — state-line derivation hook.
- `frontend/src/components/home/components/state-line.tsx` — state line UI.
- `frontend/src/components/home/components/activity-feed.tsx` — feed UI (group + rows + divider + dot + motion + empty).
- `*.test.ts(x)` siblings for each module above.

**Modify:**
- `frontend/src/components/home/home-dashboard.tsx` — mobile-gated `<StateLineSection/>` after `<HeroCard/>`, `<ActivityFeedSection/>` after `<ComingUp/>`.
- `frontend/src/stores/app-store.ts` — list/calendar navigation intents (`openListDetail`/`focusCalendarDate` + consumers).
- `frontend/src/components/lists-view.tsx` — consume `listDetailIntent` on mount.
- `frontend/src/components/calendar/calendar-module.tsx` — consume `calendarFocusDate` on mount.
- `frontend/src/api/hooks/use-auth.ts` — `await clearHomeActivity()` in `useLogout` (after `clearOfflineReadCache()`).
- `frontend/src/api/client/http-client.ts` — `await clearHomeActivity()` in `handleUnauthorized` (after `clearOfflineReadCache()`).
- `frontend/src/test/setup.ts` — include the new `useAppStore` intent fields in `resetAllStores()` (markers already cleared by the existing `beforeEach` `localStorage.clear()`).

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

/** The calendar query window the snapshot was captured under, as local-date
 * strings. The window slides daily, so the diff only compares calendar entities
 * within the overlap of the previous and current windows (§4.4). */
export interface EventWindow {
  start: string; // formatLocalDate(today)
  end: string; // formatLocalDate(today + W)
}

export interface ActivityState {
  snapshot: Snapshot;
  log: ActivityItem[];
  snapshotSavedAt: number;
  eventWindow: EventWindow; // bounds the snapshot's calendar entities were captured under
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
  rowsOverflow: number; // sub-rows beyond the calendar sub-row cap (for "and N more" inside the group)
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

  it("labels a created list 'New list' (not the doubled title)", () => {
    const items = diffSnapshots(buildSnapshot([], [list({ name: "Party" })]), {}, T);
    expect(items[0]).toMatchObject({ kind: "added", title: "Party", detail: "New list" });
  });

  it("suppresses a bare un-check (completedItems decrease, no other change)", () => {
    const prev = buildSnapshot([], [list({ totalItems: 3, completedItems: 2 })]);
    const fresh = buildSnapshot([], [list({ totalItems: 3, completedItems: 1 })]);
    expect(diffSnapshots(fresh, prev, T)).toHaveLength(0);
  });

  it("ignores window-edge churn: an event only outside the overlap is neither added nor removed", () => {
    const overlap = { start: "2026-06-22", end: "2026-07-10" };
    // prev had an event on the 21st (now below the new window's start); fresh has one on the 23rd.
    const prev = buildSnapshot([ev({ id: "old", date: new Date(2026, 5, 21) })], []);
    const fresh = buildSnapshot([ev({ id: "new", date: new Date(2026, 5, 23) })], []);
    const items = diffSnapshots(fresh, prev, T, overlap);
    // "old" aged out below start → not removed; "new" is inside overlap → added.
    expect(items.map((i) => i.kind)).toEqual(["added"]);
    expect(items[0].title).toBe("Dentist");
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

function listChanged(fresh: SnapshotEntry, prev: SnapshotEntry): boolean {
  if (fresh.title !== prev.title) return true;
  if (fresh.totalItems !== prev.totalItems) return true;
  // completedItems: only an INCREASE (checked off) is surfaced. An un-check
  // (decrease) with no other change is suppressed — there is no "updated"
  // signal (spec §4.2).
  return (fresh.completedItems ?? 0) > (prev.completedItems ?? 0);
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
  if (!prev) return "New list"; // created → summary "<name> · New list" (§5)
  if (fresh.title !== prev.title) return "renamed";
  const totalDelta = (fresh.totalItems ?? 0) - (prev.totalItems ?? 0);
  if (totalDelta > 0) return `+${totalDelta} items`;
  if (totalDelta < 0) return `${-totalDelta} removed`;
  // Only reachable when completedItems increased — listChanged() gates out the
  // title/total/uncheck cases, so there is no unspecified "updated" fallback.
  const doneDelta = (fresh.completedItems ?? 0) - (prev.completedItems ?? 0);
  return `${doneDelta} checked off`;
}

/** Calendar entities outside the overlap of the previous and current detection
 * windows are sliding-edge churn, not real adds/removes (spec §4.4). Lists are
 * unwindowed and always pass. yyyy-MM-dd strings compare chronologically. */
function inWindow(e: SnapshotEntry, overlap?: { start: string; end: string }): boolean {
  if (!overlap || e.module !== "calendar" || !e.date) return true;
  return e.date >= overlap.start && e.date <= overlap.end;
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

export function diffSnapshots(
  fresh: Snapshot,
  prev: Snapshot,
  detectedAt: number,
  overlap?: { start: string; end: string },
): ActivityItem[] {
  const out: ActivityItem[] = [];

  for (const key of Object.keys(fresh)) {
    const f = fresh[key];
    const p = prev[key];
    if (!p) {
      if (!inWindow(f, overlap)) continue; // entered at the sliding top edge — not a real add
      out.push(item(f, "added", f.module === "calendar" ? calendarDetail(f) : listDetail(f), detectedAt));
      continue;
    }
    const changed = f.module === "calendar" ? calendarChanged(f, p) : listChanged(f, p);
    if (changed) {
      out.push(item(f, "edited", f.module === "calendar" ? calendarDetail(f) : listDetail(f, p), detectedAt));
    }
  }

  for (const key of Object.keys(prev)) {
    if (fresh[key]) continue;
    const p = prev[key];
    if (!inWindow(p, overlap)) continue; // aged out below the sliding bottom edge — not a real remove
    out.push(item(p, "removed", "", detectedAt));
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

  it("preserves out-of-window calendar entries instead of dropping/demoting them", () => {
    const window = { start: "2026-06-21", end: "2026-07-19" };
    const log = [
      mk({ kind: "added", storeKey: "calendar:past", module: "calendar", date: "2026-06-01" }),  // below start
      mk({ kind: "edited", storeKey: "calendar:future", module: "calendar", date: "2026-12-31" }), // above end
    ];
    const next = reconcileLog(log, new Set(), window); // neither key is in fresh (aged out)
    expect(next).toHaveLength(2);
    expect(next.map((i) => i.kind).sort()).toEqual(["added", "edited"]); // unchanged, not demoted
  });

  it("still drops/demotes an IN-window entry that genuinely vanished", () => {
    const window = { start: "2026-06-21", end: "2026-07-19" };
    const log = [mk({ kind: "edited", storeKey: "calendar:gone", module: "calendar", date: "2026-06-25" })];
    expect(reconcileLog(log, new Set(), window)[0].kind).toBe("removed");
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

export function reconcileLog(
  log: ActivityItem[],
  freshKeys: Set<string>,
  window?: { start: string; end: string },
): ActivityItem[] {
  const out: ActivityItem[] = [];
  for (const i of log) {
    if (freshKeys.has(i.storeKey)) {
      out.push(i);
      continue;
    }
    // Absent from `fresh`. A calendar entry whose date is OUTSIDE the current query
    // window merely aged out of the fetch — it was NOT deleted. Preserve it as-is
    // (do not drop an `added`, do not demote an `edited`) until normal 48h pruning.
    // Without this, the window-aware diff's suppression is undone here (§4.4).
    if (window && i.module === "calendar" && i.date && (i.date < window.start || i.date > window.end)) {
      out.push(i);
      continue;
    }
    if (i.kind === "added") {
      // genuinely absent within the window → drop the phantom
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
export const FEED_ENTRY_CAP = 20; // max top-level entries (calendar group counts as one) — bounds length
export const CALENDAR_SUBROW_CAP = 10; // max sub-rows inside the expanded calendar group
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

  it("collapses identical changes to a recurring series into one sub-row", () => {
    const feed = buildFeed(
      [cal({ storeKey: "calendar:r_1", recurringEventId: "r", title: "Soccer", kind: "edited", detail: "5:00 PM" }),
       cal({ storeKey: "calendar:r_2", recurringEventId: "r", title: "Soccer", kind: "edited", detail: "5:00 PM" })],
      0, 20,
    );
    const g = feed.groups.find((x) => x.module === "calendar");
    expect(g?.rows).toHaveLength(1);
  });

  it("keeps DISTINCT changes to the same series as separate sub-rows", () => {
    const feed = buildFeed(
      [cal({ storeKey: "calendar:r_1", recurringEventId: "r", title: "Soccer", kind: "edited", detail: "5:00 PM" }),
       cal({ storeKey: "calendar:r_2", recurringEventId: "r", title: "Soccer", kind: "removed", detail: "" })],
      0, 20,
    );
    const g = feed.groups.find((x) => x.module === "calendar");
    expect(g?.rows).toHaveLength(2); // different change signature → not collapsed
  });

  it("caps calendar sub-rows and reports per-group overflow", () => {
    const many = Array.from({ length: 14 }, (_, i) =>
      cal({ storeKey: `calendar:e${i}`, title: `Event ${i}`, detail: `d${i}` }),
    );
    const feed = buildFeed(many, 0, 20, 10); // subRowCap = 10
    const g = feed.groups.find((x) => x.module === "calendar");
    expect(g?.rows).toHaveLength(10);
    expect(g?.rowsOverflow).toBe(4);
    expect(g?.summary).toContain("14 added"); // count is over ALL items, not the shown 10
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

// Deterministic order: newest first, then title, then storeKey — so a same-cycle
// batch (every delta shares detectedAt) never reshuffles between renders (§5).
function byRecency(a: ActivityItem, b: ActivityItem): number {
  return b.detectedAt - a.detectedAt || a.title.localeCompare(b.title) || a.storeKey.localeCompare(b.storeKey);
}

// One sub-row per distinct series change. The signature includes everything the
// sub-row actually renders — kind, title (label), member (dot), detail — so two
// instances collapse ONLY when they would display identically; any visible change
// (retitle, reassignment, time move) stays a separate row (§4.2).
function seriesSignature(i: ActivityItem): string | null {
  return i.recurringEventId
    ? `${i.recurringEventId}|${i.kind}|${i.title}|${i.memberId ?? ""}|${i.detail ?? ""}`
    : null;
}

function calendarSummary(items: ActivityItem[]): string {
  const added = items.filter((i) => i.kind === "added").length;
  const changed = items.filter((i) => i.kind === "edited").length;
  const removed = items.filter((i) => i.kind === "removed").length;
  const parts: string[] = [];
  if (added) parts.push(`${added} added`);
  if (changed) parts.push(`${changed} changed`);
  if (removed) parts.push(`${removed} removed`);
  return `Calendar · ${parts.join(", ")}`;
}

export function buildFeed(
  log: ActivityItem[],
  lastSeen: number,
  entryCap: number,
  subRowCap: number = entryCap,
): Feed {
  // Calendar: collapse series by (recurringEventId, kind, detail), then group all
  // calendar items together. Non-recurring events are never collapsed.
  const seenSig = new Set<string>();
  const calItems: ActivityItem[] = [];
  for (const i of log.filter((x) => x.module === "calendar").sort(byRecency)) {
    const sig = seriesSignature(i);
    if (sig) {
      if (seenSig.has(sig)) continue;
      seenSig.add(sig);
    }
    calItems.push(i);
  }

  const groups: FeedGroup[] = [];
  if (calItems.length === 1) {
    const i = calItems[0];
    groups.push({
      id: "calendar", module: "calendar",
      summary: `${i.title}${i.detail ? ` · ${i.detail}` : ""}`,
      rows: [rowOf(i)], rowsOverflow: 0, newest: i.detectedAt,
    });
  } else if (calItems.length > 1) {
    const shown = calItems.slice(0, subRowCap);
    groups.push({
      id: "calendar", module: "calendar",
      summary: calendarSummary(calItems), // count over ALL items, not just the shown sub-rows
      rows: shown.map(rowOf),
      rowsOverflow: Math.max(0, calItems.length - subRowCap),
      newest: Math.max(...calItems.map((i) => i.detectedAt)),
    });
  }

  // Lists: one group per list.
  for (const i of log.filter((x) => x.module === "lists")) {
    groups.push({
      id: i.storeKey, module: "lists",
      summary: `${i.title}${i.detail ? ` · ${i.detail}` : ""}`,
      rows: [rowOf(i)], rowsOverflow: 0, newest: i.detectedAt,
    });
  }

  groups.sort((a, b) => b.newest - a.newest || MODULE_RANK[a.module] - MODULE_RANK[b.module] || a.summary.localeCompare(b.summary));

  const overflow = Math.max(0, groups.length - entryCap);
  const capped = groups.slice(0, entryCap);

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
  it("is false when never hidden (hiddenAt=0) on the same day", () => {
    const now = day(1);
    // hiddenAt=0 must NOT read as "hidden for decades" → not meaningful.
    expect(isMeaningfulOpen({ coldStart: false, now, hiddenAt: 0, lastSeen: now - 5000 })).toBe(false);
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
  // Require hiddenAt > 0: the default 0 means "never hidden", which would
  // otherwise read as "hidden since 1970" and make every open meaningful.
  if (o.hiddenAt > 0 && o.now - o.hiddenAt > MEANINGFUL_GAP_MS) return true;
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
// A DEDICATED database — NOT the offline cache's `family-hub-offline`.
// `idb-keyval.createStore` calls `indexedDB.open(dbName)` with no version, so it
// can only create its object store during the DB's initial `onupgradeneeded`. The
// offline cache already created `family-hub-offline` at v1 with `query-cache`, so
// adding a second store to that DB would never fire an upgrade and every
// transaction would throw (silently swallowed). A separate DB avoids that entirely.
export const ACTIVITY_DB_NAME = "family-hub-home-activity";
export const ACTIVITY_STORE_NAME = "home-activity";
export const ACTIVITY_STATE_KEY = "activity-state";
```

- [ ] **Step 2: Write the failing test** (mirror `persister.test.ts` DI fake)

```ts
import { describe, expect, it, vi } from "vitest";
import { clearHomeActivity, createHomeActivityStore, type IdbKeyvalLike } from "./store";
import { getHiddenAt, getLastSeen, setHiddenAt, setLastSeen } from "./markers";
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

const state: ActivityState = {
  snapshot: { "lists:l1": { storeKey: "lists:l1", module: "lists", title: "Groceries" } },
  log: [], snapshotSavedAt: 5,
  eventWindow: { start: "2026-06-21", end: "2026-07-19" },
};

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

describe("clearHomeActivity (singleton)", () => {
  it("clears the localStorage markers, not only IndexedDB", async () => {
    setLastSeen(123);
    setHiddenAt(456);
    await clearHomeActivity(); // jsdom has no indexedDB → the IDB clear no-ops via the guard
    expect(getLastSeen()).toBe(0);
    expect(getHiddenAt()).toBe(0);
  });
});
```

> Why a DI Map fake and **not** a `fake-indexeddb` integration test (push-back on that review note): `fake-indexeddb` is **not a dependency** (verified), and the repo's IDB tests use Map-based DI fakes (`persister.test.ts`). The original concern — "does a second object store get created in the existing `family-hub-offline` DB?" — is **eliminated by using a separate database** (Step 1): there is no shared-DB upgrade path left to integration-test. Adding `fake-indexeddb` would be new scope for a hazard the design no longer has.

- [ ] **Step 3: Run it, expect FAIL.**

- [ ] **Step 4: Implement `store.ts`** (same guard/DI shape as `persister.ts`)

```ts
import { clear, createStore, del, get, set, type UseStore } from "idb-keyval";
import {
  ACTIVITY_DB_NAME, ACTIVITY_STATE_KEY, ACTIVITY_STORE_NAME,
  HIDDEN_AT_KEY, LAST_SEEN_KEY,
} from "./constants";
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

/** Wipe ALL persisted activity — the IndexedDB store AND both localStorage markers.
 * Called from useLogout() and handleUnauthorized(); never rejects. Neither caller
 * does a blanket localStorage.clear(), so clearing the markers here is what stops
 * one account's divider baseline (lastSeen/hiddenAt) leaking to the next account on
 * a shared device (spec §9). */
export const clearHomeActivity = async (): Promise<void> => {
  await appStore.clear(); // IndexedDB snapshot + log
  try {
    localStorage.removeItem(LAST_SEEN_KEY);
    localStorage.removeItem(HIDDEN_AT_KEY);
  } catch {
    // storage unavailable (private mode / webview) — nothing persisted to leak
  }
};
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
import { beforeEach, describe, expect, it, vi } from "vitest";
import type { CalendarEvent, ListSummary } from "@/lib/types";
import type { ActivityState } from "@/lib/home-activity/types";
import { useActivityFeed } from "./use-activity-feed";

let events: CalendarEvent[] = [];
let lists: ListSummary[] = [];
let querySettled = true; // toggle to exercise the "not yet loaded" gate
// Stable refetch spies (the mock factory re-runs per render, so inline vi.fn()s
// would not be assertable across renders).
const eventsRefetch = vi.fn(async () => ({ data: { data: events } }));
const listsRefetch = vi.fn(async () => ({ data: { data: lists } }));

vi.mock("@/api", async (orig) => {
  const actual = await orig<typeof import("@/api")>();
  return {
    ...actual,
    useCalendarEvents: () => ({
      data: querySettled ? { data: events } : undefined,
      isSuccess: querySettled,
      refetch: eventsRefetch,
    }),
    useLists: () => ({
      data: querySettled ? { data: lists } : undefined,
      isSuccess: querySettled,
      refetch: listsRefetch,
    }),
  };
});

beforeEach(() => {
  eventsRefetch.mockClear();
  listsRefetch.mockClear();
});

function wrapper({ children }: { children: ReactNode }) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false, gcTime: Infinity } } });
  return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
}

describe("useActivityFeed", () => {
  it("surfaces a newly-seen list change as a feed group", async () => {
    querySettled = true;
    events = [];
    lists = [{ id: "l1", name: "Groceries", kind: "grocery", totalItems: 0, completedItems: 0 }];
    const io = makeMemoryIo(); // seeds an empty prior snapshot so the list reads as 'added'
    const { result } = renderHook(() => useActivityFeed({ io, nowProvider: () => 1000 }), { wrapper });
    await waitFor(() => expect(result.current.feed.groups.length).toBeGreaterThan(0));
    expect(result.current.feed.groups[0].summary).toContain("Groceries");
  });

  it("does not snapshot or diff while a source query is unsettled", async () => {
    querySettled = false; // queries still loading
    events = [];
    lists = [{ id: "l1", name: "Groceries", kind: "grocery", totalItems: 0, completedItems: 0 }];
    const io = makeMemoryIo();
    const { result } = renderHook(() => useActivityFeed({ io, nowProvider: () => 1000 }), { wrapper });
    // No "added" dump from partial data; saveState is never called with a fresh snapshot.
    await waitFor(() => expect(io.loadState).toHaveBeenCalled());
    expect(result.current.feed.groups).toHaveLength(0);
    expect(io.saveState).not.toHaveBeenCalled();
  });
});

describe("useActivityFeed — orchestration", () => {
  it("does NOT advance the divider on a background data-change cycle", async () => {
    querySettled = true;
    events = [];
    lists = [{ id: "l1", name: "Groceries", kind: "grocery", totalItems: 0, completedItems: 0 }];
    const io = makeMemoryIo();
    const now = vi.fn(() => 1000);
    const { rerender } = renderHook(() => useActivityFeed({ io, nowProvider: now }), { wrapper });
    await waitFor(() => expect(io.saveState).toHaveBeenCalledTimes(1)); // cold-start open
    const seenAfterOpen = io.getLastSeen(); // advanced to 1000 by the open
    // A later data change (auto cycle) must refresh the log but not advance the marker.
    now.mockReturnValue(5000);
    lists = [...lists, { id: "l2", name: "Camping", kind: "todo", totalItems: 1, completedItems: 0 }];
    rerender();
    await waitFor(() => expect(io.saveState).toHaveBeenCalledTimes(2));
    expect(io.getLastSeen()).toBe(seenAfterOpen); // divider baseline did not move
  });

  it("refetches BOTH sources on return-to-visible before detecting", async () => {
    querySettled = true;
    events = [];
    lists = [];
    const io = makeMemoryIo();
    renderHook(() => useActivityFeed({ io, nowProvider: () => 1000 }), { wrapper });
    await waitFor(() => expect(io.loadState).toHaveBeenCalled());
    // jsdom visibilityState defaults to "visible" → the listener takes the visible branch.
    document.dispatchEvent(new Event("visibilitychange"));
    await waitFor(() => expect(eventsRefetch).toHaveBeenCalled());
    expect(listsRefetch).toHaveBeenCalled();
  });

  it("serializes writes: a slow earlier save cannot overwrite a newer cycle", async () => {
    querySettled = true;
    events = [];
    lists = [{ id: "l1", name: "A", kind: "todo", totalItems: 0, completedItems: 0 }];
    let state: ActivityState | null = {
      snapshot: {}, log: [], snapshotSavedAt: 0,
      eventWindow: { start: "2000-01-01", end: "2100-01-01" },
    };
    const completed: number[] = [];
    let call = 0;
    const io = {
      loadState: vi.fn(async () => state),
      saveState: vi.fn(async (s: ActivityState) => {
        const n = ++call;
        if (n === 1) await new Promise((r) => setTimeout(r, 50)); // make the FIRST save slow
        state = s;
        completed.push(n);
      }),
      getLastSeen: () => 0, setLastSeen: () => {},
      getHiddenAt: () => 0, setHiddenAt: () => {},
    };
    const now = vi.fn(() => 1000);
    const { rerender } = renderHook(() => useActivityFeed({ io, nowProvider: now }), { wrapper });
    // Queue a second cycle with newer data while the first save is still pending.
    now.mockReturnValue(2000);
    lists = [{ id: "l1", name: "A", kind: "todo", totalItems: 5, completedItems: 0 }];
    rerender();
    await waitFor(() => expect(completed).toEqual([1, 2])); // mutex preserved order despite the slow save
    expect(state?.snapshot["lists:l1"]?.totalItems).toBe(5); // newest state is final, not overwritten
  });
});

// minimal in-memory IO double for the hook's injected dependencies
function makeMemoryIo() {
  let state: ActivityState | null = {
    snapshot: {}, log: [], snapshotSavedAt: 0,
    eventWindow: { start: "2000-01-01", end: "2100-01-01" },
  };
  let lastSeen = 0;
  let hiddenAt = 0;
  return {
    loadState: vi.fn(async () => state),
    saveState: vi.fn(async (s: ActivityState) => { state = s; }),
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
import { useCallback, useEffect, useMemo, useRef, useState } from "react";
import { useCalendarEvents, useLists } from "@/api";
import {
  ACTIVITY_WINDOW_MS, CALENDAR_SUBROW_CAP, FEED_ENTRY_CAP,
  FEED_EVENT_WINDOW_DAYS, STALE_RESEED_MS,
} from "@/lib/home-activity/constants";
import { diffSnapshots, mergeLog, pruneLog, reconcileLog } from "@/lib/home-activity/diff";
import { buildFeed } from "@/lib/home-activity/group";
import {
  getHiddenAt, getLastSeen, isMeaningfulOpen, setHiddenAt, setLastSeen,
} from "@/lib/home-activity/markers";
import { buildSnapshot } from "@/lib/home-activity/normalize";
import { loadActivityState, saveActivityState } from "@/lib/home-activity/store";
import type { ActivityState, EventWindow, Feed } from "@/lib/home-activity/types";
import type { CalendarEvent, ListSummary } from "@/lib/types";
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
// Stable identities so the detection effect does not re-fire every render while a
// query is still loading (a fresh `?? []` would be a new reference each time).
const EMPTY_EVENTS: CalendarEvent[] = [];
const EMPTY_LISTS: ListSummary[] = [];

/** Overlap of the previous and current detection windows (yyyy-MM-dd strings). */
function overlapOf(prev: EventWindow, curr: EventWindow): EventWindow {
  return {
    start: prev.start > curr.start ? prev.start : curr.start,
    end: prev.end < curr.end ? prev.end : curr.end,
  };
}

export function useActivityFeed({
  io = defaultIo,
  nowProvider,
}: { io?: ActivityIo; nowProvider?: () => number } = {}) {
  // Stabilize nowProvider so the visibility listener is not torn down each render.
  const nowRef = useRef(nowProvider ?? (() => Date.now()));
  nowRef.current = nowProvider ?? nowRef.current;
  const now = useCallback(() => nowRef.current(), []);

  // Memoize the query range by a local-date STRING — startOfDay(new Date()) is a
  // new reference every render, which would churn the query key and effects.
  const todayStr = formatLocalDate(new Date(now()));
  const range = useMemo<EventWindow>(() => {
    const today = startOfDay(new Date(now()));
    return { start: formatLocalDate(today), end: formatLocalDate(addDays(today, FEED_EVENT_WINDOW_DAYS)) };
  }, [todayStr, now]);

  const eventsQuery = useCalendarEvents({ startDate: range.start, endDate: range.end });
  const listsQuery = useLists();
  const settled = eventsQuery.isSuccess && listsQuery.isSuccess;
  const events = eventsQuery.data?.data ?? EMPTY_EVENTS;
  const lists = listsQuery.data?.data ?? EMPTY_LISTS;

  const [feed, setFeed] = useState<Feed>(EMPTY_FEED);
  const [meaningfulOpenId, setMeaningfulOpenId] = useState(0); // bump → reset ephemeral expansion
  const coldStartRef = useRef(true);
  // Mutex: every cycle (cold start, data change, return-to-visible) is chained
  // through this promise so a cycle's load→diff→save runs to completion before the
  // next begins. The newest-queued cycle therefore writes LAST and wins — a slow
  // earlier save can never land after a newer one (the generation guard alone did
  // not prevent this: two saves already in flight could resolve out of order).
  const queueRef = useRef<Promise<void>>(Promise.resolve());
  // The divider baseline is frozen at the `lastSeen` of the most recent meaningful
  // OPEN and reused by every cycle in between. A background data-change cycle or a
  // quick peek must NOT move the divider, so renders never reread the advanced
  // marker — they read this ref.
  const displayBaselineRef = useRef<number | null>(null);

  const render = useCallback(
    (log: ActivityState["log"]) =>
      setFeed(buildFeed(log, displayBaselineRef.current ?? 0, FEED_ENTRY_CAP, CALENDAR_SUBROW_CAP)),
    [],
  );

  // One cycle's critical section. Serialized by `queueRef`, so no other cycle's
  // load/save can interleave with this one.
  const runCycle = useCallback(
    async (trigger: "auto" | "visible") => {
      const prior = await io.loadState();

      // Gate: never diff partial data. Render whatever is persisted and wait.
      if (!settled) {
        if (prior) {
          if (displayBaselineRef.current === null) displayBaselineRef.current = io.getLastSeen();
          render(prior.log);
        }
        return;
      }

      const ts = now();
      const fresh = buildSnapshot(events, lists);

      // First run / long absence → reseed silently (no "added" dump).
      if (!prior || ts - prior.snapshotSavedAt > STALE_RESEED_MS) {
        await io.saveState({ snapshot: fresh, log: [], snapshotSavedAt: ts, eventWindow: range });
        io.setLastSeen(ts);
        displayBaselineRef.current = ts; // nothing is "new" yet
        coldStartRef.current = false;
        setFeed(EMPTY_FEED);
        return;
      }

      // Window-bounds-aware diff AND reconcile: an event that merely aged out of the
      // sliding window must be neither reported (diff) nor dropped/demoted (reconcile)
      // — it is preserved until normal 48h pruning (§4.4).
      const overlap = overlapOf(prior.eventWindow, range);
      const deltas = diffSnapshots(fresh, prior.snapshot, ts, overlap);
      let log = mergeLog(prior.log, deltas); // merge BEFORE reconcile so add+remove net-cancels
      log = reconcileLog(log, new Set(Object.keys(fresh)), range); // preserve out-of-window entries
      log = pruneLog(log, ts, ACTIVITY_WINDOW_MS);
      await io.saveState({ snapshot: fresh, log, snapshotSavedAt: ts, eventWindow: range });

      // Classify the OPEN, not the data change. Only a real open advances the divider.
      const isOpenEvent = trigger === "visible" || coldStartRef.current;
      const meaningful =
        isOpenEvent &&
        isMeaningfulOpen({
          coldStart: coldStartRef.current, now: ts,
          hiddenAt: io.getHiddenAt(), lastSeen: io.getLastSeen(),
        });
      // Freeze the baseline at THIS open's pre-advance lastSeen; later non-open
      // cycles reuse it, so the divider holds until the next real open.
      if (meaningful || displayBaselineRef.current === null) {
        displayBaselineRef.current = io.getLastSeen();
      }
      render(log);
      if (meaningful) {
        io.setLastSeen(ts); // advance the persisted marker AFTER capturing the baseline
        setMeaningfulOpenId((n) => n + 1); // reset ephemeral expansion on a real open
      }
      coldStartRef.current = false;
    },
    [io, now, settled, events, lists, range, render],
  );

  // Enqueue a cycle behind the mutex. We deliberately run every queued cycle (no
  // gen-skip): cycles are cheap in-memory diffs, and a meaningful-open cycle must
  // never be skipped because a data-change cycle queued right after it.
  const detect = useCallback(
    (trigger: "auto" | "visible") => {
      const run = queueRef.current.then(() => runCycle(trigger));
      queueRef.current = run.catch(() => {}); // keep the chain alive across errors
      return run;
    },
    [runCycle],
  );

  // Stable handles for the visibility listener so it never re-registers on data change.
  const detectRef = useRef(detect);
  detectRef.current = detect;
  const refetchRef = useRef<() => Promise<void>>(async () => {});
  refetchRef.current = async () => {
    await Promise.all([eventsQuery.refetch(), listsQuery.refetch()]);
  };

  // Cold-start + data-change cycles. `detect` changes identity when events/lists/
  // range/settled change, which is the data-change trigger.
  useEffect(() => {
    void detect("auto");
  }, [detect]);

  // Hide → record hiddenAt. Visible → refetch (focus refetch is off app-wide),
  // THEN detect as an open. Registered once; reads the latest detect/refetch via refs.
  useEffect(() => {
    async function onVisibility() {
      if (document.visibilityState === "hidden") {
        io.setHiddenAt(now());
      } else {
        await refetchRef.current();
        void detectRef.current("visible");
      }
    }
    document.addEventListener("visibilitychange", onVisibility);
    return () => document.removeEventListener("visibilitychange", onVisibility);
  }, [io, now]);

  return { feed, meaningfulOpenId };
}
```

> Why this shape (closing Codex blocking #2–#5): **(2)** detection is gated on `settled` and uses stable `EMPTY_*` constants, so a not-yet-loaded query is never read as an empty snapshot (no "added" dump) nor as a removed module, and the effect does not re-fire on `?? []` identity churn. **(3)** a meaningful open is `trigger === "visible" || coldStartRef.current` — a background data-change cycle is `"auto"` after cold start, so it refreshes the log without advancing `lastSeen`; the rendered divider uses a frozen `displayBaseline`, so a later harmless cycle cannot re-flag rows as "earlier." **(4)** return-to-visible explicitly `refetch()`es before detecting because `refetchOnWindowFocus` is off. **(5)** a `queueRef` promise-queue **mutex** serializes cycles so each `load→diff→save` completes before the next begins — the newest-queued cycle writes last and wins, which a generation guard alone could not guarantee (two saves already in flight can resolve out of order). The visibility listener is registered once (refs hold the latest `detect`/`refetch`), and `nowProvider` is stabilized — no per-render teardown. `meaningfulOpenId` is returned so the feed can reset ephemeral expansion on a real open.

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
    { id: "calendar", module: "calendar", summary: "Calendar · 2 added", newest: 30, rowsOverflow: 0, rows: [
      { storeKey: "calendar:a", kind: "added", label: "Dentist", detail: "Tue 9:00 AM", module: "calendar", memberId: "m1" },
      { storeKey: "calendar:b", kind: "edited", label: "Soccer", detail: "5:00 PM", module: "calendar" },
    ] },
    { id: "lists:l1", module: "lists", summary: "Groceries · +3 items", newest: 10, rowsOverflow: 0, rows: [
      { storeKey: "lists:l1", kind: "edited", label: "Groceries", detail: "+3 items", module: "lists", entityId: "l1" },
    ] },
  ],
  dividerAfter: 0,
  overflow: 0,
};

describe("ActivityFeed", () => {
  it("renders groups + divider; expander toggles aria-expanded and sub-row tabbability", () => {
    render(<ActivityFeed feed={feed} onSelectRow={vi.fn()} />);
    expect(screen.getByText("Calendar · 2 added")).toBeInTheDocument();
    expect(screen.getByText(/earlier/i)).toBeInTheDocument();
    const expander = screen.getByRole("button", { name: /Calendar/ });
    expect(expander).toHaveAttribute("aria-expanded", "false");
    // Sub-rows stay in the DOM (animated collapse) but are not tabbable while collapsed.
    expect(screen.getByText(/Dentist/).closest("button")).toHaveAttribute("tabindex", "-1");
    fireEvent.click(expander);
    expect(expander).toHaveAttribute("aria-expanded", "true");
    expect(screen.getByText(/Dentist/).closest("button")).toHaveAttribute("tabindex", "0");
  });

  it("tints a leading member dot when a color resolver is provided", () => {
    render(<ActivityFeed feed={feed} onSelectRow={vi.fn()} memberColorOf={(id) => (id === "m1" ? "#ff0000" : undefined)} />);
    expect(document.querySelector('span[style*="background"]')).toBeTruthy();
  });

  it("collapses expansion when the meaningful-open epoch changes", () => {
    const { rerender } = render(<ActivityFeed feed={feed} onSelectRow={vi.fn()} meaningfulOpenId={1} />);
    fireEvent.click(screen.getByRole("button", { name: /Calendar/ }));
    expect(screen.getByRole("button", { name: /Calendar/ })).toHaveAttribute("aria-expanded", "true");
    rerender(<ActivityFeed feed={feed} onSelectRow={vi.fn()} meaningfulOpenId={2} />);
    expect(screen.getByRole("button", { name: /Calendar/ })).toHaveAttribute("aria-expanded", "false");
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

type MemberColorOf = (memberId: string | undefined) => string | undefined;

interface ActivityFeedProps {
  feed: Feed;
  onSelectRow: (row: FeedRow) => void;
  /** Resolve a member's dot color from its id. Wired from `useFamilyMemberMap` in
   * the dashboard so this component stays presentational/provider-free in tests. */
  memberColorOf?: MemberColorOf;
  /** Bumped on each meaningful open; remounts groups so expansion is ephemeral (§5). */
  meaningfulOpenId?: number;
}

// Shared field styling. min-h-11 = 44px touch target (spec §9). Easing/durations
// from the shipped motion system; motion-reduce collapses to instant (no plugin).
const EASE = "ease-[cubic-bezier(0.32,0.72,0,1)]";

export function ActivityFeed({ feed, onSelectRow, memberColorOf, meaningfulOpenId = 0 }: ActivityFeedProps) {
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
            {/* key includes the epoch so a meaningful open remounts (collapses) the group */}
            <ActivityGroup
              key={`${group.id}:${meaningfulOpenId}`}
              group={group}
              onSelectRow={onSelectRow}
              memberColorOf={memberColorOf}
            />
            {index === feed.dividerAfter && (
              <div className="my-2 flex items-center gap-2 text-xs text-muted-foreground/70 transition-opacity motion-reduce:transition-none">
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

function MemberDot({ color }: { color?: string }) {
  if (!color) return null;
  return (
    <span aria-hidden className="mr-2 inline-block size-2 shrink-0 rounded-full" style={{ backgroundColor: color }} />
  );
}

function ActivityGroup({
  group, onSelectRow, memberColorOf,
}: {
  group: FeedGroup;
  onSelectRow: (r: FeedRow) => void;
  memberColorOf?: MemberColorOf;
}) {
  const [open, setOpen] = useState(false);
  const press = usePressable();
  const expandable = group.rows.length > 1;

  if (!expandable) {
    const row = group.rows[0];
    return (
      <button
        type="button"
        className={cn(press.className, "flex min-h-11 w-full items-center justify-between rounded-2xl px-1 py-2 text-left")}
        onPointerDown={press.onPointerDown}
        onClick={() => onSelectRow(row)}
      >
        <span className="flex items-center text-sm">
          <MemberDot color={memberColorOf?.(row.memberId)} />
          {group.summary}
        </span>
      </button>
    );
  }

  return (
    <div>
      <button
        type="button"
        aria-expanded={open}
        className={cn(press.className, "flex min-h-11 w-full items-center justify-between rounded-2xl px-1 py-2 text-left")}
        onPointerDown={press.onPointerDown}
        onClick={() => setOpen((v) => !v)}
      >
        <span className="text-sm">{group.summary}</span>
        <span
          aria-hidden
          className={cn("text-muted-foreground transition-transform duration-[250ms] motion-reduce:transition-none", EASE)}
          style={{ transform: open ? "rotate(90deg)" : "none" }}
        >
          ▸
        </span>
      </button>
      {/* grid-rows 0fr→1fr animates height under motion-safe; under reduced motion the
          height snaps and only the opacity fades (spec §6 "opacity-only"). When closed
          the sub-list is `inert` + aria-hidden, so it is fully non-interactive and out
          of the a11y tree — not merely untabbable. */}
      <div
        className={cn("grid motion-safe:transition-[grid-template-rows] motion-safe:duration-[250ms]", `motion-safe:${EASE}`)}
        style={{ gridTemplateRows: open ? "1fr" : "0fr" }}
      >
        <ul
          inert={!open}
          aria-hidden={!open}
          className={cn(
            "overflow-hidden pl-3 transition-opacity duration-[250ms]",
            EASE,
            open ? "opacity-100" : "opacity-0",
          )}
        >
          {group.rows.map((row) => (
            <SubRow key={row.storeKey} row={row} onSelectRow={onSelectRow} memberColorOf={memberColorOf} tabbable={open} />
          ))}
          {group.rowsOverflow > 0 && (
            <li className="px-1 py-2 text-xs text-muted-foreground/70">and {group.rowsOverflow} more</li>
          )}
        </ul>
      </div>
    </div>
  );
}

function SubRow({
  row, onSelectRow, memberColorOf, tabbable,
}: {
  row: FeedRow;
  onSelectRow: (r: FeedRow) => void;
  memberColorOf?: MemberColorOf;
  tabbable: boolean;
}) {
  const press = usePressable(); // sub-rows are pressable too (spec §6)
  return (
    <li>
      <button
        type="button"
        tabIndex={tabbable ? 0 : -1}
        className={cn(press.className, "flex min-h-11 w-full items-center justify-between rounded-xl px-1 py-2 text-left text-sm")}
        onPointerDown={press.onPointerDown}
        onClick={() => onSelectRow(row)}
      >
        <span className="flex items-center">
          <MemberDot color={memberColorOf?.(row.memberId)} />
          {markerFor(row.kind)} {row.label}
        </span>
        {row.detail && <span className="text-muted-foreground">{row.detail}</span>}
      </button>
    </li>
  );
}

function markerFor(kind: FeedRow["kind"]): string {
  return kind === "added" ? "+" : kind === "removed" ? "−" : "~";
}
```

> Notes: every interactive element (group row, sub-row) uses `usePressable` and a `min-h-11` (44px) target. Expansion is animated via a pure-CSS `grid-template-rows` transition (no `tailwindcss-animate` dependency) and the chevron rotates; both honor `prefers-reduced-motion` via `motion-reduce:transition-none`. The member dot resolves `memberId → color` through the `memberColorOf` prop (wired from `useFamilyMemberMap` in Task 11), so it is implemented in v1, not deferred. Expansion resets on the `meaningfulOpenId` epoch (the group key), satisfying "ephemeral expansion" across background→foreground.

- [ ] **Step 4: Run it, expect PASS.**

- [ ] **Step 5: Commit**

```bash
git add frontend/src/components/home/components/activity-feed.tsx frontend/src/components/home/components/activity-feed.test.tsx
git commit -m "feat(home): ActivityFeed component with expand/collapse + divider"
```

---

## Task 11: Deep-link navigation intents + integrate into `HomeDashboard`

Two parts: (A) entity-level deep-link plumbing via the app-store navigation-intent pattern (a bare `setActiveModule` cannot open a list detail — `selectedListId` is component-local in `ListsView`), and (B) mounting the sections inside `HomeDashboard` behind a **synchronous** `useIsMobile` gate (the `App.tsx` desktop→Calendar redirect is a post-render effect, so Home mounts once on desktop without a synchronous gate — spec §7/§9).

**Files:**
- Create: `frontend/src/lib/home-activity/navigation.ts` + `.test.ts` (pure deep-link resolver)
- Modify: `frontend/src/stores/app-store.ts` (list/calendar navigation intents) + test
- Modify: `frontend/src/components/lists-view.tsx` (consume list-detail intent)
- Modify: `frontend/src/components/calendar/calendar-module.tsx` (consume calendar-focus intent)
- Modify: `frontend/src/components/home/home-dashboard.tsx` (gated sections + wiring)
- Test: `frontend/src/components/home/home-dashboard.test.tsx` (extend existing)

- [ ] **Step 1: Pure deep-link resolver** (`navigation.ts` + test)

A list row opens that list's detail; an **in-window** calendar row opens its event sheet (handled by the dashboard's existing `handleEventClick`); an **out-of-window** calendar row focuses the Calendar on its date. Keep the mapping pure so it is unit-testable without the dashboard.

```ts
// navigation.ts
import { getEventKey } from "@/lib/time-utils";
import type { CalendarEvent } from "@/lib/types";
import type { FeedRow } from "./types";

export type FeedSelection =
  | { type: "open-event"; event: CalendarEvent }
  | { type: "open-list"; listId: string }
  | { type: "focus-calendar"; date: string }
  | { type: "switch-module"; module: "calendar" | "lists" };

export function resolveFeedSelection(row: FeedRow, inWindowEvents: CalendarEvent[]): FeedSelection {
  if (row.module === "lists") {
    // A removed list has no detail to open → land on the Lists module rather than
    // request a deleted id (which would 404).
    if (row.kind === "removed" || !row.entityId) return { type: "switch-module", module: "lists" };
    return { type: "open-list", listId: row.entityId };
  }
  // A removed calendar event's sheet is gone → focus its day (or the module).
  if (row.kind === "removed") {
    return row.date ? { type: "focus-calendar", date: row.date } : { type: "switch-module", module: "calendar" };
  }
  const key = row.storeKey.replace("calendar:", "");
  const match = inWindowEvents.find((e) => getEventKey(e) === key);
  if (match) return { type: "open-event", event: match };
  if (row.date) return { type: "focus-calendar", date: row.date };
  return { type: "switch-module", module: "calendar" };
}
```

```ts
// navigation.test.ts (essentials)
it("opens the list detail for a list row", () => {
  expect(resolveFeedSelection({ storeKey: "lists:l1", module: "lists", kind: "edited", label: "x", entityId: "l1" }, []))
    .toEqual({ type: "open-list", listId: "l1" });
});
it("opens the event sheet for an in-window calendar row", () => {
  const ev = { id: "e1", title: "Dentist", date: new Date(2026, 5, 23) } as CalendarEvent;
  const sel = resolveFeedSelection({ storeKey: "calendar:e1", module: "calendar", kind: "added", label: "Dentist" }, [ev]);
  expect(sel).toEqual({ type: "open-event", event: ev });
});
it("focuses the calendar date for an out-of-window calendar row", () => {
  expect(resolveFeedSelection({ storeKey: "calendar:e9", module: "calendar", kind: "added", label: "Camp", date: "2026-07-15" }, []))
    .toEqual({ type: "focus-calendar", date: "2026-07-15" });
});
it("does NOT open a deleted list — a removed list row lands on the Lists module", () => {
  expect(resolveFeedSelection({ storeKey: "lists:l1", module: "lists", kind: "removed", label: "Old", entityId: "l1" }, []))
    .toEqual({ type: "switch-module", module: "lists" });
});
it("a removed calendar row focuses its day, not a deleted event sheet", () => {
  expect(resolveFeedSelection({ storeKey: "calendar:e1", module: "calendar", kind: "removed", label: "Gone", date: "2026-06-23" }, [{ id: "e1" } as CalendarEvent]))
    .toEqual({ type: "focus-calendar", date: "2026-06-23" });
});
```

- [ ] **Step 2: App-store navigation intents** (mirror the existing `start*Draft`/`consume*Draft` pattern in `app-store.ts`)

```ts
// add to AppState:
listDetailIntent: string | null;       // list id to open on next ListsView mount
calendarFocusDate: string | null;      // yyyy-MM-dd to focus on next CalendarModule mount
openListDetail: (listId: string) => void;
consumeListDetailIntent: () => string | null;
focusCalendarDate: (date: string) => void;
consumeCalendarFocusDate: () => string | null;

// implementation (same shape as consumeMealPlacementDraft):
listDetailIntent: null,
calendarFocusDate: null,
openListDetail: (listId) => set({ listDetailIntent: listId, activeModule: "lists" }),
consumeListDetailIntent: () => { const v = get().listDetailIntent; set({ listDetailIntent: null }); return v; },
focusCalendarDate: (date) => set({ calendarFocusDate: date, activeModule: "calendar" }),
consumeCalendarFocusDate: () => { const v = get().calendarFocusDate; set({ calendarFocusDate: null }); return v; },
```

> Test (`app-store.test.ts`): `openListDetail("l1")` sets `activeModule === "lists"` and `consumeListDetailIntent()` returns `"l1"` then `null`. **Also add `useAppStore` reset for these new fields to `resetAllStores()` in `src/test/setup.ts`** (per the CLAUDE.md store-reset gotcha — `useAppStore` is already reset there, just include the new fields).

- [ ] **Step 3: Consume the intents**

`lists-view.tsx` — open the intended list once on mount (after the existing `selectedListId` state):
```tsx
const consumeListDetailIntent = useAppStore((s) => s.consumeListDetailIntent);
useEffect(() => {
  const id = consumeListDetailIntent();
  if (id) setSelectedListId(id);
}, [consumeListDetailIntent]);
```
`calendar-module.tsx` — focus the intended date once on mount (reusing the existing `setDate` action):
```tsx
const consumeCalendarFocusDate = useAppStore((s) => s.consumeCalendarFocusDate);
useEffect(() => {
  const date = consumeCalendarFocusDate();
  if (date) setDate(parseLocalDate(date)); // setDate already in useCalendarActions()
}, [consumeCalendarFocusDate, setDate]);
```

- [ ] **Step 4: Mount sections in `home-dashboard.tsx` behind a synchronous mobile gate**

Hooks cannot be conditional, so the organizer hooks live in **child components** that are only *rendered* on mobile — off-mobile they never mount, so the wider queries + persistence never run (spec §9).

```tsx
import { useIsMobile } from "@/hooks";
import { useFamilyMemberMap } from "@/api";
import { useActivityFeed } from "./hooks/use-activity-feed";
import { useStateLine } from "./hooks/use-state-line";
import { StateLine } from "./components/state-line";
import { ActivityFeed } from "./components/activity-feed";
import { resolveFeedSelection } from "@/lib/home-activity/navigation";
import { useAppStore } from "@/stores";
import { colorMap, type CalendarEvent } from "@/lib/types";
import type { FeedRow } from "@/lib/home-activity/types";

function StateLineSection({ now }: { now: Date }) {
  const { choresRemaining, dinnerTitle } = useStateLine({ now });
  return <StateLine choresRemaining={choresRemaining} dinnerTitle={dinnerTitle} />;
}

function ActivityFeedSection({
  now, inWindowEvents, onOpenEvent,
}: { now: Date; inWindowEvents: CalendarEvent[]; onOpenEvent: (e: CalendarEvent) => void }) {
  const { feed, meaningfulOpenId } = useActivityFeed({ nowProvider: () => now.getTime() });
  const memberMap = useFamilyMemberMap();
  const handleSelect = (row: FeedRow) => {
    const sel = resolveFeedSelection(row, inWindowEvents);
    const store = useAppStore.getState();
    if (sel.type === "open-event") onOpenEvent(sel.event);
    else if (sel.type === "open-list") store.openListDetail(sel.listId);
    else if (sel.type === "focus-calendar") store.focusCalendarDate(sel.date);
    else store.setActiveModule(sel.module);
  };
  // member.color is a FamilyColor identifier ("coral", "teal", …); resolve it to a
  // hex through colorMap (family.ts), the same path the rest of the UI uses.
  const memberColorOf = (id: string | undefined) => {
    const member = id ? memberMap.get(id) : undefined;
    return member ? colorMap[member.color]?.hex : undefined;
  };
  return (
    <ActivityFeed
      feed={feed}
      onSelectRow={handleSelect}
      memberColorOf={memberColorOf}
      meaningfulOpenId={meaningfulOpenId}
    />
  );
}
```

In `HomeDashboard` (which already computes `now`, `today`, `comingUp`, `handleEventClick`):
```tsx
const isMobile = useIsMobile();
// ...after <HeroCard /> :
{isMobile && <StateLineSection now={now} />}
// ...after <ComingUp /> :
{isMobile && (
  <ActivityFeedSection
    now={now}
    inWindowEvents={[...today, ...comingUp]}
    onOpenEvent={handleEventClick}
  />
)}
```

- [ ] **Step 5: Tests** (`home-dashboard.test.tsx`, extend existing)

```tsx
// Use the EXISTING `setViewportWidth(width)` helper already defined in
// home-dashboard.test.tsx (it sets window.innerWidth + mocks matchMedia, which is
// what useIsMobile reads). The existing suite does NOT mock useIsMobile, so do not
// introduce vi.mocked(useIsMobile).

// Mobile (≤768): the feed region mounts.
it("renders the activity feed region on mobile", async () => {
  setViewportWidth(768);
  render(<HomeDashboard nowOverride={new Date(2026, 5, 21, 12)} />);
  expect(await screen.findByText(/Since you last opened/i)).toBeInTheDocument();
});

// Desktop (≥769): the organizer never mounts (synchronous gate, not the redirect).
it("does NOT mount the feed/state-line on desktop", () => {
  setViewportWidth(1024);
  render(<HomeDashboard nowOverride={new Date(2026, 5, 21, 12)} />);
  expect(screen.queryByText(/Since you last opened/i)).not.toBeInTheDocument();
});
```
> The entity-level deep-link itself is covered by `navigation.test.ts` (resolver) + the `app-store` intent test + `ActivityFeed`'s `onSelectRow` test — no need to reconstruct a fully-seeded feed in the dashboard test.

- [ ] **Step 6: Run** — `npm test -- --run src/components/home src/lib/home-activity/navigation.test.ts src/stores/app-store.test.ts`.

- [ ] **Step 7: Commit**

```bash
git add frontend/src/lib/home-activity/navigation.ts frontend/src/lib/home-activity/navigation.test.ts \
        frontend/src/stores/app-store.ts frontend/src/stores/app-store.test.ts \
        frontend/src/components/lists-view.tsx frontend/src/components/calendar/calendar-module.tsx \
        frontend/src/components/home/home-dashboard.tsx frontend/src/components/home/home-dashboard.test.tsx \
        frontend/src/test/setup.ts
git commit -m "feat(home): entity deep-links + mobile-gated state line & activity feed"
```

---

## Task 12: Clear activity on logout + 401

> Spec §9 requires the clear on **both** paths, and `clearHomeActivity()` (Task 6) now wipes the IDB store **and** the `lastSeen`/`hiddenAt` markers — so a store-only clear cannot leave one account's divider baseline behind. `http-client.test.ts` **already exists** (it tests the 401 handler), so the 401 clear must be tested there too.

**Files:**
- Modify: `frontend/src/api/hooks/use-auth.ts` (`useLogout`, after `await clearOfflineReadCache();`)
- Modify: `frontend/src/api/client/http-client.ts` (`handleUnauthorized`, after `await clearOfflineReadCache();`)
- Test: `frontend/src/api/hooks/use-auth.test.tsx` (extend) **and** `frontend/src/api/client/http-client.test.ts` (extend — it exists)

- [ ] **Step 1: Write the failing tests** (both paths clear activity)

Mirror each suite's **existing** clearOfflineReadCache test exactly (same `createWrapper()`, `originalLocation`, mocked `reload`, and `invocationCallOrder` assertion) — §9 requires the clear to be **awaited before the reload**, not merely called.

```tsx
// In use-auth.test.tsx (the useLogout describe already has `originalLocation` + createWrapper):
import * as activityStore from "@/lib/home-activity/store";
it("clears persisted home activity (store + markers) before the logout reload", async () => {
  const activitySpy = vi.spyOn(activityStore, "clearHomeActivity").mockResolvedValue();
  const reload = vi.fn();
  Object.defineProperty(window, "location", { configurable: true, value: { ...originalLocation, reload } });
  const { result } = renderHook(() => useLogout(), { wrapper: createWrapper() });
  await result.current();
  expect(activitySpy).toHaveBeenCalled();
  expect(activitySpy.mock.invocationCallOrder[0]).toBeLessThan(reload.mock.invocationCallOrder[0]);
});
```

```ts
// In http-client.test.ts (the handleUnauthorized describe already has `originalLocation`
// and imports AUTH_TOKEN_STORAGE_KEY):
import * as activityStore from "@/lib/home-activity/store";
it("clears persisted home activity (store + markers) before the 401 reload", async () => {
  const activitySpy = vi.spyOn(activityStore, "clearHomeActivity").mockResolvedValue();
  const reload = vi.fn();
  Object.defineProperty(window, "location", { configurable: true, value: { ...originalLocation, reload } });
  localStorage.setItem(AUTH_TOKEN_STORAGE_KEY, "token-123"); // token present → real session expiry → reloads
  await handleUnauthorized();
  expect(activitySpy).toHaveBeenCalled();
  expect(activitySpy.mock.invocationCallOrder[0]).toBeLessThan(reload.mock.invocationCallOrder[0]);
});
```

- [ ] **Step 2: Run them, expect FAIL.**

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

- [ ] **Step 4: Run both, expect PASS.**

- [ ] **Step 5: Commit** (include `http-client.test.ts` — the 401 test)

```bash
git add frontend/src/api/hooks/use-auth.ts frontend/src/api/client/http-client.ts \
        frontend/src/api/hooks/use-auth.test.tsx frontend/src/api/client/http-client.test.ts
git commit -m "fix(home): clear persisted activity (store + markers) on logout and 401"
```

---

## Task 13: Test isolation + full verification

**Files:**
- Modify: `frontend/src/test/setup.ts` (only if a gap remains — see below)

- [ ] **Step 1: Confirm test isolation — likely no new code needed**

The `lastSeen`/`hiddenAt` **markers are localStorage**, and `setup.ts` already calls `localStorage.clear()` in `beforeEach` (verified) — so an `afterEach` `removeItem` for them would be **redundant**; do **not** add it. The real isolation risk is the new **`useAppStore` navigation-intent fields** (`listDetailIntent`, `calendarFocusDate`): `resetAllStores()` resets `useAppStore` by name, so make sure that reset includes the new fields (handled in Task 11 Step 2). The idb activity store is dependency-injected in unit tests, so it never touches a shared singleton there.

> Net: Task 13 usually changes nothing in `setup.ts`. Only touch it if a new persisted surface escapes `beforeEach`'s `localStorage.clear()` + `resetAllStores()`.

- [ ] **Step 2: Run the full unit suite**

Run: `npm test -- --run`
Expected: all green, including the new `src/lib/home-activity/*` and `src/components/home/*` tests.

- [ ] **Step 3: Lint + typecheck + build**

Run: `npm run lint && npm run build`
Expected: no Biome errors; `tsc` clean; Vite build succeeds.

- [ ] **Step 4: Commit**

```bash
# Only if setup.ts actually changed (see Step 1 — usually it does not):
git add frontend/src/test/setup.ts
git commit -m "test(home): cover activity-store test isolation"
```

- [ ] **Step 5: Manual smoke (optional, recommended before PR)**

Run the app (`npm run dev`), open the mobile home surface, and verify: the state line shows chores/dinner when present; making a list change in another tab then returning to Home surfaces it under "Since you last opened" (the visibility handler refetches, so a change from another device shows on return); a quick background→foreground does not advance the "new" divider; expanding a calendar group then backgrounding/foregrounding collapses it again; tapping a list row opens that list's detail; logging out and back in shows an empty feed.

---

## Notes for the implementer

- **Reuse, don't reinvent:** `getEventKey` (composite calendar key), `formatLocalDate`/`getWeekStartSunday` (device-local dates), `usePressable` (press + haptics), `useFamilyMemberMap` (member→color), the app-store `start*Draft`/`consume*Draft` navigation-intent pattern (deep-links), and the `createOfflineReadCache` DI shape (mirrored by `createHomeActivityStore`) all already exist. Do not add new date libraries, a second persister abstraction, or `fake-indexeddb`.
- **Separate IndexedDB database.** The activity store uses `family-hub-home-activity`, NOT the offline cache's `family-hub-offline` — `idb-keyval.createStore` can't add a second object store to an existing DB (no version bump). This is the #1 trap; do not "consolidate" the databases.
- **Don't diff partial or sliding-window data.** Gate detection on both queries settled, and pass the previous/current window overlap to `diffSnapshots` so the daily-sliding calendar window doesn't fabricate adds/removes at the edges.
- **Divider advances on an OPEN, never on a data-change cycle.** Classify meaningful opens only on cold start / return-to-visible; render against a `displayBaselineRef` re-frozen only at a real open; serialize cycles through the promise-queue mutex.
- **No backend.** Everything is derived on-device. If you find yourself wanting an `/api/activity` endpoint, stop — that is the deferred AI-phase work (spec §12).
- **Device-local only.** Do not introduce family-timezone math; the FE has no helper for it (spec §10.4).
- **Feed = calendar + lists.** Chores and meals belong to the state line, never the feed (spec anchor 7).
- **Member-color dot (spec §6) is implemented in v1** via the `memberColorOf` prop wired from `useFamilyMemberMap` (Tasks 10–11), not deferred.
