# Architecture — Python → Pine Script v6

This doc maps Simon's Python `ict_v3/*` to the single Pine Script v6 file `pine/ICT_v6.pine`.

## Why a single file (not a multi-file library)

Pine Script libraries exist (`//@version=6\nlibrary("X")`) but importing them across
TradingView's "Add to chart" UI is friction-heavy for a single-user. **Single
self-contained strategy file** keeps the deliverable copy-pastable. Reusable types
and functions are organized into clearly-marked **sections** within the one file.

## Drawing budget

Pine v6 caps drawing object pools per script:

| Object | Default | Max | We use |
|---|---|---|---|
| labels | 50 | 500 | 500 |
| boxes | 50 | 500 | 500 |
| lines | 50 | 500 | 500 |

Strategy declaration:

```pine
strategy("ICT v6 NQ", overlay=true,
    max_labels_count=500, max_boxes_count=500, max_lines_count=500,
    max_bars_back=2000,
    initial_capital=100000, default_qty_type=strategy.fixed,
    default_qty_value=1, commission_type=strategy.commission.cash_per_contract,
    commission_value=2, slippage=2, pyramiding=0)
```

To stay under budget on a 6-year 1-min chart (~600k bars) we apply **lifecycle rules**:

- **FVG box**: deleted when filled OR when older than `fvg_max_age_bars` (default 480 = 8 hours on 1m).
- **OB box**: deleted when broken OR when older than `ob_max_age_bars` (default 1440 = 1 day on 1m).
- **Sweep label**: deleted when older than `sweep_keep_bars` (default 240).
- **Swing pivot label**: capped at most-recent `swing_max_keep` per side (default 80).
- **Kill zone**: rendered via `bgcolor()` not boxes — no budget cost.
- **OTE zone**: deleted when setup expires (60 bars) or fills.

## Multi-timeframe (HTF) data

D1 bias is pulled with `request.security`:

```pine
[d1_bias_str, d1_dr_high, d1_dr_low] = request.security(
    syminfo.tickerid, "D",
    [bias_signal, dr_high, dr_low],
    lookahead = barmerge.lookahead_off)
```

Lookahead off ensures **no future leakage** — bias is computed on the prior day's
closed bar exactly like the Python `compute_daily_bias()` (frozen at 00:00 ET).

For W1/M1 HTF filter (optional, off by default — Python v8 verdict was "noise"):
identical pattern with `"W"` and `"M"`.

## Type system

Pine v6 UDTs:

```pine
type FVG
    int       idx
    int       dir         // +1 bull, -1 bear
    float     fvg_low, fvg_high, atr_at
    int       conf_idx    // FVG confirmed at idx + 1
    bool      filled
    box       bx          // visual ref

type OB
    int       idx, dir
    float     low, high, body_ratio, displacement_atr
    int       displacement_idx
    string    state       // 'fresh' | 'tested' | 'broken'
    bool      breaker
    box       bx

type SwingPivot
    int       idx, side   // side: +1=high, -1=low
    int       confirmed_idx
    float     price
    label     lb

type EQCluster
    int       side, touch_count, most_recent_idx
    float     level
    line      ln
    label     lb

type Sweep
    int       idx, source_kind   // 1=EQL, 2=EQH, 3=swing_low, 4=swing_high
    float     level
    label     lb
    line      ln

type OTESetup
    int       direction, sweep_idx, disp_idx, place_idx, expire_idx
    float     limit_price, stop, target, ote_low, ote_high
    float     sweep_level, disp_extreme
    box       ote_bx
    line      stop_ln, target_ln, limit_ln
    bool      filled

type Trade
    string    setup
    int       entry_bar, direction
    float     entry_price, stop_price, initial_stop, target_price
    string    bias, kill_zone
    int       contracts, initial_contracts
    int       partials_filled
    float     partial_pnl_usd
    bool      moved_to_be
    line      entry_ln, stop_ln, target_ln
    label     entry_lb
```

## Module mapping (Python → Pine)

| Python module | Pine section |
|---|---|
| `timezones.py` | `// ═══ KILL-ZONE TIME WINDOWS ═══` (constants + `in_window_et()` helper using `hour(time, "America/New_York")`) |
| `sessions.py` | `// ═══ SESSIONS / KILL ZONES ═══` (active KZ string, lunch, hard cutoff) |
| `bias.py` | `// ═══ D1 BIAS ENGINE ═══` (HTF-resolved via request.security; on-D1 swing+MSS detection) |
| `htf_bias.py` | `// ═══ W1 / M1 OPTIONAL HTF FILTER ═══` (only used if input flag) |
| `structures.py` | `// ═══ INTRADAY STRUCTURES ═══` (pivots via ta.pivot, EQL/EQH cluster, sweep, displacement+FVG, engulfing, micro-MSS) |
| `order_blocks.py` | `// ═══ ORDER BLOCKS / BREAKERS ═══` |
| `opening_gaps.py` | `// ═══ NDOG / NWOG ═══` (computed from D1 + W1 securities) |
| `model.py` (state machine) | `// ═══ STATE MACHINE & STRATEGY ENTRIES ═══` |
| `model.py` (R-ladder + chandelier) | `// ═══ TRADE MANAGEMENT ═══` (using strategy.exit with ladder + manual trail) |
| `power_of_3.py` | omitted by default (Python verdict: noise) |
| `silver_bullet.py` | preset config flag |
| `unicorn.py` | preset config flag |
| `news_blackout.py` | omitted (Python verdict: noise) |

