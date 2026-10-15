# Auction Scout — Project Spec (v0.1, Retail first)

> Working name. Rename freely.
> Owner: Ben. Builder: Claude Code (local instance).
> Status: **Draft for review.** Nothing built yet.

---

## Part A — Plain-English summary (for Ben)

**What it is:** a private gold-making assistant for World of Warcraft. It watches the auction house all day, remembers prices, and tells you when something is cheap enough to buy and make a profit.

**The four pieces** (think of a small shop with a scout, a notebook, a bulletin board, and a runner):

| Piece | Lives on | Job | Analogy |
|---|---|---|---|
| **Collector** | Ubuntu home server | Downloads Blizzard's auction data every hour and saves it | The scout who walks the market every hour |
| **Analyzer + Web page** | Ubuntu home server | Builds price history, finds deals, shows reports, sends phone alerts | The notebook and the bulletin board |
| **Helper** | Windows gaming PC | Carries the deal list into the game, carries scan results out | The runner between the shop and the market |
| **Add-on** | Inside WoW | Fresh scan at the AH, highlights deals, one click to buy | Your assistant standing at the counter with you |

**What it finds (first version):**
- **Vendor snipes:** items listed for less than a vendor will pay → guaranteed profit.
- **Disenchant snipes:** items listed for less than the mats they disenchant into → profit on average.

**What it will NOT do:**
- ❌ Buy anything by itself — **every purchase is your click** (Blizzard rule).
- ❌ Log in or scan while you're away (that's botting).
- ❌ Be reachable from the public internet.

**Scope of this version:** **Retail (Midnight), Area 52 (US) only.** Classic Era and WoW Forever come later, and the design leaves room for them.

---

## Part B — Technical spec (for Claude Code)

### B1. Goals & non-goals

**Goals (v1):**
1. Hourly collection of US commodity auctions and Area 52 connected-realm auctions from the Blizzard Game Data API.
2. Durable price history (per item, per hour).
3. Vendor-snipe and disenchant-snipe detection with configurable thresholds.
4. Web dashboard (LAN + private remote access) and phone push alerts.
5. In-game add-on: fresh AH scan, deal highlighting, one-click-per-item buying, posting price hints.
6. Windows helper that syncs data between server and add-on.

**Non-goals (v1):** Classic Era, WoW Forever, flip/"buy low sell high" alerts, event/patch planner, loot lookup, multi-user support, public hosting. (See B12 roadmap.)

**Design constraint:** keep game-specific logic behind a "game flavor" boundary (`retail`, later `classic_era`, `forever`) so new flavors are additive.

### B2. Architecture

```
 Blizzard API ──hourly──> [Collector] ──> [Database] <── [Analyzer] ──> [Web UI / Alerts]
  (us.api.blizzard.com)    (Ubuntu)        (Ubuntu)       (Ubuntu)         (Ubuntu)
                                               ^  |
                                   scan upload |  | deal list + values (HTTP, LAN/Tailscale)
                                               |  v
                                          [Helper .exe] (Windows)
                                               ^  |
                           SavedVariables file |  | Data.lua file
                                               |  v
                                          [WoW Add-on] (in game)
```

**Why the helper exists:** WoW add-ons have no network access. They can only (a) read Lua files in their own folder at load/`/reload`, and (b) write a SavedVariables file on logout/`/reload`. The helper bridges those files to the server.

### B3. Tech stack (recommended defaults — builder may propose alternatives with reasons)

