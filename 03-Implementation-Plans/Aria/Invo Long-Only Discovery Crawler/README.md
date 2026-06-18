# Invo Long-Only Discovery Crawler — Implementation Plan

> **Goal:** Build a one-off/recurring crawler that discovers broad Invo trader portfolios,
> fetches their closed position history, filters to long-only trades, and produces a
> ranked leaderboard of the best traders for Aria to follow (longs only).

**Repo:** `HMardani7/invo-reader` (VPS: `/root/apps/invo-reader/`, sandbox: `/srv/hermes-repos/invo-reader/`)
**Plan location:** `/home/hermes/workbench/invo_long_only_crawler/`
**Date:** 2026-06-06

---

## Why This Matters

Aria only copies **long positions** (Bitget spot mode). The Invo "trending" leaderboard shows total P&L
(including shorts), so top-ranked traders like `crypto_rocket` (97.9% WR, +781% PnL) are useless for Aria
because their profitable activity is almost entirely shorts.

This crawler strips away the short-side noise and answers:
> "Who are the best long-only traders on Invo, regardless of whether we currently follow them?"

---

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                   Invo Reader Container               │
│                                                      │
│  ┌──────────┐    ┌─────────────┐    ┌────────────┐  │
│  │ watcher   │    │ crawler.js  │    │ server.js   │  │
│  │ (polling) │    │ (NEW)       │    │ (health)    │  │
│  └──────────┘    └──────┬──────┘    └────────────┘  │
│                         │                            │
│              ┌──────────▼──────────┐                 │
│              │  browser (Playwright)│                 │
│              │  captured auth token │                 │
│              └──────────┬──────────┘                 │
│                         │                            │
│         ┌───────────────▼───────────────┐            │
│         │    Invo API (api.invoapp.com) │            │
│         │    /v1_0/trending/*           │            │
│         │    /v1_0/portfolios/*         │            │
│         │    /v1_0/posts/*              │            │
│         │    /v1_0/search/*             │            │
│         └───────────────────────────────┘            │
│                                                      │
│  Output: /app/data/long_only_leaderboard.json        │
└──────────────────────────────────────────────────────┘
```

The crawler reuses the existing Invo Reader's:
- **Browser session** — already logged in, Playwright page object
- **Captured auth token** — `capturedAuthToken` from `watcher.js`
- **API call pattern** — `page.evaluate(fetch(...))` for authenticated API calls

It does NOT:
- Modify the notification polling loop
- Interfere with Aria signal forwarding
- Require separate login/credentials

---

## Crawler Logic

### Phase 1 — Discovery

Collect candidate portfolios from multiple sources:

1. **Trending portfolios** — `/v1_0/trending/get_portfolios_pl`
   - Request: `{ page: N, size: 50 }` (paginate up to 10 pages = 500 portfolios)
2. **Trending users** — `/v1_0/trending/get_users`
   - Request: `{ page: N, size: 50 }` (paginate up to 5 pages)
   - Then fetch each user's portfolios via `/v1_0/portfolios/v2/get_users_portfolios`
     - Request: `{ userId: "<id>", page: { page: 1, size: 20 } }`
3. **User search** — `/v1_0/search/users`
   - Request: `{ query: "", page: N, size: 50 }` (paginate)
   - Empty query returns all searchable users (broader than trending)

**Deduplication:** Merge all sources by `portfolio.id`. Keep unique portfolios only.

### Phase 2 — Investment History Fetch

For each unique portfolio (up to a configurable max, e.g., 200):

1. `/v1_0/posts/get_investment_history`
   - Request: `{ userId: "<ownerId>", portfolioId: "<portfolioId>", page: 1, size: 50, baseOnly: true }`
   - Response: posts with `post.update` containing trade details
   - Paginate until we've seen all closed positions or hit a max-depth limit

2. From each post, extract:
   - `post.update.baseId` — stable position key
   - `post.update.directionLong` — true/false
   - `post.update.entrySim` / `post.update.lastSim` — entry/current simulated price
   - `post.update.isOpen` / `post.update.isAdded` — position state
   - `post.update.createdAt` — timestamp
   - `post.update.entrySize` / `post.update.remainingAdds` — sizing
   - `post.update.leverage` — source leverage
   - `post.assetData.symbol` — coin ticker

### Phase 3 — Long-Only Scoring

For each trader/portfolio, compute from closed long positions only:

| Metric | Formula |
|--------|---------|
| `long_trades` | Count of closed long positions (min 10 for inclusion) |
| `long_winrate` | `wins / long_trades` |
| `long_cumulative_pnl_pct` | Sum of closed PnL percentages |
| `long_avg_pnl_pct` | Mean of individual PnL percentages |
| `long_median_pnl_pct` | Median of individual PnL percentages |
| `long_max_win` | Best individual long PnL |
| `long_max_loss` | Worst individual long PnL |
| `long_recent_pnl_pct` | Sum PnL from last 30 days only |
| `long_short_mix` | `long_trades / total_trades` (what % of activity is longs) |
| `current_open_longs` | Count of currently open long positions |
| `avg_source_leverage` | Average leverage used |
| `top_long_symbols` | Most-traded coins on long side |

**Ranking formula (composite score):**
```
score = (long_avg_pnl_pct * 0.3) +
        (long_winrate * 0.25) +
        (long_cumulative_pnl_pct / max_cumulative * 0.2) +
        (long_recent_pnl_pct / max_recent * 0.15) +
        (long_short_mix * 0.10)
```
Only portfolios with ≥10 closed long positions are scored.

### Phase 4 — Output

Write `/app/data/long_only_leaderboard.json`:

```json
{
  "generated_at": "2026-06-06T12:00:00Z",
  "source": "invo-api-crawl",
  "total_portfolios_crawled": 200,
  "total_long_traders_qualified": 45,
  "rankings": [
    {
      "rank": 1,
      "username": "tyron",
      "name": "Tyron",
      "portfolio_title": "Chef tyron💲",
      "portfolio_id": "23307af7-...",
      "long_trades": 32,
      "long_winrate": 0.969,
      "long_cumulative_pnl_pct": 93.82,
      "long_avg_pnl_pct": 2.93,
      "long_median_pnl_pct": 2.15,
      "long_max_win": 28.5,
      "long_max_loss": -5.2,
      "long_recent_pnl_pct": 12.4,
      "long_short_mix": 0.67,
      "current_open_longs": 3,
      "avg_source_leverage": 8.2,
      "top_long_symbols": ["TAO", "SOL", "ZEC", "BCH"],
      "composite_score": 0.872,
      "recommendation": "follow",
      "caveats": "Heavy on SOL/TAO; monitor if market rotates"
    }
  ],
  "aria_suggested_follows": ["tyron", "1000x", "bitshark"],
  "aria_suggested_watch": ["glitty"],
  "aria_suggested_avoid": ["crypto_rocket", "luxora", "kzs"]
}
```

---

## Files to Touch

| File | Action | Purpose |
|------|--------|---------|
| `src/crawler.js` | **CREATE** | Main crawler implementation |
| `src/config.js` | MODIFY | Add crawler config section |
| `src/server.js` | MODIFY | Add `/crawl` and `/leaderboard` endpoints |
| `src/index.js` | MODIFY | Register crawl endpoint on startup |
| `.env.example` | MODIFY | Document new env vars |

### `src/crawler.js` — Full Module Design

```javascript
// Modules follow existing patterns: config, logger, browser, store
const config = require('./config');
const logger = require('./logger');

/**
 * Invo Long-Only Discovery Crawler
 *
 * Usage:
 *   const result = await crawl({ maxPortfolios: 200, minLongTrades: 10 });
 *   // result written to /app/data/long_only_leaderboard.json
 */

