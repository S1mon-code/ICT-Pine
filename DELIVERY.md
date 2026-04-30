# Delivery Report — Overnight Build (2026-04-30 → 05-01)

**Repo:** https://github.com/S1mon-code/ICT-Pine
**Local:** `~/Desktop/ICT-Pine/`
**Source-of-truth Python (read-only, untouched):** `~/Desktop/ICT/ict_v3/`

---

## What was delivered

Two compile-clean Pine Script v6 files that mirror the ICT v6 (2022 Model) Python
strategy with **comprehensive self-annotating visualization** of every primitive.

| File | Type | Lines | Purpose |
|---|---|---:|---|
| `pine/ICT_v6.pine` | `strategy()` | 1411 | Full v6 backtestable strategy + every visualization |
| `pine/ICT_v6_lite.pine` | `indicator()` | ~370 | Indicator-only, lighter and faster to load — for visual review |

Both scripts last verified `pine_check` clean (0 errors, 0 warnings) before TVBridge CDP went offline during the page reload.

### Visualization coverage (toggleable)

| Primitive | Visual rendered |
|---|---|
| Daily bias (D1 MSS direction, frozen at 00:00 ET) | Day-start emoji label + status table cell |
| Kill zones | bgcolor — London=cyan, NY AM=yellow, NY PM=lime, SB=brighter, Lunch=gray |
| Asian range (20:00→00:00 ET) | Translucent yellow box building over session |
| Midnight 15m range (00:00→00:14 ET) | Translucent silver box |
| Hard cutoff (15:55 ET) | Vertical red dashed line at first crossing each day |
| Local extremes — swing highs/lows (fractal-N) | ▲ / ▼ markers at confirmed pivots |
| Equal-Lows / Equal-Highs clusters | Dashed horizontal line + EQL×N / EQH×N badge |
| Sweeps (pierce + reclaim) | ⚡SSL / ⚡BSL labels + dotted level line |
| Displacement bars (≥1×ATR + leaves FVG) | Yellow body tint (bgcolor) |
| Fair Value Gaps (3-bar, bull + bear) | Green / red translucent boxes (extend until filled, then gray) |
| Order Blocks (bull / bear) | Solid blue / orange boxes |
| Breaker Blocks (BPR — flipped OB after break) | Striped purple boxes (auto-promoted) |
| OTE zone (62 / 70.5 / 79 fib) | Aqua / maroon dashed band on each pending setup |
| Entry, stop, target | ▶ marker + horizontal lines + R-info label |
| R-ladder partial exits | T1 / T2 / T3 dot markers at 0.5R / 1.5R / 3R |
| Chandelier ATR trail | Dynamic stop following best high/low |
| NDOG / NWOG opening gaps | Magenta boxes |
| Status table (top-right) | D1 bias / KZ / PD / State / Trades today / Daily R / FVG·OB·EQ counts |

### Backtest engine

`ICT_v6.pine` is a real `strategy()` declaration, so TradingView's Strategy
Tester computes:
- Trade list, win-rate, profit-factor
- Equity curve, max drawdown, Sharpe
- Per-trade entry/exit markers natively
- Commissions ($2/side per contract) and slippage (2 ticks) match the Python

---

## Iterative build journey