| Concern | Choice | Notes |
|---|---|---|
| Server language | Python 3.12+ | |
| Web framework | FastAPI + Jinja2 templates + HTMX | Server-rendered; no JS build step |
| Database | SQLite (WAL mode) to start | Move to Postgres only if size/speed demands |
| Scheduling | systemd timers | Collector hourly; analyzer after each collection |
| Charts | Lightweight JS lib from CDN (e.g. uPlot or Chart.js) | |
| Phone alerts | **ntfy** (self-hosted on the same server, or ntfy.sh) | Ben installs the ntfy phone app |
| Remote access | **Tailscale** | No port forwarding; never expose to public internet |
| Helper | Python, packaged to a single `.exe` with PyInstaller | Runs at Windows login (Startup folder or Task Scheduler) |
| Add-on | Lua + XML, WoW Retail client | |
| Config | `config.yaml` + `.env` for secrets | |

### B4. Data sources

**Blizzard Game Data API** (developer account at develop.battle.net; OAuth client-credentials flow).
- Token: `POST https://oauth.battle.net/token` (`grant_type=client_credentials`).
- Commodities (region-wide): `GET /data/wow/auctions/commodities?namespace=dynamic-us` — updates ~hourly; **costs 25 quota units per call**.
- Realm auctions: `GET /data/wow/connected-realm/{id}/auctions?namespace=dynamic-us`.
  - Area 52 connected-realm ID: look up at startup via the connected-realm search/realm endpoints and cache it (expected `3676`, **verify**).
- Items: `GET /data/wow/item/{id}?namespace=static-us` → name, quality, item class/subclass, level, `sell_price`, binding, etc. Cache permanently; refresh on game patch.
- Use `Last-Modified` / `If-Modified-Since` to skip unchanged snapshots (note: commodities still charges full quota).
- Rate limits: respect documented limits (currently ~36,000 req/hour, 100 req/sec); implement backoff on 429.

**Builder must verify** all endpoints, namespaces, fields and limits against the current Blizzard developer docs before coding, and note any differences in `docs/api-notes.md`.

### B5. Data model (minimum)

- `items` — item_id, name, quality, class, subclass, inventory_type, base item level, vendor_sell_price, expansion (if derivable), is_commodity, last_refreshed.
- `snapshots` — id, source (`commodities` | `realm:<id>`), flavor, fetched_at, last_modified, listing_count, raw_file_path.
- `listings_latest` — the most recent snapshot's listings only (replaced each hour): auction_id, item_id, bonus_list (sorted string), modifiers, quantity, unit_price or buyout, time_left.
- `price_hourly` — item_key, hour, min_price, p10_price, median_price, total_quantity, listing_count. **Kept forever.**
- `price_daily` — rolled up from hourly after 90 days (optional optimization).
- `sell_through_hourly` — item_key, hour, est_units_disappeared (see B6.3).
- `de_table` — rules mapping (expansion, quality, item class, item-level band) → expected mats & quantities. Loaded from `data/disenchant.yaml`, user-editable.
- `de_results` — actual disenchant outcomes logged by the add-on (for calibration).
- `deals` — detected deals per analyzer run (type, item, listing, cost, expected value, profit, ROI, created_at, status).
- `scan_uploads` — in-game scan results uploaded by the helper.

**Item key:** `item_id` for commodities; `item_id + sorted bonus_list` for gear (bonus IDs change item level, which changes value). v1 may treat unknown bonus IDs as opaque strings.

**Raw snapshots:** store gzipped JSON on disk for 7 days (debugging/reprocessing), then delete. Only aggregates are kept long-term.

### B6. Analysis rules

All thresholds live in `config.yaml`.

**B6.1 Vendor snipe**
- Condition: `buyout_per_unit < vendor_sell_price` **and** `(vendor_sell_price − buyout_per_unit) × quantity ≥ min_profit_vendor` (default 2g).
- Caveat: retail vendor price for gear may vary with bonus IDs/item level. The **add-on re-checks the true vendor price in game** (via the item link) before highlighting. Server-side result is a "candidate."