// ─── API Helpers ──────────────────────────────────────────────────────────

/**
 * Call Invo API through the browser page using captured auth token.
 * Mirrors watcher.js pattern: page.evaluate(fetch(...))
 */
async function callApi(page, path, body = {}) { ... }

/**
 * Paginate an endpoint, collecting all results.
 * Handles: stop on empty page, stop when oldest item < cutoff date
 */
async function paginateApi(page, path, bodyFactory, maxPages = 10) { ... }

// ─── Discovery Phase ──────────────────────────────────────────────────────

/**
 * Fetch trending portfolios from /trending/get_portfolios_pl
 * Returns: [{ portfolio_id, owner_id, username, title, plSnapshot, winRate, ... }]
 */
async function discoverTrendingPortfolios(page) { ... }

/**
 * Fetch trending users, then their portfolios
 * Returns: same shape as above
 */
async function discoverTrendingUsers(page) { ... }

/**
 * Search users, then their portfolios
 * Returns: same shape as above
 */
async function discoverSearchUsers(page) { ... }

// ─── Investment History Phase ─────────────────────────────────────────────

/**
 * Fetch all investment history posts for a portfolio.
 * Filters to posts with directionLong=true.
 * Deduplicates by baseId (stable position key).
 * Returns: [{ baseId, directionLong, entrySim, lastSim, isOpen, pnlPct, ... }]
 */
async function fetchPortfolioHistory(page, portfolioId, ownerId) { ... }

// ─── Scoring Phase ────────────────────────────────────────────────────────

/**
 * Compute long-only metrics from a set of closed long positions.
 * Only includes portfolios with ≥ minLongTrades closed longs.
 */
function scoreLongPositions(positions, minLongTrades = 10) { ... }

/**
 * Rank portfolios by composite score.
 */
function rankPortfolios(scored) { ... }