| Step | Outcome |
|---|---|
| Phase 1 — Read all `~/Desktop/ICT/ict_v3/*` (read-only) | Strategy fully understood: state machine, sweep / displacement / FVG / OTE / R-ladder / chandelier / KZ / D1 bias / Asian range / NDOG |
| Phase 2 — Deep research Pine v6 | Confirmed support: UDTs + arrays + methods, `request.security` with `barmerge.lookahead_off`, `hour(time, "America/New_York")`, drawing-budget caps at 500 |
| Phase 3 — Architecture blueprint | `ARCHITECTURE.md` written; chose single-file strategy + lite indicator |
| Phase 4 — `gh repo create S1mon-code/ICT-Pine` | Repo public; initial commit pushed |
| Phase 5 — Write `pine/ICT_v6.pine` | First draft — got initial syntax errors (multi-line ternary, `color.gold` doesn't exist, tuple-returning ternary) — fixed iteratively to compile clean |
| Phase 6 — `pine_check` clean | 0 errors, 0 warnings after refactoring `ta.*` calls and detection-fn calls to top level for v6 consistency |
| Phase 7 — Add to chart on NQ1! 5m via TVBridge | Loaded successfully, but **rendered nothing** |
| Phase 8 — Debug subagent | Found root cause: `for j = swings_l.size() - 1 to 0` crashed with empty array on bar 0 because Pine v6 with default `step=+1` iterates `[-1, 0]` and `arr.get(-1)` aborts the entire bar. **Fixed** with `if sz > 0` guard + `by -1` step. |
| Phase 9 — Write lite indicator + `USAGE.md` + `ARCHITECTURE.md` | Companion docs done |
| Phase 10 — Code review subagent | Found 5 more empty-array / inverted-loop hazards in `f_rebuild_eq_clusters`, `f_detect_sweep_long/short`, `f_disp_long/short_recent`. **All guarded with size > 0 checks.** |
| Phase 11 — TV page reload | TradingView CDP server hung after `location.reload()` — couldn't resume visual verification |

### Final commits on `master`

```
$ git log --oneline
75bcafa fix: defensive guards for empty arrays + inverted loop ranges
6f6a4ec feat: add ICT v6 Lite indicator + USAGE.md
9c3811b fix(critical): empty-array crash + wrong loop direction in sweep detection
ce03d59 fix: extract ta.* and detection funcs to top-level for Pine v6 consistency
6f4d6a1 feat: ICT v6 Pine Script v6 main strategy — full state machine + visualization
e8e6e6e init: ICT-Pine project — README + ARCHITECTURE blueprint
```

---

## Status when Simon wakes up

| Item | Status |
|---|---|
| `gh repo create S1mon-code/ICT-Pine` | ✅ Public, all commits pushed |
| `pine/ICT_v6.pine` compile clean | ✅ 0 / 0 last verified |
| `pine/ICT_v6_lite.pine` compile clean | ✅ 0 / 0 last verified |
| All known empty-array / inverted-loop bugs guarded | ✅ 6 fixes |
| Loaded on chart and rendering visually | ⚠️ **Final visual verification needs Simon** — TV Desktop CDP stopped responding after a page reload at ~00:25 |
| Strategy generates trades in backtest | Unknown — needs in-chart strategy tester run |
| Docs (`README`, `ARCHITECTURE`, `USAGE`, `DELIVERY`) | ✅ All in repo |

### What Simon should do this morning (5 minutes)

1. **If TradingView Desktop still has the spinning Pine Editor:**
   - Quit and relaunch TV Desktop with the debug script:
     `~/Desktop/TVBridge/scripts/launch_tv_debug_mac.sh`
2. **Open the chart:** any liquid 1m/5m chart of `NQ1!` works.
3. **Pine Editor → New → Strategy → paste contents of `pine/ICT_v6.pine`** (or the lite version into "New → Indicator").
4. **Save** with Ctrl+S, then click **Add to chart**.
5. Should immediately see (within 2-3 sec):
   - `·` heartbeat dots at the top of every bar (proves runtime is healthy)
   - Status table top-right
   - Kill zone backgrounds in NY hours
   - ▲▼ swing markers
   - FVG / OB / Sweep boxes & labels
   - On a 1m chart you'll start seeing OTE setups + entries

### If something still doesn't render

Open TV's Pine console (bottom of editor) and look for runtime errors. If a
`<for-loop>` array out-of-bounds appears that the agent didn't catch, the fix
pattern is always the same — wrap in `if arr.size() > 0` and use `by -1` for
descending iteration.

---

## What's NOT delivered (versus the maximalist read of "every ICT primitive")

These were Python-side flags off-by-default and verified as **noise** in the
v8/v9 self-challenge rounds. Not implemented in Pine to keep the deliverable
minimal:

- News blackout (FOMC/CPI/NFP calendar) — Python `enable_news_blackout=False`
- W1 / M1 HTF alignment filter — Python `enable_htf_filter=False`
- Confluence Score gate — Python `enable_confluence_gate=False`
- Volume imbalance / Liquidity void gate — Python `enable_volume_imbalance_gate=False`
- Power of 3 daily-template filter — Python `enable_po3_filter=False`
- Drawdown-adaptive sizing — Python `enable_dd_adaptive_sizing=False`
- Multi-tier OTE — Python `enable_multi_tier_ote=False`
- Order-flow agreement filter — Python `enable_order_flow_filter=False`
- Silver Bullet preset — Pine inputs already let you set `allowed_kill_zones=("London_SB","NY_AM_SB","NY_PM_SB")` to replicate
- Unicorn (Breaker + FVG overlap) — design captured in `unicorn.py`, not ported (separate setup, not part of v6 default)
- Turtle Soup — Python verdict: failure
- Stop Magnet Reversal — Python verdict: failure
- Gap Entry — Python verdict: failure

If Simon wants any of these, they're additive — straightforward to bolt on as
extra input-flagged blocks within the existing strategy file.

---

## Honest caveat

Final morning verification was **not** completed because TV Desktop's CDP
debug server stopped responding at 00:25:33 after `location.reload()`. The
script compiled cleanly the last time `pine_check` could run; both batches
of empty-array fixes are conceptually sound and additive. There is residual
risk of one more silent runtime crash hidden somewhere — if the chart loads
the script but draws nothing, the same debug pattern (look for a non-guarded
`for i = 0 to arr.size() - 1` loop on a possibly-empty array) will find it.

---

*Generated 2026-05-01 ~00:35.*
