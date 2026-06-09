# Blocks — Code Review

**File**: `index.html` (3,631 lines, ~123 KB, single-file app)
**Branch**: `preview`
**Date**: 2026-06-01

A pragmatic, single-author private tool. Most of the code is well-structured. The
findings below are sorted by severity.

---

## Strengths

- **Real-time sync done right.** `_clientId` echo prevention (L1487), immutable
  backups (L1393), throttled auto-backup (AUTO_BACKUP_MIN_INTERVAL_MS, L1304),
  separate Firestore + IndexedDB paths. Clean.
- **XSS-safe by default.** User-controlled strings (`row.label`, `block.label`,
  tag chips) are consistently set via `textContent` (25 occurrences) — not
  `innerHTML`. Good.
- **rAF-batched render** (L1809-1817) avoids layout thrash on 26 call sites.
- **State migration path** from v1 (calendar) → v2 (weekday) and v3 (grid/data
  kinds) is present.
- **Auth gate + Firestore rules agree** — both allowlist `dan.croucher@gmail.com`.
- **`firebaseInit()` early-returns** if SDK fails to load (L1252) — no crash.
- **No `eval`, no `document.write`, no `outerHTML`, no `insertAdjacentHTML`**.
- **Row reordering** is properly debounced through render.
- **No duplicate function declarations** and no shadowed `let`/`const`.

---

## Critical / high

### 1. XSS in tag datalist (L1654)
```js
dl.innerHTML = existing.map(t => `<option value="${t}">`).join('');
```
Tag strings are user-controlled, stored unescaped in Firestore, and interpolated
into an attribute value with no escaping. A tag like `"><img src=x onerror=…`
breaks out of the attribute.

**Today**: single approved user, so not exploitable in practice.
**Tomorrow**: SPEC mentions "approved user email" as the *initial* deployment
model, implying multi-user is planned. **Fix now.**

Suggested fix — build the datalist with `createElement`:
```js
existing.forEach(t => {
  const o = document.createElement('option');
  o.value = t;
  dl.appendChild(o);
});
```

### 2. `applyRemoteState` overwrites unsaved local state (L1500-1529)
```js
if (data.rows)     { rows           = data.rows; }
if (data.blocks)   { blocks         = data.blocks; }
```
If a local change is in the 200 ms debounce window of `saveState()` and a remote
update arrives, the local change is silently lost. With 200 ms debounce this is
unlikely to bite, but a drag that lasts seconds (a real risk during
move/resize) easily overlaps a remote snapshot. **Race condition** with no
conflict resolution.

Suggested fix: include the local change in a 3-way merge, or at minimum show a
"remote overwrite" toast and let the user re-issue their change. Easier fix:
queue local mutations and replay on conflict.

### 3. `snapshotState` mixes `serverTimestamp()` sentinel with IndexedDB
serialization (L1381-1391, L1473)
```js
updatedAt: firebase.firestore.FieldValue.serverTimestamp(),
...
db.transaction(STORE, 'readwrite').objectStore(STORE).put(snapshot, STATE_KEY);
```
`snapshotState()` is passed to **both** Firestore (which understands the
sentinel) and IndexedDB (structured-clone). The Firestore sentinel is a plain
object with a method-shaped marker; it may serialize as `{}` or throw on
`put()`. You only see this when the Firestore `set` succeeds *and* the
IndexedDB write fails — easy to miss in testing.

Suggested fix: build two snapshots, or strip server timestamps before
IndexedDB:
```js
const persistable = { ...snapshot, updatedAt: Date.now() };
db.transaction(...).put(persistable, STATE_KEY);
```

---

## Medium

### 4. `migrateV1ToV2` is anchor-dependent (L3537-3556)
```js
const calIsWeekend = (col) => (col % 7) === 5 || (col % 7) === 6;
```
This assumes calendar col 0 = Sunday. If the v1 data was indexed against a
project-start year that began on a different weekday, the migrated blocks land
in the wrong weekday column. Latent — only matters when migrating legacy v1
data.

Suggested fix: store `projectStart` (an ISO date) on the v1 doc and recompute
weekday columns from that anchor.

### 5. `sanitizeState` is too permissive (L1375-1379)
```js
function sanitizeState(data) {
  if (!data || typeof data !== 'object') return null;
  if (!Array.isArray(data.rows) || !Array.isArray(data.blocks)) return null;
  return data;
}
```
Validates only the top-level arrays. A malformed individual row/block (missing
`id`, `start`, or `duration` of wrong type) crashes `normalizeBlocks`,
`render()`, etc. Firestore rules are stricter, but they only check that
`rows`/`blocks` are lists and `version` is int.

Suggested fix: validate each row/block shape and either drop or coerce bad
entries.

### 6. Firestore rules don't validate numeric ranges
```js
allow create, update: if isAllowedUser()
  && request.resource.data.version is int
  && request.resource.data.rows is list
  && request.resource.data.blocks is list;
```
No bound checks on `contingencyPct` (0–100?), `panelWidth` (0–window?),
`zoom` (must be in `MAP_ZOOM_LEVELS`?), or block `duration`/`start` (must be
non-negative integers). A compromised approved account can poison the doc and
brick the client for everyone.

Suggested fix:
```js
&& request.resource.data.contingencyPct is int
&& request.resource.data.contingencyPct >= 0
&& request.resource.data.contingencyPct <= 100
```