// ─── Output Phase ─────────────────────────────────────────────────────────

/**
 * Write leaderboard JSON to /app/data/long_only_leaderboard.json
 * Also writes a compact suggested-follows list for Aria.
 */
function writeLeaderboard(rankings, metadata) { ... }

// ─── Main Entry Point ─────────────────────────────────────────────────────

/**
 * Main crawl entry point.
 *
 * @param {object} options
 * @param {number} options.maxPortfolios - Max portfolios to deep-crawl (default 200)
 * @param {number} options.minLongTrades - Min closed longs required (default 10)
 * @param {number} options.maxHistoryPages - Max history pages per portfolio (default 5)
 * @param {string}  options.outputPath - Output file path (default /app/data/long_only_leaderboard.json)
 * @returns {object} { rankings, metadata, error }
 */
async function crawl(options = {}) {
  const {
    maxPortfolios = 200,
    minLongTrades = 10,
    maxHistoryPages = 5,
    outputPath = path.join(config.data.dir, 'long_only_leaderboard.json'),
  } = options;

  // 1. Get browser page + verify auth token
  // 2. Discovery: collect unique portfolios from all sources
  // 3. Deduplicate + sort by potential (use plSnapshot as rough sort)
  // 4. Deep-crawl: fetch investment history for top N portfolios
  // 5. Score: filter to long-only, compute metrics
  // 6. Rank: sort by composite score
  // 7. Write: output JSON file
  // 8. Return: summary
}

module.exports = { crawl, fetchPortfolioHistory, scoreLongPositions, rankPortfolios, writeLeaderboard };
```

### `src/config.js` — Add Crawler Section

```javascript
crawler: {
  maxPortfolios: parseInt(process.env.CRAWLER_MAX_PORTFOLIOS, 10) || 200,
  minLongTrades: parseInt(process.env.CRAWLER_MIN_LONG_TRADES, 10) || 10,
  maxHistoryPages: parseInt(process.env.CRAWLER_MAX_HISTORY_PAGES, 10) || 5,
  requestDelayMs: parseInt(process.env.CRAWLER_REQUEST_DELAY_MS, 10) || 200,
  discoveryPages: parseInt(process.env.CRAWLER_DISCOVERY_PAGES, 10) || 10,
  outputFile: process.env.CRAWLER_OUTPUT_FILE || 'long_only_leaderboard.json',
},
```

### `src/server.js` — Add Endpoints

```javascript
// GET /crawl — trigger a one-off crawl (async, returns immediately with status)
app.get('/crawl', async (req, res) => {
  if (crawlInProgress) {
    return res.json({ status: 'already_running', started_at: crawlStartedAt });
  }
  crawlInProgress = true;
  crawlStartedAt = new Date().toISOString();
  // Fire async — don't block the response
  crawler.crawl({ maxPortfolios: config.crawler.maxPortfolios })
    .then(result => {
      lastCrawlResult = result;
      crawlInProgress = false;
    })
    .catch(err => {
      lastCrawlError = err.message;
      crawlInProgress = false;
    });
  res.json({ status: 'started', started_at: crawlStartedAt });
});