## State machine

Same 7 states as Python `simulate_v3`, encoded as `int` `state_int`:

```
0 IDLE             — waiting for KZ + bias
1 ARMED            — bias set + in KZ → watch for sweep
2 SWEPT            — sweep detected → watch for displacement
3 MSS_OK           — displacement + FVG confirmed → place OTE limit
4 OTE_PENDING      — limit at 70.5% awaiting fill / reactive confirmation
5 FILLED           — trade open → manage to target/stop/cutoff
6 CLOSED           — record + reset to IDLE
```

States 3 and 4 are merged in code (we directly create OTE setup once displacement
is confirmed). State 6 is implicit (just resets).

## Backtest fidelity vs. Python

What we **match exactly**:
- D1 bias direction (bull/bear/neutral)
- Sweep detection (single fractal swing low/high; EQL/EQH cluster prioritized)
- Displacement: ≥1×ATR body + leaves 3-bar FVG of ≥0.2×ATR
- OTE 70.5% sweet spot
- Stop = sweep ± 1×ATR buffer (capped at 2×ATR)
- R-ladder 0.5R / 1.5R / 3R, 33/33/34
- Chandelier ATR trail at 1×ATR
- Hard cutoff & max hold
- Daily limits: 3 trades, -2R stop, +3R lock

What's **approximated** (tractable in Pine):
- ATR — Pine `ta.atr(14)` is RMA-based, equivalent to Wilder.
- Reactive entry — check engulfing / micro-MSS in OTE band on bar close.
- Multi-tier OTE — disabled (Python default also disables: `enable_multi_tier_ote=False`).
- News blackout — disabled (Python default).
- HTF W1/M1 — disabled (Python default; toggleable input).
- Slippage 2 ticks per side — strategy declaration.
- Commission $2 per side per contract — strategy declaration.

What's **changed by necessity**:
- The Python uses `datetime_ct` and `datetime_et` columns; Pine uses NY time directly via
  `hour(time, "America/New_York")`. Same semantic — DST handled by TradingView.
- Trading day boundary for daily-state — uses ET calendar date.

## Visualization rules

Each primitive has a toggle. Default **everything ON** so the chart speaks for itself.

```
show_kz             — bgcolor for London / NY AM / NY PM / SB / London Close / NY Lunch
show_d1_bias        — top-left status label + day-start arrow
show_swings         — ▲/▼ at confirmed pivots
show_eql_eqh        — horizontal line at cluster level + touch-count
show_fvg            — green/red boxes (extending until filled)
show_ob             — solid blue/orange boxes
show_breaker        — purple striped boxes (BPR)
show_sweep          — ⚡ + horizontal sweep line (3-day fade)
show_displacement   — yellow bar tint (background)
show_ote            — 62/79 dashed band + 70.5 solid line (only on active setup)
show_entry          — ▶ at entry, +stop/target lines, R-label
show_partials       — T1/T2/T3 markers + chandelier trail line
show_asian          — translucent gold box overnight
show_midnight       — translucent silver box 00:00-00:14
show_ndog           — magenta box at gap
show_hard_cutoff    — vertical red dashed line at 15:55 ET
```

## Performance

On 1-min NQ (~390 bars/day RTH + ETH ~1440/day) for 6 years, Pine evaluates left-to-right
and our state is O(1) per bar (we keep arrays bounded). Should compile and load.

## Testing strategy

1. Compile clean (no `pine_check` errors).
2. On NQ1! 5m chart, visually confirm:
   - Kill zones render correctly at expected ET times.
   - D1 bias label updates daily.
   - At least one full trade lifecycle visible: sweep → FVG → OTE → entry → R-ladder → exit.
3. Strategy report shows non-zero trades.
4. Compare aggregate trade count and winrate ballpark to Python (715 trades / ~80% WR
   over 6 years; on 1 year of NQ1! 1m we'd expect ~120 trades).

Pine ≠ Python identically because of:
- Continuous contract roll handling differs.
- Granular tick-level fills not simulatable in Pine.
- Reactive entry confirmation has small bar-close vs intra-bar timing differences.

That's expected and documented.
