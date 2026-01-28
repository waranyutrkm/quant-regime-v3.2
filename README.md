# Quant Regime v3.2 — Optimized Market Breadth Search Engine

> Language note:
> - Explanations: Thai
> - Anything “on top” (code/formula/commands): English (copy-ready)

---

## 0) What is this project?

Quant Regime v3.2 is a research/backtest UI that:
1) Pulls daily OHLC data from Binance Futures (top liquid USDT pairs)
2) Builds a **dynamic universe** (rolling list of top-liquidity coins per day)
3) Computes a **Market Breadth** signal using EMA cross logic per coin
4) Smooths breadth with EMA
5) Converts signal → exposure using one of multiple regime models
6) Applies **volatility targeting** to scale exposure
7) Backtests portfolio NAV and produces risk metrics
8) Runs a **grid search** across parameter ranges and ranks results

---

## 1) Repository structure (recommended)

```bash
quant-regime-v3/
├─ src/
│  ├─ App.jsx
│  ├─ main.jsx
│  ├─ index.css
├─ index.html
├─ package.json
└─ README.md
````

---

## 2) Setup & Run locally (Vite)

### 2.1 Create project

```bash
npm create vite@latest quant-regime-v3 -- --template react
cd quant-regime-v3
```

### 2.2 Install dependencies

```bash
npm install
npm install recharts lucide-react
```

### 2.3 Replace files

* Replace `src/App.jsx` with your Quant Regime code
* Ensure `index.html` exists as standard Vite root HTML
* Ensure `src/main.jsx` mounts App

### 2.4 Run dev server

```bash
npm run dev
```

Open the local URL shown in terminal.

---

## 3) Data pipeline (Step-by-step)

### Step 1 — Scan market universe

* API:

```text
GET https://fapi.binance.com/fapi/v1/ticker/24hr
```

* Filter: symbols ending with `"USDT"`
* Sort by: `quoteVolume` descending
* Take top N (default: 80)

**Goal:** Choose the most liquid coins in Futures market.

---

### Step 2 — Download daily OHLC (close + volume proxy)

For each selected symbol, request up to 450 daily bars:

```text
GET https://fapi.binance.com/fapi/v1/klines?symbol=BTCUSDT&interval=1d&limit=450
```

You store for each bar:

* `date = ISO day`
* `close = k[4]`
* `quoteVolume = k[7]`

**Goal:** Build per-symbol time series.

---

### Step 3 — Align all symbols into a unified daily timeline

* Collect all distinct dates across all symbols
* Create a master list of dates (sorted ascending)
* For each date:

  * Create `symbols` dictionary containing any available coin values that day
  * Set `market_proxy` using BTCUSDT close (acts as benchmark / underlying return series)

**Output shape (conceptual):**

```js
{
  date: "2025-01-01",
  market_proxy: 42000.0,
  symbols: {
    BTCUSDT: { close: 42000, quoteVolume: 123456789 },
    ETHUSDT: { close: 2200,  quoteVolume: 987654321 },
    ...
  }
}
```

---

## 4) Core indicators and formulas

### 4.1 EMA (Exponential Moving Average)

EMA is computed as:

```text
alpha = 2 / (period + 1)
EMA[0] = price[0]
EMA[t] = alpha * price[t] + (1 - alpha) * EMA[t-1]
```

#### Example (copy-ready)

```text
period = 3
alpha  = 2/(3+1) = 0.5

prices: [10, 12, 11, 13]

EMA0 = 10
EMA1 = 0.5*12 + 0.5*10 = 11
EMA2 = 0.5*11 + 0.5*11 = 11
EMA3 = 0.5*13 + 0.5*11 = 12
```

---

### 4.2 Breadth signal (Market Breadth)

For each day `t`:

1. Select the **top-liquidity subset** of coins for that day:

```text
countToTake = floor(totalCoinsThatDay * (universeTopPct/100))
```

2. For each selected coin:

* Compute bull condition:

```text
isBull = EMA_fast(coin)[t] > EMA_slow(coin)[t]
```

3. Compute breadth as either:

* Volume-weighted breadth (default):

```text
breadth[t] = sum(volume_i for bullish coins) / sum(volume_i for all selected coins)
```

* Equal-weight breadth:

```text
breadth[t] = bullishCount / selectedCount
```

#### Example (volume-weighted)

```text
Selected coins: A,B,C

