# Product Requirements Document — XTB Stock Analysis Dashboard v4
**Type:** Detailed / As-Built Reference
**Version:** 4.0.0
**Status:** Implemented

---

## 1. Product Overview

A fully client-side stock analysis dashboard that helps retail traders identify high-probability entry setups across a personal watchlist. No server, no database, no build step. All computation runs in the browser using two free-tier APIs. The tool scores each stock 0–100 and classifies it as LONG, SHORT, or WAIT based on a weighted technical and fundamental ruleset.

**Target user:** Retail trader with a watchlist of 6–20 stocks who wants a morning briefing before the market opens — which stocks are actionable, what size to take, and where to place stops.

**Core constraint:** Must run from a static file server (or locally via `python -m http.server`). No login, no backend, no cost beyond free API tiers.

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Markup / Layout | HTML5 + Tailwind CSS v3 (CDN) |
| Logic | Vanilla ES6 JavaScript (module pattern via `const Obj = {}`) |
| Charts | Lightweight Charts v4.1.3 (CDN) |
| Data — quotes & fundamentals | Finnhub REST API (free tier: 60 req/min) |
| Data — OHLCV history | Twelve Data REST API (free tier: 8 credits/min) |
| Persistence | `localStorage` (API keys, watchlist, 24h data cache) |
| Hosting | Any static file server; deployed on Vercel |

---

## 3. File Architecture

```
index.html          Main shell — Tailwind config, all CSS, all markup, tooltip JS
js/
  config.js         All numeric thresholds + localStorage helpers + cache helpers
  api.js            Finnhub + Twelve Data fetchers, rate limiting, fetchWithTimeout
  technicals.js     Pure calc* functions — no DOM, no side effects
  structure.js      Market structure, liquidity sweep, Fair Value Gap
  scoring.js        calcCompositeScore(), calcScoreBreakdown(), getConfluenceSetup()
  app.js            Orchestrator — loadDashboard(), refreshData(), processTicker()
  ui.js             All DOM rendering — dashboard, entry cards, DCA advisor, overlays
  charts.js         Lightweight Charts integration, chart toggle + cleanup
```

---

## 4. Data Pipeline

```
Page load
  → loadDashboard()
      → cache exists AND age < 24h?
            YES → renderDashboard() from cache, return
            NO  → refreshData()
                    → validateApiKey() [both keys]
                    → fetchAllOHLCV(tickers) [Twelve Data, 8/batch, 65s cooldown]
                    → fetchSectorETFs(sectors) [parallel]
                    → fetchTickerData(ticker) × N [Finnhub, 6/batch, 1.5s gap]
                    → processTicker() × N
                        → technicals.js → structure.js → scoring.js
                    → setCache(marketData, chartData)
                    → renderDashboard()
```

### Rate Limiting
- **Twelve Data:** 8 symbols per batch request, 65-second cooldown between batches
- **Finnhub:** 6 tickers per batch, 1.5-second gap between batches, 60 req/min ceiling
- **Timeouts:** `fetchWithTimeout` — 15s default, 30s for Twelve Data (larger payloads)
- **HTTP 429:** Throws `"rate limit exceeded"` — caught in app.js with user-visible error

---

## 5. Indicators Computed Per Ticker

| Indicator | Source | Period / Config |
|---|---|---|
| RSI | `calcRSI(closes)` | 14-day |
| ATR + ATR% | `calcATR(h,l,c)` | 14-day |
| EMA50 + EMA200 | `calcEMASeries(closes, n)` | 50 / 200 |
| Trend | `calcTrend(closes, ema50s, ema200s)` | Bullish / Neutral / Bearish |
| AVWAP | `calcAVWAP(c,v,days)` | 5d / 14d / 30d / 6m (126d) |
| POC | `calcPOC(c,v,days)` | 5d / 14d, 20 bins |
| ADX | `calcADX(h,l,c)` | 14-day |
| BB Bandwidth | `calcBBBandwidth(closes)` | 20-day, 2σ |
| Fibonacci | `calcFibonacci(h,l,c)` | 126-day swing, levels 0.236/0.382/0.5/0.618/0.786 |
| Market Structure | `calcStructure(highs, lows)` | 14-bar pivots, HH/HL vs LH/LL |
| Liquidity Sweep | `calcSweep(h,l,c)` | BUY_SWP / SELL_SWP / NONE |
| Fair Value Gap | `calcFVG(h,l,c)` | BULL / BEAR / null + distance% + fill% |
| RS vs SPY | `calcRSvsSPY(closes, spyCloses)` | 30-day |
| Volume Ratio | `calcVolumeRatio(volumes)` | Recent vs average |
| MACD | `calcMACD(closes)` | 12/26/9 EMA |
| OBV Direction | `calcOBVDirection(c,v,n)` | 5d / 14d / 30d (positive/negative/flat) |
| Pivot R1/S1 | `calcPivots(h,l,c)` | Standard floor pivot |
| 52W Position | `calc52WPosition(price,h52,l52)` | % in 52-week range |
| % Below ATH | `calcPctBelowATH(price,h52)` | % below 52-week high |

### Fundamentals (from Finnhub)
`pe_trailing`, `pe_forward`, `dividend_yield`, `market_cap`, `beta`, `earnings_date`, `days_to_earnings`, `eps_growth_yoy`, `short_ratio`, `analyst_rating`, `price_target_upside`

---

## 6. Scoring System

### Composite Score (`calcCompositeScore`)
Produces a 0–100 score. Points are additive; penalties are subtractive. Final score is clamped to [0, 100].

