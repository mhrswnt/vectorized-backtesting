# Backtesting Strategies on Indian IT Stocks (vectorbt)

Vectorised backtests of a moving-average crossover strategy on five Indian IT large caps, built with [vectorbt](https://vectorbt.dev). The notebook goes from a single-parameter backtest to a walk-forward optimisation that picks parameters by Sharpe ratio.

**Universe (NSE, daily close via Yahoo Finance):** `TCS.NS`, `INFY.NS`, `HCLTECH.NS`, `WIPRO.NS`, `TECHM.NS`
**Period:** 2022-10-21 → 2026-10-21 (~986 trading days) for the multi-stock sections; 2022-01-01 → 2026-01-01 for the TCS walk-forward section.

## What's in the notebook

All code lives in [`building_stratagies_vec_bt.ipynb`](building_stratagies_vec_bt.ipynb).

| Step | What it does |
|---|---|
| 1. Data | Downloads daily close prices for the five stocks with `vbt.YFData` |
| 2. Crossover signals | 10-day fast MA vs 30-day slow MA. Enter on cross above, exit on cross below. Signals are shifted one bar, so a signal on day t's close trades on day t+1's close (no look-ahead) |
| 3. Baseline comparison | Strategy total return per stock vs buy-and-hold (`Portfolio.from_holding`), both net of the same costs |
| 4. Time splits | `range_split(n=4)` cuts the history into 4 windows, and the strategy runs on each with two parameter sets (10/30 and 20/30) |
| 5. Sharpe by split | `pf.sharpe_ratio()` per fast/slow window pair, split and stock, to see how stable the edge is across regimes |
| 6. Walk-forward optimisation | On TCS: 30 rolling splits (2-year window, 180-day sets). Grid-search every fast/slow pair from 10–39 days for the best **in-sample Sharpe**, then re-run the winning pair **out-of-sample**. Costs apply in both stages |

### Trading costs

Every backtest, including the buy-and-hold benchmark and the walk-forward, uses one shared `costs` dict defined at the top of the notebook:

| Parameter | Value | Meaning |
|---|---|---|
| `fees` | 0.12% | Per side, approximating STT, stamp duty and exchange charges on NSE delivery trades |
| `slippage` | 0.05% (5 bps) | Applied against you on every fill |
| `fixed_fees` | ₹7.67 | Flat charge per order |
| `init_cash` | ₹100,000 | Per stock |

Sharpe ratios are annualised with 252 trading days (`vbt.settings.returns['year_freq']`), not vectorbt's default of 365.

## Results

### Crossover (10/30) vs buy-and-hold, full period

Net of fees and slippage, signals executed on the next bar:

| Stock | Crossover total return | Buy & hold |
|---|---|---|
| HCLTECH | 77.1% | 57.6% |
| TECHM | 21.8% | 95.3% |
| INFY | 10.5% | -15.3% |
| TCS | 6.9% | -10.8% |
| WIPRO | -9.2% | 3.0% |

The crossover beat buy-and-hold on 3 of 5 stocks (HCLTECH, INFY, TCS), mostly by avoiding drawdowns in INFY and TCS. It lost clearly on TECHM and WIPRO. Costs take roughly 5–9 percentage points off each crossover result, while buy-and-hold barely moves because it makes a single trade. Trading on the next bar rather than the signal bar also lowers the results, so an earlier version of this notebook that ignored both looked noticeably better than this one (e.g. HCLTECH 96.1%, TCS 23.1%, WIPRO +4.8%).

### Sharpe ratio by time split (4 splits, ~246 days each)

The edge is not stable. For the 10/30 pair, Sharpe is positive in splits 0–1 (e.g. HCLTECH 1.06 → 2.26, INFY 0.44 → 1.50) and turns negative in splits 2–3 (e.g. INFY -1.66, WIPRO -2.28 in the last split). Trend-following did well in trending regimes and got chopped up in sideways ones.

### Walk-forward on TCS (in-sample optimised, out-of-sample tested)

Over 30 out-of-sample windows, the in-sample-best parameters produced:

- Mean out-of-sample Sharpe: **-0.51**
- Median out-of-sample Sharpe: **-0.52**
- Positive in only 9 of 30 windows (best 0.92, worst -1.58)

Picking the best Sharpe in-sample did not produce a positive Sharpe out-of-sample once costs were included. The optimiser favours fast, frequently trading pairs that look good gross and are hurt by costs, which is a typical sign of parameter overfitting. This is the main takeaway of this project.

Note that the walk-forward runs with `direction="both"` (long and short), unlike the long-only backtests above, and shorting delivery stocks is not possible in India, so treat this section as a methodology demo rather than a tradable result.

## Setup

```bash
pip install -r requirements.txt
jupyter lab building_stratagies_vec_bt.ipynb
```

Tested with Python 3.11 and vectorbt 1.2.0. Data is pulled live from Yahoo Finance, so re-running will extend the series to the current date and numbers will drift from those above.

## Limitations

- Costs are an approximation: a flat 0.12% per side plus a flat ₹7.67 per order. Real brokerage, DP charges and taxes (including capital gains tax) are not modelled exactly, and slippage is a constant 5 bps rather than depending on volume
- Fills are at the next day's close, not the open, and there is no check for liquidity or circuit limits
- Only 5 stocks, which is a subset of the Nifty IT index, and all are highly correlated
- Sharpe is the only risk-adjusted metric computed so far
- The walk-forward section is run on TCS alone

## Roadmap

- [ ] Mean-reversion strategy (e.g. z-score or Bollinger Band entries) and comparison with the crossover
- [ ] More risk-adjusted metrics: Sortino, Calmar, max drawdown
- [x] Fees and slippage
- [ ] Long-only walk-forward, to match the tradable backtests
- [ ] Walk-forward on all five stocks

## Disclaimer

For education and research only. Past backtest performance does not indicate future results and nothing here is investment advice.
