# Dynamic ORB System

An opening-range breakout (ORB) strategy on US ETFs that **switches its opening-range window** each session. It picks among 5 / 10 / 15 / 30 / 60 minutes, or no trade, and the choice comes from rules that only look at past performance. The whole study is in [`research.ipynb`](research.ipynb).

**Bottom line:** the frozen strategy passed its pre-registered 2016+ holdout, but only weakly (ΔSharpe +0.216, Newey–West p 0.023). The placebos show the gain does **not** come from switching windows, and the holdout was not blind. Treat it as unproven. The only clean test left is a forward log of sessions after 2026-09-21.

---

## The strategy

### Universe and data
- **Traded:** DIA, GLD, IEF, IWM, QQQ, SHY, SLV, SPY, TLT, USO. These are auto-discovered from `*_clean_1min.csv` in `/Users/andrew/main/trading/data/etfs/`.
- **Excluded:**
  - CPER, because its bars are too sparse.
  - **VXX**, which is a volatility reference only: it is never traded, never used as a feature, and never used in any selection.
- **Bars:** 1-minute, regular trading hours only.
  - The data come from pitrading until 2022-03-07, then IEX. The IEX feed is thin: volume is about 2 % of before, so relative volume is computed within each source only.
  - The two segments were dividend-adjusted to different dates. Cell 2 rescales the pre-2022 prices onto one back-adjusted basis.
- **Market data is never committed:** it lives outside the repo.

### Trade mechanics (config C1, the "Static ORB" winner)
| Element | Rule |
|---|---|
| Direction | Direction of the opening-range candle, taken only if the overnight gap agrees (`candle+gap`) |
| Filters | \|gap\| ≥ 0.3 ATR; opening-range width ≤ 0.5 ATR |
| Entry | A bar closes beyond the opening range + 0.02 ATR; last entry 11:30 |
| Stop | Opening-range midpoint; moves to breakeven after +2 R |
| Exit | Time exit at 14:00 |
| Costs | 1 tick of slippage per fill, $0.0035/share/side, conservative fill model |
| Eligibility | Decided before the open only: first bar by 09:32, at least 4 of the 09:30–09:34 bars, trailing bar coverage ≥ 80 %. The asset trades only if its median round-trip cost over the prior 250 sessions is ≤ 0.15 R |

### The dynamic layer: which window today?
At **09:35** each asset picks one window. Every input is known before the session, and two causality tests check this.

The frozen choice is **D\* = C1 · `ts_inc_class_h250_k2_pos`**:
- It holds a default window (the best expanding-mean window, asset class level).
- It switches only when another window's EWMA performance (half-life 250 sessions) beats the default by **2 standard errors**.
- It switches at most once per 20 sessions.
- It **stays flat** when the held window's EWMA is ≤ 0. This is the "gate".

The candidates for D\* also included argmax time-series rules and walk-forward random forests: a stacked regressor of each window's value, and a multiclass "which window wins" classifier. D\* uses neither forest.

**Benchmark S\*:** a fixed window per asset, chosen on 2008–2015: DIA 10 · GLD 30 · IEF 15 · IWM 15 · QQQ 15 · SHY 15 · SLV 15 · SPY 10 · TLT 15 · USO 15.

### Research protocol
1. **Stage A (2008–2015):** a screen of 768 combinations of mechanics and filters, scored on their static windows. Four configs go forward.
2. **Selection (2008–2015):** stationary bootstrap. D\* is the variant with the best 10th percentile of paired ΔSharpe against S\*, out of 128 dynamic variants. The data-snooping checks are SPA, Reality Check, StepM, a model confidence set and deflated Sharpe.
3. **Pre-registration:** D\*, S\*, the frozen vol-matching factor k = 1.22648 and the tests are hashed into `data_cache/preregistration_71316e0b.json` (sha256 `22d4360d…`).
4. **Holdout (2016-01-01 → 2026-09-21), tested once:**
   - d_t = k·r_D − r_S.
   - Primary test: one-sided Newey–West t of the mean of d_t.
   - It must also have the same sign before and after 2022-03-07.

## Results

| Check | Result |
|---|---|
| Switching beats the best static window in-sample, after data snooping | **No** (SPA p 1.00) |
| Pre-registered holdout: D\* beats S\* | **Pass, weak**: Sharpe 0.98 vs 0.76, ΔSR +0.216 (90 % CI −0.02 … +0.47), NW p 0.023 |
| Smallest gap the holdout can detect | ≈ 0.49 Sharpe |
| Walk-forward re-selection every year | Fails when training starts in 2009 (p 0.257) |
| Same rules on dividend-corrected prices | Fails (ΔSR +0.08, p 0.152) |
| Is it the window switching? | **No**: an old (stale) window does as well (placebo p 0.997) |
| Is it the gate picking better days? | **No**: random days within each asset-year do as well (p 0.997) |
| Per asset after Holm correction | 0 of 4 traded assets significant |
| Depends on volatility (VXX)? | No |

**Where the holdout gap comes from:** the only thing the placebos don't remove is *how many days the gate trades each asset each year*. S\* is also a weak benchmark out of sample: the all-window ensemble, with no selection at all, has Sharpe 1.02 and D\* does not beat it.

**Caveats:**
- The holdout is **not blind**. An earlier development registration's holdout was seen (it failed) before the current registration was written.
- C1 comes from an earlier project mined on the same 2008–2015 data.
- From 2022 only IWM, QQQ, SPY and TLT have enough IEX data. IEF and SHY are never tradable.

## What comes next: the forward log
Holdout data can no longer give a clean answer. The only test left is **forward**: log D\* and S\* on every session after 2026-09-21 under the frozen rule.

**Cell 14** does this. It is written and tested, and pasted in from chat; it isn't in the notebook yet. It:
- appends each new session to `data_cache/forward_log_71316e0b.csv`;
- refuses to run if anything differs from the registration.

Be realistic about what it can show. Confirming a gap of about +0.2 Sharpe would take **roughly 55–60 years** of data. The detectable gap is 1.6 after one year and 0.5 after ten. The log is therefore a **breakdown monitor**: it can show D\* losing money or the gap reversing, but it can't confirm the edge.

## Repository layout
```
research.ipynb   Cells 1–13: config, data, engine, features, dynamic layer, Stage A, forests,
                 selection + pre-registration, holdout, robustness, per-asset, VXX, conclusions
data_cache/      simulation / forest caches, the frozen pre-registration, the forward log
```

## Running
1. Run Cells 1–13 in order with `QUICK = False`. The caches make re-runs fast; a cold run takes much longer.
2. After adding new bars to the CSVs, re-run Cells 1–13, then the forward log (Cell 14).
3. Never edit the pre-registration file. Cell 8 refuses to continue if the 2008–2015 selection no longer matches it.