**B6.2 Disenchant snipe**
- Expected DE value = Σ (expected qty of each mat × mat market value) per the `de_table`.
- Mat market value = conservative price (e.g. 3-day median of hourly `p10_price`).
- Net DE value = expected DE value × (1 − AH cut 5%).
- Condition: `net_DE_value − buyout ≥ min_profit_de` (default 10g) **and** ROI ≥ `min_roi_de` (default 20%).
- Only armor/weapons of uncommon quality or better; exclude cosmetic/non-disenchantable subclasses (maintain an exclude list).
- Seed `disenchant.yaml` from public sources (e.g. Wowhead disenchanting guides). Do **not** copy code/data from other add-ons whose licenses forbid it.
- **Calibration:** once the add-on logs real DE results, show "expected vs. actual" per rule and allow adjusting.

**B6.3 Market value & sell-through (supporting metrics)**
- Market value (per item): 14-day median of hourly `p10_price` (tunable).
- Sell-through estimate: listings/quantity that vanish between consecutive snapshots before their `time_left` would expire. Label as **estimate** (can't tell sold vs. cancelled).

**B6.4 Freshness**
- Each deal stores the snapshot age. Deals older than 2 snapshots auto-expire.

### B7. Web dashboard

Pages (server-rendered, mobile-friendly):
1. **Home:** last collection time & health, counts of current deals by type, top 10 deals by profit.
2. **Vendor snipes** and **DE snipes:** sortable tables (item, qty, price, value, profit, ROI, age). Link each item to its history page and Wowhead.
3. **Item page:** price history chart (hourly min/median, quantity), sell-through estimate, current listings.
4. **Settings:** thresholds, alert rules, watchlist, DE table editor (or link to YAML with reload button).
5. **System:** collector logs, API quota usage, DB size, last helper sync.

No login needed on LAN/Tailscale for v1, but bind to LAN/Tailscale interfaces only.

### B8. Alerts

- Push via ntfy when: a single deal's profit ≥ `alert_min_profit` (default 100g), or total available deal profit ≥ `alert_total_profit` (default 500g).
- Quiet hours setting; at most one alert per hour unless a deal exceeds a "big deal" threshold.
- Alert text example: *"Area 52: 6 vendor snipes (+312g), 2 DE deals (+180g)."* with a link to the dashboard.

### B9. Windows helper

- Config: server URL (Tailscale/LAN), WoW install path (auto-detect `C:\Program Files (x86)\World of Warcraft\_retail_` with override), account folder name(s).
- Every 5 minutes (and on start):
  - **Down:** fetch `/api/addon-data?flavor=retail&realm=area-52` → write `Interface\AddOns\AuctionScout\Data.lua` atomically (write temp file, then rename).
  - **Up:** if `WTF\Account\<ACCOUNT>\SavedVariables\AuctionScout.lua` changed, parse it and POST scan results + DE logs to the server, then mark uploaded.
- Tray icon or simple window showing: last sync, server reachable yes/no.
- Never stores Blizzard API secrets.

**Data.lua contents (kept small; target < 5 MB):**
- Generated timestamp, realm, snapshot age.
- Market values for items relevant to deals (mats + candidates), not the whole market.
- Vendor and DE candidate lists (item_id, bonus key, max buy price per unit, expected value).
- DE table subset needed to evaluate fresh-scan items.
- Thresholds from config.

### B10. In-game add-on (Retail)

**Functions:**
1. **Status panel** at the AH: data age, counts of candidate deals.
2. **Fresh scan button:** full AH scan (`C_AuctionHouse.ReplicateItems` or current equivalent; throttled ~every 15 min). Re-evaluate every listing locally using Data.lua values **plus** true in-game vendor price from the item link. Also catches new deals not on the server list.
3. **Deal list:** sorted by profit; for each, a **Buy** button that performs a targeted item search then purchases **only on Ben's click** (hardware event). Commodities use the commodity purchase flow with price confirmation.
4. **Guardrails:** max price per unit = value from Data.lua; abort if current price > max; confirm dialog above a gold threshold (default 1,000g).
5. **Logging to SavedVariables:** fresh scan summary (item, price, qty, timestamp), purchases made, and **disenchant results** (item disenchanted → mats received) via loot events.
6. **Posting hint (stretch for v1):** when selling, show market value and suggested price from Data.lua.

**Rules:**
- No automation without a hardware click per purchase. No background scanning outside AH interaction. Comply with Blizzard UI add-on policy.
- Use the current `## Interface:` version for Midnight (check in game: `/dump select(4, GetBuildInfo())`).
- Builder must verify AH API calls against current documentation (warcraft.wiki.gg API pages) and note them in `docs/addon-api-notes.md`.

### B11. Security & ops

- Blizzard client ID/secret in `.env` on the server only; `.env` in `.gitignore`.
- Web UI and API bound to LAN + Tailscale only. Helper → server calls use a shared token from config.
- Nightly SQLite backup (keep 14 days).
- systemd services/timers with `Restart=on-failure`; logs via journald.
- One-command setup script for the server (`scripts/install_server.sh`) and a written step-by-step `docs/SETUP.md` **written for a non-developer** (Ben).

### B12. Milestones & acceptance criteria

Build in order. **Stop after each milestone, demo it to Ben in plain language, and get approval before continuing.**

| # | Milestone | Done when… |
|---|---|---|
| **M0** | Repo skeleton, config, API notes | Repo structure exists; `docs/api-notes.md` confirms endpoints; Blizzard credentials work (token fetch test passes). |
| **M1** | Collector | Hourly timer runs on Ubuntu; 24 h of commodities + Area 52 snapshots stored; aggregates in `price_hourly`; items cached with vendor prices. |
| **M2** | Analyzer: vendor + DE snipes | Deals table populated after each collection; unit tests for profit math, fees and thresholds; `disenchant.yaml` seeded for Midnight + recent expansions. |
| **M3** | Web dashboard + alerts | Pages in B7 work on phone via Tailscale; ntfy alert received for a test deal. |
| **M4** | Helper + add-on (read-only) | Data.lua appears in AddOns folder; add-on shows status panel and deal list at the AH; fresh scan re-evaluates deals. |
| **M5** | Add-on buying + logging | Buy buttons work with click-per-purchase and guardrails; purchases and DE results upload to the server and show on the dashboard. |

**Later (not in v1):** flip alerts (below-market + fast-selling), loot/"what is this worth" lookup, event & patch planner, Classic Era flavor (both factions + neutral AH), WoW Forever flavor.

### B13. Suggested repo layout

```
auction-scout/
  server/
    collector/        # Blizzard API client, snapshot fetch, aggregation
    analyzer/         # market value, sell-through, vendor/DE snipes
    web/              # FastAPI app, templates, static
    alerts/           # ntfy integration
    db/               # schema, migrations
  helper/             # Windows sync app
  addon/AuctionScout/ # WoW add-on (Lua/XML, .toc)
  data/disenchant.yaml
  config.example.yaml
  .env.example
  scripts/            # install, backup
  docs/               # SETUP.md, api-notes.md, addon-api-notes.md
  tests/
```

### B14. Open questions / assumptions to verify

1. Area 52 connected-realm ID and whether any other realms are connected to it (they share an AH).
2. Whether vendor sell price from the Item API varies by bonus IDs for retail gear.
3. Current Midnight disenchanting outputs and yields per quality/item-level band.
4. Exact current AH add-on API for full scan and purchase (names may have changed in Midnight).
5. Expected snapshot sizes (to size disk; estimate < 50 GB/year with 7-day raw retention).
6. Ben's WoW account folder name(s) and install path.

### B15. Working agreement for the builder

- Ben is **not a developer**. Explain decisions in plain language with analogies; keep instructions short with bullet steps.
- Ask before: spending money, opening network ports, installing system-wide software, or anything hard to undo.
- Prefer simple, boring, well-documented tools.
- Commit in small steps with clear messages; keep `docs/SETUP.md` current.
