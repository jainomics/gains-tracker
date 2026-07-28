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
       ├─ AI food analysis → Cloudflare Worker /  → Anthropic API (Claude Sonnet)
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
      { "name": "Chicken breast 200g", "cals": 330, "prot": 62, "time": "12:30" }
    ]
  },
  "body": [
    { "date": "2026-03-28", "weight": 83.2, "fat": 18.1 }
  ],
  "goals": { "cals": 2000, "prot": 150, "carbs": 0, "fat": 0, "targetWeight": null, "targetFat": null },
  "deletions": { "2026-03-28": 1 },
  "favourites": [ { "name": "Porridge", "cals": 350, "prot": 12 } ],
  "favouriteRemovals": { "Some food": true }
}
```

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
- Otherwise → union of local and remote entries, deduped by `name|cals` key

**Favourites sync:**
- Uses `favouriteRemovals` object to record explicit unfavourites
- Merge: union of both devices' favourites, filtered through merged removals list
- Re-favouriting clears the removal record so it can sync freely again
- Removal records from remote are only absorbed if the food is NOT currently in local favourites (prevents stale removals killing a re-favourite)

**Reading large files (>1MB) — `githubDecodeContent()`:**
GitHub's Contents API only inlines base64 `content` for files under 1MB. Past that, it returns `"content": ""` and `"encoding": "none"`, which broke the app once `food_logs.json` crossed 1MB (JSON.parse on empty string → "Unexpected EOF", read silently failed while writes kept succeeding). `githubRead()`, `syncWithings()`, and `syncHealth()` now all route through a shared `githubDecodeContent(data)` helper: if `data.encoding === 'base64'` it decodes inline as before; otherwise it fetches the same file via the Git Blobs API (`/git/blobs/{sha}` with `Accept: application/vnd.github.v3.raw`), which supports up to 100MB. No action needed as data grows further — just be aware this is why the read path looks like two branches, not one.

**Saving a new/updated GitHub token — `saveGithubToken()`:**
Deliberately calls `syncToGitHub()` (merge+push), **not** `pullFromGitHub()` (destructive pull). This was a real bug: a device that had been offline (expired token) and had local-only logs would, on pasting a fresh token, immediately overwrite those local logs with GitHub's older copy via a plain pull. Fixed 29 Jul 2026 after it nearly cost the user several months of iPhone food logs. Do not revert this to a plain pull without re-solving that problem.

---

## AI food logging

All logging methods go through a confirm-before-log flow — user always reviews before anything is saved.

1. **Describe food** (text) — calls Claude via Cloudflare Worker `POST /`
2. **Photo** — calls Claude via Cloudflare Worker with optional hint text
3. **Barcode** — queries Open Food Facts API (free, no key needed), good UK coverage
4. **Quick Log** — Favourites (starred) + Recent (14 days auto)
5. **Manual Entry** — name, calories, protein

**System prompts (UK-aware):**
- Text: *"You are a precise UK nutrition analyst. The user is based in the UK. Use UK portion sizes and recognise UK brands and supermarket products. Extract ALL food items described and return ONLY a JSON array. Each item: {name, cals, prot}. Integers only. When uncertain about portion size, estimate conservatively."*
- Photo: *"You are a UK nutrition analyst. Identify all food items visible. Use plate size, hand size, or any packaging as reference to estimate portions. Return ONLY a JSON array: [{name, cals, prot}]. Integers only. Be conservative when uncertain about portion size."* + optional user hint appended.

Model: `claude-sonnet-4-6` — **updated 29 Jul 2026.** The previous model, `claude-sonnet-4-20250514`, was retired by Anthropic on 15 June 2026; requests to it returned an error response with no `content` field, which the app's parser silently treated as an unparseable AI reply ("Could not parse — try being more specific") on every single request, with no indication the real problem was the model ID. If AI logging fails identically on every input regardless of complexity, check whether the model string needs updating again before assuming it's a token/secret issue — Anthropic retires old dated model snapshots periodically. Current models as of this writing: Fable 5, Opus 4.8, Sonnet 4.6, Haiku 4.5.

---

## Cloudflare Worker

Two routes, both protected by `X-Secret` header:

**`POST /`** — AI proxy. Forwards request body to Anthropic API and returns response.

**`POST /health`** — Apple Health ingestion. Receives JSON from Health Auto Export, parses steps and active energy (handles both kJ and kcal units), reads current `health.json` from GitHub, merges by date, writes back.

Environment secrets required (set in Cloudflare dashboard → Workers & Pages → gains-tracker-proxy → Settings → Variables and Secrets):
- `WORKER_SECRET` — validates incoming requests (does not expire)
- `ANTHROPIC_API_KEY` — AI food analysis (does not expire unless manually rotated)
- `GITHUB_TOKEN` — used by /health route to read/write health.json (**expires every 90 days — see Credentials section**)

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

### ⚠️ GitHub PAT expires every 90 days — in THREE separate places

This is the single biggest recurring failure mode for this app, and it caused a full day of debugging on 29 Jul 2026. **The same 90-day GitHub PAT is used in three independent locations, and renewing one does NOT renew the others:**

1. **Browser localStorage** (`gh_token`) — set via the app's **Goals tab**, per device/browser. Each device (Mac Safari, iPhone Safari) has its own separate copy — updating it on the Mac does nothing for the iPhone.
2. **`~/.zshenv` on Mac** (`GITHUB_TOKEN`) — used by the Withings cron script. Edit via `nano ~/.zshenv`.
3. **Cloudflare Worker env secret** (`GITHUB_TOKEN`) — used by the `/health` route for Apple Health syncing. Edit via Cloudflare dashboard → Workers & Pages → gains-tracker-proxy → Settings → Variables and Secrets → Edit.

**When the token expires, the symptoms differ depending on which of the three has gone stale**, and multiple can fail simultaneously without being obviously related:
- Browser copy expired → app shows "Sync error", food logging stops syncing across devices
- Mac `.zshenv` copy expired → `withings_sync.log` shows `HTTP Error 401: Unauthorized`, body data stops updating
- Cloudflare copy expired → Apple Health steps/active calories silently stop appearing in Trends, with no visible error anywhere in the app (this one is easy to miss for a long time)

**When renewing, generate ONE new classic PAT (repo scope, 90-day expiry) and paste it into all three locations in the same sitting**, rather than fixing them one at a time as symptoms surface.

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

**⚠️ macOS updates can silently break this job.** After a macOS update on 29 Jul 2026, `crontab -l` came back completely empty — the update had wiped the user's crontab entirely, and separately, Terminal needed to be re-granted **Full Disk Access** (System Settings → Privacy & Security → Full Disk Access) before `crontab -e` would even work. Neither failure produced an obvious error until the log was checked — the job just silently stopped running. **After any macOS update, check `crontab -l` shows the job is still present** before assuming anything else is wrong.

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
- **Accent:** `#5e6ad2` (Linear's indigo)
- **Green:** `#4cc38a`, **Amber:** `#e5a50a`, **Red:** `#e5484d`
- **Activity colours:** Steps `#f97316` (orange), Active kcal `#ec4899` (pink)
- **Border radius:** 4px (sm), 6px (md), 8px (lg) — sharp not rounded
- **Typography:** tight letter-spacing, uppercase tracking on labels, monospace for all numbers
- **No shadows** — elevation via borders only
- Chart colours must be hardcoded hex — Chart.js cannot read CSS variables

