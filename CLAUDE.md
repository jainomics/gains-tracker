# Gains Tracker — Claude Handoff Document

## What this is

A personal health tracking web app built as a single HTML file, hosted on GitHub Pages. It tracks daily calories and protein intake, body weight and body fat %, steps, active calories, and shows trends over time. It syncs body composition data automatically from a Withings smart scale via a nightly Python script, and syncs Apple Health activity data via Health Auto Export app.

The user is non-technical — explain concepts clearly, avoid jargon, walk through steps one at a time. They use **Safari 16.6** on Mac and iPhone. This is their first coding project.

---

## Live URLs

- **App:** https://jainomics.github.io/gains-tracker/
- **App code repo (public):** https://github.com/jainomics/gains-tracker
- **Data repo (private):** https://github.com/jainomics/gains-data
- **Cloudflare Worker:** https://gains-tracker-proxy.jainomics.workers.dev

---

## Architecture overview

```
Browser (Safari 16.6)
  └─ index.html (static, hosted on GitHub Pages)
       ├─ Food logs → GitHub API → gains-data/food_logs.json (private)
       ├─ AI food analysis → Cloudflare Worker / → Anthropic API (Claude Sonnet)
       ├─ Withings body data → gains-data/withings.json (private, read-only from app)
       ├─ Apple Health data → gains-data/health.json (private, read-only from app)
       └─ Local storage (browser cache — source of truth is always GitHub)

iPhone (Health Auto Export app, every 4 hours)
  └─ Reads Steps + Active Energy from Apple Health
       └─ POST to Cloudflare Worker /health
            └─ Worker reads/merges/writes gains-data/health.json via GitHub API

Mac (cron job, 11:40pm nightly)
  └─ ~/scripts/withings_sync.py
       ├─ Reads credentials from ~/.zshenv
       ├─ Fetches weight + body fat % from Withings API
       └─ Writes to gains-data/withings.json via GitHub API
```

---

## Key files

| File | Location | Purpose |
|---|---|---|
| `index.html` | gains-tracker repo | The entire app — HTML, CSS, JS in one file |
| `worker.js` | Cloudflare Worker | AI proxy + Apple Health ingestion endpoint |
| `withings_sync.py` | ~/scripts/ on Mac | Nightly Withings → GitHub sync script |
| `food_logs.json` | gains-data repo | Food logs, goals, favourites, body data |
| `withings.json` | gains-data repo | Body measurements from Withings scale |
| `health.json` | gains-data repo | Steps + active calories from Apple Health |
| `.withings_token.json` | ~/.withings_token.json on Mac | Withings OAuth token (auto-refreshed) |

---

## What the app tracks

- **Calories (kcal)** — daily total vs goal
- **Protein (g)** — daily total vs goal
- **Steps** — from Apple Health via Health Auto Export
- **Active calories burned** — from Apple Health via Health Auto Export
- **Body weight (kg)** and **body fat %** — from Withings scale + manual entry
- Carbs and fat are deliberately excluded — user only cares about the above.

---

## Data storage

### food_logs.json
```json
{
  "last_synced": "2026-03-28T23:30:00",
  "logs": {
    "2026-03-28": [
      { "name": "Chicken breast 200g", "cals": 330, "prot": 62, "time": "12:30", "id": "x7k2m9qr" }
    ]
  },
  "body": [
    { "date": "2026-03-28", "weight": 83.2, "fat": 18.1 }
  ],
  "goals": { "cals": 2000, "prot": 150, "carbs": 0, "fat": 0, "targetWeight": null, "targetFat": null },
  "deletions": { "2026-03-28": 1 },
  "favourites": [ { "name": "Porridge", "cals": 350, "prot": 12 } ],
  "favouriteRemovals": { "Some food": true },
  "moves": { "x7k2m9qr": true }
}
```

**Note:** Each log entry has an `id` field — an 8-character random alphanumeric string (e.g. `"x7k2m9qr"`) generated at log time. Old entries without an `id` fall back to `name|cals` dedup. New entries always have an `id`.

**`moves`** records the `id` of any entry that has been moved to a different date. During merge, any remote entry whose `id` appears in `moves` is silently skipped — preventing it from reappearing on the old date on any device, including ones that have never opened the app before.

### health.json
```json
{
  "last_synced": "2026-03-28T23:00:00",
  "entries": [
    { "date": "2026-03-28", "steps": 9241, "activeCals": 387 }
  ]
}
```

---

## Sync logic

