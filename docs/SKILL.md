---
name: volume-breakout-strategy
description: >-
  Screen and backtest liquid US equities using the Dan Zanger-style Volume Breakout
  Strategy. Use for momentum breakouts, base consolidations, Dan Zanger setups,
  breakout stock screening, volume confirmation, anti-chase checks, entry/exit
  plans, or historical performance analysis. Apply prior-bar 30-day bases,
  1.5x prior 20-day volume, next-open execution, EMA21 exits, a fixed 7% stop
  and 0.05% transaction friction per executed side.
---

# Dan Zanger Volume Breakout Strategy

## Scope and assets

This is the active **US-equities, daily-timeframe** strategy. Find a close above a
consolidation range, confirmed by volume, while limiting the distance chased above
the breakout pivot. The rules implement the user's precise specification; do not
claim independent authentication, endorsement, or guaranteed historical edge.

Use S&P 500 / Nasdaq 100 constituents with verified membership dates, or an
explicit basket of liquid leaders. The default research basket is AAPL, MSFT,
NVDA, AMZN, GOOGL, META and TSLA. Those seven symbols are a static basket, not a
historical index reconstruction or an exhaustive stock-market scan.

- [rules.json](references/rules.json): machine-readable defaults, parameter
  constraints, universe scope, execution model and metric definitions.
- [run_us_breakout.py](scripts/run_us_breakout.py): canonical standalone
  yfinance/Pandas backtest, cache, trade matching, metrics and ASCII summary.
- [requirements.txt](scripts/requirements.txt): Python dependencies.
- [validation.md](references/validation.md): test commands, acceptance checks,
  and practical limits of the execution model.

The script is self-contained: its defaults mirror `rules.json`, and package
tests enforce their agreement. The JSON describes defaults rather than silently
overriding code. Record any deliberate parameter or universe adaptation.

## Parameters

| Rule | Default | Meaning / constraint |
| --- | ---: | --- |
| `base_days` | 30 | Previous completed sessions; integer 2–1000 |
| `volume_days` | 20 | Previous completed sessions; integer 1–1000 |
| `max_base_pct` | 0.20 | Fractional width; 0.20 means 20%, positive finite |
| `min_volume_ratio` | 1.5 | Minimum relative volume; finite and >1 |
| `max_chase_pct` | 0.05 | Fraction above pivot; positive and ≤5% |
| `ema_days` | 21 | Exit EMA period; integer 1–1000 |
| `hard_stop` | 0.07 | Fixed fraction below actual entry fill; positive and ≤7% |
| `cost_rate` | 0.0005 | Combined friction per buy/sell, 0.05%; 0≤rate<1 |

The 1000-session bound is a package resource guard. Use these defaults unless
the user explicitly requests a variation; an admissible override is not a new
default. `min_volume_ratio` is a floor: volume greater than 2x is still eligible.
No extra moving-average trend, ranking, intraday, or take-profit condition is
implicitly required by the four entry conditions below.

## Data preflight

1. Record symbols, US listing exchange, requested dates, source, price basis and
   `America/New_York` session dates. Use completed regular-session daily bars.
   The CLI's exclusive end defaults to today before 17:00 New York time, or the
   next calendar date afterward; report the provider's actual final session.
2. Verify the requested universe separately. Historical index research requires
   dated constituents including removals/delistings; current membership must not
   be backfilled. The standalone script accepts explicit tickers but does not
   implement time-varying membership. Do not call its static basket a bias-free
   S&P 500 or Nasdaq 100 backtest.
3. Avoid thinly traded small caps, OTC names, wide spreads and positions that
   dominate observed volume. Obtain liquidity evidence when changing the basket.
   No numeric market-cap or minimum-liquidity cutoff was supplied for this
   specification: disclose any extra screen rather than inventing one.
4. Require unique increasing daily dates, finite positive OHLC, nonnegative
   volume, and `low <= min(open, close) <= max(open, close) <= high`. Do not sort,
   interpolate, forward-fill, or silently delete bad bars to manufacture signals.
   Investigate missing exchange sessions and halts with the data provider.
5. Use one consistent split/dividend-adjusted price basis for all OHLC. Yahoo
   `auto_adjust=True` provides the model's adjusted series; dividends are not
   credited a second time. Adjusted fills are modeled prices, not raw broker
   execution prices. Preserve volume units and audit corporate-action anomalies.
6. Require `max(base_days, volume_days, ema_days-1)+1` observations before the
   first signal (31 by default). Require positive finite base low, pivot and
   prior average volume. Invalid/missing inputs are errors, not a valid no-signal.

## Mathematical entry rules

At completed signal session `t`, define:

