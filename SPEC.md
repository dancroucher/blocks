# Blocks — Firebase Realtime Sync Spec

## What
A private project timeline app. Authorized browsers stay in sync in real-time.
Single shared document, Firebase Auth required. Initial deployment supports one approved user email.

## Terminology
- Map rows are called **resources**.
- Resources are usually individual people.
- Each resource contains one or more **tracks** (sub-rows / lanes). Blocks belong to a track within a resource.

## Architecture

### Firebase Project
- **Firestore** (not Realtime Database)
  - Database: `blocks-shared` (default region)
  - Document path: `projects/blocks`
  - One document holds the full app state (rows, blocks, counters, settings)
  - Real-time listener via `onSnapshot` — all open browsers update instantly
- **Firebase Auth required** — access is limited to an approved user email
  - Firestore rules deny public access, deny deletes, validate basic state shape, and allow immutable backup creation

### App State (Firestore document)
State version is currently **4** (`v4 = resource tracks`). `rows[].tracks` and `blocks[].trackId` are introduced in v4; older documents are migrated forward on load.
```json
{
  "version": 4,
  "rows": [
    {
      "id": 1,
      "label": "Alice",
      "capacity": 100,
      "tags": [],
      "tracks": [
        { "id": "r1-t1", "label": "Main", "height": 40 }
      ]
    }
  ],
  "blocks": [
    { "id": 1, "rowId": 1, "trackId": "r1-t1", "start": 0, "baseDuration": 3, "duration": 3, "label": "", "kind": "grid", "color": 0 }
  ],
  "nextBlockId": 10,
  "nextRowId": 6,
  "contingencyPct": 0,
  "zoom": "day",
  "panelWidth": 280,
  "collapsedRows": {},
  "projectStart": "dynamic-year-start",
  "savedAt": 1740000000000,
  "updatedAt": 1740000000000,
  "updatedBy": { "uid": "...", "email": "owner@example.com" },
  "_clientId": "browser-session-id"
}
```

### Firebase SDK (v9 compat)
- Loaded from CDN: `https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js` + `firebase-firestore-compat.js`
- Uses the **compat** API (window全局, no module bundler needed)
- Google sign-in is used through Firebase Auth
- API key remains public Firebase app config; confidentiality is enforced by Auth + Firestore Security Rules

### Persistence Strategy
1. **Auth gate**: app UI stays locked until Firebase Auth returns an approved user
2. **Firebase primary**: load from Firestore on init, save back on every change (debounced 200ms)
3. **IndexedDB local fallback**: if Firestore fails after sign-in, fall back to local IndexedDB
4. **Automatic backups**: before saves, write throttled immutable snapshots to `projects/blocks/backups/{timestamp}`
5. **IndexedDB bootstrap**: if Firestore doc doesn't exist yet, load from IndexedDB then write to Firestore
6. **Seed on first run**: if neither source has data, create 5 example blocks, save to both

### Load/Save Flow
```
init()
  → requireAuth() [Firebase Auth]
  → loadState()  [Firestore]
    → doc exists?  → restore state → listenForChanges()
    → doc missing? → loadState() [IndexedDB fallback]
                     → write to Firestore (bootstrap)
                     → listenForChanges()
  → local bootstrap if both fail
```

### ListenForChanges
```js
onSnapshot(docRef, (snap) => {
  if (!snap.exists()) return;
  const data = snap.data();
  if (data._clientId === CLIENT_ID) return; // ignore echo of own writes
  if (!sanitizeState(data)) return;
  applyRemoteState(data);
});
```
`_clientId` field prevents write echo: on save, attach the current browser session id so the listener ignores its own echo without hiding updates from other browsers.

