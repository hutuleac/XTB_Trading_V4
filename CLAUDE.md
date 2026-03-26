# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

XTB Trading Dashboard (v4.0.0) is a fully client-side stock analysis tool. No build step, no bundler, no backend. Pure HTML + JavaScript served as static files. All indicator math runs in the browser. Three external sources provide data: **Finnhub** (quotes, fundamentals), **Twelve Data** (OHLCV history), and public endpoints for **VIX** and **Fear & Greed index** (used in the dashboard header).

> Reference docs: `ARCHITECTURE.md` (deep-dive), `STOCKS_DASHBOARD_BLUEPRINT.md` (original design spec). `index.html` is 44 KB — it contains the full Tailwind config, custom CSS, and all markup inline.

---

## Running the Project

No install step. Serve the root directory as static files:

```bash
python -m http.server 8080
# or
npx serve .
```

Open `http://localhost:8080`. On first load, the setup overlay asks for both API keys and a watchlist.

There are no tests, no linter config, and no build commands.

---

## Architecture

### Module Responsibilities

| File | Role |
|---|---|
| `js/app.js` | Main orchestrator — validates keys, fetches all data, calls process pipeline, triggers render |
| `js/api.js` | Finnhub + Twelve Data fetchers, batching, rate limiting, `fetchWithTimeout` |
| `js/config.js` | All thresholds, periods, localStorage key names, position sizing tiers |
| `js/technicals.js` | Pure math functions (`calc*`) — RSI, ATR, EMA, AVWAP, POC, ADX, BB, Fibonacci, MACD, OBV, RS vs SPY |
| `js/structure.js` | Market structure (HH/HL vs LH/LL), liquidity sweep detection, Fair Value Gap detection |
| `js/scoring.js` | `calcCompositeScore()` — assembles score from technicals/fundamentals/sentiment + penalties, sets `setup_type` and `position_sizing_pct` |
| `js/ui.js` | DOM rendering, color/format helpers, tooltip generation. Holds `_marketData` and `_chartData` module state. Renders three sections: dashboard table, Entry Cards (`renderEntryCards`), DCA Advisor (`renderDCAAdvisor`) |
| `js/charts.js` | Lightweight Charts integration — candlestick, EMA/AVWAP/POC overlays, sweep & FVG markers, screenshot |

### Data Flow

```
app.js: validate keys
  → fetch OHLCV batch (Twelve Data, 8 symbols/batch, 60s cooldown between batches)
  → fetch sector ETFs in parallel
  → fetch Finnhub per ticker (batches of 6, 60 req/min limit)
  → per ticker: technicals.js → structure.js → scoring.js → store in marketData
  → cache marketData + chartData to localStorage (24h TTL)
  → ui.renderDashboard()
```

### State

- `localStorage`: `finnhub_api_key`, `twelve_data_api_key`, `watchlist_tickers`, `dashboard_cache` (with timestamp)
- `ui.js` module-level: `_marketData` (ticker → all computed fields), `_chartData` (ticker → OHLCV arrays)

### Key `marketData` Fields Per Ticker

`price`, `rsi_daily`, `atr_pct`, `ema50`, `ema200`, `avwap_5d/14d/30d/6m`, `poc_5d/14d`, `adx`, `bb_bandwidth`, `fib_position`, `structure`, `sweep`, `fvg`, `rs_spy_30d`, `macd_value`, `obv_*`, `pe_trailing`, `pe_forward`, `eps_growth_yoy`, `days_to_earnings`, `analyst_rating`, `sector_trend`, `composite_score`, `setup_type`, `position_sizing_pct`, `confluence_reasons`

---

## Common Change Patterns

**Add or adjust a scoring threshold** → `js/config.js` (all numeric thresholds live here: `ENTRY_SCORE_THRESHOLD`, `POSITION_SIZE_TIERS`, ATR limits, RSI zones)

**Add a new indicator** → `js/technicals.js` (add `calcXxx()` pure function) → call it in `app.js` `processTicker()` → store result on `marketData` object

**Change scoring weights or add a scoring condition** → `js/scoring.js` `calcCompositeScore()`

**Fetch new data from an API** → `js/api.js` (add fetch function) → call from `app.js` `processTicker()`

**Change what displays in the dashboard table** → `js/ui.js` `renderDashboard()`

