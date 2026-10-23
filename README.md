# Backtesting Strategies on Indian IT Stocks (vectorbt)

Vectorised backtests of a moving-average crossover strategy on five Indian IT large caps, built with [vectorbt](https://vectorbt.dev). The notebook goes from a single-parameter backtest to a walk-forward optimisation that picks parameters by Sharpe ratio.

**Universe (NSE, daily close via Yahoo Finance):** `TCS.NS`, `INFY.NS`, `HCLTECH.NS`, `WIPRO.NS`, `TECHM.NS`
**Period:** 2022-10-21 → 2026-10-21 (~986 trading days) for the multi-stock sections; 2022-01-01 → 2026-01-01 for the TCS walk-forward section.

## What's in the notebook

All code lives in [`building_stratagies_vec_bt.ipynb`](building_stratagies_vec_bt.ipynb).

| Step | What it does |
|---|---|
| 1. Data | Downloads daily close prices for the five stocks with `vbt.YFData` |
| 2. Crossover signals | 10-day fast MA vs 30-day slow MA. Enter on cross above, exit on cross below |
| 3. Baseline comparison | Strategy total return per stock vs buy-and-hold (`Portfolio.from_holding`) |
| 4. Time splits | `range_split(n=4)` cuts the history into 4 windows, and the strategy runs on each with two parameter sets (10/30 and 20/30) |
| 5. Sharpe by split | `pf.sharpe_ratio()` per fast/slow window pair, split and stock, to see how stable the edge is across regimes |
| 6. Walk-forward optimisation | On TCS: 30 rolling splits (2-year window, 180-day sets). Grid-search every fast/slow pair from 10–39 days for the best **in-sample Sharpe**, then re-run the winning pair **out-of-sample** |

## Results

### Crossover (10/30) vs buy-and-hold, full period

| Stock | Crossover total return | Buy & hold |
|---|---|---|
| HCLTECH | 96.1% | 57.9% |
| TECHM | 26.0% | 95.6% |
| TCS | 23.1% | -10.6% |
| INFY | 15.3% | -15.1% |
| WIPRO | 4.8% | 3.2% |

The crossover beat buy-and-hold on 4 of 5 stocks, mostly by avoiding drawdowns in TCS and INFY. TECHM is the clear exception. These runs assume **zero fees and slippage**.

### Sharpe ratio by time split (4 splits, ~246 days each)

The edge is not stable. For the 10/30 pair, Sharpe is strongly positive in splits 0–1 (e.g. HCLTECH 1.62 → 2.64, INFY 0.43 → 2.05) and turns negative in splits 2–3 (e.g. INFY -2.27, WIPRO -2.28 in the last split). Trend-following did well in trending regimes and got chopped up in sideways ones.

### Walk-forward on TCS (in-sample optimised, out-of-sample tested)

Over 30 out-of-sample windows, the in-sample-best parameters produced:

- Mean out-of-sample Sharpe: **-0.06**
- Median out-of-sample Sharpe: **0.22**
- Positive in 19 of 30 windows, with a run of strongly negative windows (Sharpe down to -1.88) in the middle

Picking the best Sharpe in-sample did not produce a reliably positive Sharpe out-of-sample. That is the typical sign of parameter overfitting, and it is the main takeaway of this project.

## Setup

```bash
pip install -r requirements.txt
jupyter lab building_stratagies_vec_bt.ipynb
```

Tested with Python 3.11 and vectorbt 1.2.0. Data is pulled live from Yahoo Finance, so re-running will extend the series to the current date and numbers will drift from those above.

## Limitations

- No transaction costs, taxes or slippage in any run
- Only 5 stocks, which is a subset of the Nifty IT index, and all are highly correlated
- Sharpe is the only risk-adjusted metric computed so far
- The walk-forward section is run on TCS alone

## Roadmap

- [ ] Mean-reversion strategy (e.g. z-score or Bollinger Band entries) and comparison with the crossover
- [ ] More risk-adjusted metrics: Sortino, Calmar, max drawdown
- [ ] Fees and slippage
- [ ] Walk-forward on all five stocks

## Disclaimer

For education and research only. Past backtest performance does not indicate future results and nothing here is investment advice.
