# Blocks — Firebase Realtime Sync Spec

## What
A private project timeline app. Authorized browsers stay in sync in real-time.
Single shared document, Firebase Auth required. Initial deployment supports one approved user email.

## Terminology
- Map rows are called **resources**.
- Resources are usually individual people.

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
- 2026-05-31 — Added Firebase Auth gate, single-user allowlist, immutable backups, safer Firestore rules
- 2026-04-12 — Firebase real-time sync added (Firestore primary + IndexedDB fallback)
