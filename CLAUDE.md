# Project: Habit Tracker + Shopping List

Two standalone PWA apps sharing a Firebase project. No build step, no framework, no dependencies beyond Firebase CDN. Each app is a single self-contained HTML file.

## File structure

```
index.html            — Habit Tracker app
shopping.html         — Shopping List app
sw.js                 — Service worker for Habit Tracker
sw-shopping.js        — Service worker for Shopping List
manifest.json         — PWA manifest for Habit Tracker
manifest-shopping.json — PWA manifest for Shopping List
icon.svg              — Habit Tracker icon (green, abstract)
icon-shopping.png     — Shopping List icon (blue cart on gray)
icon-shopping.svg     — Shopping List icon (generated, superseded by PNG)
```

## Tech stack

- **Vanilla JS + HTML + CSS** — no framework, no bundler, no npm
- **Firebase Firestore** (compat SDK v10.12.2, loaded from CDN) — real-time sync
- **localStorage** — primary local storage, Firestore is the sync layer on top
- **Service Worker** — offline caching (PWA)
- **Inline SVGs** — all icons are inline SVG strings, no icon font (except Tabler Icons CDN used in `index.html` for some decorative icons)

## Architecture pattern

Both apps follow the identical pattern:

```
load()        → read from localStorage → render()
save()        → write to localStorage + write to Firestore
attachSync()  → Firestore onSnapshot listener → on cloud update, overwrite local if cloudTs > localTs → render()
```

**Conflict resolution**: timestamp-based. The Firestore `updatedAt` server timestamp is compared against a locally stored `updatedAt`. Cloud wins if it is newer. This is last-write-wins — suitable for personal/family use, not for high-concurrency edits.

**Sharing model**: no authentication. Each device generates a random 32-char hex key stored in localStorage. Users share the key manually (copy/paste via the Sync popup) to link devices. All data for a key lives in a single Firestore document.

## Firebase config

Both apps use the **same Firebase project** and **same config block**:

```js
const FIREBASE_CONFIG = {
  apiKey: "AIzaSyCNLpnZFXsZkcDvi4cc0L5tV3KE1S_Sfhy",
  authDomain: "habit-tracker-claude-cc@1a.firebaseapp.com",
  projectId: "habit-tracker-claude-cc01a",
  storageBucket: "habit-tracker-claude-cc0la.firebasestorage.app",
  messagingSenderId: "646881871024",
  appId: "1:646881871024:web:1414bf@db728ec7160a775"
};
```

**Firestore security rules** (must be set in Firebase console):
```
match /habits/{k}   { allow read, write: if k.matches('[0-9a-f]{32}'); }
match /shopping/{k} { allow read, write: if k.matches('[0-9a-f]{32}'); }
```

## localStorage key namespaces

Each app uses its own prefix to avoid collisions:

| App           | Prefix  | Keys                                              |
|---------------|---------|---------------------------------------------------|
| Habit Tracker | `ht3_`  | `syncKey`, `updatedAt`, `habits`, `checks`, `times` |
| Shopping List | `sl1_`  | `syncKey`, `updatedAt`, `lists`, `listOrder`, `activeListId`, `notifsEnabled` |

`activeListId` and `notifsEnabled` are device-local and are **not** synced to Firestore.

## Data models

### Habit Tracker (`habits` Firestore collection)
```js
{
  habits: [{ id, name, subs: [{ id, name }] }],
  checks: { "habitId|YYYY-MM-DD": 1 },
  times:  { "habitId|YYYY-MM-DD": minutes },
  updatedAt: serverTimestamp()
}
```
- `checks` and `times` older than 365 days are pruned on load (`pruneOldData()`)
- IDs: habits use `'h' + Date.now()`, subs use `'s' + Date.now()`

### Shopping List (`shopping` Firestore collection)
```js
{
  lists: {
    "<listId>": {
      name: "Tesco",
      items: [{ id, name, qty, checked: bool, addedAt: timestamp }]
    }
  },
  listOrder: ["listId", ...],   // tab display order, synced
  updatedAt: serverTimestamp()
}
```
- IDs: lists use `'l' + Date.now()`, items use `'i' + Date.now()`
- New items are prepended (`unshift`) so newest appear first
- Items are displayed unchecked first, checked (strikethrough/dimmed) at bottom
- `listOrder` and `lists` are both synced; `activeListId` is device-local only

## CSS conventions

- CSS variables for theming: `--color-text-primary/secondary/tertiary`, `--color-background-primary/secondary`, `--color-border-primary/secondary/tertiary`
- Dark mode via `@media (prefers-color-scheme: dark)` overriding the same variables
- Accent colors: Habit Tracker `#1D9E75` (green), Shopping List `#3B82F6` (blue) + `--accent-dark` variant
- Border radius tokens: `--border-radius-md: 8px`, `--border-radius-lg: 12px`
- All user-provided strings rendered into innerHTML must go through `esc()` (HTML escaping) and `escAttr()` for attribute contexts

## Icon conventions

- Inline SVG strings are declared as constants at the top of the script block
- `ICON_TRASH` (15×15), `ICON_CHEVRON`/`ICON_X` in `index.html`
- `ICON_TRASH`, `ICON_CHECK`, `ICON_PENCIL`, `ICON_BELL`, `ICON_BELL_OFF` in `shopping.html`
- No emoji in code unless decorating an empty state

## Service worker conventions

- Each app has its own SW with a distinct cache name (`habit-tracker-vN`, `shopping-vN`)
- On activate, each SW only deletes caches **prefixed with its own name** to avoid wiping the other app's cache
- `shopping.html` always fetched network-first; other local assets cache-first
- CDN resources (Firebase SDK) are network-first with cache fallback

## Deployment

Deployed via **GitHub Pages** from the `main` branch — no CI, no build step. Pushing to `main` publishes immediately. Both apps are accessible as sibling URLs:
- `https://<user>.github.io/<repo>/`            → Habit Tracker
- `https://<user>.github.io/<repo>/shopping.html` → Shopping List

## Notifications (Shopping List only)

`shopping.html` uses the browser **Web Notifications API** to alert the user when another device adds items to a shared list.

- A bell button in the header toggles notifications on/off (requests browser permission on first enable)
- Preference stored in `sl1_notifsEnabled` localStorage key (device-local)
- Notifications only fire when `document.hidden` — if the app is in the foreground the real-time render is sufficient
- Items added locally are tracked in `locallyAddedIds` (an in-memory `Set`) so the adding device never self-notifies
- The first sync after attaching (`localUpdatedAt === 0`) is always skipped to avoid a flood on initial load
- Logic lives in `attachSync()`: diffs old vs new item IDs on each snapshot, fires `new Notification(...)` for new IDs not in `locallyAddedIds`
- No push server or FCM involved — relies entirely on the Firestore `onSnapshot` WebSocket staying alive (works while browser/PWA is backgrounded; "unrestricted battery" on Android helps keep the connection alive)

## Things to avoid

- Do not add a framework, bundler, or npm
- Do not split the apps into multiple JS/CSS files — keep each app in its single HTML file
- Do not add a backend — Firebase is the only server-side component
- Do not introduce auth — the sync key IS the auth
- Do not use `prompt()` or `alert()` for user input — use the popup overlay pattern already established
- Do not add comments explaining what code does — only add comments for non-obvious WHY
