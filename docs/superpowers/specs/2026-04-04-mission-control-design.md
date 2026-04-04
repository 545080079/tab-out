# Mission Control — Design Spec

**Date:** 2026-04-04
**Status:** Approved for implementation

## What It Is

A personal browsing dashboard + Chrome extension that answers: "What was I working on, and what did I forget about?"

It reads Chrome's browsing history, uses AI (DeepSeek) to cluster pages into "missions," and shows them on a new-tab dashboard. It also knows which tabs are currently open and can close them for you.

## Problem

Zara opens many tabs across parallel tasks, context-switches frequently, and loses track of what she was doing. Tabs pile up until the machine crashes. There's no system to surface abandoned work or clean up finished tasks.

## Solution: Two Pieces

### 1. Local Web App (the dashboard)

A lightweight local server that serves the new-tab dashboard.

**Data source:** Chrome's SQLite history database (`~/Library/Application Support/Google/Chrome/Default/History`). The server copies this file (Chrome locks it while running) and queries it for recent browsing activity.

**AI clustering:** Sends batches of recent URLs + titles to DeepSeek API to cluster into named "missions." Results are cached locally in a SQLite database. Re-analysis runs:
- On first load
- When the user clicks "Refresh now"
- Automatically every 30 minutes while the server is running

**What the dashboard shows:**
- Greeting with date and time-of-day awareness
- **Focus level meter** — visual bar showing how many parallel missions are active (more missions = more scattered)
- **Stale tabs banner** — "23 stale tabs are still open from finished or cold missions" with a one-click "Close all stale tabs" button
- **Nudge banner** — "3 missions have gone cold" prompt
- **Active missions** — touched today or yesterday, sorted by recency
- **Gone cold missions** — untouched for 2+ days
- Each mission card shows:
  - AI-generated name (e.g., "Voice AI Agents Research")
  - Status tag: Active / Cooling (1-3 days) / Abandoned (3+ days)
  - Open tabs badge (e.g., "9 tabs open") — live count from the extension
  - Summary of activity and where the user left off
  - Page chips showing key URLs
  - Action buttons (see below)

**Mission actions:**
- **Focus on this** — tells the extension to bring those tabs to front
- **Close N tabs** — closes all open tabs matching this mission
- **Close & archive** — closes tabs, saves all URLs to an archive for future reference
- **Pick back up** — brings tabs to focus (same as "Focus on this")
- **Let it go** — closes tabs and dismisses the mission from the dashboard

**Footer stats:** Total missions, open tabs, stale tabs count.

### 2. Chrome Extension (thin layer)

Manifest V3 extension with minimal permissions.

**Responsibilities:**
- Override the new tab page to load `http://localhost:3456` (the dashboard)
- Show a toolbar badge with the current number of parallel missions (color-coded: green = focused, amber = scattered, red = danger zone)
- Expose an API via `chrome.runtime.onMessageExternal` that the dashboard calls to:
  - List all open tabs (`chrome.tabs.query`)
  - Close specific tabs (`chrome.tabs.remove`)
  - Focus/bring tabs to front (`chrome.tabs.update`, `chrome.windows.update`)
  - Get the current tab count for the badge
- The extension itself contains almost no logic — it's a bridge between the dashboard and Chrome's tab APIs

**Communication flow:**
```
newtab.html (extension context)
  └── <iframe src="http://localhost:3456"> (dashboard)
        ↕ window.postMessage
newtab.html receives message → calls chrome.tabs API → posts result back
```

The extension's `newtab.html` loads the dashboard in a full-screen iframe. The dashboard communicates with the parent frame via `window.postMessage`. The parent frame has extension context (access to `chrome.tabs`, etc.) and acts as a bridge. This is necessary because navigating away from an extension page to localhost would lose chrome.* API access.

## Tech Stack

