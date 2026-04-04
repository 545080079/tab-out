# Mission Control Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local browsing dashboard + Chrome extension that clusters browsing history into "missions" using DeepSeek AI, shows them on a new-tab page, and can close stale tabs.

**Architecture:** Node.js/Express server reads Chrome's SQLite history, sends URLs to DeepSeek for clustering, caches results in its own SQLite DB, and serves a vanilla HTML dashboard. A thin Manifest V3 Chrome extension overrides the new tab page (iframe to localhost), bridges chrome.tabs API calls via postMessage, and shows a badge.

**Tech Stack:** Node.js, Express, better-sqlite3, DeepSeek API (OpenAI-compatible), vanilla HTML/CSS/JS, Chrome Extension Manifest V3, macOS Launch Agent

---

## File Structure

```
tab-mission-control/
├── server/
│   ├── index.js              # Express server entry point, starts server + scheduler
│   ├── db.js                 # SQLite setup, schema creation, query helpers
│   ├── history-reader.js     # Copies & queries Chrome's History SQLite file
│   ├── url-filter.js         # Filters noise from raw browsing data
│   ├── clustering.js         # DeepSeek API calls for mission clustering
│   ├── routes.js             # Express route handlers for /api/*
│   └── config.js             # Reads ~/.mission-control/config.json
├── dashboard/
│   ├── index.html            # The dashboard page (served by Express)
│   ├── style.css             # All dashboard styles (from mockup)
│   └── app.js                # Dashboard logic: fetch missions, postMessage to extension, UI updates
├── extension/
│   ├── manifest.json         # Manifest V3 config
│   ├── newtab.html           # New tab page with iframe + postMessage bridge
│   ├── newtab.js             # Bridge logic: listen for postMessage, call chrome.tabs.*
│   ├── background.js         # Service worker: badge updates, polling server
│   └── icons/
│       ├── icon16.png
│       ├── icon48.png
│       └── icon128.png
├── scripts/
│   └── install.js            # Setup script: creates ~/.mission-control/, config, launch agent
├── package.json
├── mockup.html               # Design reference (already exists)
└── docs/                     # Specs and plans (already exists)
```

---

### Task 1: Project Scaffolding & Config

**Files:**
- Create: `package.json`
- Create: `server/config.js`
- Create: `scripts/install.js`

- [ ] **Step 1: Initialize the Node.js project**

```bash
cd "/Users/zara/Documents/For Claude/tab-mission-control"
npm init -y
```

- [ ] **Step 2: Install dependencies**

```bash
npm install express better-sqlite3 openai
```

`openai` package works with DeepSeek since DeepSeek uses an OpenAI-compatible API. We just change the base URL.

- [ ] **Step 3: Update package.json scripts**

In `package.json`, update the `scripts` section:

```json
{
  "scripts": {
    "start": "node server/index.js",
    "install-service": "node scripts/install.js"
  }
}
```

- [ ] **Step 4: Create server/config.js**

```js
// server/config.js
// Reads configuration from ~/.mission-control/config.json
// Falls back to sensible defaults if the file doesn't exist yet

const fs = require('fs');
const path = require('path');

const CONFIG_DIR = path.join(process.env.HOME, '.mission-control');
const CONFIG_PATH = path.join(CONFIG_DIR, 'config.json');

const DEFAULTS = {
  port: 3456,
  refreshIntervalMinutes: 30,
  // How many recent URLs to send to DeepSeek per analysis batch
  batchSize: 200,
  // How many days of history to look at
  historyDays: 7,
  // DeepSeek API config
  deepseekApiKey: '',
  deepseekBaseUrl: 'https://api.deepseek.com',
  deepseekModel: 'deepseek-chat',
};

function loadConfig() {
  // Make sure the config directory exists
  if (!fs.existsSync(CONFIG_DIR)) {
    fs.mkdirSync(CONFIG_DIR, { recursive: true });
  }

  // If no config file yet, create one with defaults (minus the API key)
  if (!fs.existsSync(CONFIG_PATH)) {
    fs.writeFileSync(CONFIG_PATH, JSON.stringify(DEFAULTS, null, 2));
    return { ...DEFAULTS };
  }

  // Read and merge with defaults (so new config keys get picked up)
  const stored = JSON.parse(fs.readFileSync(CONFIG_PATH, 'utf-8'));
  return { ...DEFAULTS, ...stored };
}

const config = loadConfig();

module.exports = config;
```

- [ ] **Step 5: Create scripts/install.js**

```js
// scripts/install.js
// Sets up ~/.mission-control/ directory, config file, and macOS Launch Agent
// so the server starts automatically on login

const fs = require('fs');
const path = require('path');
const { execSync } = require('child_process');

const HOME = process.env.HOME;
const CONFIG_DIR = path.join(HOME, '.mission-control');
const LOGS_DIR = path.join(CONFIG_DIR, 'logs');
const CONFIG_PATH = path.join(CONFIG_DIR, 'config.json');
const PLIST_PATH = path.join(HOME, 'Library/LaunchAgents/com.mission-control.plist');

// The absolute path to this project
const PROJECT_DIR = path.resolve(__dirname, '..');

console.log('Setting up Mission Control...\n');

// 1. Create directories
if (!fs.existsSync(CONFIG_DIR)) fs.mkdirSync(CONFIG_DIR, { recursive: true });
if (!fs.existsSync(LOGS_DIR)) fs.mkdirSync(LOGS_DIR, { recursive: true });
console.log('Created ~/.mission-control/ and logs directory');

// 2. Create config if it doesn't exist
if (!fs.existsSync(CONFIG_PATH)) {
  const defaults = {
    port: 3456,
    refreshIntervalMinutes: 30,
    batchSize: 200,
    historyDays: 7,
    deepseekApiKey: '',
    deepseekBaseUrl: 'https://api.deepseek.com',
    deepseekModel: 'deepseek-chat',
  };
  fs.writeFileSync(CONFIG_PATH, JSON.stringify(defaults, null, 2));
  console.log('Created config at ~/.mission-control/config.json');
  console.log('  ** You need to add your DeepSeek API key there! **');
} else {
  console.log('Config already exists at ~/.mission-control/config.json');
}

// 3. Find the node binary path (needed for the launch agent)
const nodePath = execSync('which node').toString().trim();

// 4. Create macOS Launch Agent plist
const plist = `<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.mission-control</string>
  <key>ProgramArguments</key>
  <array>
    <string>${nodePath}</string>
    <string>${path.join(PROJECT_DIR, 'server/index.js')}</string>
  </array>
  <key>WorkingDirectory</key>
  <string>${PROJECT_DIR}</string>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>StandardOutPath</key>
  <string>${path.join(LOGS_DIR, 'stdout.log')}</string>
  <key>StandardErrorPath</key>
  <string>${path.join(LOGS_DIR, 'stderr.log')}</string>
</dict>
</plist>`;

fs.writeFileSync(PLIST_PATH, plist);
console.log('Created Launch Agent at ~/Library/LaunchAgents/com.mission-control.plist');

// 5. Load the launch agent
try {
  // Unload first in case it was already loaded
  execSync(`launchctl unload "${PLIST_PATH}" 2>/dev/null || true`);
  execSync(`launchctl load "${PLIST_PATH}"`);
  console.log('Launch Agent loaded — Mission Control will now start on login');
} catch (e) {
  console.log('Could not load Launch Agent automatically. Run:');
  console.log(`  launchctl load "${PLIST_PATH}"`);
}

console.log('\nDone! Next steps:');
console.log('1. Add your DeepSeek API key to ~/.mission-control/config.json');
console.log('2. Run: npm start');
console.log('3. Load the Chrome extension from the extension/ folder');
```

