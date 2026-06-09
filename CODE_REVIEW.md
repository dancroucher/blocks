# Blocks — Code Review (consolidated)

**File**: `index.html` (4,529 lines, single-file app) + `firestore.rules`
**Branch**: `preview`
**Date**: 2026-06-09 (supersedes the 2026-06-01 review; statuses of the old
findings are folded in below)

> **Status update (later the same day)** — all four suggested PRs landed on
> `preview`:
> 1. `5970fb6` — A1 rules reconciled (**deploy still required**:
>    `firebase deploy --only firestore:rules`).
> 2. `b2b5f53` — B1–B4 dead code deleted (+ C4 console noise, C6 unused var;
>    Duplicate restored to the block context menu).
> 3. `9ba7675` — A2 anchor pinned to saved `projectStart`.
> 4. `3aa1751` — A3 deferred remote apply, B5 single save convention,
>    C1 transients stripped, C2 view prefs in localStorage, C10 flag hack gone.
>
> Still open: C3 (block move/resize on Pointer Events), C5 (tag length cap),
> C7 (innerHTML footguns), C8 (auth reload), C9 (`_render` size).

---

## Resolved since 2026-06-01

- ~~#1 XSS in tag datalist~~ — fixed; datalist built with `replaceChildren` +
  `createElement` (L2178).
- ~~#3 serverTimestamp sentinel in IndexedDB~~ — fixed; `snapshotState()` is
  plain, `firestoreSnapshot()` adds the sentinel only for the Firestore path
  (L1766–1789).
- ~~#5 permissive sanitizeState~~ — fixed; per-row/track/block validation and
  coercion (L1699–1764).
- ~~#7 empty stub functions~~ — removed.
- ~~#6 Firestore rules lack range checks~~ — added in the repo rules file…
  but see **A1**: the rules file is now wrong in a worse way.
- #2 remote-overwrite race — *improved* (`_hasPendingLocalSave` guard,
  `_skipSaveOnNextPostRender` anti-ping-pong) but not closed; see **A3**.
- #4 migrateV1ToV2 anchor dependence — subsumed by **A2**, which is the same
  anchor problem and is live, not latent.

---

## A — Act now (real bugs)

### A1. `firestore.rules` no longer matches what the app writes
`snapshotState()` now sends `headerWidth` and `resourceSummaryWidth` and
(deliberately, since 28855a7) omits `rowHeights`. The rules file:

- `requiredStateKeys()` still **requires** `rowHeights` → `hasAll()` fails;
- `hasOnly([...])` doesn't include `headerWidth` / `resourceSummaryWidth` →
  fails again.

If this file is deployed, **every save and every backup is rejected**. Since
multi-client sync demonstrably works, the *deployed* rules must be an older,
looser version — meaning the f47b005 hardening (key allowlists, range checks)
is probably not in effect in production either. `git log` confirms the rules
file was last touched ~60 commits ago.

**Action**: update the rules — drop `rowHeights` from required keys (keep it
in `hasOnly` so stale clients can still write), add `headerWidth` (int,
240–720) and `resourceSummaryWidth` (int, 88–360) to both key lists with range
checks — then `firebase deploy --only firestore:rules` and verify a save
succeeds from the live app.

### A2. Year rollover silently shifts every block
`PROJECT_START_DATE` is derived from `new Date().getFullYear()` (L1583–1585)
and `saved.projectStart` is *deliberately ignored* on load (L4499). Block
`start` values are weekday indices **relative to that anchor**. On
1 Jan 2027 the anchor jumps a year forward, so every stored block renders
~261 weekday columns earlier than where it was placed. The data isn't
corrupted, but the display is, and the first save after that re-persists the
same indices against a comment claiming they're anchor-relative.

**Action**: on load, compare `saved.projectStart` with the current anchor and
shift every `block.start` by the weekday delta between the two (one-time
migration per anchor change). Keep persisting `projectStart` as now.

### A3. Remote apply can still land mid-interaction
The `_hasPendingLocalSave || _saveTimer` guard only covers the 200 ms debounce
window. During an active drag (`dragState`, `createState`, row/track resize)
nothing is pending yet, so a remote snapshot can call `applyRemoteState`,
replace `rows`/`blocks`, and orphan the `row` / `startHeights` objects captured
by `startRowResize`/`startTrackResize` — the exact detached-object bug class
ea60cf6 just fixed from the other direction. Separately, when the guard *does*
trip, the remote update is dropped entirely ("Local changes kept") rather than
deferred.

**Action**: stash the latest remote snapshot instead of applying/dropping it
when any drag state or pending save exists, and apply it on pointerup/after
save completes.

---

## B — Dead and redundant code (delete)

### B1. The side data-panel is gone but half its code remains
There is no `#data-panel` or `#panel-blocks` element in the DOM (the Data tab
renders a table view instead). Dead as a result:

- `renderPanel()`'s entire list branch (L3270–3400) — `if (!list) return`
  always returns; only the stats half (used by the table view) runs.
- `startEditingPanelRowLabel`, `startEditingPanelBlockLabel` (only called from
  the dead branch).
- `startRowDrag(…, 'panel')` source path, including
  `document.getElementById('data-panel').getBoundingClientRect()` (L4063)
  which would **throw** if it were ever reached.
