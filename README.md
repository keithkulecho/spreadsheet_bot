# Spreadsheet Bot — Multi-Timeframe SMC Research Pipeline

An event-driven research and execution stack for a **multi-timeframe Smart Money Concepts (SMC)** strategy on **XAUUSD (gold)**.

The system detects liquidity sweeps, structural breaks and supply/demand "flip" zones on 5-minute data, filters them through 1H/15M structure, and backtests the resulting retest-triggered entries. The same engine is wrapped by an optimization phase, a paper-trading phase and a live-trading phase behind a strict, pre-registered research process.

> Research status: the core edge is **not yet established**. Development data is used up and the live decisions are being driven by a small number of pre-registered holdout tests (see [Research rules](#research-rules)).

---

## Table of contents

- [How it works](#how-it-works)
- [Glossary](#glossary)
- [Repository layout](#repository-layout)
- [Quick start](#quick-start)
- [Running the pipeline (`main.py`)](#running-the-pipeline-mainpy)
- [Data](#data)
- [Dashboard](#dashboard)
- [Tests](#tests)
- [Research toolkit](#research-toolkit)
- [Research rules](#research-rules)
- [Phases](#phases)
- [Continuous integration](#continuous-integration)
- [Conventions](#conventions)

---

## How it works

```
5M OHLC
   │
   ├── Resample to 1H (macro) and 15M (internal) structure
   │
   ▼
run_backtest()  (strategy/backtest.py)
   ├── Swing detection                 strategy/swings.py
   ├── Market structure / breaks       strategy/structure.py
   ├── Liquidity levels -> sweeps      strategy/liquidity.py
   ├── Supply & demand zones           strategy/zones.py
   ├── Flip-zone signal generation     strategy/signals.py
   ├── MTF confluence guard            strategy/confluence.py   (per-signal filter)
   └── Retest -> trigger -> WIN/LOSS/OPEN lifecycle
                                       strategy/retest_engine.py
   ▼
Trade results + optional CSV output
```

`main.py` is the orchestration layer only. All strategy logic lives under `strategy/`; the backtest engine is deterministic and replay-safe (no look-ahead — a signal can only use information visible at its bar).

## Glossary

| Term | Meaning |
|------|---------|
| **SMC** | Smart Money Concepts — structure-, liquidity- and zone-based price-action method. |
| **Swing** | A confirmed local high (BSL) or low (SSL). |
| **BOS / CHOCH / INITIAL_BREAK** | Break of structure / change of character / first structural break. |
| **Liquidity sweep** | Price taking out a swing level, then reversing. |
| **Flip zone** | A supply/demand zone formed from a structural break, used as a reversal entry. |
| **MTF** | Multi-timeframe: 1H = macro context, 15M = internal context, 5M = execution. |
| **R** | Risk multiple: `r_multiple = gross_R − cost_R`, where risk = fill price to stop. |
| **Z label** | A signal that is against **both** the 1H and 15M trend. |
| **Room** | Distance to the nearest confirmed 15M swing, measured in R (`>= 1.5R` = "room pass"). |
| **MOC** | Market-on-close: signal triggers on a touch, fills at the next bar's open. |
| **RE test** | The pending-signal state machine: still pending / expired / invalidated / triggered. |

## Repository layout

```
main.py                  Orchestration: load -> MTF build -> gated backtest -> CSV
requirements.txt         Pinned dependencies (Python 3.11)

strategy/                Core strategy engine (frozen for research)
  swings.py              Swing detection
  structure.py           HH/HL/LH/LL, trend, BOS/CHOCH
  zones.py               Supply/demand zone detection + mitigation
  liquidity.py           Liquidity levels and sweeps
  signals.py             Flip-zone signal generation
  retest_engine.py       PENDING_RETEST state machine (shared with Phase 3)
  atr.py                 Causal ATR(14) (known at the fill bar's open)
  confluence.py          MTF direction / confluence filter
  mtf_structure.py       1H + 15M resampling and trend projection
  structure_targets.py   Structural take-profit (nearest confirmed 15M swing)
  premium_discount.py    Premium/discount (dealing range) context
  pattern_tags.py        Z / A / B / X tagging
  rules.py               Session / trading-hour rules
  test_*.py              Strategy-level unit tests

tools/                   Research & statistics toolkit (see below)
tests/                   Pytest suite

dashboard/               Streamlit app (Research, Phase2 Lab, Machet AI, Paper Trading)
phase_02_optimization/   Monte Carlo, overfit/PBO, walk-forward, sensitivity
phase_03_paper/          Paper-trading system (replay, positions, sessions, journal)
phase_04_live/           Live execution (broker, orders, risk, recovery, reconciliation)
gold_orderflow/          MT5 tick/order-flow probes
market_data/             Data ingestion / feature helpers

data/                    Committed datasets (see Data)
.github/workflows/       CI: check.yml (full) and toolkit.yml (fast)
```

The repository root also contains many one-off `diagnose_*.py`, `patch_*.py`, `verify_*.py` and `check_*.py` scripts accumulated during development. These are scratch/diagnostic tools, not part of the pipeline.

## Quick start

Requires **Python 3.11**.

```bash
git clone https://github.com/keithkulecho/spreadsheet_bot.git
cd spreadsheet_bot

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Run the pipeline on the committed real gold data
python main.py --data data/real_gold_data_5m.csv
```

Note: `MetaTrader5` and `tzdata` are Windows-only and install conditionally; the backtest and tests run fine on Linux without them.

## Running the pipeline (`main.py`)

```bash
python main.py --data data/real_gold_data_5m.csv \
               --spread 0.30 --slippage 0.10 --commission 0.0 \
               --output mtf_trades.csv
```

### Options

| Flag | Default | Purpose |
|------|---------|---------|
| `--data` | `your_data_file.csv` | Path to the 5M OHLC CSV. |
| `--output` | none | Save closed trades (with 1H/15M trend tags) to CSV. |
| `--no-mtf-neutral` | off | Reject signals when the 15M trend is neutral instead of allowing them. |
| `--no-direction-gate` | off | Drop the 1H/15M trend-agreement requirement (counter-trend entries allowed). |
| `--spread` | `0.0` | Round-trip spread in **price units** (gold: `0.30` = 30 cents). |
| `--slippage` | `0.0` | Adverse slippage **per side** in price units (applied on entry and exit). |
| `--commission` | `0.0` | Total round-trip commission in price units. |
| `--reward-multiple` | `2.0` | Target as a multiple of risk. |
| `--fill-mode` | `touch` | `touch` (fill at zone edge) or `market_on_close` (trigger on touch, fill next open). |
| `--target-anchor` | `zone` | `zone`, `fill` (reward-multiple × real risk), or `structure` (nearest confirmed 15M swing). |
| `--min-rr` | `1.5` | With `--target-anchor structure`, minimum reward to the swing in R. |
| `--advance-to-min-rr` | off | With `structure`, use the next swing that reaches `--min-rr` instead of skipping. |
| `--stop-atr-mult` | `0.0` | Widen the stop beyond the zone edge by this many causal ATR(14). |
| `--strict-entry-fill` | off | Fill only if price reaches the entry (incompatible with `market_on_close`). |

**Cost scale guard.** Costs are in price units, so they must match the data's price scale. The pipeline refuses to run if the round-trip cost exceeds 50% of the median 5M candle range (i.e. gold-scale costs on EURUSD-scale data), because the resulting R values would be meaningless.

**Fill / target constraints.** `--target-anchor fill|structure` and `--stop-atr-mult > 0` require `--fill-mode market_on_close`. `--min-rr` and `--advance-to-min-rr` only apply with `--target-anchor structure`.

## Data

| File | Contents |
|------|----------|
| `data/real_gold_data_5m.csv` | ~80,000 real XAUUSD 5M candles, 2025-08-20 onwards (UTC), with volume. Readable by CI. |
| `data/master_historical_data.csv` | ~5,000 synthetic 5M candles (EURUSD-scale prices) used for pipeline/guard smoke runs. |

Both use the format `timestamp,open,high,low,close[,volume]`.

> `real_gold_data_5m.csv` is the **development** dataset. Per the research rules it is used up and may not be tuned on any further. New work runs on fresh, non-overlapping holdouts.

Additional datasets are pulled from MetaTrader 5 on Windows (`pull_mt5_*.py`, `mt5_*.py`, `stage_mt5_datasets.py`); the `.db`/`.parquet` artifacts and most large CSVs are git-ignored.

## Dashboard

A native Streamlit app:

```bash
streamlit run dashboard/app.py
```

Pages:

1. **Research** — `dashboard/pages/1_📊_Research.py`
2. **Phase2 Lab** — `dashboard/pages/2_◆_Phase2_Lab.py`
3. **Machet AI** — `dashboard/pages/3_🤖_Machet_AI.py`
4. **Paper Trading** — `dashboard/pages/4_📈_Paper_Trading.py`

The dashboard uses built-in Streamlit components only (no custom HTML/CSS, no `unsafe_allow_html`).

## Tests

```bash
# Full suite (the look-ahead test is a script, run separately)
pytest -q tests --ignore=tests/test_mtf_causality.py

# Look-ahead / causality test (also runs on real data)
python tests/test_mtf_causality.py
python tests/test_mtf_causality.py data/real_gold_data_5m.csv
```

Coverage includes: signal generation, backtest fill realism and strict fills, fill-anchored and structural targets, MTF causality, pattern tags, warmup, confluence direction gate, daily-loss limits, Phase 3 sessions and Phase 4 live execution, plus the statistics toolkit (deflated Sharpe, block bootstrap, rank stability, walk-forward, session-vs-outside, random-entry baseline, ledger).

## Research toolkit

`tools/` holds the statistics and reporting scripts. The most important:

| Tool | Purpose |
|------|---------|
| `z_vs_room.py` | Core test: is the edge in the **Z** label or the **room** rule? |
| `z_vs_room_by_group.py` | Per-group breakdown of the Z-vs-room result. |
| `exit_study.py` | Trade outcomes at fixed exits (2R/5R) with Z/room/time tags. |
| `session_vs_outside.py` | H-S1 session filter test (pooled clustered OLS, in/out window). |
| `walk_forward.py` | Walk-forward / out-of-sample stability. |
| `deflated_sharpe.py` | Deflated Sharpe ratio given the number of trials. |
| `block_bootstrap.py` | Block bootstrap confidence intervals. |
| `random_entry_baseline.py` | Random-entry null baseline (dev vs holdout guards). |
| `rank_stability.py` | Rank stability of in-sample vs out-of-sample. |
| `bif.py` | Backtest overfitting / probability checks. |
| `ledger.py` | Trial ledger (`docs/ledger.csv`) of every hypothesis tested. |
| `mtf_split.py` | MTF split with reward sweep (MOC). |
| `pattern_report.py` | Pattern-tag report. |
| `structure_target_report.py` | Structural target vs fixed R. |
| `compare_fill_modes.py` | Fill-rule comparison and cost sensitivity. |
| `check_data_sessions.py` | Timestamp / DST / session sanity for a dataset. |

Example:

```bash
python tools/z_vs_room.py --data data/real_gold_data_5m.csv \
    --rows 0 --which tail --cost 0.5 --output holdout_trades.csv
```

## Research rules

Two pre-registered documents freeze the hypotheses, metrics, pass/fail lines and run logs **before** results are seen. Editing them after a result voids the test:

- **`Research rules z vs room · MD`** — Q1: is Z better than not-Z? Q2: is room ≥ 1.5R better than room < 1.5R? Measured as the pooled effect at a fixed 5R exit, room threshold T = 1.5R, with PASS requiring a positive effect and t > 2.2 plus ≥100 trades in the smaller group. Includes sign/quarter robustness notes showing neither Z nor room is confirmed on development data, so a fresh holdout decides.
- **`Research rules session filter · MD`** — H-S1: do trades signalled in the London/NY overlap (13:00–17:00 UTC) have a higher mean R? Pooled clustered OLS, SWE +0.30R, N* ≈ 310 trades per group.

A `docs/ledger.csv` trial ledger tracks every hypothesis and its data split; it is surfaced on the CI Summary page and feeds the DSR trial count.

## Phases

| Phase | Directory | Scope |
|-------|-----------|-------|
| 2 | `phase_02_optimization/` | Monte Carlo, PBO/overfit detection, walk-forward, parameter/spread sensitivity, staged validation, trial registry. |
| 3 | `phase_03_paper/` | Paper trading: market engine (replay and MT5), positions, sessions, notifications, journal, performance, persistence, a Strategy Adapter that only receives already-triggered signals. |
| 4 | `phase_04_live/` | Live execution: broker (MT5), orders, risk (daily loss limits), reconciliation, recovery, clock/time offsets, monitoring, notifications. |

Phase 3's adapter deliberately reuses `strategy/retest_engine.py` rather than re-implementing trigger logic, so there is a single source of truth for when a signal fires.

## Continuous integration

Two GitHub Actions workflows run on pushes to `main` / `feature/multi-timeframe`:

- **`.github/workflows/check.yml`** — the long check: installs pinned deps, runs the pipeline on both datasets, verifies the cost-scale guard refuses mismatched data, checks data sanity (gaps, weekend candles), runs the pytest suite and the MTF look-ahead test, checks for tracked-but-ignored files and hard-coded credentials, py-compiles the live code, and runs the MTF/target/ATR research reports.
- **`.github/workflows/toolkit.yml`** — a fast (~1 minute) check that every toolkit file is present and that the statistics tools and their guards behave.

## Conventions

- Strategy modules keep detection separate from forward simulation; the backtest is deterministic and replay-safe.
- Causal by default: features used at a bar may only use bars up to the previous close (e.g. `known_atr`).
- Costs are always expressed in **price units** and verified against the data's scale.
- Research is holdout-first: the development data is used up, rules are frozen before runs, results are logged immediately, and code under `strategy/` plus the research tools is frozen during a holdout phase.