Volumes: A=100, B=200, C=700
Bull coins: A and C

breadth = (100 + 700) / (100 + 200 + 700)
        = 800 / 1000
        = 0.80
```

---

### 4.3 Breadth smoothing (EMA on breadth)

After computing raw breadth series:

```text
smoothedBreadth = EMA(breadth, breadthSmoothPeriod)
```

This reduces noisy regime flips.

---

### 4.4 BTC daily returns (benchmark return stream)

```text
btcReturn[t] = (BTC[t] / BTC[t-1]) - 1
```

Example:

```text
BTC[t-1]=100, BTC[t]=105
btcReturn = 105/100 - 1 = 0.05 = 5%
```

---

### 4.5 Rolling volatility estimate (20-day std dev)

You compute rolling std dev of daily BTC returns over last 20 days:

```text
std = sqrt( sum((x - mean)^2) / (n-1) )
```

Example:

```text
returns: [0.01, -0.02, 0.03]
mean = (0.01 - 0.02 + 0.03) / 3 = 0.0066667

variance = ((0.01-0.0066667)^2 + (-0.02-0.0066667)^2 + (0.03-0.0066667)^2) / (3-1)
         = ((0.0033333^2) + (-0.0266667^2) + (0.0233333^2)) / 2
         = (0.00001111 + 0.00071111 + 0.00054444) / 2
         = 0.00126666 / 2
         = 0.00063333

std = sqrt(0.00063333) = 0.025166
```

---

### 4.6 Volatility targeting (Vol Scaling)

Target volatility is `volTarget` (example 0.02).

```text
volMultiplier = min(1.2, volTarget / currentVol)
```

Example:

```text
volTarget  = 0.02
currentVol = 0.04
volMultiplier = min(1.2, 0.02/0.04) = min(1.2, 0.5) = 0.5
```

Meaning: market is volatile → reduce exposure.

---

## 5) Exposure models (regime logic)

Let:

* `b = smoothedBreadth[t]`
* `th = threshold`
* `cf = confirmDays`

### 5.1 TREND_FOLLOW

```text
exposure = 1 if b >= th else 0
```

### 5.2 CONFIRMED_TREND

Require `cf` consecutive days above threshold:

```text
exposure = 1 if all(b[t-cf+1...t] >= th) else 0
```

Example:

```text
cf=3, th=0.6
breadth last 3: [0.65, 0.62, 0.61] => exposure=1
breadth last 3: [0.65, 0.58, 0.61] => exposure=0
```

### 5.3 DUAL_REGIME (Long/Short)

```text
if b >= th: exposure = 1
else if b <= 1 - th: exposure = -1
else exposure = 0
```

Example:

```text
th=0.7
b=0.75 => exposure=1
b=0.25 => since 1-th=0.3, b<=0.3 => exposure=-1
b=0.50 => exposure=0
```

### 5.4 ADAPTIVE_EXPOSURE

Maps breadth linearly to [-1, +1]:

```text
exposure = clamp( (b - 0.5)/0.5 , -1, +1 )
```

Example:

```text
b=0.75 => (0.75-0.5)/0.5=0.5 => exposure=0.5
b=0.10 => (0.10-0.5)/0.5=-0.8 => exposure=-0.8
```

---

## 6) Portfolio backtest math

### 6.1 Final exposure after vol scaling

```text
finalExp[t] = baseExposure[t] * volMultiplier[t]
```

### 6.2 Daily portfolio return

Portfolio returns are tied to BTC returns (proxy):

```text
portRet[t] = finalExp[t] * btcReturn[t]
```

Example:

```text
finalExp = 0.50
btcReturn = 0.04 (4%)
portRet = 0.5 * 0.04 = 0.02 (2%)
```

### 6.3 NAV update

```text
NAV[t] = NAV[t-1] * (1 + portRet[t])
```

Example:

```text
NAV[t-1]=10000
portRet=0.02
NAV[t]=10000*(1.02)=10200
```

---

## 7) Transaction cost model (simple)

If exposure changes meaningfully, apply fee:

```text
if abs(finalExp[t] - finalExp[t-1]) > 0.05:
    NAV[t] = NAV[t] * (1 - feePct)