- **Server:** Node.js with Express (lightweight, fast startup)
- **Database:** SQLite via `better-sqlite3` (reads Chrome history, stores cached mission data)
- **AI:** DeepSeek API for mission clustering and naming
- **Frontend:** Single HTML page with vanilla JS (no framework needed — it's a dashboard, not an app). CSS-only animations. Google Fonts for typography.
- **Extension:** Manifest V3, minimal service worker, no React
- **Auto-start:** macOS Launch Agent (plist file) to start the server on login

## Data Flow

1. Server starts, copies Chrome's History SQLite file to a temp location
2. Server queries recent browsing data (URLs, titles, visit counts, timestamps)
3. Server sends URL batches to DeepSeek with a clustering prompt
4. DeepSeek returns mission clusters with names and summaries
5. Server caches results in its own SQLite database
6. Dashboard loads, fetches cached missions from server API
7. Dashboard asks extension for currently open tabs
8. Dashboard cross-references missions with open tabs to show "N tabs open" badges
9. User clicks an action button → dashboard sends message to extension → extension executes tab operation → dashboard updates UI

## API Endpoints (Server)

- `GET /api/missions` — returns cached mission clusters with metadata
- `POST /api/missions/refresh` — triggers re-analysis of browsing history
- `POST /api/missions/:id/archive` — archives a mission's URLs
- `POST /api/missions/:id/dismiss` — dismisses a mission from the dashboard
- `GET /api/stats` — returns summary stats (total missions, page counts, etc.)

## DeepSeek Prompt Strategy

Send batches of ~100 recent URLs with titles, visit counts, and timestamps. Ask DeepSeek to:
1. Cluster them into distinct "missions" (tasks/activities the user was working on)
2. Name each mission with a short, descriptive title
3. Summarize what the user was doing and where they left off
4. Classify status: active (today), cooling (1-3 days), abandoned (3+ days)
5. Ignore noise (Google sign-in flows, session timeouts, generic homepages)

## Filtering & Noise Reduction

Before sending to DeepSeek, filter out:
- Authentication/OAuth flows (accounts.google.com, sign-in pages)
- Session timeouts and error pages
- Chrome internal pages (chrome://, chrome-extension://)
- Duplicate URLs (keep the most recent visit)
- Very short titles (< 4 chars) or generic titles ("Just a moment...")

## Persistence

- **Mission cache:** SQLite database at `~/.mission-control/missions.db`
- **Archives:** Same database, separate table for archived mission URLs
- **Dismissed missions:** Same database, dismissed mission IDs stored so they don't reappear
- **Server config:** `~/.mission-control/config.json` (DeepSeek API key, port, refresh interval)

## Auto-Start (macOS Launch Agent)

A plist file at `~/Library/LaunchAgents/com.mission-control.plist` that:
- Starts the Node.js server on login
- Restarts it if it crashes (`KeepAlive: true`)
- Runs silently in the background
- Logs to `~/.mission-control/logs/`

## Chrome Extension Details

**Manifest permissions:**
- `tabs` — to query and close tabs
- `activeWindow` — to focus windows
- `storage` — to store the server port if customized

**New tab override:**
```json
"chrome_url_overrides": {
  "newtab": "newtab.html"
}
```
`newtab.html` is an extension page that:
- Loads `http://localhost:3456` in a full-screen, borderless iframe
- Listens for `postMessage` from the iframe and translates them to `chrome.tabs.*` calls
- Posts results back to the iframe
- Shows a minimal fallback ("Server not running — start Mission Control") if localhost is unreachable

**Badge logic:**
- Extension polls the server every 60 seconds for mission count
- Badge text = number of active missions
- Badge color: green (1-3), amber (4-6), red (7+)

## Design Aesthetic

Editorial/magazine tone. Warm paper-like background. Newsreader serif for headings, DM Sans for body. Subtle paper texture overlay. Staggered fade-up animations on load. No emojis — icons only. Light mode. See mockup at `mockup.html`.

## What's NOT in Scope

- No cross-device sync
- No cloud storage — everything is local
- No Chrome Web Store publishing (side-loaded for personal use)
- No real-time tab monitoring (polling-based, not event-driven)
- No mobile version
- No integrations with task managers (Notion, Linear, etc.) — may add later