- CSS: `.data-panel`, `.panel-resize-handle`, `body.panel-resizing`,
  `.panel-blocks`, `.panel-block-item` (+ `.block-name`/`.block-meta`
  children), `.panel-name-input`, `.panel-block-name-text`, `.panel-row-label`.
  (Keep `.panel-header/.panel-stats/.panel-settings/.panel-section-*` — the
  table view reuses those.)
- `panelWidth` state: persisted, synced, range-checked in rules… and controls
  nothing. Keep writing it for rules back-compat until A1 lands, then drop it
  everywhere in one pass.

**Action**: delete the above; rename the surviving `renderPanel` to
`renderStats` to stop implying a panel exists.

### B2. Custom row-drag fallback is broken and unreachable in practice
The hand-rolled reorder machinery (`startRowDrag`, the `rowDragState` branches
in document `mousemove`/`mouseup`, `rowDragIndicator` — ~250 lines) only
activates when the SortableJS CDN fails. Under the current DOM it's broken
anyway: it queries `.row[data-row-id]`, which now matches **track lanes**, not
`.resource-group` rows, so it would drag a single lane; and its drop indicator
can never appear because `.row-reordering-indicator { display:none !important; }`
(L1160) overrides the inline `display:block`.

**Action**: delete the fallback and its CSS (`row-reordering`, `row-dragging`,
`row-lifted`, `row-drag-placeholder`, both `.row-reordering-indicator` blocks,
the duplicated conflicting `.panel-section-header.row-dragging` rules). If
offline reorder matters, vendor Sortable.min.js into the repo instead — it's
15 KB and removes the CDN failure mode entirely.

### B3. Dead functions
`togglePanel`, `renameBlock`, `cycleColor`, `duplicateBlock` have **no
callers** (the context menu no longer wires them). Note `duplicateBlock` is a
feature loss — decide whether to re-add "Duplicate" to the context menu or
delete the function; the other three just go.

### B4. Legacy `rowHeights` remnants
Persistence stopped in 28855a7, but the global `let rowHeights`, the
assignment in `applyRemoteState` (L1920 — directly above a comment saying not
to use it), `delete rowHeights[rowId]` in `deleteResource`, and the `init`
assignment all survive. Only the `sanitizeState` read (legacy seed,
L1716/L1754) is still needed. The vestigial `row.height` field (`makeRow` sets
`height: 0`) can go in the same pass once rules allow it.

### B5. Double-save convention
`schedulePostRenderWork()` calls `saveState()` after **every** render, yet
most mutators also call `saveState()` explicitly — so nearly every edit
schedules two debounced saves, and pure view changes (zoom, Escape-cancel of a
label edit) write to Firestore at all. Small but everywhere:
`deleteBlock(id); saveState();` double-saves twice over.

**Action**: pick one convention. Recommended: mutators call `saveState()`
explicitly; remove the implicit save from `schedulePostRenderWork` (first
fixing the few spots that silently rely on it: add-row click,
`updateContingency`, Sortable `onEnd` paths already save — audit the rest).
This also removes the need for the `_skipSaveOnNextPostRender` flag.

---

## C — Smaller / hygiene

1. **`block._lastClickAt` is persisted** — the dblclick detector writes a
   transient field onto block objects, which then goes to Firestore and
   IndexedDB (and currently passes rules only because blocks aren't
   key-checked). Track last-click in a local Map keyed by id, or strip `_`
   fields in `snapshotState`.
2. **View state is shared state** — `zoom`, `headerWidth`,
   `resourceSummaryWidth` sync across clients, so changing zoom on the laptop
   changes the desktop. This was the breeding ground for the recent sync-loop
   bugs. Consider moving pure view prefs to `localStorage` and slimming the
   shared doc to data.
3. **Block move/resize still uses mouse events** (`startMove`, `startResize`,
   document `mousemove`/`mouseup`) while every other handle was migrated to
   Pointer Events in 02dbe0d. Works on desktop; no touch, and it's the last
   place the wedged-handle bug class could recur. Migrate to
   `beginPointerDrag`.
4. **Console noise** — `console.log('State loaded from Firestore')` (L1825)
   and the v1-migration log (L4488) still ship.
5. **Tags**: no length cap or case normalization on input (`addTagToRow`);
   `sanitizeState` caps count (50) but not string length.
6. **Unused var** `weekDay` (L2702).
7. **`fmtDays`/`fmtBlockDuration` via innerHTML** — still numbers-only, still
   a footgun if a label ever gets interpolated. Unchanged from old #19/20.
8. **`location.reload()` on sign-in/out** — unchanged from old #21; heavy but
   harmless.
9. **`_render` is ~470 lines** — unchanged in spirit from old #8. Quality
   only; extract per-region helpers when next touching it.
10. **`applyRemoteState` forces `_renderScheduled = false`** before `render()`
    — if a rAF was already queued this double-renders. Harmless with the rAF
    batch but the flag reset is a smell; let the existing batch coalesce.

---

## Suggested sequencing

1. **PR 1 (rules)**: A1 — reconcile + deploy `firestore.rules`; verify save,
   backup, and a stale-client write.
2. **PR 2 (dead code)**: B1–B4 + C4/C6 — pure deletion, ~400+ lines and ~80
   lines of CSS, no behavior change. Do this before any further resize work;
   it shrinks the surface the sync bugs live in.
3. **PR 3 (anchor)**: A2 — projectStart migration on load. Test by faking
   `TIMELINE_BASE_YEAR + 1`.
4. **PR 4 (sync)**: A3 + B5 + C1/C2 — one save convention, deferred remote
   apply, transient fields out of the doc, view prefs local.