### Firestore Security Rules
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isAllowedUser() {
      return request.auth != null
        && request.auth.token.email in ['you@example.com'];
    }

    match /projects/blocks {
      allow read: if isAllowedUser();
      allow create, update: if isAllowedUser()
        && request.resource.data.version is int
        && request.resource.data.rows is list
        && request.resource.data.blocks is list;
      allow delete: if false;

      match /backups/{backupId} {
        allow read, create: if isAllowedUser();
        allow update, delete: if false;
      }
    }
  }
}
```

### API Key Handling
- Stored in `firebaseConfig` object in the JS
- Vercel env var `VITE_FIREBASE_API_KEY` injected at build time via `vercel env pull` (or hardcoded in preview deployments for now)
- In production (own Vercel project): set via Vercel dashboard → Settings → Environment Variables
- The API key is not a secret; security depends on Firebase Auth and Firestore rules
- Approved account: `dan.croucher@gmail.com`

## Resources & Tracks (v4)

Each resource (row) holds an ordered list of **tracks** rendered as grouped grid
rows under a single resource header. Blocks are assigned to a specific track via
`block.trackId`.

### Data model
- `row.tracks`: `[{ id, label, height }]`, 1–`MAX_TRACKS_PER_ROW` (8) entries.
  - `id`: stable string, e.g. `r{rowId}-t{n}` (or a generated unique id).
  - `label`: track name (editable inline; first track defaults to "Main").
  - `height`: px, clamped to `ROW_H_MIN` (30) … `ROW_H_MAX` (360); default `ROW_H_DEFAULT` (40).
- `block.trackId`: which track the block sits in. Falls back to the resource's first track if missing.
- Resource visual height = sum of its track heights + the add-track row, floored at `RESOURCE_HEADER_MIN_H` (56).
- Legacy rows without `tracks` are migrated to a single "Main" track on load.

### Layout
- A resource renders as a horizontal group: a sticky left **header column**
  (`--row-header-width` = 300px) beside the stacked **track lanes** (timeline).
- The header column splits into a fixed **summary** (resource name, capacity %,
  tags) and a **track list** (one label row per track) that is vertically
  aligned with its lanes.
- Sub-track dividers are per-element borders on the track label rows and lanes
  only — the summary (left) section has no horizontal lines.

### Interaction
- **Resize one track**: drag the line dividing it from the track below (handle
  present on both the timeline lane and the header label row; accent on hover).
  The bottom-most divider only resizes a track when a "+ Track" row sits below
  it; otherwise that edge is the resource resize handle.
- **Resize whole resource**: drag the handle at the bottom of the resource
  (all tracks scale together); hold **Shift** to apply to all resources.
- **Add track**: a half-height "+ Track" row (`ADD_TRACK_ROW_H` = 22px) at the
  bottom of each resource (hidden at max tracks). Left-aligned, dim grey label.
- **Reorder tracks**: drag the ⋮⋮ handle on a track label row (SortableJS).
- **Rename track**: inline-editable input on each track label row.
- **Delete track / resource**: × on each track label row; the resource delete ×
  sits in the top-left corner of the resource header.

## Implementation Notes

### JS SDK (compat)
```html
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>
```

### Firebase config (public project — no secrets rotation needed for read-only public data)
```js
firebase.initializeApp({
  apiKey: "AIzaSyD-...",
  authDomain: "blocks-shared.firebaseapp.com",
  projectId: "blocks-shared"
});
```

### File Structure
```
/blocks
  index.html        — single-file app (HTML + CSS + JS)
  firestore.rules   — locked-down production Firestore rules
  BLOCKS.md         — changelog
  SPEC.md           — this file
```

## Changelog
- 2026-06-05 — Resource-header typography tuning: larger resource and sub-track names, more padding above/below the name and between capacity and tags, fixed-height tag chips (consistent regardless of upper/lowercase), and a left-aligned, dimmer "+ Track" label
- 2026-06-04 — Sub-tracks (state v4): resources hold multiple tracks rendered as grouped grid rows with per-block `trackId`; per-track resize by dragging the dividing line (mirrors resource resize), resource-level resize (Shift = all resources), half-height "+ Track" add row, inline-editable track names, track reordering, and the resource delete button moved to the header top-left; dividers no longer cross the summary section
- 2026-06-01 — Timeline window now generates dynamically from the current year through two years ahead; Ctrl/Cmd wheel and Ctrl/Cmd +/- zoom the map around the pointer/viewport center; added Day+, and ½ Day close zoom levels
- 2026-05-31 — Added Firebase Auth gate, single-user allowlist, immutable backups, safer Firestore rules
- 2026-04-12 — Firebase real-time sync added (Firestore primary + IndexedDB fallback)
