# Usage Guide

## Quick Start

1. Open TradingView (Desktop or Web).
2. Load chart `CME_MINI:NQ1!` (or any liquid futures / crypto).
3. Set timeframe to **5 minute** (or **1 minute** for closer fidelity to Python original).
4. Click `Pine Editor` (bottom panel).
5. New → Strategy → paste full contents of `pine/ICT_v6.pine`.
6. Click `Save` (Ctrl+S).
7. Click `Add to chart`.

## What you should see

Within 2-3 seconds of adding to chart you should see:

- **Top of every bar**: small blue `·` heartbeat dots — confirms the script is running on every bar.
- **Background**: colored bands during kill-zone hours (London=cyan, NY AM=yellow, NY PM=lime, SB=brighter shade, Lunch=gray).
- **Top-right corner**: status table showing D1 bias, Kill Zone, PD pos, State, Trades today, Daily R, FVG/OB/EQ counts.
- **Dotted horizontal lines**: EQL×N (fuchsia, below) and EQH×N (cyan, above) — equal-low/high stop magnets.
- **Tiny ▲▼ markers**: swing pivots (green ▼ for swing lows, red ▲ for swing highs).
- **Green / red translucent boxes**: 3-bar Fair Value Gaps. Green=bullish, red=bearish. Gray when filled.
- **Blue / orange boxes**: Order Blocks (blue=bullish OB, orange=bearish OB).
- **Purple striped boxes**: Breaker Blocks (BPR — formerly an OB that broke + flipped role).
- **⚡SSL / ⚡BSL labels**: sweep events (Sell Side Liquidity = swept low; Buy Side Liquidity = swept high).
- **Aqua/maroon dashed band**: the OTE zone on each pending setup, with limit/stop/target lines.
- **▶ markers**: trade entries with R-info label.
- **T1/T2/T3 dots**: R-ladder partials filling at 0.5R/1.5R/3R.
- **Vertical red dashed line**: hard cutoff at 15:55 ET each day.
- **Day-start emoji label**: 🟢🔴⚪ above each new day's first bar showing D1 bias.
- **Translucent yellow box overnight**: Asian range (20:00→00:00 ET).
- **Translucent silver box**: Midnight 15-min range (00:00→00:14 ET).
- **Magenta boxes**: NDOG/NWOG opening gaps.

## Recommended workflow

### Visual review on 5m

1. Set chart to NQ1! 5m.
2. Add `ICT_v6.pine` strategy.
3. Open Strategy Tester (bottom panel) to see backtest results.
4. Toggle visualization layers under the gear icon → Inputs → "▼ Visualization" group to declutter.
5. Scroll back through history to see how setups formed.

### Live trading review on 1m

1. Switch to 1m.
2. Watch a kill zone (e.g., NY AM 8:30-11:00 ET).
3. Wait for ⚡ sweep marker.
4. Wait for OTE zone to draw with dashed limit/stop/target.
5. If ▶ entry triggers, watch for T1 → T2 → T3 fills.

### Backtest tuning

Strategy Tester → Performance Summary will show:
- Total trades
- Win rate
- Profit factor
- Max drawdown
- Sharpe (TV computes simple Sharpe; Python's calday-Sharpe is calculated differently)

To tune, adjust under Settings → Inputs:
- `▼ Strategy params` group: ATR length, displacement multiplier, FVG min size, OTE percentages, sweep lookback
- `▼ R-ladder + Trail` group: target R levels, trail multiplier
- `▼ Risk` group: risk per trade, max contracts, daily limits
- `▼ Time gates` group: which kill zones to allow

## Caveats vs. Python original

| Aspect | Python (canonical) | Pine v6 |
|---|---|---|
| Timezone | CT data → ET via offset | NY ET via `hour(time, "America/New_York")` (DST handled by TV) |
| ATR | Wilder via EMA | `ta.atr(14)` (RMA = Wilder, equivalent) |
| Continuous contract | None — per-expiry contracts | Uses TV's `NQ1!` continuous |
| Slippage / commission | 2 ticks / $2 per side | Same — set in `strategy()` declaration |
| Backtest engine | Pandas event-driven custom | TradingView's strategy engine |
| Drawing limits | Unlimited | 500 each (boxes / lines / labels) — old ones auto-deleted |

Expect **±20-30% trade count variance** vs. Python results due to engine differences.

## Drawing budget tips

If you see the chart get cluttered:
- Reduce `swing_keep_per_side` (default 80) to 30
- Reduce `fvg_max_age` (default 480) to 240
- Reduce `ob_max_age` (default 1440) to 720
- Toggle off the noisier layers (e.g. `show_swings`, `show_eq`)

If you want MORE history:
- Bump `max_bars_back` in the strategy declaration (default 2000)

## Common issues

### "Strategy report is empty"
The strategy needs at least one full setup → entry → exit cycle. Reasons it may not happen:
- Timeframe too high (use 1m or 5m)
- Visible range too short (zoom out / scroll back to load 1-2 weeks of history)
- D1 bias is `neutral` (no MSS detected on D1 yet)
- All kill zones disabled in inputs
- PD threshold too tight

### Heartbeat dots missing
If the blue `·` at the top of bars is absent, the script is silently failing. Open the Pine Editor console (Pine logs) to see runtime errors. The most common cause: an array out-of-bounds in iteration.

### No kill-zone background
- Confirm the chart is on a timeframe ≤ 1H (kill zones are minute-scale)
- Confirm `show_kz` input is on
- Confirm chart timezone is NY (TV usually shows in your local TZ; the script converts via `hour(time, "America/New_York")` so it doesn't matter, but if it's broken, check `et_h` in a debug plot)

## Source-of-truth

Original Python at `~/Desktop/ICT/ict_v3/`. This repo is a clean Pine port; it doesn't read or modify the original.