- [ ] **Step 6: Verify project structure**

```bash
ls -la server/ scripts/ package.json
```

- [ ] **Step 7: Commit**

```bash
git init
git add package.json package-lock.json server/config.js scripts/install.js
git commit -m "feat: project scaffolding with config and install script"
```

---

### Task 2: Database Layer

**Files:**
- Create: `server/db.js`

- [ ] **Step 1: Create server/db.js with schema and helpers**

```js
// server/db.js
// Manages the local SQLite database at ~/.mission-control/missions.db
// Stores cached mission clusters, archives, and dismissed missions

const Database = require('better-sqlite3');
const path = require('path');
const fs = require('fs');

const DB_DIR = path.join(process.env.HOME, '.mission-control');
const DB_PATH = path.join(DB_DIR, 'missions.db');

// Make sure the directory exists
if (!fs.existsSync(DB_DIR)) {
  fs.mkdirSync(DB_DIR, { recursive: true });
}

const db = new Database(DB_PATH);

// Enable WAL mode for better concurrent read performance
db.pragma('journal_mode = WAL');

// Create tables if they don't exist
db.exec(`
  -- Each mission is a cluster of related browsing activity
  CREATE TABLE IF NOT EXISTS missions (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    summary TEXT NOT NULL,
    status TEXT NOT NULL CHECK(status IN ('active', 'cooling', 'abandoned')),
    last_activity TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    dismissed INTEGER NOT NULL DEFAULT 0
  );

  -- URLs that belong to each mission
  CREATE TABLE IF NOT EXISTS mission_urls (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    mission_id TEXT NOT NULL REFERENCES missions(id) ON DELETE CASCADE,
    url TEXT NOT NULL,
    title TEXT NOT NULL,
    visit_count INTEGER NOT NULL DEFAULT 1,
    last_visit TEXT NOT NULL
  );

  -- Archived missions — saved for future reference after closing
  CREATE TABLE IF NOT EXISTS archives (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    mission_id TEXT NOT NULL,
    mission_name TEXT NOT NULL,
    urls_json TEXT NOT NULL,
    archived_at TEXT NOT NULL DEFAULT (datetime('now'))
  );

  -- Track when we last ran analysis so we don't re-analyze too frequently
  CREATE TABLE IF NOT EXISTS meta (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
  );
`);

// ---- Query helpers ----

// Get all non-dismissed missions, ordered by last activity
const getMissions = db.prepare(`
  SELECT * FROM missions
  WHERE dismissed = 0
  ORDER BY
    CASE status
      WHEN 'active' THEN 0
      WHEN 'cooling' THEN 1
      WHEN 'abandoned' THEN 2
    END,
    last_activity DESC
`);

// Get URLs for a specific mission
const getMissionUrls = db.prepare(`
  SELECT url, title, visit_count, last_visit
  FROM mission_urls
  WHERE mission_id = ?
  ORDER BY last_visit DESC
`);

// Insert or replace a mission (used during clustering)
const upsertMission = db.prepare(`
  INSERT INTO missions (id, name, summary, status, last_activity, updated_at)
  VALUES (@id, @name, @summary, @status, @last_activity, datetime('now'))
  ON CONFLICT(id) DO UPDATE SET
    name = @name,
    summary = @summary,
    status = @status,
    last_activity = @last_activity,
    updated_at = datetime('now')
`);

// Insert a URL for a mission
const insertMissionUrl = db.prepare(`
  INSERT INTO mission_urls (mission_id, url, title, visit_count, last_visit)
  VALUES (@mission_id, @url, @title, @visit_count, @last_visit)
`);

// Delete all URLs for a mission (used before re-inserting during refresh)
const deleteMissionUrls = db.prepare(`
  DELETE FROM mission_urls WHERE mission_id = ?
`);

// Dismiss a mission
const dismissMission = db.prepare(`
  UPDATE missions SET dismissed = 1, updated_at = datetime('now') WHERE id = ?
`);

// Archive a mission
const archiveMission = db.prepare(`
  INSERT INTO archives (mission_id, mission_name, urls_json)
  VALUES (@mission_id, @mission_name, @urls_json)
`);

// Get/set meta values (like last analysis time)
const getMeta = db.prepare(`SELECT value FROM meta WHERE key = ?`);
const setMeta = db.prepare(`
  INSERT INTO meta (key, value) VALUES (?, ?)
  ON CONFLICT(key) DO UPDATE SET value = excluded.value
`);

// Clear all missions and URLs (used before a full refresh)
function clearAllMissions() {
  db.exec(`DELETE FROM mission_urls`);
  db.exec(`DELETE FROM missions`);
}

module.exports = {
  db,
  getMissions,
  getMissionUrls,
  upsertMission,
  insertMissionUrl,
  deleteMissionUrls,
  dismissMission,
  archiveMission,
  getMeta,
  setMeta,
  clearAllMissions,
};
```

- [ ] **Step 2: Test the database initializes correctly**

```bash
cd "/Users/zara/Documents/For Claude/tab-mission-control"
node -e "const db = require('./server/db.js'); console.log('DB initialized OK'); console.log('Tables:', db.db.prepare(\"SELECT name FROM sqlite_master WHERE type='table'\").all().map(r => r.name));"
```

Expected output: DB initialized OK with tables: missions, mission_urls, archives, meta.

- [ ] **Step 3: Commit**

```bash
git add server/db.js
git commit -m "feat: SQLite database layer with schema and query helpers"
```

---

### Task 3: Chrome History Reader

**Files:**
- Create: `server/history-reader.js`
- Create: `server/url-filter.js`

- [ ] **Step 1: Create server/url-filter.js**

```js
// server/url-filter.js
// Filters out noise from raw Chrome browsing history before sending to AI
// Things like login flows, session timeouts, Chrome internal pages, etc.

// Domains to completely ignore
const BLOCKED_DOMAINS = [
  'accounts.google.com',
  'accounts.youtube.com',
  'myaccount.google.com',
  'chrome.google.com/webstore',
];

// URL patterns to filter out (regex)
const BLOCKED_PATTERNS = [
  /^chrome:\/\//,
  /^chrome-extension:\/\//,
  /^about:/,
  /^edge:\/\//,
  /\/oauth/i,
  /\/signin/i,
  /\/sign-in/i,
  /\/login/i,
  /\/auth\//i,
  /\/callback\?/i,
  /session[Tt]imeout/,
  /ServiceLogin/,
  /InteractiveLogin/,
];

// Titles that indicate noise (exact or partial matches)
const BLOCKED_TITLES = [
  'just a moment...',
  'sign in',
  'log in',
  'redirecting',
  'loading...',
  '403 forbidden',
  '404 not found',
  '500 internal server error',
];

function filterUrls(entries) {
  return entries.filter(entry => {
    const url = entry.url || '';
    const title = (entry.title || '').toLowerCase();

    // Skip entries with very short or empty titles
    if (title.length < 4) return false;

    // Skip blocked domains
    try {
      const hostname = new URL(url).hostname;
      if (BLOCKED_DOMAINS.some(d => hostname.includes(d))) return false;
    } catch {
      return false; // Invalid URL
    }

    // Skip blocked URL patterns
    if (BLOCKED_PATTERNS.some(pattern => pattern.test(url))) return false;

    // Skip blocked titles
    if (BLOCKED_TITLES.some(blocked => title.includes(blocked))) return false;

    return true;
  });
}

// Deduplicate URLs, keeping the most recent visit for each unique URL
// Also strips query params for deduplication but keeps the full URL in the result
function deduplicateUrls(entries) {
  const seen = new Map();
  for (const entry of entries) {
    // Use the URL without long query strings for dedup
    let dedupeKey;
    try {
      const parsed = new URL(entry.url);
      // Keep short query strings (they might be meaningful, like ?q=search)
      // Strip long ones (usually session tokens, OAuth state, etc.)
      if (parsed.search.length > 100) {
        dedupeKey = parsed.origin + parsed.pathname;
      } else {
        dedupeKey = parsed.origin + parsed.pathname + parsed.search;
      }
    } catch {
      dedupeKey = entry.url;
    }

    // Keep the entry with the most recent visit
    if (!seen.has(dedupeKey) || entry.last_visit > seen.get(dedupeKey).last_visit) {
      seen.set(dedupeKey, entry);
    }
  }
  return Array.from(seen.values());
}

module.exports = { filterUrls, deduplicateUrls };
```