**GitHub is the single source of truth.** Two distinct operations:

**On load — `pullFromGitHub()`**
Reads GitHub and overwrites local state entirely. No merge — GitHub wins. Local storage is a cache only.

**On any change — `syncToGitHub()`**
1. Reads current GitHub state to get SHA and pick up anything another device wrote
2. Applies local state on top, merging in any new remote entries not already present
3. Writes the merged result back to GitHub
4. Uses a pending queue (`syncPending` flag) — if a sync is in flight when another change arrives, it chains immediately after rather than dropping the change

**Debounce:** 500ms for food logging (batches rapid entries). Deletes cancel the debounce and sync immediately.

**Food log merge rules (per date):**
- If local has a deletion recorded for that date → local wins entirely, remote not pulled in
- Otherwise → union of local and remote entries, deduped by `id` if present, falling back to `name|cals` for old entries without an id
- Any remote entry whose `id` is in `moves` is skipped — prevents moved entries from resurrecting on the old date

**Favourites sync:**
- Uses `favouriteRemovals` object to record explicit unfavourites
- Merge: union of both devices' favourites, filtered through merged removals list
- Re-favouriting clears the removal record so it can sync freely again
- Removal records from remote are only absorbed if the food is NOT currently in local favourites (prevents stale removals killing a re-favourite)

---

## AI food logging

All logging methods go through a confirm-before-log flow — user always reviews before anything is saved.

1. **Describe food** (text) — calls Claude via Cloudflare Worker `POST /`
2. **Photo** — calls Claude via Cloudflare Worker with optional hint text
3. **Barcode** — queries Open Food Facts API (free, no key needed), good UK coverage
4. **Quick Log** — Favourites (starred) + Recent (14 days auto)
5. **Manual Entry** — name, calories, protein

**System prompts (UK-aware):**
- Text: *"You are a precise UK nutrition analyst. The user is based in the UK. Use UK portion sizes and recognise UK brands and supermarket products. Extract ALL food items described and return ONLY a JSON array. Each item: {name, cals, prot}. Integers only. When uncertain about portion size, estimate conservatively. For branded UK products use their actual nutritional values."*
- Photo: *"You are a UK nutrition analyst. Identify all food items visible. Use plate size, hand size, or any packaging as reference to estimate portions. Return ONLY a JSON array: [{name, cals, prot}]. Integers only. Be conservative when uncertain about portion size."* + optional user hint appended.

Model: `claude-sonnet-4-20250514`

---

## Cloudflare Worker

Two routes, both protected by `X-Secret` header:

**`POST /`** — AI proxy. Forwards request body to Anthropic API and returns response.

**`POST /health`** — Apple Health ingestion. Receives JSON from Health Auto Export, parses steps and active energy (handles both kJ and kcal units), reads current `health.json` from GitHub, merges by date, writes back.

Environment secrets required (set in Cloudflare dashboard):
- `WORKER_SECRET` — validates incoming requests
- `ANTHROPIC_API_KEY` — AI food analysis
- `GITHUB_TOKEN` — used by /health route to read/write health.json

The complete current Worker code is in `worker.js`.

---

## Credentials and secrets

All secrets are stored outside the codebase. `index.html` and `worker.js` contain zero hardcoded secrets.

| Secret | Where stored | What it does |
|---|---|---|
| GitHub PAT | Browser localStorage (`gh_token`) | Read/write to gains-data repo |
| Worker secret | Browser localStorage (`worker_secret`) | Authenticates app to Cloudflare Worker |
| Worker secret | Health Auto Export app (X-Secret header) | Authenticates health syncs to Worker |
| Anthropic API key | Cloudflare env secrets (`ANTHROPIC_API_KEY`) | AI food analysis |
| Worker secret | Cloudflare env secrets (`WORKER_SECRET`) | Validates all incoming Worker requests |
| GitHub token | Cloudflare env secrets (`GITHUB_TOKEN`) | Worker writes health.json to GitHub |
| Withings Client ID | ~/.zshenv on Mac | Withings API auth |
| Withings Client Secret | ~/.zshenv on Mac | Withings API auth |
| GitHub PAT | ~/.zshenv on Mac | Withings script writes to gains-data |

**GitHub PAT expires every 90 days.** User needs reminding to renew and paste into Goals tab. Worker secret does not expire.

Secrets are entered via the **Goals tab** — GitHub token card and Worker secret card. Both stored in browser localStorage only.

---