**Positive signals (examples):**
- RSI 30–45: +12 pts (oversold entry zone)
- Price > EMA50 and EMA50 > EMA200: +10 pts
- Bullish structure (HH/HL): +10 pts
- AVWAP support (price above AVWAP30d): +8 pts
- Bullish sweep (buy-side liquidity taken): +8 pts
- Bullish FVG present: +8 pts
- MACD bullish crossover: +6 pts
- RS vs SPY > 0 (30d outperformance): +6 pts
- ADX > 25 (trending): +5 pts
- OBV rising (30d): +5 pts
- Analyst Buy rating: +5 pts
- EPS growth YoY positive: +4 pts
- Earnings > 21 days away: +3 pts
- (Mirror negatives for short setups)

**Penalties:**
- Earnings in < 7 days: −15 pts
- Extreme ATR > 5%: −10 pts
- ADX < 15 (no trend): −5 pts
- Bearish sector trend: −5 pts

### Score Breakdown (`calcScoreBreakdown`)
Returns categorised sub-scores: `technicals`, `structure`, `fundamentals`, `sentiment`. Displayed in the score tooltip on hover.

### Confluence Setup (`getConfluenceSetup`)
Separately evaluates 6 LONG criteria and 6 SHORT criteria. Returns:
- `setup_type`: LONG / SHORT / WAIT
- `criteria_met` / `total_criteria`
- `sizing_pct`: Position sizing as % of portfolio
- `long_met[]` / `long_miss[]` / `short_met[]` / `short_miss[]` — shown in setup tooltip
- Hard WAIT blockers: earnings < 7d, extreme volatility, ADX < 15

### Position Sizing Tiers (from `CONFIG.POSITION_SIZE_TIERS`)
| Score | Size |
|---|---|
| ≥ 95 | 15% |
| ≥ 90 | 12% |
| ≥ 85 | 10% |
| ≥ 80 | 8% |
| ≥ 75 | 5% |

---

## 7. UI Components

### Primary Table
10 columns: Symbol · Price · 1D% · Score (badge with hover tooltip) · Setup (badge with criteria tooltip) · RSI · Trend · Structure · Earn · Rating. Clicking a row expands the chart panel below it.

### Detail Table
~20 columns of technical/fundamental data: ATR% · EMA50/200 · AVWAP30d · POC14d · ADX · BB BW · FVG · Fib · RS SPY · MACD · OBV · PE · EPS · Short Ratio · Price Target · Pivot R1/S1. Wrapped in `overflow-x: auto` for mobile.

### Chart Panel (per ticker, toggled on row click)
- **Price chart:** Candlesticks + EMA50 (blue) + EMA200 (orange) + AVWAP30d (purple dashed) + POC14d (gray dashed) + Fibonacci levels (5 colors) + Sweep/FVG markers. OHLCV crosshair legend top-left. Save PNG button.
- **Volume + RSI chart:** Volume histogram (green/red) + RSI line + OB/OS reference lines (70/30). Time-scale synced to price chart.
- Heights: 450px / 130px desktop; 260px / 100px mobile.
- Resize handler: Reflows on `window.resize`, cleans up on close.

### Entry Cards
Shown for tickers with `composite_score >= 75`. Per card: ticker, score, setup type, entry price (uses FVG level if within 1.5%), SL (entry − 1.5× ATR), TP1 (entry + 2.0× ATR), TP2 (entry + 3.5× ATR), R:R ratios, position size %, earnings warning, 5-step execution checklist.

### DCA Advisor
Shown for tickers below entry threshold but with RSI in [DCA_RSI_MIN, DCA_RSI_MAX] AND ATR% ≤ DCA_MAX_ATR_PCT. Per card: accumulation zone (price ± ATR), 5 tranche levels, SL (90% of zone low), TP (105% of zone high), OBV and sector warnings.

### Setup/Settings Overlays
- **Setup:** First-visit overlay, collects Finnhub key, Twelve Data key, watchlist tickers
- **Settings:** Same fields, editable any time via ⚙ button

### Header
Title, timestamp, VIX link (Yahoo Finance), Fear & Greed link (CNN), ticker count, Refresh, Settings, version badge.

---

## 8. Configuration Reference (`js/config.js`)

```js
ENTRY_SCORE_THRESHOLD: 75      // minimum score for entry cards
CACHE_TTL_HOURS: 24            // cache age before forced refresh
OHLCV_DAYS: 504                // ~2 years of history fetched
CHART_DAYS: 90                 // trading days shown in chart
DCA_MAX_ATR_PCT: 2.0           // max volatility for DCA candidates
DCA_RSI_MIN: 35                // RSI floor for DCA candidates
DCA_RSI_MAX: 65                // RSI ceiling for DCA candidates
```

---

## 9. Responsive Behaviour

| Breakpoint | Behaviour |
|---|---|
| ≥ 640px (sm) | Full desktop layout |
| < 640px | Header wraps to 2 lines · badges shrink · grids go single-column · tooltips wrap text · chart panel capped at 500px · touch-tap tooltips |

---

## 10. Known Limitations

- CNN Fear & Greed API is blocked by CORS in browser — replaced with direct link
- VIX requires external link (Yahoo Finance) rather than live badge
- Free Twelve Data tier limits to 8 symbols/min — large watchlists are slow
- No alerts, no watchlist history, no portfolio P&L tracking
- No backtesting; scoring weights are heuristic, not statistically validated
- All data is pull-on-demand; no WebSocket real-time streaming