- [ ] **Step 2: Create server/history-reader.js**

```js
// server/history-reader.js
// Reads Chrome's browsing history from its local SQLite database
// Chrome locks the file while running, so we copy it to a temp location first

const Database = require('better-sqlite3');
const fs = require('fs');
const path = require('path');
const os = require('os');
const config = require('./config');

// Chrome stores its history here on macOS
const CHROME_HISTORY_PATH = path.join(
  os.homedir(),
  'Library/Application Support/Google/Chrome/Default/History'
);

// We copy to a temp file because Chrome holds a lock on the original
const TEMP_COPY_PATH = path.join(os.tmpdir(), 'mission-control-history-copy.db');

function readRecentHistory() {
  // Copy the Chrome history file to a temp location
  if (!fs.existsSync(CHROME_HISTORY_PATH)) {
    console.error('Chrome history file not found at:', CHROME_HISTORY_PATH);
    return [];
  }

  fs.copyFileSync(CHROME_HISTORY_PATH, TEMP_COPY_PATH);

  // Open the copy (read-only for safety)
  const chromeDb = new Database(TEMP_COPY_PATH, { readonly: true });

  // Calculate the cutoff time
  // Chrome stores timestamps as microseconds since Jan 1, 1601 (Windows epoch)
  // We need to convert from Unix epoch to Chrome epoch
  const daysAgo = config.historyDays;
  const cutoffUnix = Date.now() / 1000 - (daysAgo * 24 * 60 * 60);
  // Chrome epoch offset: seconds between 1601-01-01 and 1970-01-01
  const CHROME_EPOCH_OFFSET = 11644473600;
  const cutoffChrome = (cutoffUnix + CHROME_EPOCH_OFFSET) * 1000000;

  // Query recent URLs with visit data
  const rows = chromeDb.prepare(`
    SELECT
      u.url,
      u.title,
      u.visit_count,
      datetime(u.last_visit_time / 1000000 - 11644473600, 'unixepoch', 'localtime') as last_visit,
      u.last_visit_time as last_visit_raw
    FROM urls u
    WHERE u.last_visit_time > ?
      AND u.title != ''
      AND length(u.title) > 3
    ORDER BY u.last_visit_time DESC
    LIMIT ?
  `).all(cutoffChrome, config.batchSize * 2);

  chromeDb.close();

  // Clean up the temp file
  try { fs.unlinkSync(TEMP_COPY_PATH); } catch {}

  return rows;
}

module.exports = { readRecentHistory };
```

- [ ] **Step 3: Test history reader against real Chrome data**

```bash
cd "/Users/zara/Documents/For Claude/tab-mission-control"
node -e "
const { readRecentHistory } = require('./server/history-reader');
const { filterUrls, deduplicateUrls } = require('./server/url-filter');
const raw = readRecentHistory();
console.log('Raw entries:', raw.length);
const filtered = filterUrls(raw);
console.log('After filtering:', filtered.length);
const deduped = deduplicateUrls(filtered);
console.log('After dedup:', deduped.length);
console.log('Sample:', deduped.slice(0, 5).map(e => e.title));
"
```

Expected: Numbers should decrease at each stage. Sample should show real page titles without noise.

- [ ] **Step 4: Commit**

```bash
git add server/history-reader.js server/url-filter.js
git commit -m "feat: Chrome history reader with noise filtering"
```

---

### Task 4: DeepSeek AI Clustering

**Files:**
- Create: `server/clustering.js`

- [ ] **Step 1: Create server/clustering.js**

```js
// server/clustering.js
// Sends browsing history to DeepSeek to cluster into "missions"
// Uses the OpenAI-compatible API (DeepSeek uses the same format)

const OpenAI = require('openai');
const crypto = require('crypto');
const config = require('./config');
const { readRecentHistory } = require('./history-reader');
const { filterUrls, deduplicateUrls } = require('./url-filter');
const db = require('./db');

// Initialize the DeepSeek client (OpenAI-compatible)
function getClient() {
  return new OpenAI({
    apiKey: config.deepseekApiKey,
    baseURL: config.deepseekBaseUrl,
  });
}

// Build the prompt for DeepSeek
function buildPrompt(entries) {
  const now = new Date().toISOString();
  const urlList = entries.map((e, i) =>
    `${i + 1}. [${e.title}] ${e.url} (visited ${e.visit_count}x, last: ${e.last_visit})`
  ).join('\n');

  return `You are analyzing a user's recent browsing history to identify distinct "missions" — tasks or activities they were working on.

Current time: ${now}

Here are their recent browsing entries:
${urlList}

Cluster these into distinct missions. For each mission, provide:
1. A short, specific name (e.g., "LiveKit Voice Agent Research" not "YouTube Watching")
2. A 1-2 sentence summary of what the user was doing and where they left off
3. Status: "active" (last activity today), "cooling" (1-3 days ago), "abandoned" (3+ days ago)
4. Which URL indices belong to this mission (as a comma-separated list of numbers)

Rules:
- Group by intent, not by domain. A YouTube video about voice AI and a GitHub repo about voice AI are the same mission.
- Ignore background noise: social media scrolling (X/Twitter home feed, notifications), email checking, and generic browsing aren't missions unless there's a clear focused activity.
- Be specific in names. "GitHub Work" is bad. "Frontend Slides PR Review" is good.
- If a URL doesn't fit any mission, skip it.

Respond in this exact JSON format:
{
  "missions": [
    {
      "name": "Mission Name",
      "summary": "What the user was doing and where they left off.",
      "status": "active|cooling|abandoned",
      "url_indices": [1, 2, 3]
    }
  ]
}

Return ONLY the JSON, no markdown formatting.`;
}

// Parse DeepSeek's response into mission objects
function parseResponse(responseText, entries) {
  // Strip markdown code fences if present
  let cleaned = responseText.trim();
  if (cleaned.startsWith('```')) {
    cleaned = cleaned.replace(/^```(?:json)?\n?/, '').replace(/\n?```$/, '');
  }

  const parsed = JSON.parse(cleaned);

  return parsed.missions.map(mission => {
    // Generate a stable-ish ID based on the mission name
    const id = crypto.createHash('md5')
      .update(mission.name.toLowerCase())
      .digest('hex')
      .slice(0, 12);

    // Resolve URL indices to actual URL entries
    const urls = (mission.url_indices || [])
      .filter(i => i >= 1 && i <= entries.length)
      .map(i => entries[i - 1]);

    // Find the most recent activity among the mission's URLs
    const lastActivity = urls.length > 0
      ? urls.reduce((latest, u) => u.last_visit > latest ? u.last_visit : latest, urls[0].last_visit)
      : new Date().toISOString();

    return {
      id,
      name: mission.name,
      summary: mission.summary,
      status: mission.status,
      last_activity: lastActivity,
      urls,
    };
  });
}

