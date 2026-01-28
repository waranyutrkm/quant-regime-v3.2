# Quant Regime v3.2  
## Optimized Market Breadth Grid Search Engine (with Volatility Targeting & Trading Cost)

---

## 1. Overview

**Quant Regime v3.2** is a quantitative research and backtesting engine designed to:

- Detect market regimes using **cross-sectional market breadth**
- Convert regime signals into **systematic portfolio exposure**
- Apply **volatility targeting** for risk normalization
- Evaluate robustness through **multi-parameter grid search**
- Explicitly model **trading cost based on portfolio turnover**
- Analyze **maximum exposure risk** over time

The system is implemented as a fully client-side React application using real-time data from **Binance Futures**.

---

## 2. High-Level Workflow

```

Binance API
↓
Dynamic Universe Selection (Top Liquidity)
↓
EMA Calculation (Fast / Slow)
↓
Market Breadth Construction
↓
Breadth Smoothing (EMA)
↓
Regime → Exposure Mapping
↓
Volatility Targeting
↓
Trading Cost Adjustment
↓
Portfolio NAV Simulation
↓
Grid Search Optimization

```

---

## 3. Data Pipeline

### 3.1 Market Scan

- Endpoint:
```

GET /fapi/v1/ticker/24hr

```

- Filter: `symbol.endsWith("USDT")`
- Sort by: `quoteVolume`
- Select: **Top 80 symbols (default)**

This universe is **dynamic** and recalculated for every backtest run.

---

### 3.2 OHLC Data Download

For each selected symbol:

```

GET /fapi/v1/klines
interval = 1d
limit    = 450

```

Stored fields:
- `date`
- `close`
- `quoteVolume`

---

### 3.3 Time Alignment

- All symbols are aligned into a single daily timeline
- Missing prices are forward-filled
- `BTCUSDT` is used as the **market proxy / benchmark**

---

## 4. Indicator Calculations

### 4.1 Exponential Moving Average (EMA)

For any price series `P[t]`:

```

α = 2 / (period + 1)
EMA[t] = α * P[t] + (1 - α) * EMA[t-1]

```

EMA values are **precomputed and cached** for all symbols and periods used in grid search.

---

### 4.2 Market Breadth

At each day `t`, for a selected universe:

- A symbol is **Bullish** if:
```

EMA_fast[t] > EMA_slow[t]

```

#### Volume-Weighted Breadth (default)

```

Breadth[t] = Σ(Volume_bullish) / Σ(Volume_total)

```

Breadth is bounded between `[0, 1]`.

---

### 4.3 Breadth Smoothing

To reduce noise:

```

SmoothedBreadth[t] = EMA(Breadth, smooth_period)

```

---

## 5. Regime → Exposure Models

Let:
- `b = SmoothedBreadth[t]`
- `th = threshold`

### 5.1 TREND_FOLLOW

```

Exposure = 1   if b ≥ th
Exposure = 0   otherwise

```

---

### 5.2 CONFIRMED_TREND

Requires confirmation over `N` days:

```

Exposure = 1 if b[t-N+1 ... t] ≥ th
Exposure = 0 otherwise

```

---

### 5.3 DUAL_REGIME (Long / Short)

```

Exposure =  1   if b ≥ th
Exposure = -1   if b ≤ 1 - th
Exposure =  0   otherwise

```

---

### 5.4 ADAPTIVE_EXPOSURE

Continuous exposure scaling:

```

Exposure = clamp((b - 0.5) / 0.5, -1, 1)

```

---

## 6. Volatility Targeting

Volatility is computed from BTC returns:

```

BTC_Return[t] = (BTC[t] / BTC[t-1]) - 1

```

Rolling 20-day volatility:

```

σ[t] = StdDev(BTC_Return[t-20 ... t-1])

```

Volatility multiplier:

```

VolMult[t] = min(1.2, TargetVol / σ[t])

```

Final exposure:

```

FinalExposure[t] = BaseExposure[t] * VolMult[t]

```

---

## 7. Portfolio Return & NAV

Daily portfolio return:

```

PortfolioReturn[t] = FinalExposure[t] * BTC_Return[t]

```

NAV update:

```

NAV[t] = NAV[t-1] * (1 + PortfolioReturn[t])

```

---

## 8. Trading Cost Model (Turnover-Based)

Transaction cost is applied **proportionally to exposure change**:

```

Turnover[t] = |FinalExposure[t] - FinalExposure[t-1]|

```

If:
```

Turnover[t] > 0.01

```

Fee cost:

```

FeeCost[t] = Turnover[t] * NAV[t] * (FeePct / 100)

```

NAV after fee:

```

NAV[t] = NAV[t] - FeeCost[t]

```

This avoids over-penalizing micro-adjustments while realistically pricing regime shifts.

---

## 9. Exposure Risk Tracking

### 9.1 Period Max Exposure (Rolling 30 Days)

Tracks the maximum absolute exposure within each logging window:

```

PeriodMaxExp = max(|FinalExposure|)

```

### 9.2 Global Max Exposure (All-Time)

Tracks worst-case leverage risk:

```

GlobalMaxExp = max(|FinalExposure[t]|) for all t

```

Both metrics are stored and displayed in results and logs.

---

## 10. Performance Metrics

### 10.1 CAGR

```

Years = TotalDays / 365
CAGR = (FinalNAV / InitialCapital)^(1 / Years) - 1

```

---

### 10.2 Maximum Drawdown (MDD)

```

Peak[t] = max(NAV[0...t])
DD[t] = (NAV[t] - Peak[t]) / Peak[t]
MDD = min(DD[t])

```

---

### 10.3 Sortino Ratio

Uses downside deviation only:

```

Sortino = CAGR / (StdDev(NegativeReturns) * √365)

```

---

## 11. Grid Search Optimization

The system performs an exhaustive grid search over:

- Universe Size (%)
- EMA Fast
- EMA Slow
- Breadth Threshold
- Confirmation Days
- Breadth Smoothing
- Strategy Type

Each parameter combination is fully backtested.

Results are ranked by **CAGR** and analyzed for **local robustness** by comparing neighboring parameter sets.

---

## 12. Output & Visualization

### Provided Outputs:
- Full equity curve vs BTC benchmark
- Drawdown profile
- Exposure time series
- Holdings snapshots (top bullish assets)
- Max exposure risk metrics
- Searchable & sortable result table

---

## 13. Research Notes & Limitations

- BTC proxy introduces **basis risk**
- No funding rate or slippage modeled
- Grid search is **in-sample**
- Results may appear optimistic due to volatility targeting

---

## 14. Recommended Extensions

- Walk-forward / rolling out-of-sample testing
- Multi-asset basket PnL instead of BTC proxy
- Funding rate & slippage modeling
- Regime stability constraints

---

## 15. Disclaimer

This project is intended for **research and educational purposes only**.  
It does not constitute financial advice or a trading recommendation.
