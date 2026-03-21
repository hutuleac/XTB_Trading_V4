# XTB Trading Dashboard — v4.0.0

> Real-time S&P 500 stock analysis in your browser. 20+ technical indicators, composite scoring, and actionable LONG / SHORT / WAIT signals — no server, no build step.

![Version](https://img.shields.io/badge/version-4.0.0-blue) ![No Build](https://img.shields.io/badge/build-none-brightgreen) ![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## Overview

XTB Trading Dashboard is a fully client-side stock analysis tool for active traders. It fetches live price data and fundamentals from two free APIs, calculates a suite of technical and fundamental indicators entirely in the browser, and surfaces a **Composite Score (0–100)** alongside a clear **LONG / SHORT / WAIT** signal for every ticker on your watchlist.

No server. No build step. No data ever leaves your browser except to the two API providers.

---

## Features at a Glance

| Category | What's Included |
|---|---|
| **20+ Indicators** | RSI, EMA 50/200, ATR, ADX, AVWAP, POC, Bollinger Bandwidth, Fibonacci, Market Structure, Liquidity Sweeps, FVG, RS vs SPY |
| **Composite Score** | Weighted 0–100 across technicals, fundamentals, and sentiment |
| **Signal Detection** | LONG / SHORT / WAIT with tiered position sizing |
| **Interactive Charts** | Candlestick + EMA/AVWAP/POC/Fibonacci overlays, RSI sub-chart, sweep & FVG markers |
| **Watchlist** | Add/remove tickers live, persisted in localStorage |
| **Zero Infrastructure** | Pure HTML + JavaScript, CDN-loaded libraries |

---

## Data Sources

The dashboard uses two APIs. Both offer free tiers sufficient for personal use.

### Finnhub
**Purpose:** Live quotes, company profile, earnings calendar, analyst ratings, EPS data.

- Free tier: 60 calls/minute
- Key stored in browser `localStorage` as `finnhub_api_key`
- Sign up: [finnhub.io](https://finnhub.io)

### Twelve Data
**Purpose:** OHLCV historical time series — 504 trading days (~2 years) per ticker.

- Free tier: 8 credits/minute (dashboard auto-throttles with a 65-second cooldown between batches)
- Key stored in browser `localStorage` as `twelve_data_api_key`
- Sign up: [twelvedata.com](https://twelvedata.com)

**Both keys are stored only in your browser's `localStorage`. They are never sent to any server other than the respective API endpoints.**

---

## API Key Setup

1. Sign up at [finnhub.io](https://finnhub.io) and copy your API key
2. Sign up at [twelvedata.com](https://twelvedata.com) and copy your API key
3. Open the dashboard — the setup overlay appears on first launch
4. Enter both keys and your initial watchlist tickers
5. Keys are saved automatically and survive page reloads

---

## Technical Indicators

All indicators are calculated client-side from raw OHLCV data. No pre-calculated values are fetched from any API.

| Indicator | Period / Config | Description |
|---|---|---|
| **RSI** | 14 | Relative Strength Index — momentum oscillator (0–100). Oversold <30, overbought >70. Entry zones differ per setup direction. |
| **EMA 50** | 50-day | Short-term trend baseline. Price above = near-term bullish bias. |
| **EMA 200** | 200-day | Long-term trend filter. Price above = macro bullish; below = macro bearish. One of the highest-weight scoring inputs. |
| **ATR** | 14 | Average True Range — absolute volatility in price units. Used for risk sizing. |
| **ATR%** | ATR / Price | Normalised volatility. Above 5% triggers a penalty and WAIT block. |
| **ADX** | 14 | Average Directional Index — trend strength (0–100). Below 20 = weak trend; above 25 = strong trend. |
| **Bollinger Band Bandwidth** | 20, 2σ | Measures squeeze conditions. Below 1.5% triggers a WAIT block (breakout pending, direction unknown). |
| **AVWAP 5d** | 5-day | Anchored VWAP — short-term mean reversion level. |
| **AVWAP 14d** | 14-day | Anchored VWAP — medium swing level. |
| **AVWAP 30d** | 30-day | Anchored VWAP — primary monthly level. Heavily weighted in scoring. |
| **AVWAP 6m** | 6-month | Anchored VWAP — institutional level. |
| **POC** | Volume profile | Point of Control — price level with highest volume over the lookback window. Acts as magnet/support/resistance. |
| **Fibonacci Levels** | 0.236 / 0.382 / 0.5 / 0.618 / 0.786 | Retracement levels from the recent swing high/low. Used to identify value zones. |
| **Market Structure** | Swing pivots | Higher Highs + Higher Lows = bullish; Lower Highs + Lower Lows = bearish. Core direction filter. |
| **Liquidity Sweeps** | Wick analysis | Detects wicks that breach a prior high/low and reverse — signals stop-hunt events. |
| **Fair Value Gaps (FVG)** | 3-candle pattern | Imbalance zones where price moved too fast and left an unfilled gap. Potential re-entry areas. |
| **Relative Strength vs SPY** | 30-day | Compares ticker's 30-day return to SPY's return. Positive = outperforming the market. |
| **Volume Ratio** | 20-day avg | Current session volume divided by 20-day average. > 1.5 = above-average interest. |
| **MACD** | 12/26/9 | Trend-following momentum. Signal line crossovers indicate momentum shifts. |
| **OBV** | Cumulative | On-Balance Volume — confirms or diverges from price trend using volume flow. |

---

## Scoring System (0–100)

Each ticker receives a Composite Score across three weighted categories. Higher scores indicate better confluence of conditions for a trade setup.

### Technical — max 50 points

| Condition | Points |
|---|---|
| RSI in entry zone (35–65 depending on direction) | +10 |
| Market structure bullish/aligned | +15 |
| Price above EMA 200 | +10 |
| Price above AVWAP 30d | +10 |
| Price in Fibonacci value zone (23.6%–61.8%) | +5 |

### Fundamental — max 30 points

| Condition | Points |
|---|---|
| Earnings date > 21 days away (safe zone) | +10 |
| EPS growth YoY > 0% | +10 |
| Forward P/E < Trailing P/E (improving valuation) | +10 |

### Sentiment — max 10 points

| Condition | Points |
|---|---|
| 30-day Relative Strength vs SPY > 0 | +10 |

### Penalties

| Condition | Adjustment |
|---|---|
| Earnings within 7 days | −20 |
| ATR% > 5% (extreme volatility) | −15 |
| Bearish market structure + price below EMA 200 | −15 |

---

## Setup Detection & Signal Interpretation

### LONG Signal

Requires **4 of 6** confluence conditions:

1. Price above EMA 200
2. RSI between 40–65 (not overbought, not deeply oversold)
3. Bullish market structure (HH/HL pattern)
4. Price above AVWAP 30d
5. Price in Fibonacci support zone (value area)
6. Positive RS vs SPY

A LONG signal means the stock has sufficient technical and structural alignment for a potential long entry. It does not mean buy immediately — use your own entry trigger.

### SHORT Signal

Requires **4 of 6** confluence conditions:

1. Price below EMA 200
2. RSI between 35–60 (not oversold, not deeply overbought)
3. Bearish market structure (LH/LL pattern)
4. Price below AVWAP 30d
5. Price in Fibonacci resistance zone
6. Negative RS vs SPY

### WAIT Signal

Automatically assigned when **any** of the following are true, regardless of score:

- Earnings within 10 days (gap risk)
- ATR% > 5% (too volatile to size properly)
- Bollinger squeeze < 1.5% bandwidth (breakout imminent, direction unknown)
- Weak trend: neutral structure + ADX < 20
- Price trapped between EMA 50 and EMA 200 (compression zone)

WAIT does not mean the stock is bad — it means conditions are not clean enough for a rule-based entry. Check again after the blocker resolves.

### Position Sizing Tiers

The score directly maps to suggested position size as a percentage of your normal allocation:

| Score | Suggested Size |
|---|---|
| 95+ | 15% |
| 90–94 | 12% |
| 85–89 | 10% |
| 80–84 | 8% |
| 75–79 | 5% |
| Below 75 | WAIT |

These are relative weights, not fixed dollar amounts. Adjust to your own risk management rules.

---

## Project Structure

```
XTB_Trading_V4/
├── index.html          # Single-page app — all UI and DOM structure
├── ARCHITECTURE.md     # Detailed module design and data flow
├── README.md           # This file
└── js/
    ├── config.js       # Parameters, thresholds, localStorage helpers
    ├── api.js          # Finnhub & Twelve Data fetching, validation, rate limiting
    ├── technicals.js   # Pure math: RSI, ATR, EMA, AVWAP, POC, ADX, BB, Fib, RS, MACD, OBV
    ├── structure.js    # Market structure detection, liquidity sweeps, FVG
    ├── scoring.js      # Composite score calculation + LONG/SHORT/WAIT logic
    ├── charts.js       # Lightweight Charts rendering + screenshot export
    ├── ui.js           # DOM rendering, tooltips, watchlist management
    └── app.js          # Main orchestrator — validate → fetch → process → render
```

**No build. No bundler. No node_modules.** Pure HTML + JavaScript.

External libraries (Tailwind CSS, Lightweight Charts) load from CDN on page open.

---

## Data Flow

```
Page Load
  └── API keys in localStorage?
        ├── NO  → Setup overlay (enter Finnhub key + Twelve Data key + watchlist)
        └── YES → loadDashboard()
                    ├── Validate Finnhub key (test quote request)
                    ├── Fetch SPY benchmark data (Twelve Data)
                    ├── Fetch all watchlist tickers in parallel
                    │     ├── Finnhub: quote, profile, earnings, analyst grades, EPS
                    │     └── Twelve Data: 504-day OHLCV history
                    ├── Calculate 20+ indicators per ticker (client-side)
                    ├── Score each ticker (0–100)
                    ├── Detect LONG / SHORT / WAIT
                    └── Render dashboard rows + interactive charts
```

Non-critical data failures (e.g. analyst grades unavailable) degrade gracefully — the ticker still renders with partial data rather than failing completely.

---

## Configuration Reference

Key parameters in `js/config.js`:

| Parameter | Default | Purpose |
|---|---|---|
| `OHLCV_DAYS` | 504 | Trading days of history fetched (~2 years) |
| `RSI_PERIOD` | 14 | RSI lookback window |
| `EMA_SHORT` | 50 | Short EMA period |
| `EMA_LONG` | 200 | Long EMA period |
| `ATR_PERIOD` | 14 | ATR lookback window |
| `ADX_PERIOD` | 14 | ADX lookback window |
| `AVWAP_WINDOWS` | 5d, 14d, 30d, 6m | AVWAP anchor periods |
| `CHART_DAYS` | 90 | Days displayed in the interactive chart |
| `BB_PERIOD` | 20 | Bollinger Bands period |
| `BB_STD` | 2 | Bollinger Bands standard deviation multiplier |

---

## Versioning

This project uses semantic versioning: `v{major}.{minor}.{patch}`

| Increment | When |
|---|---|
| **Major** | New project generation (V1 → V2 → V3 → V4) |
| **Minor** | New indicator, new feature, significant UI change |
| **Patch** | Bug fix, threshold adjustment, documentation update |

The README version always matches the dashboard version. When the app version is updated in `js/config.js`, the README badge and heading should be updated in the same commit.

**Current version: v4.0.0** (2026-03-21)

---

## Changelog

### v4.0.0 — 2026-03-21
- Migrated data sources from Financial Modeling Prep to **Finnhub + Twelve Data**
- Dual API key setup (Finnhub for fundamentals/quotes, Twelve Data for OHLCV history)
- Automatic rate limit handling for Twelve Data free tier (65-second batch cooldown)
- Versioning scheme established at v4.x.x to match project generation

### v3.2.0 — 2026-03-18
- Configuration parameters consolidated in `config.js`
- localStorage helpers for both API keys

### v3.1.0 — 2026-03-16
- 20+ technical indicators implemented in `technicals.js`
- Composite scoring system with penalties
- LONG / SHORT / WAIT signal detection with position sizing tiers

---

## Disclaimer

> **This tool is for educational and research purposes only.**
>
> It does not constitute financial advice, investment recommendations, or solicitation to buy or sell any securities. Trading involves substantial risk of loss. Always do your own research and consult a qualified financial advisor. The authors are not responsible for any financial losses.
