# CLAUDE.md

## Commands

```bash
npm run start         # Build in watch mode (dev, both Chrome + Firefox targets)
npm run build         # Production build + zip archives for both targets
npm run check-types   # TypeScript type checking (use this, not npx tsc)
npm run lint          # Lint
npm run lint -- --fix # Lint and auto-fix all auto-fixable errors
npm test              # Run tests
npm test -- --testPathPattern=tabUtil  # Run a single test file by pattern
```

## Architecture

Tab Wrangler is a browser extension with three compiled entry points:

- `app/background.ts` — service worker; owns the tab-closing loop, writes `tabTimes` to
  `chrome.storage.local`, updates the badge
- `app/popup.tsx` / `app/options.tsx` — React UIs; both boot the same `App.tsx` shell

**Build** uses a dual-compiler webpack setup producing `dist/chrome/` (Manifest v3 service worker)
and `dist/firefox/` (Manifest v3 background scripts array). `webpack.config.js` is dev;
`webpack.production.config.js` wraps it and adds a zip plugin.

### Communication between background and UI

There is no message-passing protocol between background and UI (beyond a single `"reload"` command).
All coordination goes through storage events:

- Background writes → storage → UI React Query hooks invalidate → re-render
- UI mutates settings/locks → storage → background `onChanged` listener reacts

### Storage layout

| Area | Key | Contents |
|------|-----|----------|
| `sync` | `persist:settings` | All user settings (pause state, thresholds, whitelist, locked IDs, …) |
| `local` | `persist:localStorage` | Saved/closed tabs, statistics, install date |
| `local` | `tabTimes` | `{ [tabId]: lastAccessedTimestamp }` — written frequently, kept separate |
| `local` | `pausedAt` | Timestamp when paused; absent when running |

`app/js/storage.ts` wraps all storage access. Every mutating function acquires `ASYNC_LOCK`
(5s max) to prevent race conditions. React Query hooks (`useStorageSyncPersistQuery`,
`useStorageLocalPersistQuery`) subscribe to `chrome.storage.onChanged` and self-invalidate.

### Settings system (`app/js/settings.ts`)

Singleton with synchronous `get(key)` / async `set(key, value)` / `subscribe(key, fn)`. Loaded from
`chrome.storage.sync` at startup. In React components, use the `useSetting(key)` hook (backed by
`useSyncExternalStore`) rather than reading `settings.get()` directly, so components re-render on
change.

### Tab-closing logic (`app/js/tabUtil.ts`)

`findTabsToCloseCandidates(tabTimes, tabs)` is the core function. It filters out locked tabs
(pinned, grouped if `filterGroupedTabs`, audible if `filterAudio`, in a window with a pinned tab if
`filterPinnedWindows`, whitelisted, manually locked, window-locked), respects `minTabs` /
`minTabsStrategy` (`"givenWindow"` counts per window; `"allWindows"` counts across all windows), and
returns the oldest eligible tabs up to the configured `maxTabs` limit.

The per-tab lock decision lives in `getTabLockStatus(tab, options)` (and the boolean wrapper
`isTabLocked`). It is pure and has no access to sibling tabs, so any *window-level* rule must be
computed by the caller and threaded in as a flag — see the pinned-window filter below.

### UI layer

- `LockTab/` — open-tabs view with countdown timers, lock controls, window grouping
- `CorralTab/` — closed-tabs view; uses `react-virtualized` for large lists
- `OptionsTab/` — settings form
- React Query handles storage-as-server-state; `UndoContext` provides undo for tab operations

### i18n

All user-visible strings go through `chrome.i18n.getMessage(key)`. Messages live in
`_locales/{locale}/messages.json`. When adding a new string, add it to `_locales/en/messages.json`
with a `description` field (used by translators on Crowdin) and `placeholders` for any substitution
values.

---

## Feature: Pinned-window filter (`filterPinnedWindows`)

### Summary

When enabled, **no tab in any window that contains at least one pinned tab is auto-closed** — the
pinned tab itself *and* all of its non-pinned siblings in that window are exempt. This is distinct
from the always-on rule that protects the pinned tab alone. It is exposed as a checkbox in Options
("Don't close any tabs in windows that have a pinned tab") and **defaults to ON**. Affected tabs also
render as locked (reason "Window pinned") in the open-tabs (Lock) UI.

### Why the design looks the way it does

The natural place for this rule is the per-tab lock check `getTabLockStatus`, but that function is
**pure and per-tab** — it cannot see a tab's siblings, so it cannot answer "does this tab's window
contain a pinned tab?" on its own. Rather than pass the whole tab list into the lock check (which
would change its shape and every call site substantially), the rule is split:

- **Callers compute the window context.** Whoever holds the in-scope tabs builds
  `windowIdsWithPinnedTab = new Set(tabs.filter(t => t.pinned).map(t => t.windowId))` and passes a
  per-tab boolean `windowHasPinnedTab` into the lock check.