---

## App structure

**Pages:** Today, Trends, Body, Goals

**Today page:**
- Macro cards showing calories and protein vs goal with progress bars
- Date navigator — arrows to step days, tap date to pick, "Today" pill when on past date
- Logging tabs: Describe Food (AI text), Quick Log (favourites + recent), Photo, Barcode, Manual Entry
- Today's log list with star (favourite) and delete per entry
- Past dates can be logged retrospectively — all logging methods work on any selected date

**Trends page:**
- 7-day summary table (always last 7 days, colour-coded vs goals)
- Period selector: 7d / 30d / 3m / 1y
- Calories & Protein chart (dual axis)
- Macro breakdown doughnut
- Body composition chart (weight + body fat %)
- Activity trend chart (steps + active kcal, dual axis)
- Nutrition ↔ Body insights (recomp scoring)

**Body page:** Withings auto-sync status, manual body measurement entry, measurement history

**Goals page:**
- Calorie/protein targets, body goals, progress
- **Backup your data** (added 29 Jul 2026) — one-tap export of full local state (logs, body, goals, favourites, health) as JSON to clipboard, for the user to paste into Notes as a safety copy before touching sync settings. Added after a near-miss where several months of iPhone-only food logs almost got overwritten by a destructive token-save pull (see Sync logic section above).
- GitHub token entry, Worker secret entry

---

## Recomposition scoring (Trends page)

User goal is body recomposition — lose fat while maintaining or gaining muscle. Requires 5+ days food data and 3+ body measurements.

- **Recomp working** — fat dropping, weight stable or up
- **Cutting** — fat and weight both dropping
- **Bulking** — fat and weight both rising
- **Maintaining** — minimal change in both
- **Mixed signal** — unclear pattern

---

## Deliberate decisions

