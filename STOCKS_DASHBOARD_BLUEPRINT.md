# S&P 500 Dashboard — Best Practices Blueprint

Adapted from Pioniex Futures Monitor v4.5. Each concept maps a proven crypto-futures pattern to its equity equivalent. Pick what fits your existing board and ignore what doesn't apply.

---

## 1. Fast Decision Table

**Purpose:** Instant daily scan — one row per ticker, all critical info visible without clicking.

### Column Set

| Column | Crypto original | Stocks equivalent | Notes |
|--------|----------------|-------------------|-------|
| Ticker | Symbol + exchange badge | Symbol + sector badge | e.g. AAPL · Tech |
| Price | Mark price | Last close | |
| Δ% 1d | Δ% 24h | Daily % change | Green >+0.5%, Red <−0.5% |
| Δ% 5d | — | Weekly % change | Useful for swing context |
| Score | Composite 0–10 badge | Same (see Section 2) | s-high ≥8, s-mid 6–7.9, s-low <6 |
| Setup | LONG X/4 · SHORT X/4 | LONG X/4 · SHORT X/4 | % confluence met |
| RSI | 4H RSI(14) | Daily RSI(14) | Green <30, Red >70 |
| Volume | Volume spike flag | Volume vs 20d avg | ≥2× avg = spike |
| Short Int% | OI% 7d change | Short interest % of float | Rising SI + buy flow = squeeze risk |
| Earnings | Funding rate | Days to next earnings | Red if ≤5 days |
| Struct Daily | Structure 4H | Structure Daily | Bullish / Bearish / Neutral |
| Struct Weekly | Structure 30d | Structure Weekly | Macro context |
| Rec | calcRecommendation() | Same logic (see Section 3) | STRONG LONG / LONG / WAIT / SHORT |

### Key Design Rules
- Keep it scannable: max 12 columns visible by default, hide others behind "Show All" toggle
- Score badge color = instant priority signal (green = high, yellow = mid, red = low)
- Expandable row on click: show full indicator breakdown inline

---

## 2. Composite Scoring Engine (0–10)

**Activation threshold:** Score ≥ 7.5 → generate entry parameters.
**Direction:** determined by macro conditions (see Section 3).

### Weight Table

| Component | Max Points | Stocks Conditions |
|-----------|-----------|-------------------|
| **Trend Macro** | +2.0 | Full (+2.0): 3 of 4 conditions met → [price > SMA50, price > SMA200, sector ETF bullish structure, OBV30d positive] · Partial (+0.8): 2 of 4 |
| **Trend Swing** | +1.5 | Price > AVWAP5d + OBV5d rising + daily buy flow positive |
| **Pressure** | +2.0 | Full (+2.0): volume flow >+5% + short interest declining + OBV5d positive · Partial (+0.8): 2 of 3 |
| **OBV Quality** | +1.0 | All 3 timeframes aligned: OBV5d, OBV14d, OBV30d all positive (LONG) or all negative (SHORT) · Bounce/correction signals = +0.5 |
| **Setup** | +2.0 | Best (+2.0): support sweep + buy flow + SI declining · Good (+1.0): POC5d confluence · Moderate (+0.5): POC14d + daily structure alignment |
| **EMA Alignment** | +0.5 | EMA50 > EMA200 for LONG · EMA50 < EMA200 for SHORT (daily chart) |
| **FVG Proximity** | +0.5 | Price near an intact daily Fair Value Gap, structure confirms direction |
| **POC Confluence** | +0.5 | Volume POC 5d ≈ volume POC 14d within 0.5% |
| **TOTAL** | **10.0** | |

### Penalties

| Condition | Penalty |
|-----------|---------|
| RSI > 75 or < 25 | −0.5 |
| Earnings within 5 days | −0.5 |
| Sector structure conflicts with stock structure | −0.5 |
| High short interest + rising (SHORT squeeze risk on LONG) | −0.5 to −1.0 |

### Score → Status Mapping

| Score | Status | Action |
|-------|--------|--------|
| ≥ 7.5 | ★ ACTIVE | Generate entry card |
| 6.5–7.4 | ⚡ NEAR ACTIVE | Monitor, set alert |
| < 6.5 | Waiting | No action |