```

Example:

```text
feePct = 0.0004 (0.04%)
NAV after return = 10200
NAV after fee = 10200*(1 - 0.0004) = 10195.92
```

---

## 8) Drawdown metrics

### 8.1 Peak

```text
peak[t] = max(peak[t-1], NAV[t])
```

### 8.2 Drawdown

```text
dd[t] = (NAV[t] - peak[t]) / peak[t]
```

### 8.3 Max drawdown

```text
MDD = min(dd[t])  (most negative)
```

Example:

```text
NAV series: [100, 110, 105, 120, 90]
peaks:      [100, 110, 110, 120, 120]
dd:         [0, 0, (105-110)/110=-0.04545, 0, (90-120)/120=-0.25]
MDD = -0.25 = -25%
```

---

## 9) CAGR (Correct compounded annual growth rate)

Let:

* `startNAV = capital`
* `endNAV = final NAV`
* `years = dataDays / 365`

```text
CAGR = (endNAV / startNAV)^(1/years) - 1
CAGR% = CAGR * 100
```

Example:

```text
startNAV=10000
endNAV=20000
dataDays=730 => years=2

CAGR = (20000/10000)^(1/2)-1
     = (2)^(0.5)-1
     = 1.41421356-1
     = 0.41421356
CAGR% = 41.42%
```

---

## 10) Sortino Ratio (annualized downside risk)

1. Compute downside returns only:

```text
downRets = { portRet[t] | portRet[t] < 0 }
```

2. Downside deviation (daily):

```text
downStd = std(downRets)
```

3. Annualize downside deviation:

```text
downAnn = downStd * sqrt(365)
```

4. Sortino:

```text
Sortino = CAGR / downAnn
```

Example (conceptual):

```text
CAGR = 0.50
downStd = 0.01
downAnn = 0.01 * sqrt(365) = 0.191
Sortino = 0.50 / 0.191 = 2.62
```

---

## 11) Profit Factor

```text
grossProfit = sum(portRet[t] for portRet[t] > 0)
grossLoss   = abs(sum(portRet[t] for portRet[t] < 0))
profitFactor = grossProfit / max(grossLoss, tiny)
```

Example:

```text
positive: [0.02, 0.01] => grossProfit=0.03
negative: [-0.01, -0.02] => grossLoss=0.03
PF=0.03/0.03=1.0
```

---

## 12) Grid Search logic (Step-by-step)

### Step 1 — Build parameter ranges

From UI inputs:

* universeTopPct (start/end/step)
* emaFast (start/end/step)
* emaSlow (start/end/step)
* threshold (start/end/step)
* confirmDays (start/end/step)
* breadthSmooth (start/end/step)
* strategy (one model or RUN_ALL)

Range generator:

```text
range = [start, start+step, ..., end]
```

---

### Step 2 — Cache EMA for all coins and all EMA periods

This is a major speed optimization:

* For every coin:

  * Fill missing closes with last known close
  * Compute EMA for every needed fast/slow period once
  * Store in `emaCache[symbol][period][t]`

---

### Step 3 — For each combo, backtest

For each combination:

1. Compute daily breadth series using dynamic universe selection
2. Smooth breadth
3. For each day:

   * compute base exposure by strategy rules
   * apply vol scaling
   * update NAV
   * track DD, curve, logs
4. Compute metrics (CAGR, MDD, Sortino, PF)
5. Store result row

---

### Step 4 — Rank results

You currently sort by:

```text
CAGR descending
```

(If you truly want “risk-adjusted CAGR”, you should sort by Sortino or a combined score; see notes below.)

---

## 13) About Rolling Window (Important)

คำถาม: “มันควรมี rolling window ด้วยไหม?”

**คำตอบแบบ research:** ควรมี “Rolling Window Evaluation” ถ้าคุณต้องการให้ผลลัพธ์น่าเชื่อถือขึ้น และลด overfitting

### Recommended approach (standard)

1. Split data into rolling segments:

* Train window: e.g. 365 days
* Test window: e.g. 90 days
* Step forward: e.g. 30 days

2. For each rolling step:

* Grid-search on TRAIN only
* Select best params
* Apply to TEST
* Save test performance

3. Aggregate out-of-sample performance:

* Mean/median CAGR, MDD, Sortino
* Worst-case test window

This is the “walk-forward optimization” pattern.

---

## 14) Why your results look unrealistically high (explain in Thai)

จากผลลัพธ์ที่คุณโชว์:

* CAGR 2000%+
* MDD -5.87%
* Sortino 100+

**เหตุผลที่มักเกิดได้ในระบบนี้:**

1. You are effectively trading BTC with dynamic exposure but without realistic slippage/spread
2. Vol targeting reduces drawdown dramatically and amplifies CAGR if returns align
3. Fee model is too small and only charged on exposure delta > 0.05
4. Binance futures data universe + volume weighting can bias breadth to strong coins
5. If short exposure works perfectly in down markets, metrics explode

**Research fix checklist:**

* Add slippage and funding (for futures)
* Charge fee proportional to turnover (abs(delta exposure))
* Use rolling/walk-forward evaluation
* Add max leverage constraint, liquidation logic (optional)
* Use realistic “tradeable basket” assumption instead of BTC-only proxy

---

## 15) Outputs in the UI (what they mean)

### Grid Dashboard

Shows top ranked parameter combos:

* MODEL (strategy)
* EMA (fast/slow)
* UNIV% (top liquidity percent)
* CAGR%
* SORTINO
* MAX DD%

### Alpha Analyzer

Shows equity curve:

* Portfolio Return % (pnl)
* BTC Benchmark % (btc)

### Holdings History

Every ~30 days when exposure ≠ 0:

* date
* exposure
* universe size
* top bullish coins (by volume)
* NAV

---

## 16) Known limitations (current design)

1. Portfolio return uses BTC proxy only (not a true basket of coins)
2. No funding rates (futures)
3. No slippage model
4. No true rebalancing PnL from multi-asset holdings
5. Grid search is in-sample only (no walk-forward yet)

---

## 17) Copy-ready “Math Summary” block

```text
EMA:
alpha = 2/(p+1)
EMA[0] = x[0]
EMA[t] = alpha*x[t] + (1-alpha)*EMA[t-1]

Breadth (volume):
breadth[t] = sum(vol_i for bullish i) / sum(vol_i for all i)

Bullish rule:
bull_i[t] = 1 if EMA_fast_i[t] > EMA_slow_i[t] else 0

BTC return:
r[t] = BTC[t]/BTC[t-1] - 1

Rolling vol:
vol[t] = std(r[t-20...t-1])

Vol multiplier:
m[t] = min(1.2, volTarget / vol[t])

Exposure:
finalExp[t] = baseExp[t] * m[t]

Portfolio return:
portRet[t] = finalExp[t] * r[t]

NAV:
NAV[t] = NAV[t-1] * (1 + portRet[t])

Fee:
if abs(finalExp[t] - finalExp[t-1]) > 0.05:
  NAV[t] = NAV[t] * (1 - feePct)

Drawdown:
peak[t] = max(peak[t-1], NAV[t])
dd[t] = (NAV[t] - peak[t]) / peak[t]
MDD = min(dd[t])

CAGR:
years = days/365
CAGR = (NAV_end / NAV_start)^(1/years) - 1

Sortino:
downRets = {portRet[t] < 0}
downAnn = std(downRets) * sqrt(365)
Sortino = CAGR / downAnn