// GET /leaderboard — return latest crawl results
app.get('/leaderboard', (req, res) => {
  const filePath = path.join(config.data.dir, config.crawler.outputFile);
  if (fs.existsSync(filePath)) {
    res.json(JSON.parse(fs.readFileSync(filePath, 'utf-8')));
  } else {
    res.json({ error: 'No leaderboard yet. Trigger /crawl first.' });
  }
});
```

---

## Run Modes

### Mode 1 — One-off manual crawl

```bash
docker exec invo-reader-xvfb node src/crawler.js
```
Useful for ad-hoc discovery. Runs inside the container, reuses the live auth token.

### Mode 2 — HTTP-triggered crawl

```bash
curl http://<vps>:8090/crawl
```
Starts async crawl. Check progress via:
```bash
curl http://<vps>:8090/crawl/status   # (if we add this)
curl http://<vps>:8090/leaderboard    # latest results
```

### Mode 3 — Periodic refresh (future)

Add a cron-like scheduler in the reader that re-crawls every 6–12 hours.
Could use `setInterval()` in `index.js` or Hermes cron.

---

## Pitfalls & Design Decisions

### PITFALL: Auth Token Expiry Mid-Crawl

The crawl can take many minutes (200 portfolios × 3–5 API calls each). The JWT access
token expires every ~10 minutes. **Mitigation:**
- Before each API call batch, check if the token is close to expiry
- If stale, call `refreshAuthToken(page)` (same function used by `watcher.js`)
- Keep the crawl state serializable so it can survive a token refresh

### PITFALL: Rate Limiting

Invo's API may throttle heavy usage. **Mitigation:**
- Configurable `CRAWLER_REQUEST_DELAY_MS` (default 200ms between calls)
- Exponential backoff on 429 responses
- Max 5 history pages per portfolio (configurable)

### PITFALL: Browser Session Contention

The crawler shares the browser page with the notification watcher. **Mitigation:**
- The crawler is designed to be a one-off, not continuous
- It's triggered manually or on a schedule, not per-tick
- For production periodic use, consider a separate browser context

### PITFALL: investment_history Only Has Recent Data

Invo's `/posts/get_investment_history` may only return the most recent N positions
per portfolio. Very old positions may be missing. **Mitigation:**
- Document this limitation in the output
- Score includes a `recent_pnl_pct` metric that's honest about the data window
- For traders with long histories, we capture what we can

### PITFALL: directionLong Field May Not Exist on All Positions

Some position updates may lack the `directionLong` field. **Mitigation:**
- Fall back to `post.assetData.side` or extract from `post.content` text
- If side cannot be determined, exclude the position rather than guess

### PITFALL: Browser Crash During Long Crawl

A 200-portfolio crawl is hours of browser uptime with many `page.evaluate()` calls.
**Mitigation:**
- Save intermediate state to disk every N portfolios
- On restart, resume from checkpoint
- Or keep the crawl small enough to complete in one session (~200 portfolios)

---

## Verification

### Pre-Deployment Test

```bash
# From the invo-reader sandbox or VPS:
docker exec invo-reader-xvfb node -e "
const crawler = require('./src/crawler');
// Quick test: crawl just 3 portfolios
crawler.crawl({ maxPortfolios: 3, minLongTrades: 1 }).then(r => {
  console.log('Test crawl complete:', JSON.stringify(r.metadata));
}).catch(e => console.error(e));
"
```

### Post-Deployment Verification

```bash
# 1. Trigger crawl
curl http://localhost:8090/crawl

# 2. Wait for completion (check /status for crawl state)
curl http://localhost:8090/status | jq .crawl

# 3. Get leaderboard
curl http://localhost:8090/leaderboard | jq '.rankings[:5] | .[] | {username, long_winrate, long_avg_pnl_pct}'

# 4. Verify output file
docker exec invo-reader-xvfb ls -la /app/data/long_only_leaderboard.json
docker exec invo-reader-xvfb head -c 500 /app/data/long_only_leaderboard.json
```

### Aria Integration Verification

```bash
# Aria reads the JSON file directly (it's on the same VPS, different container):
docker exec hyperliquid-copy-bot uv run python -c "
import json
with open('/app/data/long_only_leaderboard.json') as f:
    data = json.load(f)
for r in data['rankings'][:5]:
    print(f\"{r['username']:15} long_wr={r['long_winrate']:.1%} avg_pnl={r['long_avg_pnl_pct']:+.2f}% rec={r['recommendation']}\")
print('Suggested follows:', data['aria_suggested_follows'])
"
```

Note: Aria's container would need the Invo Reader's data directory mounted or accessible.
Alternatively, Aria could call `http://invo-reader-xvfb:8090/leaderboard` on the Docker network.

---

## Tasks

| # | Task | File | Estimated Effort |
|---|------|------|-----------------|
| 1 | Add crawler config section | `src/config.js` | 5 min |
| 2 | Create crawler module with API helpers | `src/crawler.js` | 30 min |
| 3 | Implement discovery phase (trending + search) | `src/crawler.js` | 20 min |
| 4 | Implement investment history fetch | `src/crawler.js` | 25 min |
| 5 | Implement scoring + ranking | `src/crawler.js` | 20 min |
| 6 | Implement output writer | `src/crawler.js` | 10 min |
| 7 | Add HTTP endpoints to server | `src/server.js` | 15 min |
| 8 | Register endpoints in index.js | `src/index.js` | 5 min |
| 9 | Update .env.example | `.env.example` | 5 min |
| 10 | Test crawl (3 portfolios) | manual | 15 min |
| 11 | Full crawl (200 portfolios) + verify | manual | 30 min |
| 12 | Deploy to VPS, restart container | ops | 10 min |

Total estimated: ~3 hours of implementation + 1 hour of testing.

---

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| Reuse watcher.js auth pattern, not separate login | Avoids credential duplication; crawler runs in same container |
| Output to JSON file, not DB | Simpler for Aria to consume; no schema migration burden |
| HTTP-triggered, not continuous | Crawler is expensive (100s of API calls); continuous would risk rate limits |
| Min 10 closed long positions for ranking | Fewer = noise; 10 gives statistical minimum for win rate |
| Composite score weights avg PnL heaviest | Consistency matters more than one lucky trade |
| Don't modify watcher.js polling loop | Crawler is independent; no risk of breaking signal forwarding |
| Save intermediate state for long crawls | Browser crashes during crawling are rare but recovery is free with checkpointing |
