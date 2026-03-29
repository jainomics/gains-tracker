# Gains Tracker — Claude Handoff Document

## What this is

A personal health tracking web app built as a single HTML file, hosted on GitHub Pages. It tracks daily calories and protein intake, body weight and body fat %, and shows trends and correlations over time. It also syncs body composition data automatically from a Withings smart scale via a nightly Python script.

The user is non-technical — explain concepts clearly, avoid jargon, walk through steps one at a time. They use **Safari 16.6** on Mac and iPhone. This is their first coding project.

---

## Live URLs

- **App:** https://jainomics.github.io/gains-tracker/
- **App code repo (public):** https://github.com/jainomics/gains-tracker
- **Data repo (private):** https://github.com/jainomics/gains-data
- **Cloudflare Worker (AI proxy):** https://gains-tracker-proxy.jainomics.workers.dev

---

## Architecture overview

```
Browser (Safari 16.6)
  └─ index.html (static, hosted on GitHub Pages)
       ├─ Food logs → GitHub API → gains-data/food_logs.json (private)
       ├─ AI food analysis → Cloudflare Worker → Anthropic API (Claude Sonnet)
       ├─ Withings body data → gains-data/withings.json (private, read-only from app)
       └─ Local storage (browser cache, syncs with GitHub on every save)

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
| `withings_sync.py` | ~/scripts/ on Mac | Nightly Withings → GitHub sync script |
| `food_logs.json` | gains-data repo | Food logs, goals, favourites, body data |
| `withings.json` | gains-data repo | Body measurements from Withings scales |
| `.withings_token.json` | ~/.withings_token.json on Mac | Withings OAuth token (auto-refreshed) |

---

## What the app tracks

- **Calories (kcal)** — daily total vs goal
- **Protein (g)** — daily total vs goal
- That is it. Carbs and fat are deliberately excluded — user only cares about these two.

---

## Data storage

All data lives in `gains-data/food_logs.json` with this structure:

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
  "favourites": [
    { "name": "Porridge", "cals": 350, "prot": 12 }
  ]
}
```

Body data also exists separately in `gains-data/withings.json`, written by the Python script.

---

## Sync logic

The app syncs to GitHub 1.5 seconds after any change (debounced). On load it reads from GitHub and merges with local storage.

**Merge rules:**
- Remote wins if it has more entries for a date AND no deletions recorded for that date
- Local wins if any deletions are recorded for that date (prevents deleted entries coming back)
- Favourites are merged by name — both local and remote contributions kept

**Delete fix:** When a user deletes an entry, it is tracked in `state.deletions[date]` and syncs immediately (not debounced). This prevents the 1.5s debounce window from pulling the deleted entry back from remote.

---

## AI food logging

Three logging methods all go through a confirm-before-log flow:

1. **Describe food** (text) — calls Claude via Cloudflare Worker
2. **Photo** — calls Claude via Cloudflare Worker with optional hint text
3. **Barcode** — queries Open Food Facts API (free, no key needed), good UK coverage

After AI analysis, the user sees an editable card with calories and protein. They can adjust numbers before confirming. Nothing logs automatically without review.

**System prompts (UK-aware):**
- Text: *"You are a precise UK nutrition analyst. The user is based in the UK. Use UK portion sizes and recognise UK brands and supermarket products. Extract ALL food items described and return ONLY a JSON array. Each item: {name, cals, prot}. Integers only. When uncertain about portion size, estimate conservatively."*
- Photo: *"You are a UK nutrition analyst. Identify all food items visible. Use plate size, hand size, or any packaging as reference to estimate portions. Return ONLY a JSON array: [{name, cals, prot}]. Integers only. Be conservative when uncertain about portion size."* + optional user hint appended.

**Cloudflare Worker:** Proxies requests to Anthropic API. Requires `X-Secret` header matching `WORKER_SECRET` environment secret stored in Cloudflare. API key stored as `ANTHROPIC_API_KEY` in Cloudflare environment secrets. Model: `claude-sonnet-4-20250514`.

---

## Credentials and secrets

All secrets are stored outside the codebase. The `index.html` file contains zero hardcoded secrets.

| Secret | Where stored | What it does |
|---|---|---|
| GitHub PAT (`ghp_...`) | Browser localStorage (`gh_token`) | Read/write to gains-data repo |
| Worker secret | Browser localStorage (`worker_secret`) | Authenticates app to Cloudflare Worker |
| Anthropic API key | Cloudflare environment secrets (`ANTHROPIC_API_KEY`) | AI food analysis |
| Worker secret | Cloudflare environment secrets (`WORKER_SECRET`) | Validates incoming requests to Worker |
| Withings Client ID | ~/.zshenv on Mac | Withings API auth |
| Withings Client Secret | ~/.zshenv on Mac | Withings API auth |
| GitHub PAT | ~/.zshenv on Mac | Withings script writes to gains-data |