// Main function: read history, call DeepSeek, store results
async function analyzeBrowsingHistory() {
  console.log('Starting browsing history analysis...');

  // 1. Read and filter history
  const raw = readRecentHistory();
  const filtered = filterUrls(raw);
  const entries = deduplicateUrls(filtered);

  // Limit to configured batch size
  const batch = entries.slice(0, config.batchSize);

  if (batch.length === 0) {
    console.log('No browsing history to analyze');
    return [];
  }

  console.log(`Analyzing ${batch.length} URLs...`);

  // 2. Call DeepSeek
  const client = getClient();
  const prompt = buildPrompt(batch);

  const response = await client.chat.completions.create({
    model: config.deepseekModel,
    messages: [{ role: 'user', content: prompt }],
    temperature: 0.3, // Low temperature for consistent clustering
    max_tokens: 4000,
  });

  const responseText = response.choices[0].message.content;

  // 3. Parse the response
  const missions = parseResponse(responseText, batch);
  console.log(`Found ${missions.length} missions`);

  // 4. Store in database
  // Clear old missions (we regenerate from scratch each time)
  db.clearAllMissions();

  const insertTransaction = db.db.transaction((missions) => {
    for (const mission of missions) {
      db.upsertMission.run({
        id: mission.id,
        name: mission.name,
        summary: mission.summary,
        status: mission.status,
        last_activity: mission.last_activity,
      });

      for (const url of mission.urls) {
        db.insertMissionUrl.run({
          mission_id: mission.id,
          url: url.url,
          title: url.title,
          visit_count: url.visit_count,
          last_visit: url.last_visit,
        });
      }
    }
  });

  insertTransaction(missions);

  // 5. Record when we last analyzed
  db.setMeta.run('last_analysis', new Date().toISOString());
  console.log('Analysis complete and cached');

  return missions;
}

module.exports = { analyzeBrowsingHistory };
```

- [ ] **Step 2: Test clustering with real data (requires DeepSeek API key)**

Before running this test, make sure your DeepSeek API key is in `~/.mission-control/config.json`.

```bash
cd "/Users/zara/Documents/For Claude/tab-mission-control"
node -e "
const { analyzeBrowsingHistory } = require('./server/clustering');
analyzeBrowsingHistory()
  .then(missions => {
    console.log('Missions found:', missions.length);
    missions.forEach(m => console.log('-', m.name, '(' + m.status + ')', '-', m.urls.length, 'urls'));
  })
  .catch(err => console.error('Error:', err.message));
"
```

Expected: Should print mission names that match real browsing patterns (similar to what we saw in the mockup).

- [ ] **Step 3: Commit**

```bash
git add server/clustering.js
git commit -m "feat: DeepSeek AI clustering for browsing history"
```

---

### Task 5: API Routes

**Files:**
- Create: `server/routes.js`

- [ ] **Step 1: Create server/routes.js**

```js
// server/routes.js
// Express route handlers for the Mission Control API
// The dashboard fetches data from these endpoints

const express = require('express');
const db = require('./db');
const { analyzeBrowsingHistory } = require('./clustering');

const router = express.Router();

// GET /api/missions — return all active (non-dismissed) missions with their URLs
router.get('/missions', (req, res) => {
  const missions = db.getMissions.all();

  // Attach URLs to each mission
  const result = missions.map(mission => ({
    ...mission,
    urls: db.getMissionUrls.all(mission.id),
  }));

  res.json(result);
});

// POST /api/missions/refresh — trigger a new analysis of browsing history
// This is called when the user clicks "Refresh now" on the dashboard
let isRefreshing = false;

router.post('/missions/refresh', async (req, res) => {
  // Prevent multiple simultaneous refreshes
  if (isRefreshing) {
    return res.status(429).json({ error: 'Already refreshing, please wait' });
  }

  isRefreshing = true;
  try {
    const missions = await analyzeBrowsingHistory();
    res.json({ success: true, count: missions.length });
  } catch (err) {
    console.error('Refresh failed:', err);
    res.status(500).json({ error: err.message });
  } finally {
    isRefreshing = false;
  }
});

// POST /api/missions/:id/dismiss — hide a mission from the dashboard
router.post('/missions/:id/dismiss', (req, res) => {
  const { id } = req.params;
  db.dismissMission.run(id);
  res.json({ success: true });
});

// POST /api/missions/:id/archive — save mission URLs for reference, then dismiss
router.post('/missions/:id/archive', (req, res) => {
  const { id } = req.params;

  // Get the mission and its URLs
  const missions = db.getMissions.all();
  const mission = missions.find(m => m.id === id);
  if (!mission) {
    return res.status(404).json({ error: 'Mission not found' });
  }

  const urls = db.getMissionUrls.all(id);

  // Archive it
  db.archiveMission.run({
    mission_id: id,
    mission_name: mission.name,
    urls_json: JSON.stringify(urls),
  });

  // Dismiss it from the active view
  db.dismissMission.run(id);

  res.json({ success: true });
});

// GET /api/stats — summary statistics for the footer
router.get('/stats', (req, res) => {
  const missions = db.getMissions.all();
  const totalUrls = missions.reduce((sum, m) => {
    return sum + db.getMissionUrls.all(m.id).length;
  }, 0);

  const abandoned = missions.filter(m => m.status === 'abandoned').length;
  const lastAnalysis = db.getMeta.get('last_analysis');

  res.json({
    totalMissions: missions.length,
    totalUrls,
    abandonedMissions: abandoned,
    lastAnalysis: lastAnalysis ? lastAnalysis.value : null,
  });
});

module.exports = router;
```

- [ ] **Step 2: Commit**

```bash
git add server/routes.js
git commit -m "feat: API routes for missions, refresh, dismiss, archive"
```

---

### Task 6: Express Server Entry Point

**Files:**
- Create: `server/index.js`

- [ ] **Step 1: Create server/index.js**

```js
// server/index.js
// Main entry point for the Mission Control server
// Serves the dashboard, API routes, and runs periodic analysis

const express = require('express');
const path = require('path');
const config = require('./config');
const routes = require('./routes');
const db = require('./db');
const { analyzeBrowsingHistory } = require('./clustering');

const app = express();

// Parse JSON request bodies (for POST endpoints)
app.use(express.json());

// Serve the dashboard's static files (HTML, CSS, JS)
app.use(express.static(path.join(__dirname, '..', 'dashboard')));

// Mount API routes under /api
app.use('/api', routes);