### 7. Empty stub functions at the end of file (L3624-3625)
```js
function applyPanelWidth() {}
function setupPanelResize() {}
```
Dead code. The SPEC describes a resizable panel, but the implementation never
landed. Either implement or remove.

### 8. `_render` is a 452-line monolith (L2056-2507)
Too much in one function: header rendering, row rendering, block rendering,
today indicator, weekend shading, tag chips, capacity input, etc. Hard to test
or modify safely. **Same story** for `renderPanel` (154 lines) and
`renderCombinedBlocksView`.

Suggested fix: extract per-region helpers (`renderHeader`, `renderRows`,
`renderBlocks`, `renderTodayIndicator`) called from a top-level `_render`.

### 9. `render` is called 26 times across the file
Every state mutation triggers a re-render. rAF batching keeps it cheap, but
compound operations (e.g. `deleteResource` does `render()` + `renderPanel()`)
double-paint. Track the latest set of "dirty regions" and re-render only those.

### 10. `nextRowId` initial value is 6, but `rows` is hard-coded with IDs 1-5
`let nextRowId = 6;` (L1543) and 5 default rows (L1533-1538). This works but
the next ID is implicit from the array literal. Easy to break on a refactor.
`makeRow` already takes the ID; consider `nextRowId++` inline.

---

## Minor

### 11. `console.log` in production (9 found)
- 8 are conditional `console.warn` for errors — fine.
- 1 `console.log('State loaded from Firestore')` (L1427) — useful for
  debugging, but pollutes the console in production. Gate behind a debug flag
  or remove.

### 12. `console.log('migrated', ...)` (L3589)
Same — the migration only runs once, so the message is just noise after the
first migration. Replace with a one-time flag.

### 13. `addTagToRow` has no input length / charset limit (L1597-1604)
Tags persist to Firestore and re-render on every device. No upper bound on
length, no normalization (uppercase vs `Frontend` vs `frontend` become
different tags). Consider `tag = tag.toLowerCase().slice(0, 32)` and trim
whitespace.

### 14. Tag chips duplicate on every `buildTagInput` call (L1652-1654)
The `datalist` is appended to `document.body` and never removed. The
`innerHTML` is reset, but creating a fresh `<datalist>` is leaky if the
function is called many times. The current `if (!dl)` guard handles it.

### 15. `ALLOWED_EMAILS` is case-folded in `isAllowedUser` (L1319) but
Firestore rules compare raw `request.auth.token.email`. If Google ever returns
the email with different casing than the rule, sign-in works in the UI but
every Firestore request is denied. Test this on a real sign-in.

### 16. `duplicateBlock` (L3429-3436) does not copy `kind` explicitly beyond
the default in `createBlock`. The `clone.kind` inherits from `block.kind ||
'grid'`, which is fine, but the rest of the block (`baseDuration`, custom
`color`, any future fields) needs to be revisited each time `createBlock`
gains a new field. Consider a `cloneBlock(source, opts)` helper.

### 17. `setSyncStatus('saving', 'Loading')` is set inside `showSignedIn`
(L1372) but the actual load happens in `loadState()`. The status may say
"Loading" while the real status is "Saved" for a moment. Cosmetic.

### 18. `let nextRowId = 6` vs the default `rows` array
Adding/removing default rows requires editing both the array and the constant.
Refactor to compute the next ID from the array length.

### 19. `fmtBlockDuration` is called via `innerHTML` (L1934, L2379, L2703,
L3056, L3076, L3091) — currently safe because it only formats numbers, but
it's a footgun if a future contributor adds a label or string. Consider
returning a `DocumentFragment` or using `textContent` + sibling `<span>`s.

### 20. `renderMapControls` (L1818) uses `innerHTML` with template literal.
All interpolations are computed strings (`MAP_ZOOM_LEVELS[i]`, numeric values).
Safe today, but again a footgun.

### 21. `setupAuthUI` (L1346) calls `location.reload()` after sign-in and
sign-out (L1354, L1362). Reloading on sign-out is a heavy hammer that throws
away all client state. A targeted UI swap (show auth screen, hide app) would
be cheaper and feel snappier.

### 22. Drag state machine is implicit
`dragState`, `createState`, `rowDragState`, `panState` are global mutable
objects. A bug in any handler can leave a stale drag state and lock the UI.
Consider a small `DragController` class with explicit `begin`/`end` methods
and a final `tearDown` on every path.

---

## Summary of severity

| # | Severity | Area |
|---|----------|------|
| 1 | High | XSS in tag datalist (L1654) |
| 2 | High | Local-vs-remote race (L1500) |
| 3 | High | `serverTimestamp` in IndexedDB (L1388, L1473) |
| 4 | Med  | migrateV1ToV2 anchor bug (L3537) |
| 5 | Med  | Permissive sanitizeState (L1375) |
| 6 | Med  | Firestore rules missing range checks |
| 7 | Med  | Empty stub functions (L3624-3625) |
| 8 | Med  | `_render` is 452 lines (L2056-2507) |
| 9 | Med  | 26 `render()` call sites |
| 10–22 | Low / nit | see above |

The codebase is in good shape for a personal tool. Items 1, 2, 3 are the only
ones I'd treat as real bugs; the rest are quality-of-life.

---

## Suggested first PR

1. Replace the tag datalist `innerHTML` with `createElement` (item 1).
2. Add range checks to Firestore rules (item 6).
3. Strip `serverTimestamp` before the IndexedDB `put` (item 3).
4. Remove the empty stub functions (item 7).
