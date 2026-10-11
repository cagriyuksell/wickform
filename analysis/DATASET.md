# Labeled trade dataset

The rule-based strategies generate candidate trades. Each candidate is reviewed by hand and labeled. The target is a **meta-labeling** model: given a candidate, should it be taken?

A sample of 30 labeled trades is in [`data/labeled_trades_sample.csv`](data/labeled_trades_sample.csv).

## How labels are produced

1. The backtest engine produces a trade: setup, entry, stop, target, outcome.
2. The reviewer opens it in the **labeling card**. The chart is **masked after the exit** (blind review), so the outcome does not influence the label.
3. The reviewer records:
   - **Decision:** was this a valid setup to take? (`take` / `skip` / `unsure`) The decision is about the setup, not the result. A valid setup that hit its stop is still `take`.
   - **Checklist:** were the order block, trend, entry and stop correct?
   - **Strategy errors:** what the rule set did wrong (wrong CHoCH, wrong zone…).
   - **LTF confirmation:** when the CHoCH was wrong, did a valid one form elsewhere? If yes, the strategy took the wrong break. If no, there was no confirmation and the setup should be skipped.
   - **Chart-anchored reasons:** every reason is tied to a candle (time, price, timeframe), so a model can locate it. Free-text notes are kept for the reviewer but are **not** model inputs.
   - **Alternative entry:** where the trade should have been taken, with exact entry, stop and target prices.
4. A labeling rule is checked automatically: OB ✓ + trend ✓ + LTF confirmation → `take`; otherwise `skip`. Labels that contradict it are flagged for review.

## Leakage controls

| Risk | Control |
|---|---|
| Labeler sees the outcome | Blind review. The chart is masked after the exit, recomputed per timeframe. |
| Model tuned on the evaluation data | Locked exam period. Setups on or after 2026-07-01 are hidden until the final test. This sample contains none of them. |
| Look-ahead in features | Suggested zones and trend are computed only from candles up to the entry. |
| Random train/test split on time series | Purged walk-forward validation (planned). |
| Repeated setups inflating the sample | Repeats are flagged (`repeat_setup`) and excluded from training. |

## Columns

| Column | Type | Meaning |
|---|---|---|
| `trade_no` | int | Trade number within the backtest run |
| `strategy` | text | `smc` (Smart-Money-Concepts structure) or `harmonik` (harmonic patterns) |
| `symbol`, `timeframe` | text | Instrument (perpetual swaps) and main timeframe |
| `side`, `order_type` | text | `long` / `short`; `limit` / `market` |
| `setup_time_utc`, `entry_time_utc`, `exit_time_utc` | time | When the setup formed, was filled and closed |
| `entry`, `stop`, `target`, `exit` | price | Order levels and the exit price |
| `outcome` | text | `stop`, `target`, `missed` (never filled), `timeout` |
| `planned_R` | float | Reward-to-risk at entry |
| `R_net` | float | Result in R after fees and slippage |
| `MFE_R`, `MAE_R` | float | Maximum favourable / adverse excursion in R |
| `bars_held`, `confidence` | int | Duration; strategy confidence (1 + liquidity sweep + RSI divergence + HTF alignment) |
| `confirm_tf`, `entry_zone`, `target_type`, `harmonic_pattern` | text | Strategy's own context |
| **`label`** | text | **Target:** `take` / `skip` / `unsure` |
| `check_ob_correct`, `check_trend_correct`, `check_entry_correct`, `check_stop_correct` | bool | Reviewer checklist |
| `strategy_errors` | list | `wrong_choch`, `wrong_ob`, `wrong_entry`, `wrong_stop`, `wrong_target`, `repeat_setup` |
| `ltf_confirmed` | bool | When the CHoCH was wrong: did a valid one form? |
| `tags` | list | Standard tags only (e.g. `entry early`, `liquidity not swept`) |
| `correct_choch_*` | time / price / tf / dir | Where the correct change-of-character was |
| `alt_*` | side / prices / R:R / time | Reviewer's alternative entry |
| `reason_entry*` | text / prices | Entry type, its zone, timeframe and additional confluences |
| `reason_confirm_tf`, `reason_stop*`, `reason_target*`, `reason_trend` | various | Chart-anchored reasons; trend `1` up, `-1` down |

## Current size

The second backtest run has 411 trades. 159 are SMC: 120 unlocked and 39 in the locked exam period. Labeling of the unlocked SMC trades is in progress. The sample contains 30 of them: 12 `take`, 18 `skip`.