// Start the server
app.listen(config.port, () => {
  console.log(`Mission Control running at http://localhost:${config.port}`);

  // Run initial analysis if we haven't analyzed recently
  const lastAnalysis = db.getMeta.get('last_analysis');
  const thirtyMinutesAgo = new Date(Date.now() - 30 * 60 * 1000).toISOString();

  if (!lastAnalysis || lastAnalysis.value < thirtyMinutesAgo) {
    console.log('Running initial analysis...');
    analyzeBrowsingHistory().catch(err =>
      console.error('Initial analysis failed:', err.message)
    );
  } else {
    console.log('Using cached analysis from', lastAnalysis.value);
  }

  // Schedule periodic re-analysis
  const intervalMs = config.refreshIntervalMinutes * 60 * 1000;
  setInterval(() => {
    console.log('Running scheduled analysis...');
    analyzeBrowsingHistory().catch(err =>
      console.error('Scheduled analysis failed:', err.message)
    );
  }, intervalMs);
});
```

- [ ] **Step 2: Test the server starts**

```bash
cd "/Users/zara/Documents/For Claude/tab-mission-control"
timeout 5 node server/index.js || true
```

Expected: Should print "Mission Control running at http://localhost:3456" and then either run analysis or use cache.

- [ ] **Step 3: Commit**

```bash
git add server/index.js
git commit -m "feat: Express server with auto-analysis scheduling"
```

---

### Task 7: Dashboard Frontend

**Files:**
- Create: `dashboard/index.html`
- Create: `dashboard/style.css`
- Create: `dashboard/app.js`

This is the largest task. The dashboard is based on the approved mockup at `mockup.html` but wired up to real data from the API and extension.

- [ ] **Step 1: Create dashboard/style.css**

Extract all the styles from `mockup.html` into this file. This is a direct copy of the `<style>` block from the mockup with no changes — the mockup's design was already approved.

```bash
mkdir -p dashboard
```

Copy the CSS from `mockup.html` `<style>` tag verbatim into `dashboard/style.css`. The full CSS is in the mockup — it includes all variables, card styles, animations, toast, banner, etc. No modifications needed.

- [ ] **Step 2: Create dashboard/index.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>New Tab</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Newsreader:ital,opsz,wght@0,6..72,300;0,6..72,400;0,6..72,500;1,6..72,300;1,6..72,400&family=DM+Sans:wght@300;400;500;600&display=swap" rel="stylesheet">
<link rel="stylesheet" href="style.css">
</head>
<body>
<div class="container">
  <header>
    <div class="header-left">
      <h1 id="greeting"></h1>
      <div class="date" id="dateDisplay"></div>
    </div>
    <div class="scatter-meter">
      <div class="scatter-label">Focus level</div>
      <div class="scatter-bar" id="scatterBar"></div>
      <div class="scatter-caption" id="scatterCaption"></div>
    </div>
  </header>

  <div class="tab-cleanup-banner" id="cleanupBanner" style="display:none">
    <div class="tab-cleanup-left">
      <div class="tab-cleanup-icon">
        <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" d="M3.75 9.776c.112-.017.227-.026.344-.026h15.812c.117 0 .232.009.344.026m-16.5 0a2.25 2.25 0 0 0-1.883 2.542l.857 6a2.25 2.25 0 0 0 2.227 1.932H19.05a2.25 2.25 0 0 0 2.227-1.932l.857-6a2.25 2.25 0 0 0-1.883-2.542m-16.5 0V6A2.25 2.25 0 0 1 6 3.75h3.879a1.5 1.5 0 0 1 1.06.44l2.122 2.12a1.5 1.5 0 0 0 1.06.44H18A2.25 2.25 0 0 1 20.25 9v.776" /></svg>
      </div>
      <div class="tab-cleanup-text">
        <strong id="staleTabCount">0 stale tabs</strong> are still open from finished or cold missions. Free up your browser?
      </div>
    </div>
    <button class="tab-cleanup-btn" id="closeAllStaleBtn">Close all stale tabs</button>
  </div>

  <div class="nudge-banner" id="nudgeBanner" style="display:none">
    <div class="nudge-icon">
      <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" d="M12 9v3.75m9-.75a9 9 0 1 1-18 0 9 9 0 0 1 18 0Zm-9 3.75h.008v.008H12v-.008Z" /></svg>
    </div>
    <div class="nudge-text" id="nudgeText"></div>
  </div>

  <div class="active-section" id="activeSection" style="display:none"></div>
  <div class="abandoned-section" id="abandonedSection" style="display:none"></div>

  <footer>
    <div class="footer-stats">
      <div class="stat"><div class="stat-num" id="statMissions">0</div><div class="stat-label">Missions</div></div>
      <div class="stat"><div class="stat-num" id="statTabs">0</div><div class="stat-label">Open tabs</div></div>
      <div class="stat"><div class="stat-num" id="statStale">0</div><div class="stat-label">Stale tabs</div></div>
    </div>
    <div class="last-refresh">
      <span id="lastRefreshTime">Loading...</span> &middot; <button class="refresh-btn" id="refreshBtn">Refresh now</button>
    </div>
  </footer>
</div>

<div class="toast" id="toast">
  <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" d="M9 12.75 11.25 15 15 9.75M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z" /></svg>
  <span id="toastText"></span>
</div>

<script src="app.js"></script>
</body>
</html>
```

- [ ] **Step 3: Create dashboard/app.js**

