# Plan: Replacing the current timeline with @gravity-ui/timeline (or its behaviour)

Reference: https://gravity-ui.com/libraries/timeline · https://github.com/gravity-ui/timeline

## What we want

1. **Dynamic (continuous) zoom** — smooth wheel/pinch zoom instead of 5 fixed steps.
2. **Adaptive ruler** — the header shows the right granularity (days ↔ weeks ↔ months) as
   the zoom level changes, like the Gravity ruler does.
3. **Keep the zoom dropdown** — Month / Week / Day / Day+ / ½ Day become named presets
   that jump the continuous zoom to a known px-per-day value.

## What the library gives us vs. what we'd lose

| Capability | Gravity timeline | Our current timeline |
|---|---|---|
| Rendering | Canvas, virtualized, fast | DOM (divs per block), CSS-painted grid |
| Zoom | Continuous, smooth, `zoomLevels` config | 5 fixed `dayPx()` steps |
| Ruler | Auto day/week/month ticks from real time | Hand-built month/week/day header rows |
| Time model | **Continuous real time (ms timestamps)** | **Weekday-only columns (weekends removed)** |
| Events | `from`/`to` render on axes/tracks | Blocks with capacity scaling, contingency tails, holiday extensions |
| Interactions | click/select/hover, marker group-zoom, Drag integrations | drag-create, drag-move, edge-resize, dependency arrows, context menus, per-track resize |
| Integration | React (`useTimeline`) or direct JS (`new Timeline()` + `timeline.init(canvas)`) | Vanilla JS, single-file, no build step |

**The two hard mismatches:**

1. **Weekday-only grid.** Gravity renders real time, so Sat/Sun occupy width and its ruler
   labels real dates. We'd either (a) accept weekends appearing (blocks spanning Fri→Mon
   would render wider than 2 "days of work"), or (b) map our weekday-column domain onto a
   fake continuous domain (1 col = 24h of virtual time) and **replace the ruler renderer**
   so labels come from `colToDate()` — at which point we're no longer using the part of the
   library we wanted most (its date-aware ruler).
2. **Interaction depth.** Blocks here are editable objects: inline label inputs, capacity
   tails, holiday logic, dependency handles, right-click menus, per-track lanes with DOM
   row headers that stay sticky. Rebuilding all of that inside a canvas is a rewrite of
   ~2/3 of the app, not a swap of the timeline layer.

## Recommendation: two-phase approach

### Phase 1 (recommended, low risk): adopt the *behaviour*, not the library

Implement Gravity-style continuous zoom + adaptive ruler natively in the existing DOM
timeline. This gets everything in "What we want" without rewriting interactions.

1. **Continuous `dayPx`.**
   - Replace the 5 zoom constants with a single float `dayPxValue` (clamped ~3 → 320).
   - `dayPx()` returns `dayPxValue`; wheel/ctrl+wheel multiplies it by ~1.12 per notch,
     anchored under the cursor (we already anchor correctly after the last change).
   - Persist `dayPxValue` in view prefs (keep reading old `zoom` values as fallbacks).

2. **Adaptive ruler (the day/week/month logic).** Choose header rows from px-per-day:
   - `dayPx >= 24` → month row + week row + day-letter row + date row (current Day view).
   - `8 <= dayPx < 24` → month row + week row (current Week view).
   - `dayPx < 8` → month row only (current Month view); collapse to quarter/year labels
     below ~1.5px/day if we ever allow zooming that far out.
   - Reuse the existing month/week/day header builders — they already take `dayPx()`;
     the change is only *which rows render*, driven by thresholds instead of the enum.
   - Grid paint (`painted-day/week/month` backgrounds) picks its repeating gradient from
     the same thresholds so gridlines always match the finest visible header row.

3. **Keep the dropdown as presets.** Month=5px, Week=16px, Day=40px, Day+=160px, ½ Day=320px.
   Selecting one animates/jumps `dayPxValue` to that preset; the button label shows the
   nearest preset name (or e.g. "Day −" between presets).

4. **Smoothness details.**
   - Debounced re-render already exists (rAF-batched `render()`); continuous zoom will
     re-render per wheel notch — acceptable, but add a fast path that only rescales
     `left/width` styles + header rebuild (skip Sortable setup) while a zoom gesture is
     in flight, then do a full render on gesture end.
   - Remove the 140ms wheel throttle; scale per-event `deltaY` instead.

Estimated scope: contained to `dayPx`/`setZoom`/header-render/grid-paint code paths in
index.html. No dependencies, no build step, all interactions untouched.

### Phase 2 (optional, later): evaluate embedding Gravity for the *header/minimap*

If we still want the library itself:

- Use the **direct JS API** (`new Timeline({settings, viewConfiguration})` +
  `timeline.init(canvas)`) via an ESM CDN import — no React needed.
- Embed it as a **read-only overview strip** (minimap) above the grid: real-time domain,
  our milestones as `markers`, holiday ranges as `sections`, blocks as `events`.
  Sync its visible range ↔ our `scrollLeft`/`dayPxValue` both ways.
- This sidesteps both mismatches: the canvas never needs our editing interactions, and
  weekends in the minimap are fine (it's an overview).
- Only if that proves out would a full replacement (blocks as canvas events with custom
  renderers + DragHandler integration) be worth costing; today it's a rewrite of block
  editing, dependencies, tracks, and the data-pane sync for little user-visible gain over
  Phase 1.

## Suggested implementation order (Phase 1)

1. `dayPxValue` state + persistence + preset mapping (dropdown keeps working). 
2. Wheel zoom → continuous, anchored (build on existing `setZoom` anchor math).
3. Threshold-driven header rows + grid paint selection.
4. Fast-path rescale during gestures; full render on end.
5. QA: block drag/resize/create at arbitrary `dayPx`, dependency arrows, milestone lane,
   today line, "Today" jump, week-position preservation across preset jumps.

## Risks

- A few places assume the zoom enum (`isDayLikeZoom()`, `daysPerCell()`, week background
  offset, half-day AM/PM row) — each needs a threshold equivalent.
- Saved prefs migration: old `zoom: 'week'` → `dayPxValue: 16`.
- Very small `dayPx` (<3px) makes block borders/labels degenerate; clamp and hide labels
  below a readability threshold (Gravity does the same via virtualization/culling).
