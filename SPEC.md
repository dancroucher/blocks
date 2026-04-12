# Blocks — Firebase Realtime Sync Spec

## What
A shared project timeline app. All browsers stay in sync in real-time.
Single shared document, no auth required.

## Architecture

### Firebase Project
- **Firestore** (not Realtime Database)
  - Database: `blocks-shared` (default region)
  - Document path: `projects/blocks`
  - One document holds the full app state (rows, blocks, counters, settings)
  - Real-time listener via `onSnapshot` — all open browsers update instantly
- **No auth** — public read/write allowed (anyone with the URL can edit)
  - Firestore rules: allow read, write;

### App State (Firestore document)
```json
{
  "version": 2,
  "rows": [...],
  "blocks": [...],
  "nextBlockId": 10,
  "nextRowId": 6,
  "contingencyPct": 0,
  "zoom": "day",
  "panelWidth": 280,
  "collapsedRows": {},
  "projectStart": "2026-01-01T00:00:00.000Z",
  "savedAt": 1740000000000,
  "updatedAt": 1740000000000
}
```

### Firebase SDK (v9 compat)
- Loaded from CDN: `https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js` + `firebase-firestore-compat.js`
- Uses the **compat** API (window全局, no module bundler needed)
- Anonymous sign-in not used — uses a hardcoded API key for public access

### Persistence Strategy
1. **Firebase primary**: load from Firestore on init, save back on every change (debounced 200ms)
2. **IndexedDB local fallback**: if Firestore fails (offline/no key), fall back to local IndexedDB
3. **IndexedDB bootstrap**: if Firestore doc doesn't exist yet, load from IndexedDB then write to Firestore
4. **Seed on first run**: if neither source has data, create 5 example blocks, save to both

### Load/Save Flow
```
init()
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
  if (data._source === 'local') return; // ignore echo of own writes
  applyRemoteState(data);
});
```
`_source` field prevents write echo: on save, attach `_source: 'local'` to the doc write so the listener ignores its own echo.

### Firestore Security Rules
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /projects/{doc} {
      allow read, write: if true;
    }
  }
}
```

### API Key Handling
- Stored in `firebaseConfig` object in the JS
- Vercel env var `VITE_FIREBASE_API_KEY` injected at build time via `vercel env pull` (or hardcoded in preview deployments for now)
- In production (own Vercel project): set via Vercel dashboard → Settings → Environment Variables
- Note: API key is intentionally permissive for public read/write — accept the trade-off for simplicity

## Implementation Notes

### JS SDK (compat)
```html
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
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
  BLOCKS.md         — changelog
  SPEC.md           — this file
```

## Changelog
- 2026-04-12 — Firebase real-time sync added (Firestore primary + IndexedDB fallback)