```js
// dashboard/app.js
// Main dashboard logic: fetches missions from API, renders cards,
// communicates with the Chrome extension via postMessage for tab operations

// ---- Extension bridge ----
// When running inside the extension's iframe, we talk to the parent frame
// which has access to chrome.tabs API

let extensionAvailable = false;
let openTabs = []; // Current open tabs from the extension

// Send a message to the extension (parent frame) and wait for a response
function sendToExtension(action, data = {}) {
  return new Promise((resolve) => {
    // If not in an iframe (dev mode), resolve with empty data
    if (window.parent === window) {
      resolve({ success: false, reason: 'not-in-extension' });
      return;
    }

    const messageId = Math.random().toString(36).slice(2);

    function handleResponse(event) {
      if (event.data && event.data.messageId === messageId) {
        window.removeEventListener('message', handleResponse);
        resolve(event.data);
      }
    }

    window.addEventListener('message', handleResponse);
    window.parent.postMessage({ action, messageId, ...data }, '*');

    // Timeout after 3 seconds
    setTimeout(() => {
      window.removeEventListener('message', handleResponse);
      resolve({ success: false, reason: 'timeout' });
    }, 3000);
  });
}

// Fetch open tabs from the extension
async function fetchOpenTabs() {
  const result = await sendToExtension('getTabs');
  if (result.success) {
    extensionAvailable = true;
    openTabs = result.tabs || [];
  }
  return openTabs;
}

// Close tabs by matching URLs
async function closeTabsByUrls(urls) {
  const result = await sendToExtension('closeTabs', { urls });
  if (result.success) {
    // Re-fetch tabs after closing
    await fetchOpenTabs();
  }
  return result;
}

// Focus tabs by matching URLs (bring to front)
async function focusTabsByUrls(urls) {
  return sendToExtension('focusTabs', { urls });
}

// ---- UI helpers ----

function showToast(message) {
  const toast = document.getElementById('toast');
  document.getElementById('toastText').textContent = message;
  toast.classList.add('visible');
  setTimeout(() => toast.classList.remove('visible'), 2500);
}

function timeAgo(dateStr) {
  const diff = Date.now() - new Date(dateStr).getTime();
  const minutes = Math.floor(diff / 60000);
  const hours = Math.floor(diff / 3600000);
  const days = Math.floor(diff / 86400000);

  if (minutes < 1) return 'just now';
  if (minutes < 60) return `${minutes} min ago`;
  if (hours < 24) return `${hours} hr${hours > 1 ? 's' : ''} ago`;
  if (days === 1) return 'yesterday';
  return `${days} days ago`;
}

function getGreeting() {
  const hour = new Date().getHours();
  if (hour < 12) return 'Good morning, Zara';
  if (hour < 17) return 'Good afternoon, Zara';
  return 'Good evening, Zara';
}

function getDateDisplay() {
  return new Date().toLocaleDateString('en-US', {
    weekday: 'long', year: 'numeric', month: 'long', day: 'numeric'
  });
}

// Count how many open tabs match a mission's URLs
function countOpenTabsForMission(missionUrls) {
  if (!extensionAvailable) return 0;

  return openTabs.filter(tab => {
    return missionUrls.some(mu => {
      try {
        const tabHost = new URL(tab.url).hostname;
        const missionHost = new URL(mu.url).hostname;
        // Match by domain for broader matching
        return tabHost === missionHost ||
          tab.url.includes(mu.url) ||
          mu.url.includes(tab.url);
      } catch {
        return false;
      }
    });
  }).length;
}

// Get the actual tab objects that match a mission's URLs
function getOpenTabsForMission(missionUrls) {
  if (!extensionAvailable) return [];

  return openTabs.filter(tab => {
    return missionUrls.some(mu => {
      try {
        const tabHost = new URL(tab.url).hostname;
        const missionHost = new URL(mu.url).hostname;
        return tabHost === missionHost ||
          tab.url.includes(mu.url) ||
          mu.url.includes(tab.url);
      } catch {
        return false;
      }
    });
  });
}

// ---- Icons (SVG strings) ----
const ICONS = {
  tabs: '<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" d="M3 8.25V18a2.25 2.25 0 0 0 2.25 2.25h13.5A2.25 2.25 0 0 0 21 18V8.25m-18 0V6a2.25 2.25 0 0 1 2.25-2.25h13.5A2.25 2.25 0 0 1 21 6v2.25m-18 0h18" /></svg>',
  close: '<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" d="M6 18 18 6M6 6l12 12" /></svg>',
  archive: '<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" d="M20.25 7.5l-.625 10.632a2.25 2.25 0 0 1-2.247 2.118H6.622a2.25 2.25 0 0 1-2.247-2.118L3.75 7.5m6 4.125l2.25 2.25m0 0l2.25 2.25M12 13.875l2.25-2.25M12 13.875l-2.25 2.25M3.375 7.5h17.25c.621 0 1.125-.504 1.125-1.125v-1.5c0-.621-.504-1.125-1.125-1.125H3.375c-.621 0-1.125.504-1.125 1.125v1.5c0 .621.504 1.125 1.125 1.125Z" /></svg>',
  focus: '<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" d="m4.5 19.5 15-15m0 0H8.25m11.25 0v11.25" /></svg>',
};

// ---- Rendering ----

function renderMissionCard(mission, openTabCount) {
  const urls = mission.urls || [];
  const displayUrls = urls.slice(0, 4); // Show max 4 page chips

  const tabBadge = openTabCount > 0
    ? `<span class="open-tabs-badge">${ICONS.tabs} ${openTabCount} tab${openTabCount > 1 ? 's' : ''} open</span>`
    : '';

  // Build action buttons based on mission status and open tabs
  let actionsHtml = '<div class="actions">';

  if (mission.status === 'cooling' || mission.status === 'abandoned') {
    actionsHtml += `<button class="action-btn primary" data-action="focus" data-mission-id="${mission.id}">${ICONS.focus} Pick back up</button>`;
  }

  if (openTabCount > 0 && mission.status !== 'abandoned') {
    actionsHtml += `<button class="action-btn close-tabs" data-action="close-tabs" data-mission-id="${mission.id}">${ICONS.close} Close ${openTabCount} tab${openTabCount > 1 ? 's' : ''}</button>`;
  }

  if (openTabCount > 0 && mission.status === 'abandoned') {
    actionsHtml += `<button class="action-btn close-tabs" data-action="archive" data-mission-id="${mission.id}">${ICONS.archive} Close &amp; archive</button>`;
  }

  if (mission.status === 'abandoned') {
    actionsHtml += `<button class="action-btn danger" data-action="dismiss" data-mission-id="${mission.id}">Let it go</button>`;
  }

  actionsHtml += '</div>';

  // Only show actions row if there are any buttons
  const hasActions = mission.status !== 'active' || openTabCount > 0;

  return `
    <div class="mission-card" data-mission-id="${mission.id}">
      <div class="status-bar ${mission.status}"></div>
      <div class="mission-content">
        <div class="mission-top">
          <span class="mission-name">${mission.name}</span>
          <span class="mission-tag ${mission.status}">${mission.status === 'active' ? 'Active' : mission.status === 'cooling' ? timeAgo(mission.last_activity) : timeAgo(mission.last_activity)}</span>
          ${tabBadge}
        </div>
        <div class="mission-summary">${mission.summary}</div>
        <div class="mission-pages">
          ${displayUrls.map(u => `<span class="page-chip" title="${u.url}">${u.title.slice(0, 40)}${u.title.length > 40 ? '...' : ''}</span>`).join('')}
        </div>
        ${hasActions ? actionsHtml : ''}
      </div>
      <div class="mission-meta">
        <div class="mission-time">${timeAgo(mission.last_activity)}</div>
        <div class="mission-page-count">${urls.length}</div>
        <div class="mission-page-label">page${urls.length !== 1 ? 's' : ''}</div>
      </div>
    </div>
  `;
}

function renderScatterBar(missionCount) {
  const bar = document.getElementById('scatterBar');
  const caption = document.getElementById('scatterCaption');
  const total = 10;
  let dots = '';

  for (let i = 0; i < total; i++) {
    const filled = i < missionCount;
    const high = filled && missionCount > 5;
    dots += `<div class="scatter-dot${filled ? ' filled' : ''}${high ? ' high' : ''}"></div>`;
  }
  bar.innerHTML = dots;

  let level = 'focused';
  if (missionCount > 6) level = 'high scatter';
  else if (missionCount > 3) level = 'moderate';
  caption.textContent = `${missionCount} parallel mission${missionCount !== 1 ? 's' : ''} — ${level}`;
}

async function renderDashboard() {
  // Set greeting and date
  document.getElementById('greeting').textContent = getGreeting();
  document.getElementById('dateDisplay').textContent = getDateDisplay();

  // Fetch missions from API
  let missions = [];
  try {
    const res = await fetch('/api/missions');
    missions = await res.json();
  } catch (err) {
    console.error('Failed to fetch missions:', err);
  }

  // Fetch open tabs from extension
  await fetchOpenTabs();

  // Split into active and abandoned
  const active = missions.filter(m => m.status === 'active' || m.status === 'cooling');
  const abandoned = missions.filter(m => m.status === 'abandoned');

  // Calculate tab stats
  let totalOpenTabs = openTabs.length;
  let staleTabs = 0;

  // Count stale tabs (tabs matching cooling or abandoned missions)
  const nonActive = missions.filter(m => m.status !== 'active');
  nonActive.forEach(m => {
    staleTabs += countOpenTabsForMission(m.urls || []);
  });

  // Render scatter bar
  renderScatterBar(missions.length);

  // Render stale tabs banner
  if (staleTabs > 0) {
    document.getElementById('cleanupBanner').style.display = 'flex';
    document.getElementById('staleTabCount').textContent =
      `${staleTabs} stale tab${staleTabs !== 1 ? 's' : ''}`;
  }

  // Render nudge banner
  if (abandoned.length > 0) {
    document.getElementById('nudgeBanner').style.display = 'flex';
    document.getElementById('nudgeText').innerHTML =
      `<strong>${abandoned.length} mission${abandoned.length > 1 ? 's have' : ' has'} gone cold.</strong> You started ${abandoned.length > 1 ? 'them' : 'it'} but haven't been back in days. Pick one to finish, or let ${abandoned.length > 1 ? 'them' : 'it'} go.`;
  }

  // Render active missions
  if (active.length > 0) {
    const section = document.getElementById('activeSection');
    section.style.display = 'block';
    section.innerHTML = `
      <div class="section-header">
        <h2>Active today</h2>
        <div class="section-line"></div>
        <div class="section-count">${active.length} mission${active.length !== 1 ? 's' : ''}</div>
      </div>
      <div class="missions">
        ${active.map(m => renderMissionCard(m, countOpenTabsForMission(m.urls || []))).join('')}
      </div>
    `;
  }

  // Render abandoned missions
  if (abandoned.length > 0) {
    const section = document.getElementById('abandonedSection');
    section.style.display = 'block';
    section.innerHTML = `
      <div class="section-header">
        <h2>Gone cold</h2>
        <div class="section-line"></div>
        <div class="section-count">${abandoned.length} mission${abandoned.length !== 1 ? 's' : ''}</div>
      </div>
      <div class="missions">
        ${abandoned.map(m => renderMissionCard(m, countOpenTabsForMission(m.urls || []))).join('')}
      </div>
    `;
  }

  // Update footer stats
  document.getElementById('statMissions').textContent = missions.length;
  document.getElementById('statTabs').textContent = totalOpenTabs;
  document.getElementById('statStale').textContent = staleTabs;

  // Update last refresh time
  try {
    const statsRes = await fetch('/api/stats');
    const stats = await statsRes.json();
    document.getElementById('lastRefreshTime').textContent =
      stats.lastAnalysis ? `Last analyzed ${timeAgo(stats.lastAnalysis)}` : 'Not yet analyzed';
  } catch {}
}