---

## 3. Direction Conditions

Derived once per ticker, reused by signal cards, scoring, and entry logic.

```
// Macro (daily/weekly)
bullMac = count of: [price > SMA50, price > SMA200, sector ETF = Bullish, OBV30d > 0]
bearMac = count of: [price < SMA50, price < SMA200, sector ETF = Bearish, OBV30d < 0]
direction = bullMac >= 3 ? "LONG" : bearMac >= 3 ? "SHORT" : null

// Swing (5-day)
sBull = price > AVWAP5d  AND  OBV5d > 0  AND  flow > 0
sBear = price < AVWAP5d  AND  OBV5d < 0  AND  flow < 0
dBull = price < AVWAP5d  AND  OBV5d > 0    // divergence: recovery signal
dBear = price > AVWAP5d  AND  OBV5d < 0    // divergence: weakness signal

// Pressure
buyP = flow > +5%  AND  shortInterest declining  AND  OBV5d > 0
selP = flow < −5%  AND  shortInterest rising      AND  OBV5d < 0
sqR  = flow > +5%  AND  shortInterest rising      // Squeeze risk: shorts trapped

// OBV multi-timeframe alignment
aAcc = OBV5d > 0  AND  OBV14d > 0  AND  OBV30d > 0   // full accumulation
aDis = OBV5d < 0  AND  OBV14d < 0  AND  OBV30d < 0   // full distribution
bnce = OBV30d < 0  AND  OBV14d < 0  AND  OBV5d > 0   // bounce in downtrend
corr = OBV30d > 0  AND  OBV14d > 0  AND  OBV5d < 0   // pullback in uptrend
```

---

## 4. Entry Decision Card

Generated when score ≥ 7.5. One card per qualifying ticker.

### Fields

```
┌─────────────────────────────────────────────────────────┐
│  AAPL — LONG  |  Score: 8.42/10                         │
├─────────────────────────────────────────────────────────┤
│  Entry (FVG/POC/Price)      →  $189.40                  │
│  Stop Loss (1.5 × ATR)      →  $185.20   ATR = $2.80    │
│  Take Profit 1  (R:R 1:2)   →  $194.80                  │
│  Take Profit 2  (R:R 1:3.5) →  $199.20                  │
│  Position Size              →  8% of portfolio           │
│  R:R TP1                    →  1 : 2.0                  │
│  R:R TP2                    →  1 : 3.5                  │
│  Trail trigger              →  $194.80 (at TP1)         │
│  Trail offset               →  $1.40  (0.5 × ATR)       │
├─────────────────────────────────────────────────────────┤
│  Execution Rules                                        │
│  1. Wait for daily close above entry price              │
│  2. Hard SL at $185.20 — ATR(14d) = $2.80              │
│  3. At TP1: close 50%, move SL to breakeven             │
│  4. Trail remaining 50% with $1.40 offset toward TP2   │
│  5. Cancel if OBV flips negative or sector breaks down  │
└─────────────────────────────────────────────────────────┘
```

### Position Size Tiers (no leverage — use portfolio %)

| Score | Position Size |
|-------|--------------|
| ≥ 9.5 | 15% |
| ≥ 9.0 | 12% |
| ≥ 8.5 | 10% |
| ≥ 8.0 | 8% |
| ≥ 7.5 | 5% |

Adjust tiers to your own risk tolerance. These are starting defaults.

### ATR-Based SL/TP Formulas

```
slDist  = ATR(14, daily) × 1.5
entry   = FVG_TOP (if price within 1.5% of bull FVG) else current price
sl      = entry − slDist              (LONG)  |  entry + slDist         (SHORT)
tp1     = entry + slDist × 2.0        (LONG)  |  entry − slDist × 2.0   (SHORT)
tp2     = entry + slDist × 3.5        (LONG)  |  entry − slDist × 3.5   (SHORT)
trail   = 0.5 × ATR trailing offset after TP1 hit
```

---

## 5. Signal Cards (10 Categories)

One card per ticker with 10 signal rows. Each row: `[STATUS, color-class, reason text]`.

