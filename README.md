# Intraday Momentum Strategy for SPY

Case study for the WUTIS Algorithmic Trading Superday.

Replication and extension of Zarattini, Aziz and Barbon (2025), *Beat the Market: An Effective Intraday Momentum Strategy for S&P500 ETF (SPY)*.

## Summary

- I implemented the paper's strategy on 1-minute SPY data from 2016 to September 2026 and checked it against two example days from the paper.
- With the paper's settings and volatility-targeted sizing, the Sharpe ratio after costs is **1.07 in the train period (2016–2021)** and **0.75 in the test period (2022–2026)**. This is lower than the 1.33 reported in the paper for 2007–2024.
- At similar volatility the strategy roughly keeps pace with buy-and-hold SPY. It does not clearly beat it on my sample.
- My own variation (position size tilted by the 5-day RSI) improved the train Sharpe to 1.21 but lowered the test Sharpe to 0.55. I report it as a failed idea and explain why.
- The main lesson: on fewer than five years of test data, small design changes move the Sharpe ratio between 0.55 and 1.19, so single backtest numbers should be read with caution.

## Repository structure

| File | Purpose |
|---|---|
| `01_download_data.ipynb` | Downloads 1-minute SPY bars from Alpaca and saves them as a parquet file. Run once. |
| `02_backtest.ipynb` | Loads the saved data and runs everything else: strategy, backtest, metrics, charts, own variation. |
| `README.md` | This file. |

### Structure of `02_backtest.ipynb`

1. **Settings and data.** All parameters in one block; loading and reshaping the data into a grid with one row per day and one column per minute.
2. **Strategy functions.**
   - `noise_band()` computes the upper and lower boundary.
   - `compute_position()` turns the band and VWAP into a position (+1, 0, −1) at each half-hour.
   - `run_backtest()` returns daily returns after costs.
   - `performance()` and `summarize()` compute the metrics for train and test.
   - `plot_day()` draws one day with band, VWAP and position.
3. **Sanity check** against two days shown in the paper.
4. **Baseline results**: table, equity curves, bar charts of the three metrics.
5. **Parameter sensitivity** on the train period.
6. **Own variation**: RSI-tilted sizing.
7. **Second hypothesis**: trend filter on short trades.
8. **Ablation**: switching off one building block at a time.

## How to run

1. Create a free Alpaca paper-trading account and generate API keys.
2. In Google Colab, store them under Secrets as `ALPACA_KEY` and `ALPACA_SECRET`. The keys are never written in the code.
3. Run `01_download_data.ipynb` once. It saves `spy_1min.parquet` to Google Drive.
4. Run `02_backtest.ipynb` from top to bottom.

The data file is not included in the repository.

## The strategy

**Hypothesis.** Intraday demand and supply imbalances persist, so an unusually large move from the open tends to continue until the close.

**Signal.** For each minute of the day, the average absolute move from the open at that minute over the previous 14 days defines a "noise area" around the open. The boundaries are anchored to the higher (upper) or lower (lower) of today's open and yesterday's close, to account for overnight gaps.

**Execution.** At every HH:00 and HH:30 from 10:00, the strategy is long if the price is above both the upper boundary and the VWAP, short if it is below both the lower boundary and the VWAP, and flat otherwise. All positions are closed at 16:00.

**Sizing.** Exposure is 2% divided by SPY's daily volatility over the previous 14 days, capped at 4x leverage.

**Costs.** $0.0035 commission plus $0.001 slippage per share, per trade.

**Avoiding look-ahead bias.** The band and the volatility estimate use only previous days. A position decided at a given minute earns returns only from the next minute on.

## Data and backtest design

- **Source:** Alpaca, 1-minute bars, regular session 09:30–15:59 New York time.
- **Period:** January 2016 to September 2026 (2,677 full trading days; 24 early-close days removed).
- **Train:** 2016–2021. **Test:** 2022–2026. The first 100 days are used as warm-up.
- **Metrics:** annualized return, annualized volatility and Sharpe ratio (252 trading days, risk-free rate set to zero).

## Results

### Baseline

| | | Ann. return | Ann. volatility | Sharpe |
|---|---|---|---|---|
| Base (100% sizing) | Train | 6.8% | 6.4% | 1.06 |
| | Test | 4.7% | 6.7% | 0.72 |
| Baseline (vol. sizing) | Train | 16.0% | 14.9% | 1.07 |
| | Test | 10.3% | 14.6% | 0.75 |
| SPY buy and hold | Train | FILL IN | FILL IN | FILL IN |
| | Test | 10.6% | 17.2% | 0.67 |

Observations:

