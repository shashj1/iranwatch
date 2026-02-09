
# Iran Watch

Automated daily OSINT briefing on US military posture toward Iran. Pulls live data from free APIs, runs it through Claude for IC-style analysis, and publishes a static intelligence dashboard to GitHub Pages.

**Live site →** [shashj1.github.io/iranwatch](https://shashj1.github.io/iranwatch/)

## What it does

A single Python script (`update.py`) runs once daily via GitHub Actions at 0500 UTC. It:

1. **Queries airplanes.live** for all military-tagged aircraft broadcasting ADS-B globally, filters to a bounding box covering Europe through South Asia (lat 10–55°N, lon 10°W–70°E), and identifies airframes by ICAO type code, registration, and hex address.
2. **Queries Polymarket and Metaculus** for Iran-related prediction markets — US-strike, Israel-strike, nuclear, and diplomatic categories — with deduplication and 24-hour probability change tracking.
3. **Scrapes CENTCOM RSS** for the latest press releases.
4. **Sends everything to Claude Haiku 4.5** with an IC-analyst system prompt. Claude produces a structured JSON briefing: threat level, key judgments, overnight summary, activity groups, market analysis, diplomatic context, and I&W indicator updates.
5. **Generates a self-contained HTML dashboard** (no external dependencies, no JavaScript frameworks) with an interactive canvas map, market cards, and activity feed. Writes `index.html` to the repo root.
6. **GitHub Actions commits and pushes** the updated `index.html`, which GitHub Pages serves.

Total cost per day: ~$0.01–0.03 in Claude API usage. All other APIs are free.

## Data sources

| Source | What it provides | Cost | Auth |
|---|---|---|---|
| [airplanes.live](https://airplanes.live) `/mil` endpoint | All military-tagged aircraft globally — type, registration, hex, position, altitude, speed. Unfiltered (shows aircraft that OpenSky suppresses). | Free | None |
| [Polymarket](https://polymarket.com) Gamma API | Prediction market probabilities, volume, event slugs. Searched across `iran`, `middle-east`, `geopolitics`, `us-foreign-policy` tags. | Free | None |
| [Metaculus](https://www.metaculus.com) API v2 | Community forecasting probabilities. Fetches specific curated question IDs (e.g. Q41594 "Will the US attack Iran before April 2026?") plus keyword search. | Free | None |
| [CENTCOM](https://www.centcom.mil) RSS | Latest press releases from US Central Command. | Free | None |
| [Anthropic API](https://console.anthropic.com) | Claude Haiku 4.5 for analysis. ~1 call/day. | ~$0.01–0.03/day | API key |

OpenSky Network is retained as a fallback if airplanes.live is unavailable.

## Coverage area

The bounding box spans **lat 10–55°N, lon 10°W–70°E** — covering:

- **West to 10°W**: Strait of Gibraltar, Morón and Rota (Spain), Lajes (Azores), western Mediterranean
- **North to 55°N**: Ramstein, Mildenhall, Lakenheath, Prestwick — USAF-in-Europe bases and the transatlantic refuelling corridor
- **South to 10°N**: Horn of Africa, Djibouti (Camp Lemonnier)
- **East to 70°E**: Afghanistan, Pakistan

This captures tankers staging through European bases en route to the Gulf — a gap that previously caused Iran Watch to miss aircraft visible to OSINT accounts like @DefenceGeek and @MATA_osint.

## Aircraft detection

airplanes.live pre-tags military aircraft in its database (`dbFlags & 1`), so detection does not rely on callsign guessing. The script additionally maintains:

- **~80 known military callsign prefixes** (RCH, FORTE, DOOM, ETHYL, GOLD, TABOR, ASCOT, etc.) for categorisation and the OpenSky fallback path
- **ICAO type code mapping** (~60 types: C-17, KC-135, RQ-4B, B-52, F-35, etc.) resolved from airplanes.live's `t` field
- **Hex-to-country mapping** for identifying nationality from ICAO24 address ranges (US: AE0000–AFF999, UK: 43C000–43CFFF, etc.)
- **70+ civilian airline exclusions** to prevent false positives on the OpenSky fallback
- **Location resolver** with 50+ reference points (bases, cities, water bodies, countries, European staging bases) that converts raw coordinates to descriptions like "over the Persian Gulf, ~40 mi from Al Udeid AB"

## Contextual analysis

The Claude system prompt includes explicit instructions to contextualise military movements rather than treating all traffic as Iran-related:

- Tanker movements from European bases may support routine deployments, NATO exercises, or diplomatic travel
- VIP transports near the Caucasus may relate to diplomatic visits (e.g. VP travel to Armenia)
- Fighter deployments from CONUS may be routine rotational replacements
- Only flag movements as Iran-significant when they match I&W patterns: tanker surges, bomber repositioning, ISR orbit changes, SEAD/DEAD package assembly

The analytical framework draws on Cynthia Grabo's *Anticipating Surprise: Analysis for Strategic Warning* (DIA, 2002).

## Prediction markets

Markets are categorised and colour-coded:

- 🔴 **US STRIKE** — Markets on whether the US directly strikes Iran
- 🟠 **ISRAEL STRIKE** — Markets on Israeli strikes against Iranian nuclear facilities
- 🔴 **NUCLEAR** — Iranian enrichment and weapons capability
- 🔵 **DIPLOMATIC** — Ceasefire, negotiations, agreements
- 🟠 **CONFLICT** — General military escalation

Aggressive deduplication strips date variants ("by March 2026", "by April 2026") to avoid showing five versions of the same question. Markets are sorted with US-strike first.

The script stores `history.json` between runs. On each update it computes:
- **Probability deltas** (▲/▼ badges showing movement since yesterday)
- **Volume spike detection** (flagged if trading volume exceeds 3× the previous day)
- **Alert banners** for moves ≥5 percentage points

## Setup

### Prerequisites

- A GitHub account
- An [Anthropic API key](https://console.anthropic.com/) (free tier is sufficient)

### Steps

1. Fork or clone this repo.

2. Add your API key as a GitHub Actions secret:
   - Go to **Settings → Secrets and variables → Actions**
   - Add `ANTHROPIC_API_KEY` with your Claude API key

3. Optionally add OpenSky credentials for the fallback path:
   - `OPENSKY_CLIENT_ID`
   - `OPENSKY_CLIENT_SECRET`

4. Enable GitHub Pages:
   - **Settings → Pages → Source**: Deploy from a branch → `main` → `/ (root)`

5. The workflow runs automatically at 05:00 UTC daily. To trigger manually:
   - **Actions → Update Iran Watch → Run workflow**

### Local development

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
python3 update.py
open index.html
```

Requires Python 3.10+ and `requests`. No other dependencies.

## Architecture

Everything lives in a single file (`update.py`, ~1,700 lines) that generates a single output (`index.html`). No build tools, no frontend framework, no database.

```
update.py
├── Configuration
│   ├── ME_BBOX (bounding box)
│   ├── MIL_PREFIXES (~80 callsign prefixes)
│   ├── CALLSIGN_AIRFRAMES (prefix → airframe/role)
│   └── _ICAO_TYPE_MAP (ICAO type → name/role)
│
├── Data fetchers
│   ├── fetch_opensky()        → airplanes.live /mil endpoint
│   ├── _fetch_opensky_fallback() → OpenSky Network fallback
│   ├── fetch_polymarket()     → Gamma API, categorised + deduped
│   ├── fetch_metaculus()      → API v2, known IDs + search
│   └── fetch_centcom_rss()    → CENTCOM XML feed
│
├── Analysis
│   ├── generate_analysis()    → Claude API call with IC prompt
│   └── generate_fallback_analysis() → Offline fallback
│
├── Output
│   ├── generate_html()        → Injects data into HTML template
│   └── HTML_TEMPLATE          → Embedded ~400-line HTML/CSS/JS
│
├── Utilities
│   ├── describe_location()    → Coords → human-readable place
│   ├── identify_airframe()    → Callsign → aircraft type
│   ├── _resolve_icao_type()   → ICAO code → aircraft type
│   └── _country_from_hex()    → Hex address → nationality
│
└── main()
    ├── Load history.json (yesterday's state)
    ├── Fetch all four data sources
    ├── Tag aircraft as new/returning
    ├── Compute market deltas + alerts
    ├── Save today's state to history.json
    ├── Generate Claude analysis
    └── Write index.html
```

## Files

| File | Purpose |
|---|---|
| `update.py` | The entire application — fetchers, analysis, HTML generation |
| `index.html` | Generated output — the dashboard (do not edit, overwritten each run) |
| `history.json` | Previous run's aircraft + market data for change detection |
| `.github/workflows/update.yml` | GitHub Actions workflow (daily cron + manual trigger) |
| `README.md` | This file |

## Limitations

- **Single daily snapshot**: Runs once at 0500 UTC. Aircraft transiting at other times are missed. OSINT accounts monitor 24/7.
- **ADS-B only**: Military aircraft frequently fly with transponders off. This is always a partial picture.
- **No ACARS/Mode-S**: Flight plans, cargo routing, and radio communications are not captured.
- **No hex database enrichment**: airplanes.live provides type and registration, but we don't cross-reference against community databases (like MilFlyBot) for unit assignments or deployment history.
- **Prediction market coverage**: Polymarket's tag system means some relevant markets may not be discovered. Metaculus known IDs must be manually curated.
- **No historical trending**: Only yesterday's data is compared. Longer-term trend lines (weekly/monthly probability curves) are not yet implemented.

## Licence

MIT