- **`getTabLockStatus` just consumes the boolean.** It gains two options, `filterPinnedWindows` and
  `windowHasPinnedTab`, and returns the new `"pinnedWindow"` lock reason when both are true. The
  check is placed immediately **after** the `tab.pinned` check so a pinned tab keeps reporting
  `"pinned"` and only its siblings report `"pinnedWindow"`.

This works for **both** `minTabsStrategy` modes with zero special-casing, because
`findTabsToCloseCandidates` already receives exactly the relevant `tabs` slice: one window's tabs for
`"givenWindow"`, all tabs for `"allWindows"`. Computing the pinned-window set from that slice is
correct either way. Excluded tabs drop out of `unlockedTabs`, so they also correctly stop counting
toward `minTabs`.

### Data flow

```
Setting (sync storage: filterPinnedWindows)
        │
        ├── background close loop ─→ findTabsToCloseCandidates(tabTimes, tabs)
        │       computes windowIdsWithPinnedTab from `tabs`,
        │       calls isTabLocked(tab, { …, filterPinnedWindows, windowHasPinnedTab })
        │
        └── Lock UI ─→ WindowCard (knows its window's tabs) computes windowHasPinnedTab,
                passes it to OpenTabRow → settings.getTabLockStatus(tab, { windowHasPinnedTab })
                → renders "Window pinned" lock reason.
            LockTab computes a windowIdsWithPinnedTab set across all tabs for the
                cross-window unlocked-tab count (the "allWindows" minTabs badge).
```

### Settings-singleton wrappers

`settings.getTabLockStatus` / `isTabLocked` / `isTabManuallyLockable` take an **optional**
`{ windowHasPinnedTab?: boolean } = {}` second arg, defaulting to `false`. This keeps every existing
call site compiling unchanged; only the UI sites that have window context pass the flag. The core
`getTabLockStatus`/`isTabLocked` in `tabUtil.ts` keep the two fields **required** so the compiler
forces every direct caller to be explicit.

### Deliberately scoped-out surfaces (known limitations, not bugs)

These surfaces call the per-tab lock check without window context and intentionally pass
`windowHasPinnedTab: false`, so they do **not** reflect the pinned-window rule. Reflecting it would
require an extra window query or a larger refactor for little benefit:

- **Toolbar action icon** (`app/background.ts`, `updateIcon`) — the locked/unlocked icon for the
  active tab.
- **Context-menu "Lock Tab" checkbox** (`app/js/menus.ts`) — uses `settings.isTabLocked(tab)` with
  no opts.
- **`ChronoSorter`** (`app/js/LockTab/LockTab.tsx`) — the "time until close" sort only receives two
  tabs and no window context, so pinned-window tabs are not pushed to the bottom. The default sorter
  is tab-order, so this is cosmetic.

### Files changed

| File | Change |
|------|--------|
| `app/js/tabUtil.ts` | New `"pinnedWindow"` `TabLockStatus` reason; `filterPinnedWindows` + `windowHasPinnedTab` options on `getTabLockStatus`/`isTabLocked`; `findTabsToCloseCandidates` computes `windowIdsWithPinnedTab` and threads the flag. |
| `app/js/settings.ts` | `filterPinnedWindows: boolean` in `SettingsSchema` + `SETTINGS_DEFAULTS` (default `true`); optional `{ windowHasPinnedTab }` on the singleton lock wrappers. |
| `app/js/OptionsTab/OptionsTab.tsx` | New checkbox next to `filterAudio`/`filterGroupedTabs`. |
| `app/js/LockTab/WindowCard.tsx` | Computes `windowHasPinnedTab` from its window's tabs; uses it in the unlocked count and passes it to `OpenTabRow`. |
| `app/js/LockTab/OpenTabRow.tsx` | New `windowHasPinnedTab` prop; threads it into the lock-status / manually-lockable checks; renders the `"pinnedWindow"` reason. |
| `app/js/LockTab/LockTab.tsx` | Cross-window unlocked-tab count accounts for pinned windows. |
| `app/background.ts` | Icon `lockOptions` gains the two new fields (`windowHasPinnedTab: false`). |
| `_locales/en/messages.json` | `options_option_filterPinnedWindows_label`, `tabLock_lockedReason_pinnedWindow`, `tabLock_lockedReason_pinnedWindow_tooltip`. |
| `app/js/tabUtil.test.ts` | New `getTabLockStatus` and `findTabsToCloseCandidates` cases; `defaultOptions`/`mockSettings` extended. |

### Change log

- **2026-06-27** — Added the `filterPinnedWindows` feature (default ON). Window-level rule threaded
  through the per-tab lock check via a `windowHasPinnedTab` flag; new `"pinnedWindow"` lock reason
  surfaced in the Lock UI. Toolbar icon, context menu, and chrono sorter intentionally not wired up.
  All 73 tests green; `check-types` and `lint` clean.
