# Stock Dashboard Live

> A zero-infrastructure stock analysis dashboard — open in your browser, enter an API key, start trading smarter.

![Version](https://img.shields.io/badge/version-3.1.0-blue) ![No Build](https://img.shields.io/badge/build-none-brightgreen) ![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## What It Does

Stock Dashboard Live pulls real-time data from the [Financial Modeling Prep](https://financialmodelingprep.com/) (FMP) API and runs **20+ technical indicators entirely in your browser** — no server, no backend, no build step.

For each stock on your watchlist, it calculates a **Composite Score (0–100)** and automatically generates an actionable **LONG / SHORT / WAIT** signal.

---

## Features

### Live Analysis
| Feature | Details |
|---|---|
| **20+ Technical Indicators** | RSI, ATR, EMA50/200, AVWAP, POC, ADX, Bollinger Bandwidth, Fibonacci, Market Structure, Sweeps, FVG |
| **Composite Score (0–100)** | 9 weighted criteria across technicals, fundamentals, and sentiment |
| **Confluence Setup Detection** | LONG / SHORT / WAIT signals with position sizing (60% or 100%) |
| **Relative Strength** | Benchmarked vs SPY |

### Interactive Charts
- Candlestick chart with EMA50, EMA200, AVWAP, POC, and Fibonacci overlays
- Volume + RSI sub-chart with synchronized time scale
- Live OHLCV crosshair tooltip
- Sweep and FVG markers
- One-click PNG screenshot export

### User Experience
- Dark-themed responsive UI (Tailwind CSS)
- Editable watchlist — add/remove tickers without leaving the page
- API key saved in `localStorage` — never sent anywhere except FMP
- Full error recovery: "Change API Key" and "Retry" buttons
- Loading progress indicators per ticker

---

## Composite Scoring System

Each ticker receives a score out of 100 across three categories:

```
Technical  (max 50 pts)
  +10  RSI in entry zone (30–45)
  +10  Market structure = Bullish
  +10  Price above EMA200
  +10  Price above AVWAP 30d
  +10  Fibonacci value zone (23.6%–61.8%)

Fundamental  (max 30 pts)
  +10  Earnings > 21 days away
  +10  EPS growth > 0%
  +10  Forward P/E < Trailing P/E

Sentiment  (max 10 pts)
  +10  Relative Strength > SPY

Penalties
  -20  Earnings < 7 days
  -15  ATR% > 5%  (extreme volatility)
  -15  Bearish structure + below EMA200
```

### Setup Detection

**LONG** requires 4 of 6:
1. Price > EMA200
2. RSI in 35–55
3. Bullish structure
4. Price > AVWAP 30d
5. Fibonacci value zone
6. ADX favorable

**WAIT** auto-blocks on any of:
- Earnings within 10 days
- ATR% > 5%
- Neutral structure + ADX < 20
- Price trapped between EMA50 and EMA200
- Bollinger squeeze (bandwidth < 1.5%)

---

## Project Structure

```
stock-dashboard-live/
├── index.html          # Single-page app (entry point)
├── ARCHITECTURE.md     # Detailed data flow & API documentation
├── README.md           # This file
└── js/
    ├── config.js       # Parameters, localStorage helpers
    ├── api.js          # FMP Stable API fetching (7 endpoints)
    ├── technicals.js   # Pure math: RSI, ATR, EMA, AVWAP, POC, ADX, BB, Fib, RS
    ├── structure.js    # Market structure, liquidity sweeps, FVG detection
    ├── scoring.js      # Composite score + confluence setup logic
    ├── charts.js       # Lightweight Charts rendering & screenshot
    ├── ui.js           # DOM rendering, tooltips, settings, summaries
    └── app.js          # Main orchestrator — validate → fetch → process → render
```

**No build. No bundler. No node\_modules.** Pure HTML + JavaScript.

---

## Quick Start

### Option 1 — Open Locally

```bash
git clone https://github.com/YOUR_USERNAME/stock-dashboard-live.git
cd stock-dashboard-live

# Serve with Python
python -m http.server 8080

# Or Node.js
npx serve .
```

Open `http://localhost:8080`, enter your FMP API key, done.

> You can also just double-click `index.html` — it works without a server.

### Option 2 — Cloudflare Pages *(Recommended)*

1. Fork this repo
2. Go to [Cloudflare Pages](https://pages.cloudflare.com/) → Connect repo
3. Build output directory: `/` (root) — no build command
4. Deploy → enter your API key → done

### Option 3 — GitHub Pages

1. Fork this repo
2. Settings → Pages → Source: `main` branch, `/ (root)` folder
3. Your dashboard lives at `https://YOUR_USERNAME.github.io/stock-dashboard-live/`

---

## API Key Setup

This dashboard uses the **[Financial Modeling Prep](https://financialmodelingprep.com/)** Stable API.

1. Sign up at financialmodelingprep.com
2. Copy your API key
3. Open the dashboard → paste the key in the setup screen
4. The key is stored only in your browser's `localStorage`

### FMP Endpoints Used

| Endpoint | Purpose |
|---|---|
| `historical-price-eod` | OHLCV data (504 days) — **required** |
| `profile` | Company fundamentals |
| `key-metrics-ttm` | Forward P/E ratio |
| `earnings-calendar` | Days to next earnings |
| `income-statement` | EPS growth |
| `shares-float` | Short interest % |
| `grades-consensus` | Analyst ratings |

All tickers are fetched in parallel. If a non-critical endpoint fails, the ticker is still shown with partial data.

---

## Data Flow

```
Page Load
  └── API key in localStorage?
        ├── NO  → Setup overlay (enter key + watchlist)
        └── YES → loadDashboard()
                    ├── Validate API key
                    ├── Fetch SPY benchmark
                    ├── Fetch all tickers in parallel (7 endpoints each)
                    ├── Calculate 20+ indicators per ticker
                    ├── Score each ticker (0–100)
                    ├── Detect LONG / SHORT / WAIT
                    └── Render dashboard
```

---

## Deployment Notes

| Host | Config |
|---|---|
| **Cloudflare Pages** | Build command: *(none)* · Output: `/` |
| **GitHub Pages** | Source: `main` branch, root folder |
| **Any static host** | Upload all files — no server-side processing needed |

All external libraries (Tailwind CSS, Lightweight Charts) load from CDN.

---

## Disclaimer

> **This tool is for educational and research purposes only.**
>
> It does not constitute financial advice, investment recommendations, or solicitation to buy or sell any securities. Trading involves substantial risk of loss. Always do your own research and consult a qualified financial advisor. The authors are not responsible for any financial losses.
