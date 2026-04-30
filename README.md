# ICT-Pine

Pine Script v6 implementation of Simon's ICT v6 (2022 Model) strategy from `~/Desktop/ICT/`.
Original Python: 5,000 LOC, 715 trades, calday-Sharpe **+4.68**, t-stat **18.02** on NQ futures.

## What this gives you

A single TradingView strategy script (`pine/ICT_v6.pine`) that, on any chart,
**self-annotates** every primitive of the ICT canon:

| Primitive | Visual |
|---|---|
| Daily bias (bull/bear/neutral, frozen at 00:00 ET) | Top-left status label + day-start label |
| Kill zones (London / NY AM / NY PM / SB / London Close) | Colored background bands |
| NY Lunch avoid + hard cutoff (15:55 ET) | Gray band + vertical red line |
| Asian range (20:00→00:00 ET) | Translucent box |
| Midnight 15m range (00:00→00:14 ET) | Translucent box |
| Local extremes (swing highs/lows, fractal-N) | ▲ / ▼ markers |
| Equal-Lows / Equal-Highs clusters (stop magnets) | Horizontal multi-touch line + count badge |
| Sweep events (pierce + reclaim of stop pool) | ⚡ marker + sweep level line |
| Displacement bars (≥1×ATR + leaves FVG) | Yellow body highlight |
| Fair Value Gaps (bullish + bearish 3-bar) | Green/red boxes (extend until filled) |
| Order Blocks (bull / bear) | Solid blue/orange boxes |
| Breaker Blocks (BPR — flipped OB after break) | Striped purple boxes |
| OTE zone (62 / 70.5 / 79 fib) on each setup | Dashed band + 70.5% sweet spot line |
| Entry / Stop / Target | ▶ + horizontal lines + R-label |
| R-ladder partial exits (0.5R / 1.5R / 3R) | T1/T2/T3 dots on chart |
| Chandelier ATR trail | Yellow stair-step line on runner |
| NDOG / NWOG opening gaps | Magenta boxes |

## Strategy in one paragraph

> When the daily bias is bull (or bear), and we're in a kill zone, we wait for a **sweep**
> of a recent stop pool (swing low / swing high). Then we wait for a **displacement** bar
> (≥1×ATR body that leaves a 3-bar FVG). At the **70.5%** retracement of sweep→displacement
> we place a limit order. On fill (with engulfing/MSS confirmation), we manage with an
> **R-ladder** (0.5R/1.5R/3R partials, 33%/33%/34%) plus **chandelier ATR trail** on the
> runner. Hard cutoff at 14:30 ET (configurable to 15:55). 0.5% risk per trade, 5
> contracts max.

## Quick Start

1. Open TradingView Desktop on chart `NQ1!` (E-mini Nasdaq-100 continuous).
2. Pine Editor → New → Strategy → paste contents of `pine/ICT_v6.pine`.
3. Save → Add to chart.
4. Set timeframe to **1m or 5m** for entries; the script auto-pulls D1 bias via `request.security`.
5. Toggle visualization layers under Settings → Inputs.

## Files

```
pine/
  ICT_v6.pine        Main strategy (full v6 logic + comprehensive viz)
  ICT_v6_lite.pine   Indicator-only version (no strategy.* calls — for visual review)

docs/
  ARCHITECTURE.md    How the Pine code maps to the Python modules
  USAGE.md           Tuning, performance notes, known limits
  PARAMS.md          Default v6 config + alternates (silver bullet, unicorn)
```

## Source-of-truth

Original Python at `~/Desktop/ICT/ict_v3/` — read-only reference. This repo never touches it.

## License

Personal / non-commercial. The original ICT methodology is © Inner Circle Trader.