**Adjust DCA Advisor filters** → `js/config.js` (`DCA_MAX_ATR_PCT`, `DCA_RSI_MIN`) + `js/ui.js` `renderDCAAdvisor()`

**Change chart display window** → `js/config.js` `CHART_DAYS` (default 90 trading days)

---

## Rate Limiting

- **Twelve Data** free tier: 8 credits/min → app batches 8 symbols at a time with an automatic 65-second cooldown between batches
- **Finnhub** free tier: 60 req/min → app batches 6 tickers at a time
- `fetchWithTimeout`: 15s default, 30s for Twelve Data (larger payloads)
- HTTP 429 → throws with message "rate limit exceeded" — caught in `app.js` with user-visible error

---

# context-mode — MANDATORY routing rules

You have context-mode MCP tools available. These rules are NOT optional — they protect your context window from flooding. A single unrouted command can dump 56 KB into context and waste the entire session.

## BLOCKED commands — do NOT attempt these

### curl / wget — BLOCKED
Any Bash command containing `curl` or `wget` is intercepted and replaced with an error message. Do NOT retry.
Instead use:
- `ctx_fetch_and_index(url, source)` to fetch and index web pages
- `ctx_execute(language: "javascript", code: "const r = await fetch(...)")` to run HTTP calls in sandbox

### Inline HTTP — BLOCKED
Any Bash command containing `fetch('http`, `requests.get(`, `requests.post(`, `http.get(`, or `http.request(` is intercepted and replaced with an error message. Do NOT retry with Bash.
Instead use:
- `ctx_execute(language, code)` to run HTTP calls in sandbox — only stdout enters context

### WebFetch — BLOCKED
WebFetch calls are denied entirely. The URL is extracted and you are told to use `ctx_fetch_and_index` instead.
Instead use:
- `ctx_fetch_and_index(url, source)` then `ctx_search(queries)` to query the indexed content

## REDIRECTED tools — use sandbox equivalents

### Bash (>20 lines output)
Bash is ONLY for: `git`, `mkdir`, `rm`, `mv`, `cd`, `ls`, `npm install`, `pip install`, and other short-output commands.
For everything else, use:
- `ctx_batch_execute(commands, queries)` — run multiple commands + search in ONE call
- `ctx_execute(language: "shell", code: "...")` — run in sandbox, only stdout enters context

### Read (for analysis)
If you are reading a file to **Edit** it → Read is correct (Edit needs content in context).
If you are reading to **analyze, explore, or summarize** → use `ctx_execute_file(path, language, code)` instead. Only your printed summary enters context. The raw file content stays in the sandbox.

### Grep (large results)
Grep results can flood context. Use `ctx_execute(language: "shell", code: "grep ...")` to run searches in sandbox. Only your printed summary enters context.

## Tool selection hierarchy

1. **GATHER**: `ctx_batch_execute(commands, queries)` — Primary tool. Runs all commands, auto-indexes output, returns search results. ONE call replaces 30+ individual calls.
2. **FOLLOW-UP**: `ctx_search(queries: ["q1", "q2", ...])` — Query indexed content. Pass ALL questions as array in ONE call.
3. **PROCESSING**: `ctx_execute(language, code)` | `ctx_execute_file(path, language, code)` — Sandbox execution. Only stdout enters context.
4. **WEB**: `ctx_fetch_and_index(url, source)` then `ctx_search(queries)` — Fetch, chunk, index, query. Raw HTML never enters context.
5. **INDEX**: `ctx_index(content, source)` — Store content in FTS5 knowledge base for later search.

## Subagent routing

When spawning subagents (Agent/Task tool), the routing block is automatically injected into their prompt. Bash-type subagents are upgraded to general-purpose so they have access to MCP tools. You do NOT need to manually instruct subagents about context-mode.

## Output constraints

- Keep responses under 500 words.
- Write artifacts (code, configs, PRDs) to FILES — never return them as inline text. Return only: file path + 1-line description.
- When indexing content, use descriptive source labels so others can `ctx_search(source: "label")` later.

## ctx commands

| Command | Action |
|---------|--------|
| `ctx stats` | Call the `ctx_stats` MCP tool and display the full output verbatim |
| `ctx doctor` | Call the `ctx_doctor` MCP tool, run the returned shell command, display as checklist |
| `ctx upgrade` | Call the `ctx_upgrade` MCP tool, run the returned shell command, display as checklist |