**GitHub PAT expires every 90 days.** User needs reminding to renew and paste into Goals tab. Worker secret does not expire.

The user enters secrets via the **Goals tab** — there is a card for the GitHub token and a separate card for the Worker secret. Both stored in browser localStorage only.

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

**Always run this check on any new JS before delivering:**

```python
import re
with open('macros_tracker.html', 'r') as f:
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

## Cloudflare Worker code

The complete Worker code (no secrets in code — all via env):

```javascript
export default {
  async fetch(request, env) {
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'POST, OPTIONS',
          'Access-Control-Allow-Headers': 'Content-Type, X-Secret'
        }
      });
    }

    const secret = request.headers.get('X-Secret');
    if (secret !== env.WORKER_SECRET) {
      return new Response('Unauthorized', { status: 401 });
    }

    const body = await request.json();

    const response = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': env.ANTHROPIC_API_KEY,
        'anthropic-version': '2023-06-01'
      },
      body: JSON.stringify(body)
    });

    const data = await response.json();

    return new Response(JSON.stringify(data), {
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      }
    });
  }
};
```

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

## Design language

Linear-inspired design system:

- **Font:** Geist and Geist Mono (Google Fonts)
- **Background:** `#0a0a0a` (near-black, not pure black)
- **Surface levels:** `#111111`, `#161616`, `#1c1c1c`, `#222222`
- **Borders:** `0.5px solid rgba(255,255,255,0.06)` — hairlines
- **Accent:** `#5e6ad2` (Linear's indigo)
- **Green:** `#4cc38a`, **Amber:** `#e5a50a`, **Red:** `#e5484d`
- **Border radius:** 4px (sm), 6px (md), 8px (lg) — sharp not rounded
- **Typography:** tight letter-spacing, uppercase tracking on labels, monospace for all numbers
- **No shadows** — elevation via borders only
- Chart colours must be hardcoded hex — Chart.js cannot read CSS variables

---

## App structure

**Pages:** Today, Trends, Body, Goals

**Logging tabs (Today page):**
1. Describe Food — AI text → confirm flow
2. Quick Log — Favourites (starred, permanent) + Recent (14 days, auto)
3. Photo — camera/upload + optional hint → AI → confirm flow
4. Barcode — camera scan or manual → Open Food Facts → confirm flow
5. Manual Entry — name, calories, protein

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

- **Calories and protein only** — carbs and fat excluded by user preference
- **Confirm before logging** — no auto-logging, user always reviews AI estimates
- **GitHub private repo for data** — `gains-data` is private; `gains-tracker` is public but contains zero personal data or secrets
- **Cloudflare Worker for AI** — API key cannot safely live in browser code
- **No OneDrive** — Azure app registration does not support personal Microsoft 365 accounts
- **Open Food Facts for barcodes** — free, good UK coverage, no key required
- **Safari 16.6 support** — hard constraint, drives all JS decisions
- **Manual body entry still available** — Withings auto-sync is supplementary

---

## Known limitations

- AI logging only works when served from GitHub Pages or a web server — not when opened as a local file
- Withings cron requires Mac to be awake at 11:40pm
- No real-time sync between devices — syncs on load and 1.5s after changes
- Barcode database gaps — Open Food Facts patchy for newer products; photo tab is the fallback
- GitHub PAT expires every 90 days — user must renew manually

---

## How to make and deploy changes

1. Get the latest `index.html` from the user or this conversation
2. Make edits
3. Run the Safari 16.6 JS check (see constraints section)
4. Deliver the updated `index.html`
5. User uploads to `github.com/jainomics/gains-tracker` replacing existing `index.html`
6. Wait ~60 seconds for GitHub Pages to deploy
7. Hard refresh: Cmd+Shift+R on Mac, close/reopen tab on iPhone

To test locally before uploading:
```
cd ~/Downloads && python3 -m http.server 8080
```
Then visit `http://localhost:8080/index.html`

---

## Ideas discussed for future development

- Apple Health integration (complex, deprioritised — no public API for web apps)
- Better AI accuracy — meal context, confidence flagging
- Making the app a product (aspirational, not immediate)
- Improved mobile UX — bottom tab bar native feel
- Streak tracking and habit data