| # | Signal | Futures original | Stocks equivalent | Statuses |
|---|--------|-----------------|-------------------|----------|
| 1 | **Trend Macro** | AVWAP14d/30d + CVD30d + Structure30d | SMA50/200 + Sector ETF + OBV30d | BULL · BEAR · NEUTRAL |
| 2 | **Trend Swing** | AVWAP5d + CVD5d + Flow | AVWAP5d + OBV5d + daily flow | BULLISH · BEARISH · DIV BULL · DIV BEAR |
| 3 | **Pressure** | Flow + OI + CVD5d | Volume flow + Short interest + OBV5d | BUY STRONG · SELL STRONG · SQUEEZE RISK · SHORT ACTIVE · BALANCED |
| 4 | **OBV Quality** | CVD 5d/14d/30d alignment | OBV 5d/14d/30d alignment | FULL ACCUM · FULL DISTRIB · BOUNCE · PULLBACK · MIXED |
| 5 | **Setup** | Sweep + POC + FVG | Support sweep + POC + FVG + earnings catalyst | LONG VALID · SHORT VALID · LONG@POC · SHORT@POC · WAIT |
| 6 | **Risk** | RSI extreme + OI + structure conflict | RSI extreme + earnings window + sector conflict | HIGH · MEDIUM · LOW |
| 7 | **DCA Suitability** | Grid Bot (lateral CVD) | OBV lateral + low ATR% + RSI 40–60 | RECOMMENDED · POSSIBLE · AVOID |
| 8 | **FVG** | Fair Value Gap proximity (4H) | Fair Value Gap proximity (daily) | INSIDE · NEAR · FAR · ★ (structure confirmed) |
| 9 | **EMA Trend** | EMA50/200 on 4H | EMA50/200 on daily | BULL · BEAR · BULL PULLBACK · BEAR BOUNCE · NEUTRAL |
| 10 | **Volume Spike** | Current vs 20-candle avg (≥2×) | Current vs 20-day avg (≥2×) | BULL SPIKE · BEAR SPIKE · ELEVATED · NORMAL |

### Color Coding Convention
- `bull` class → green text
- `bear` class → red text
- `neutral` class → muted/gray text
- `warn` class → yellow/amber text (risk, earnings, squeeze)

---

## 6. Score Analysis Table

Ranked table of all tickers by score. Quick overview before diving into entry cards.

### Columns

| Column | Content |
|--------|---------|
| **Asset** | Ticker + sector |
| **Score / 10** | Numeric, colored badge (s-high / s-mid / s-low) |
| **Bar** | Visual progress bar: width = score/10 × 100% |
| **Direction** | LONG (green) · SHORT (red) · — |
| **Position Size** | % of portfolio (from size tiers) or — |
| **Status** | ★ ACTIVE · ⚡ NEAR ACTIVE · waiting for confluence |

### Score Bar Colors
```
score >= 8.0  →  #50fa7b  (green)
score >= 6.0  →  #f1fa8c  (yellow)
score <  6.0  →  #ff5555  (red)
```

---

## 7. DCA / Accumulation Advisor

Replaces the Grid Bot Advisor. For stocks that score below entry threshold but are good for accumulation.

### Per-Ticker Card

```
┌─────────────────────────────────────────────────────────┐
│  MSFT  |  $415.20  |  Profile: Stable                   │
│  Viability: ✅ ACCUMULATE — OBV lateral, RSI neutral    │
├─────────────────────────────────────────────────────────┤
│  Accumulation Zone    →  $398 – $425                    │
│  Tranches             →  5 equal buys                   │
│  Cost per tranche     →  $100 (of $500 capital)         │
│  Avg cost target      →  ~$411.50                       │
│  Stop Loss            →  $358  (10% below lower bound)  │
│  Take Profit          →  $446  (5% above upper bound)   │
│  Worst-case drawdown  →  −14.3% if SL hits              │
├─────────────────────────────────────────────────────────┤
│  ⚠ Earnings in 8 days — pause new entries              │
├─────────────────────────────────────────────────────────┤
│  Setup Checklist                                        │
│  [ ] Set first buy limit: $425                          │
│  [ ] Set 4 more tranches at: $418 / $411 / $404 / $398  │
│  [ ] Set hard stop: $358                                │
│  [ ] Set take profit: $446                              │
└─────────────────────────────────────────────────────────┘
```