```text
PivotPrice_t  = max(High[t-30], ..., High[t-1])
BaseLow_t     = min(Low[t-30], ..., Low[t-1])
PriorAvgVol_t = mean(Volume[t-20], ..., Volume[t-1])

BaseWidth_t = PivotPrice_t / BaseLow_t - 1

EntryCandidate_t =
    BaseWidth_t <= 0.20
    AND Close[t] > PivotPrice_t
    AND Volume[t] >= PriorAvgVol_t * 1.5
    AND Close[t] <= PivotPrice_t * 1.05
```

All base and volume windows strictly exclude `t`: use `shift(1).rolling(...)`.
Equality passes base-width, volume and chase limits; equality with the pivot
fails the breakout. Compare unrounded values. The code compares width via
`pivot <= base_low * (1+max_base_pct)` to retain the boundary convention.

Confirm a candidate only after the close. When flat, queue an entry for the next
available trading session's open. No pyramiding and no same-close entry fills.
The 5% rule applies to the signal close; a subsequent opening gap can execute
beyond that distance. Do not retrospectively cancel the signal using future data.

## Execution and exits

1. **Entry**: buy at the next session's adjusted open. Record signal date, fill
   date, fill price, quantity and cost. A signal on the final bar stays pending.
2. **Fixed hard stop**: `Stop = ExecutedEntryPrice * 0.93`. Keep this price fixed
   for the life of the position. Do not ratchet it upward with later highs.
3. **Gap stop**: when the next open is at/below the stop, sell at that open.
   Otherwise, a daily low touching/breaching the stop sells at the stop price in
   the daily OHLC model. Protection is active on the entry session too. Gaps and
   costs can cause realized losses greater than 7%.
4. **Dynamic exit**: at a completed close, `Close[t] < EMA21[t]` queues a full exit
   at `Open[t+1]`. Equality does not exit. EMA is recursive, alpha `2/(21+1)`,
   seeded with the first supplied close; compare implementations with identical
   starting data. A known next-open exit precedes later intraday stop checks.
5. **Friction**: charge `notional * 0.0005` on each executed buy and sell. This is
   one combined slippage/commission cash cost, not two separate 0.05% charges per
   side. Do not add other fee assumptions without stating the adaptation.
6. **Portfolio convention**: default total capital is USD 100,000, divided equally
   among the requested tickers. Each sleeve reinvests its own cash, allows
   fractional shares, and stays in cash when flat. No borrowing, pyramiding or
   cross-sleeve rebalancing. This is a simulation allocation, not a personal
   position-sizing recommendation.
7. **End of data**: mark open positions at the final close; do not force a sale.
   Report open shares and pending orders separately. Trade win/PF/holding metrics
   use completed trades; total return, CAGR and MDD include marked open holdings.

## Workflow and output

From this skill directory, with Python 3.10+:

```sh
python -m pip install -r scripts/requirements.txt
python scripts/run_us_breakout.py --output-dir /tmp/volume-breakout-results
python scripts/run_us_breakout.py --offline --end YYYY-MM-DD --cache-dir /path/to/cache
python -m unittest discover -s tests -v
```

The CLI supports `--tickers`, `--start`, `--end` (exclusive), `--initial-cash`,
`--cost-rate`, `--cache-dir`, `--output-dir`, `--refresh` and `--offline`.
Use a new/empty result directory. Cache keys include ticker, interval and
adjustment policy; verify file hashes. A failed refresh can reuse a verified
cache for the exact request. Provider fallback uses `Ticker.history`; neither
fallback supplies invented prices nor bypasses proxy/TLS verification.

For screening, report as-of session, market/universe evidence, input parameters,
pivot/base low/prior volume, each condition's actual value and pass/fail, and
whether the next-open order is pending. Distinguish missing data from no signal.

For backtests, return the ASCII table of total return, CAGR, win rate, profit
factor, closing-equity MDD, closed trade count, mean calendar holding days and
final equity. Portfolio return and drawdown derive from summed sleeve equity;
never average per-ticker drawdowns. Portfolio win/PF use all completed trades'
net dollar P&L. Zero-trade statistics are N/A; positive profits with no losses
give infinite PF, represented as `inf` in exported JSON.

Export daily ledgers, `trades.csv`, `portfolio_equity.csv`, `summary.json`,
`summary.txt` and `data_manifest.json` when an output directory is supplied.
Do not infer future returns from historical statistics or claim same-day fills
were observed from daily OHLC alone. This package does not submit broker orders.

## Anti-patterns

- Signal-day highs/lows/volume inside reference windows; future lows used for stops.
- Buying at a close that was needed to confirm the signal, or filling terminal orders.
- Replacing the fixed entry-price stop with a trailing stop, alternate moving
  average, anchored price line, partial-profit ladder, or discretionary breakeven rule.
- Treating the current seven-stock basket as historical index membership.
- Ignoring missing/delisted stocks, spreads, thin volume or corporate actions.
- Averaging ticker MDDs, excluding unrealized losses from equity, or counting
  incomplete trades as wins. Changing anchors/parameters after inspecting results.