// ---- Event handlers ----

// Delegate click events for mission action buttons
document.addEventListener('click', async (e) => {
  const btn = e.target.closest('[data-action]');
  if (!btn) return;

  const action = btn.dataset.action;
  const missionId = btn.dataset.missionId;
  const card = btn.closest('.mission-card');
  const missionName = card.querySelector('.mission-name').textContent;

  if (action === 'close-tabs') {
    // Get the mission's URLs and close matching tabs
    const res = await fetch('/api/missions');
    const missions = await res.json();
    const mission = missions.find(m => m.id === missionId);
    if (mission) {
      const urls = mission.urls.map(u => u.url);
      await closeTabsByUrls(urls);
      // Remove the badge and close button from the card
      const badge = card.querySelector('.open-tabs-badge');
      if (badge) badge.remove();
      btn.remove();
      showToast(`Closed tabs from "${missionName}"`);
    }
  }

  if (action === 'archive') {
    const res = await fetch('/api/missions');
    const missions = await res.json();
    const mission = missions.find(m => m.id === missionId);
    if (mission) {
      const urls = mission.urls.map(u => u.url);
      await closeTabsByUrls(urls);
      await fetch(`/api/missions/${missionId}/archive`, { method: 'POST' });
      card.classList.add('closing');
      setTimeout(() => card.remove(), 400);
      showToast(`Archived "${missionName}"`);
    }
  }

  if (action === 'dismiss') {
    const mission = card.querySelector('.open-tabs-badge');
    if (mission) {
      // If tabs are still open, close them first
      const res = await fetch('/api/missions');
      const missions = await res.json();
      const m = missions.find(m => m.id === missionId);
      if (m) await closeTabsByUrls(m.urls.map(u => u.url));
    }
    await fetch(`/api/missions/${missionId}/dismiss`, { method: 'POST' });
    card.classList.add('closing');
    setTimeout(() => card.remove(), 400);
    showToast(`Dismissed "${missionName}"`);
  }

  if (action === 'focus') {
    const res = await fetch('/api/missions');
    const missions = await res.json();
    const mission = missions.find(m => m.id === missionId);
    if (mission) {
      await focusTabsByUrls(mission.urls.map(u => u.url));
      showToast(`Focused on "${missionName}"`);
    }
  }
});

// Close all stale tabs button
document.getElementById('closeAllStaleBtn').addEventListener('click', async () => {
  const res = await fetch('/api/missions');
  const missions = await res.json();
  const stale = missions.filter(m => m.status !== 'active');
  const allUrls = stale.flatMap(m => (m.urls || []).map(u => u.url));

  await closeTabsByUrls(allUrls);

  // Remove all badges and close buttons
  document.querySelectorAll('.open-tabs-badge').forEach(el => el.remove());
  document.querySelectorAll('.action-btn.close-tabs').forEach(el => el.remove());
  document.getElementById('cleanupBanner').style.display = 'none';
  document.getElementById('statStale').textContent = '0';

  showToast('Closed all stale tabs. Breathing room restored.');
});

// Refresh button
document.getElementById('refreshBtn').addEventListener('click', async () => {
  document.getElementById('lastRefreshTime').textContent = 'Refreshing...';
  try {
    await fetch('/api/missions/refresh', { method: 'POST' });
    await renderDashboard();
    showToast('Dashboard refreshed with latest browsing data');
  } catch (err) {
    showToast('Refresh failed — check if server is running');
  }
});

// ---- Initialize ----
renderDashboard();
```

- [ ] **Step 4: Test the dashboard loads in browser**

```bash
cd "/Users/zara/Documents/For Claude/tab-mission-control"
npm start &
sleep 2
open http://localhost:3456
```

Expected: Dashboard should load with real mission data (if DeepSeek API key is configured) or be empty but styled correctly.

- [ ] **Step 5: Commit**

```bash
git add dashboard/
git commit -m "feat: dashboard frontend with mission cards, tab actions, and extension bridge"
```

---

### Task 8: Chrome Extension

**Files:**
- Create: `extension/manifest.json`
- Create: `extension/newtab.html`
- Create: `extension/newtab.js`
- Create: `extension/background.js`

- [ ] **Step 1: Create extension icon placeholders**

```bash
mkdir -p extension/icons
```

Generate simple placeholder icons (solid colored squares — can be replaced with real icons later):

```bash
cd "/Users/zara/Documents/For Claude/tab-mission-control"
node -e "
const fs = require('fs');

// Generate a minimal PNG (1x1 pixel, scaled up by the browser)
// This is a valid minimal PNG with an orange pixel
function createMinimalPng(size) {
  // For real icons, replace these with actual designed PNGs
  // This creates a valid but tiny PNG that Chrome will accept
  const canvas = Buffer.alloc(size * size * 4);
  for (let i = 0; i < size * size; i++) {
    canvas[i * 4] = 200;     // R
    canvas[i * 4 + 1] = 113; // G
    canvas[i * 4 + 2] = 58;  // B
    canvas[i * 4 + 3] = 255; // A
  }
  return canvas;
}

// We'll use data URLs in manifest instead — just create a simple SVG as placeholder
const svg = '<svg xmlns=\"http://www.w3.org/2000/svg\" width=\"128\" height=\"128\"><rect width=\"128\" height=\"128\" rx=\"16\" fill=\"%23c8713a\"/><text x=\"64\" y=\"78\" text-anchor=\"middle\" fill=\"white\" font-family=\"sans-serif\" font-size=\"64\" font-weight=\"bold\">M</text></svg>';

// Chrome needs actual PNG files, so let's create minimal ones
// For now, we'll note that real icons need to be added before loading the extension
console.log('Icon placeholders created. Replace with real PNG icons before loading extension.');
fs.writeFileSync('extension/icons/icon.svg', svg);
"
```

- [ ] **Step 2: Create extension/manifest.json**

```json
{
  "manifest_version": 3,
  "name": "Mission Control",
  "version": "1.0.0",
  "description": "Your browsing mission dashboard — replaces new tab page",
  "permissions": [
    "tabs",
    "activeTab",
    "storage"
  ],
  "chrome_url_overrides": {
    "newtab": "newtab.html"
  },
  "background": {
    "service_worker": "background.js"
  },
  "action": {
    "default_title": "Mission Control"
  }
}
```

- [ ] **Step 3: Create extension/newtab.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Mission Control</title>
<style>
  /* Full-screen iframe, no borders */
  * { margin: 0; padding: 0; }
  body { overflow: hidden; background: #f8f5f0; }
  iframe {
    width: 100vw;
    height: 100vh;
    border: none;
  }
  /* Fallback shown if server is not running */
  .fallback {
    display: none;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100vh;
    font-family: 'Segoe UI', system-ui, sans-serif;
    color: #9a918a;
    text-align: center;
    gap: 12px;
  }
  .fallback h2 {
    color: #1a1613;
    font-weight: 400;
    font-size: 20px;
  }
  .fallback p {
    font-size: 14px;
    max-width: 400px;
    line-height: 1.6;
  }
  .fallback code {
    background: #e8e2da;
    padding: 4px 12px;
    border-radius: 4px;
    font-size: 13px;
  }
</style>
</head>
<body>

<iframe id="dashboard" src="http://localhost:3456"></iframe>

<div class="fallback" id="fallback">
  <h2>Mission Control is not running</h2>
  <p>Start the server to see your dashboard:</p>
  <code>cd ~/Documents/For\ Claude/tab-mission-control && npm start</code>
  <p>Or run <code>npm run install-service</code> to start it automatically on login.</p>
</div>

<script src="newtab.js"></script>
</body>
</html>
```