### Viability Logic

| Condition | Verdict |
|-----------|---------|
| OBV lateral + RSI 40–60 + ATR% ≤ 2% | ✅ ACCUMULATE |
| OBV weakly trending + RSI 35–65 | ⚠ POSSIBLE |
| Strong trend (OBV directional) OR RSI > 70 OR ATR% > 4% | ❌ AVOID |

### Dynamic Warnings
- ATR% > 4% → high volatility, reduce tranche size
- RSI > 70 → overbought, wait for pullback before first tranche
- Earnings within 2 weeks → pause, high IV risk
- Sector ETF breakdown → reassess thesis

---

## 8. Indicator Reference (Stocks Adaptations)

| Indicator | Timeframe | Notes |
|-----------|-----------|-------|
| RSI | Daily × 60 bars | 14-period; >70 overbought, <30 oversold |
| ATR | Daily | 14-period; used for all SL/TP sizing |
| EMA 50 / 200 | Daily | Golden/death cross; same logic as crypto |
| SMA 50 / 200 | Daily | Alternative to EMA for macro trend; use one consistently |
| POC + AVWAP | 5d / 14d / 30d | Volume-profile POC + anchored VWAP from session opens |
| OBV | 5d / 14d / 30d | Cumulative volume direction; proxy for CVD |
| Market Structure | Daily / Weekly | HH+HL = Bullish · LH+LL = Bearish; same logic |
| FVG | Last 100 × daily candles | Imbalance zones; same detection as crypto |
| Volume Spike | Daily | Current vs 20-day avg; ≥ 2× = spike |
| Short Interest % | Weekly | From broker/data feed; rising SI + buy flow = squeeze setup |
| Sector ETF Trend | Weekly | SPY sector ETFs (XLK, XLF, XLE…) as macro filter |
| Earnings Date | Calendar | Avoid new entries within 5 days; flag in Fast Decision table |

---

## 9. Dashboard Section Order (Recommended Layout)

```
[TOPBAR]
  Title · Version badge · Last update · Next refresh countdown · Refresh button · Config

[SECTION 1: FAST DECISION TABLE]
  Compact overview — all tickers, key signals at a glance
  Expandable rows for breakdown

[SECTION 2: FULL METRICS TABLE]  ← collapsible, defaults to 12 cols
  All raw indicator values per ticker
  "Show All" toggle for extended columns

[SECTION 3: SIGNAL CARDS]
  10-signal card per ticker (one card = one accordion/expandable)

[SECTION 4: SCORE ANALYSIS]
  Ranked table: Asset | Score | Bar | Direction | Size% | Status
  Expandable score breakdown rows

[SECTION 5: ACTIVE ENTRY CARDS]  ← only shown when score ≥ 7.5
  Full entry card per qualifying ticker
  Entry / SL / TP1 / TP2 / Size% / R:R / Execution checklist

[SECTION 6: DCA ADVISOR]
  Per-ticker accumulation cards (OBV-lateral tickers)
  Viability badge · Tranche plan · SL/TP · Warnings · Checklist

[SECTION 7: REFERENCE GUIDE]  ← collapsible
  Indicator glossary (all terms defined)
  Score weights table
  Color legend
```

---

## 10. Implementation Notes

- **Single config object** — keep all thresholds (score activation, RSI levels, ATR multipliers, position size tiers) in one `CFG` object. Never hardcode numbers inline.
- **Pure calculation functions** — separate indicator math from rendering. No DOM access inside calculation functions.
- **Derive conditions once** — compute all boolean conditions (`bullMac`, `sBull`, `buyP`, etc.) in a single `deriveConditions()` helper, reused by both signal cards and scoring. Avoid duplicate logic.
- **Graceful fallback** — if a data source fails for a ticker, show `—` rather than crashing the row.
- **Version badge** — single `APP_VERSION` constant, rendered in topbar on init.
- **Auto-refresh** — 20-minute interval with visible countdown; allow manual refresh button.
- **Earnings data** — most critical stock-specific addition. Fetch from a calendar API or hardcode weekly. Flag in Fast Decision table and suppress entry signals within 5 days.