- Volatility sizing (average leverage 2.66) scales return and volatility by a similar factor and leaves the Sharpe ratio almost unchanged. The paper's headline return is mostly a result of leverage.
- The edge is smaller out of sample, and smaller than in the paper. My sample does not include 2008, the paper's best year, and it includes the period after publication.

### Sanity check against the paper

| Day | Paper | This implementation |
|---|---|---|
| 31 Jan 2022 | Long from 10:30 to the close, +1.23% | Long from 10:30 to the close, +1.26% |
| 20 Jan 2022 | Long at 10:00, stopped out at 13:00, about 0% | Long at 10:00, flat from 13:00, +0.02% |

### Parameter sensitivity (train period only)

Train Sharpe for lookbacks of 7 to 90 days and band multipliers of 0.8 to 1.5 ranged from 0.72 to 1.07, without a stable region of better settings. Because the surface is flat and noisy, I kept the paper's defaults (14 days, multiplier 1) instead of picking the best cell.

## Own variation: RSI-tilted position sizing

**Idea.** The paper (section 4.5) argues that after market declines option dealers tend to be short gamma and amplify intraday trends. It reports that the strategy earns more when the 5-day RSI is low, but does not trade on it. My variation scales the position by a factor between 0.5 and 1.5 depending on yesterday's RSI: larger after declines, smaller after rallies. Signals, costs and the leverage cap are unchanged.

**Evidence on the train period.** Average daily strategy return by yesterday's RSI:

| RSI | Train | Test |
|---|---|---|
| 0–30 | 20.8 bp | −1.4 bp |
| 30–50 | 9.0 bp | 3.8 bp |
| 50–70 | 1.1 bp | 13.8 bp |
| 70–100 | 2.6 bp | 0.7 bp |

**Result.**

| | | Ann. return | Ann. volatility | Sharpe |
|---|---|---|---|---|
| Baseline | Train | 16.0% | 14.9% | 1.07 |
| | Test | 10.3% | 14.6% | 0.75 |
| RSI-tilted | Train | 19.0% | 15.3% | 1.21 |
| | Test | 7.5% | 15.1% | 0.55 |

**Why it failed.** The relationship that was clear in 2016–2021 reversed in 2022–2026, so the rule placed its largest positions where the strategy no longer earned. Either the pattern was noise (about 200 to 300 days per group), or the market changed.

### Second hypothesis: trend filter on short trades

I tested whether short trades are less reliable when SPY is above its 200-day average. On the train period, short trades earned as much as long trades in uptrends (3.7 against 3.8 basis points per day), so the hypothesis was rejected and not taken to the test period.

### Ablation

| Variant | Train Sharpe | Test Sharpe | Train return | Test return |
|---|---|---|---|---|
| Baseline | 1.07 | 0.75 | 16.0% | 10.3% |
| Without VWAP condition | 0.96 | 0.68 | 14.6% | 9.2% |
| Without gap adjustment | 0.93 | 1.19 | 15.4% | 21.3% |
| Without short trades | 1.27 | 0.69 | 10.4% | 5.8% |
| No trades before 10:30 | 0.84 | 0.85 | 11.7% | 11.2% |

- The VWAP condition helps in both periods.
- No variant is better in both periods, so I did not change the baseline.
- Removing the gap adjustment gives a much better test result but a worse train result. I did not adopt it, because selecting it would mean choosing on the test period. It is the design choice the result is most sensitive to.

## Limitations

- **Short sample.** Under six years of train and five years of test data. Sharpe ratios are estimated with an uncertainty of roughly ±0.4 to ±0.5.
- **Prices are not adjusted for dividends,** which understates buy-and-hold by roughly 1 to 2 percentage points a year.
- **Execution is idealized.** Trades fill at the 1-minute close with fixed slippage.
- **Financing costs are ignored:** margin interest on leverage and borrowing fees on short positions.
- **Early-close days are dropped.**
- **The test period was looked at more than once** (baseline, RSI variation, ablation). All attempts are reported here.
- **RSI is only a proxy** for dealer gamma positioning; I did not use options data.

## Further improvements

- Validate the gap-adjustment finding on fresh data: another ETF such as QQQ, or SPY before 2016.
- Use walk-forward testing instead of one fixed split.
- Replace the RSI proxy with a direct measure of dealer positioning, or with the VIX.
- Model execution more realistically (bid-ask spread, fills at the next bar's open).
- Add financing costs.

## Reference

Zarattini, C., Aziz, A. and Barbon, A. (2025). *Beat the Market: An Effective Intraday Momentum Strategy for S&P500 ETF (SPY).*