- [ ] **Step 4: Create extension/newtab.js**

```js
// extension/newtab.js
// Bridge between the dashboard (iframe) and Chrome's tab APIs
// The iframe sends postMessage requests, this script translates them
// to chrome.tabs.* calls and sends the results back

const iframe = document.getElementById('dashboard');
const fallback = document.getElementById('fallback');

// Check if the server is running — show fallback if not
iframe.addEventListener('load', () => {
  // If the iframe loaded successfully, hide fallback
  try {
    // Can't access iframe content due to cross-origin, but if it loaded
    // without error, we're good
    fallback.style.display = 'none';
    iframe.style.display = 'block';
  } catch (e) {
    // Ignore cross-origin errors — that means it loaded fine
  }
});

iframe.addEventListener('error', () => {
  iframe.style.display = 'none';
  fallback.style.display = 'flex';
});

// Also check with a fetch as a backup
fetch('http://localhost:3456', { mode: 'no-cors' })
  .catch(() => {
    iframe.style.display = 'none';
    fallback.style.display = 'flex';
  });

// Listen for messages from the dashboard iframe
window.addEventListener('message', async (event) => {
  // Only accept messages from our dashboard
  if (event.origin !== 'http://localhost:3456') return;

  const { action, messageId } = event.data;

  // Route the action to the appropriate Chrome API
  let response = { messageId, success: true };

  try {
    if (action === 'getTabs') {
      // Get all open tabs across all windows
      const tabs = await chrome.tabs.query({});
      response.tabs = tabs.map(t => ({
        id: t.id,
        url: t.url,
        title: t.title,
        windowId: t.windowId,
        active: t.active,
      }));
    }

    else if (action === 'closeTabs') {
      // Close tabs whose URLs match any in the provided list
      const urlsToClose = event.data.urls || [];
      const allTabs = await chrome.tabs.query({});

      const tabIdsToClose = allTabs
        .filter(tab => {
          return urlsToClose.some(url => {
            try {
              const tabHost = new URL(tab.url).hostname;
              const targetHost = new URL(url).hostname;
              return tabHost === targetHost;
            } catch {
              return tab.url.includes(url) || url.includes(tab.url);
            }
          });
        })
        .map(tab => tab.id);

      if (tabIdsToClose.length > 0) {
        await chrome.tabs.remove(tabIdsToClose);
      }
      response.closedCount = tabIdsToClose.length;
    }

    else if (action === 'focusTabs') {
      // Find and focus the first matching tab
      const urlsToFocus = event.data.urls || [];
      const allTabs = await chrome.tabs.query({});

      const matchingTab = allTabs.find(tab => {
        return urlsToFocus.some(url => {
          try {
            return new URL(tab.url).hostname === new URL(url).hostname;
          } catch {
            return false;
          }
        });
      });

      if (matchingTab) {
        await chrome.tabs.update(matchingTab.id, { active: true });
        await chrome.windows.update(matchingTab.windowId, { focused: true });
      }
    }

  } catch (err) {
    response.success = false;
    response.error = err.message;
  }

  // Send the response back to the iframe
  iframe.contentWindow.postMessage(response, 'http://localhost:3456');
});
```

- [ ] **Step 5: Create extension/background.js**

```js
// extension/background.js
// Service worker that manages the toolbar badge
// Polls the server periodically to get mission count and update badge color

const SERVER_URL = 'http://localhost:3456';
const POLL_INTERVAL_MS = 60000; // Check every 60 seconds

async function updateBadge() {
  try {
    const res = await fetch(`${SERVER_URL}/api/stats`);
    const stats = await res.json();
    const count = stats.totalMissions;

    // Set badge text to mission count
    await chrome.action.setBadgeText({ text: count > 0 ? String(count) : '' });

    // Set badge color based on scatter level
    let color;
    if (count <= 3) color = '#3d7a4a';      // Green — focused
    else if (count <= 6) color = '#b8892e';  // Amber — moderate
    else color = '#b35a5a';                  // Red — high scatter

    await chrome.action.setBadgeBackgroundColor({ color });

  } catch {
    // Server not running — clear badge
    await chrome.action.setBadgeText({ text: '' });
  }
}

// Update badge immediately on install/startup
chrome.runtime.onInstalled.addListener(updateBadge);
chrome.runtime.onStartup.addListener(updateBadge);

// Poll periodically
setInterval(updateBadge, POLL_INTERVAL_MS);

// Also update when a tab is created or removed (quick feedback)
chrome.tabs.onCreated.addListener(updateBadge);
chrome.tabs.onRemoved.addListener(updateBadge);
```

- [ ] **Step 6: Commit**

```bash
git add extension/
git commit -m "feat: Chrome extension with new tab override, tab bridge, and badge"
```

---

### Task 9: Extract Dashboard CSS from Mockup

**Files:**
- Modify: `dashboard/style.css`

The mockup HTML has all the approved styles. We need to extract them into the separate CSS file.

- [ ] **Step 1: Copy the CSS from mockup.html into dashboard/style.css**

Read `mockup.html`, extract everything between `<style>` and `</style>`, write it to `dashboard/style.css`. This is the full approved design — no changes needed.

- [ ] **Step 2: Verify the dashboard still looks correct**

```bash
cd "/Users/zara/Documents/For Claude/tab-mission-control"
npm start &
sleep 2
open http://localhost:3456
```

Compare with mockup. Should look identical in layout and style.

- [ ] **Step 3: Commit**

```bash
git add dashboard/style.css
git commit -m "feat: extract dashboard CSS from approved mockup"
```

---

### Task 10: Setup, Config, and End-to-End Test

**Files:**
- Modify: `~/.mission-control/config.json` (add DeepSeek API key)

- [ ] **Step 1: Open config file for API key**

```bash
open ~/.mission-control/config.json
```

Add the DeepSeek API key to the `deepseekApiKey` field.

- [ ] **Step 2: Run the full server and test**

```bash
cd "/Users/zara/Documents/For Claude/tab-mission-control"
npm start
```

Open http://localhost:3456 in browser. Verify:
- Dashboard loads with greeting and date
- Missions appear (may take a few seconds for first analysis)
- Page chips show real URLs
- Footer stats update
- "Refresh now" button triggers re-analysis

- [ ] **Step 3: Install the Chrome extension**

1. Open Chrome
2. Go to `chrome://extensions`
3. Enable "Developer mode" (toggle in top right)
4. Click "Load unpacked"
5. Select the `extension/` folder inside the project
6. Open a new tab — should show the Mission Control dashboard

- [ ] **Step 4: Test tab operations**

With the extension loaded:
- Open a new tab — dashboard should show in the new tab
- The toolbar badge should show the mission count
- Click "Close N tabs" on a mission — tabs should close
- Click "Let it go" — mission should disappear

- [ ] **Step 5: Install as launch agent**

```bash
cd "/Users/zara/Documents/For Claude/tab-mission-control"
npm run install-service
```

Verify it starts on next login by restarting or checking:
```bash
launchctl list | grep mission-control
```

- [ ] **Step 6: Final commit**

```bash
git add -A
git commit -m "feat: Mission Control v1 — complete with server, dashboard, and extension"
```