- **Calories and protein only for food** — carbs and fat excluded by user preference
- **Confirm before logging** — no auto-logging, user always reviews AI estimates
- **GitHub private repo for data** — `gains-data` is private; `gains-tracker` is public but contains zero personal data or secrets
- **Cloudflare Worker for AI and health ingestion** — API keys cannot safely live in browser code
- **GitHub is source of truth** — local storage is a cache only, pulled fresh on every load (except immediately after a token save, which merges instead — see Sync logic)
- **No OneDrive** — Azure app registration does not support personal Microsoft 365 accounts
- **Open Food Facts for barcodes** — free, good UK coverage, no key required
- **Safari 16.6 support** — hard constraint, drives all JS decisions in index.html
- **Manual body entry still available** — Withings auto-sync is supplementary
- **Caching left as-is — no-cache fix built but NOT adopted** — after repeated instances of deployed fixes appearing not to work because Safari kept serving a stale cached copy, a fix was built (no-cache meta tags in `<head>`) that would make every load fetch the latest version automatically. On 29 Jul 2026 the user decided not to deploy it and to keep the page's caching behaviour as-is. **Do not re-propose or redeploy this fix unprompted** — the code is available in chat history if the user asks for it later. Because of this decision, every deploy still requires a manual `?v=N` cache-busting reload (see "How to make and deploy changes" below) — this is not a leftover step to be cleaned up, it's the current expected workflow.

---

## Known limitations

- AI logging only works when served from GitHub Pages or a web server — not when opened as a local file
- Withings cron requires Mac to be awake at 11:40pm
- Health Auto Export background sync unreliable when iPhone is locked — open app to force sync
- Barcode database gaps — Open Food Facts patchy for newer products; photo tab is the fallback
- **GitHub PAT expires every 90 days, in three independent locations** — see Credentials section above for the full list and symptoms
- **macOS updates can silently wipe the Withings cron job and revoke Full Disk Access** — check `crontab -l` after any macOS update
- **Anthropic retires old dated model snapshots periodically** — if AI logging fails on every input with no useful error, check whether `claude-sonnet-4-6` (currently in use) has itself been superseded
- **Page caching is left at default (by user choice)** — a no-cache fix exists but was not adopted; every deploy requires a manual `?v=N` cache-busting reload on each device, or a deployed fix will appear not to have worked

---

## Troubleshooting playbook

Quick diagnostic order when something breaks, learned from a full debugging session on 29 Jul 2026 where multiple unrelated things had failed at once:

1. **Check the sync status indicator** (top right of app) — "Sync error" points at the browser's GitHub token.
2. **Test the GitHub token directly**, bypassing the app, via Safari's JS console (Develop menu → Show JavaScript Console):
   ```
   fetch('https://api.github.com/repos/jainomics/gains-data/contents/food_logs.json', {headers: {'Authorization': 'token ' + localStorage.getItem('gh_token'), 'Accept': 'application/vnd.github.v3+json'}, cache: 'no-store'}).then(function(r){return r.text()}).then(console.log)
   ```
   A 401 confirms an expired/wrong token in the browser specifically. A 200 with `"encoding": "none"` and empty `"content"` means the file has crossed 1MB (should be a non-issue after the `githubDecodeContent` fix, but useful to know the signature).
3. **Test the AI Worker directly**, same console, to isolate AI logging failures from sync failures (they use completely different credentials):
   ```
   fetch('https://gains-tracker-proxy.jainomics.workers.dev', {method:'POST', headers:{'Content-Type':'application/json','X-Secret':localStorage.getItem('worker_secret')}, body: JSON.stringify({model:'claude-sonnet-4-6', max_tokens:50, messages:[{role:'user', content:'say hi'}]})}).then(function(r){console.log('STATUS:', r.status); return r.text()}).then(function(t){console.log(t)})
   ```
   Status 200 with real text back = Worker secret and Anthropic key both fine. A response with no `content` field usually means a retired/invalid model ID.
4. **Check what model the currently-loaded page is actually calling** (rules out stale cache serving old code):
   ```
   document.querySelector('script:not([src])').textContent.match(/model:\s*'([^']+)'/g)
   ```
5. **Withings script:** run manually to see live errors before waiting for cron: `source ~/.zshenv && /usr/bin/python3 ~/scripts/withings_sync.py`, then `crontab -l` to confirm the job is still scheduled.
6. **After every deploy, always reload with a new `?v=N` suffix before judging whether a fix worked.** The page does not skip caching (a fix for this exists but the user chose not to adopt it — see "Deliberate decisions"), so a plain reload will very likely still serve the old cached version and make a working fix look broken. This is a required step, not a fallback. Harmless — origin-scoped localStorage is untouched by query strings.

---

## How to make and deploy changes

1. Get the latest `index.html` from the user or this conversation
2. Make edits
3. Run the Safari 16.6 JS check (see constraints section)
4. Deliver the updated `index.html`
5. User uploads to `github.com/jainomics/gains-tracker` replacing existing `index.html`
6. Wait ~60 seconds for GitHub Pages to deploy
7. **Reload with a fresh cache-busting suffix — required every time**, e.g. `https://jainomics.github.io/gains-tracker/?v=N` (bump N higher than any previously used). The page does not skip caching (see "Deliberate decisions"), so a plain reload will very likely still serve the old version. Do this on every device that needs the update — Mac and iPhone each cache independently.

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
- Consider an in-app reminder/banner when the GitHub PAT is approaching 90 days old, rather than relying on the user to remember — the multi-location expiry (browser + Mac + Cloudflare) has now caused a full day of debugging once