## Apple Health sync (Health Auto Export)

- **App:** Health Auto Export by Lybron Sobers (iOS, Premium tier required for automations)
- **Metrics exported:** Step Count, Active Energy
- **Format:** JSON, Summarized, Daily aggregation
- **Date range:** Yesterday
- **Frequency:** Every 4 hours (background — may not fire if phone locked)
- **Endpoint:** `https://gains-tracker-proxy.jainomics.workers.dev/health`
- **Auth header:** `X-Secret: [worker secret]`
- **Unit note:** Active Energy should be set to **kcal** in the app. Worker handles both kJ and kcal but kcal is preferred.
- To force an immediate sync: open Health Auto Export and tap **Export Now**

---

## Withings sync script

Lives at `~/scripts/withings_sync.py` on Mac. Runs via cron at 11:40pm:

```
40 23 * * * source ~/.zshenv && /usr/bin/python3 ~/scripts/withings_sync.py >> ~/withings_sync.log 2>&1
```

**What it does:**
- First run: fetches 730 days of history
- Subsequent runs: incremental sync with dynamic buffer (days since last sync + 2 days)
- Averages multiple weigh-ins on the same day
- Tracks `first_run_complete` flag in JSON to distinguish first vs subsequent runs
- Writes to `gains-data/withings.json` via GitHub API
- Uses Python stdlib only

**Credentials** read from `~/.zshenv` (not `.zshrc` — cron does not load `.zshrc`):
```
export WITHINGS_CLIENT_ID="..."
export WITHINGS_CLIENT_SECRET="..."
export GITHUB_TOKEN="..."
```

**SSL note:** Had to create a symlink for SSL certificates on Mac (Homebrew Python issue):
```
ln -s /opt/homebrew/etc/ca-certificates/cert.pem /opt/homebrew/etc/openssl@3/cert.pem
```

Check the log: `cat ~/withings_sync.log`

---

## Critical Safari 16.6 constraints

This is the most important technical constraint. The app MUST work in Safari 16.6. **Every JavaScript change must be checked against these rules before delivering:**

- **No arrow functions** (`=>`) anywhere in JS
- **No optional chaining** (`?.`)
- **No nullish coalescing** (`??`)
- **No template literals** (backticks) — use string concatenation
- **Use `var` throughout** — not `const` or `let`
- **No regex literals in certain positions** — use string methods instead
- **No quotes inside onclick attributes** — use index-based approaches or `this` references
- **Use `function()` not arrow functions** in forEach, filter, map, sort etc.

**Note:** The Worker (`worker.js`) runs on Cloudflare's V8 engine, not Safari — modern JS (arrow functions, const, etc.) is fine there.

**Always run this check on any new `index.html` JS before delivering:**

```python
import re
with open('index.html', 'r') as f:
    html = f.read()
js_start = html.index('<script>')
js_end = html.index('</script>', js_start)
js = html[js_start:js_end]
lines = js.split('\n')
issues = []
for i, line in enumerate(lines, 1):
    if re.search(r'[^=<>!]\s*=>\s*', line): issues.append((i, 'ARROW', line.strip()[:80]))
    if '?.' in line: issues.append((i, 'OPTCHAIN', line.strip()[:80]))
    if '??' in line: issues.append((i, 'NULLISH', line.strip()[:80]))
    if 'onclick="' in line:
        for m in re.findall(r'onclick="([^"]*)"', line):
            if m.count("'") % 2 != 0: issues.append((i, 'ODD QUOTES IN ONCLICK', line.strip()[:80]))
if issues:
    for item in issues: print(item)
else:
    print("All clear")
```

---

## Design language

Linear-inspired design system:

- **Font:** Geist and Geist Mono (Google Fonts)
- **Background:** `#0a0a0a` (near-black, not pure black)
- **Surface levels:** `#111111`, `#161616`, `#1c1c1c`, `#222222`
- **Borders:** `0.5px solid rgba(255,255,255,0.06)` — hairlines
- **Accent:** `#5e6ad2` (Linear's indigo) — CSS var `--accent`
- **Green:** `#4cc38a`, **Amber:** `#e5a50a`, **Red:** `#e5484d`
- **Activity colours:** Steps `#f97316` (orange), Active kcal `#ec4899` (pink)
- **Border radius:** 4px (sm), 6px (md), 8px (lg) — sharp not rounded
- **Typography:** tight letter-spacing, uppercase tracking on labels, monospace for all numbers
- **No shadows** — elevation via borders only
- Chart colours must be hardcoded hex — Chart.js cannot read CSS variables
- **Note:** `--accent2` is used in some inline styles but is not defined as a CSS variable — it resolves to nothing and is effectively transparent. Avoid using it; use hardcoded hex or `--accent` instead.

---

## App structure

**Pages:** Today, Trends, Body, Goals

**Today page:**
- Macro cards showing calories and protein vs goal with progress bars
- Date navigator — arrows to step days, tap date to pick, "Today" pill when on past date
- Logging tabs: Describe Food (AI text), Quick Log (favourites + recent), Photo, Barcode, Manual Entry
- Today's log list with edit (✎), star (favourite), and delete per entry — edit opens an inline row to change name, kcal, protein, or date; moving to a different date records the entry's `id` in `moves`
- Past dates can be logged retrospectively — all logging methods work on any selected date

**Trends page:**
- 7-day summary table (always last 7 days, colour-coded vs goals)
- Period selector: 7d / 30d / 3m / 1y
- 4 stat boxes: Avg daily calories, Avg daily protein, Avg active calories, Total steps (label reads "this week" on 7d, "this period" otherwise)
- Single combined chart: Calories (left axis, indigo), Steps + Active kcal (right axis, orange and pink)
- Body composition chart (weight + body fat %)
- Nutrition ↔ Body insights (recomp scoring)

**Body page:** Withings auto-sync status, manual body measurement entry, measurement history

**Goals page:** Calorie/protein targets, body goals, progress, GitHub token entry, Worker secret entry

---

## Recomposition scoring (Trends page)

User goal is body recomposition — lose fat while maintaining or gaining muscle. Requires **5+ days food data and 3+ body measurements** (the placeholder text in the HTML incorrectly says "2 weeks" — the actual threshold in code is 5 days / 3 measurements).

- **Recomp working** — fat dropping >0.3%, weight >= -0.5kg
- **Cutting** — fat dropping, weight also dropping >0.5kg
- **Bulking** — fat up >0.3%, weight up
- **Maintaining** — fat change <=0.3%, weight change <=0.5kg
- **Mixed signal** — anything else

---

## Deliberate decisions

- **Calories and protein only for food** — carbs and fat excluded by user preference
- **Confirm before logging** — no auto-logging, user always reviews AI estimates
- **GitHub private repo for data** — `gains-data` is private; `gains-tracker` is public but contains zero personal data or secrets
- **Cloudflare Worker for AI and health ingestion** — API keys cannot safely live in browser code
- **GitHub is source of truth** — local storage is a cache only, pulled fresh on every load
- **No OneDrive** — Azure app registration does not support personal Microsoft 365 accounts
- **Open Food Facts for barcodes** — free, good UK coverage, no key required
- **Safari 16.6 support** — hard constraint, drives all JS decisions in index.html
- **Manual body entry still available** — Withings auto-sync is supplementary
- **Date keys always use local date parts** — never `toISOString()` which returns UTC and causes date collisions around midnight for UK users

---

## Known limitations

- AI logging only works when served from GitHub Pages or a web server — not when opened as a local file
- Withings cron requires Mac to be awake at 11:40pm
- Health Auto Export background sync unreliable when iPhone is locked — open app to force sync
- Barcode database gaps — Open Food Facts patchy for newer products; photo tab is the fallback
- GitHub PAT expires every 90 days — user must renew manually in Goals tab

---

## How to make and deploy changes

1. Get the latest `index.html` from the user or this conversation
2. Make edits
3. Run the Safari 16.6 JS check (see constraints section)
3b. Run `node --check` on the extracted JS to catch syntax errors the linter won't catch (broken string concatenation, mismatched quotes, etc.) — a syntax error silently breaks the entire app
4. Deliver the updated `index.html`
5. User uploads to `github.com/jainomics/gains-tracker` replacing existing `index.html`
6. Wait ~60 seconds for GitHub Pages to deploy
7. Hard refresh: Cmd+Shift+R on Mac, close/reopen tab on iPhone

To test locally before uploading:
```
cd ~/Downloads && python3 -m http.server 8080
```
Then visit `http://localhost:8080/index.html`

For Worker changes: edit in Cloudflare dashboard → Deploy. No file to upload.

---

## Ideas for future development

- Better AI accuracy — meal context, confidence flagging
- Streak tracking and habit data
- Improved mobile UX — bottom tab bar native feel
- Making the app a product (aspirational, not immediate)
