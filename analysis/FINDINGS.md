# Findings

Each claim below was tested on the project's own data before a feature built on it was kept.
Negative results are reported as well. They show which signals are **not** worth modelling.

---

## 1. RSI divergence alone has no edge

**Question:** Does a classic RSI divergence predict a reversal? A divergence is a higher price high with a lower RSI high, or the reverse at lows. The pivot is confirmed *n* bars after the second extreme.

**Data:** 20 liquid coins · ~4 years of daily candles, ~1 year of 4h candles · confirmation delay of 1, 2, 3 and 5 bars.

**Measure:** direction and return 10 bars after the signal; how often the second extreme was later broken.

| Confirmation delay (daily) | 1 | 2 | 3 | **5** |
|---|---|---|---|---|
| Win rate after 10 bars | 54 % | 51 % | 48 % | **58 %** |
| Mean return after 10 bars | −4.9 % | −3.7 % | −3.5 % | **−0.6 %** |
| Extreme broken afterwards | 55 % | 53 % | 50 % | **44 %** |

On 4h the extreme was broken afterwards in 74 / 66 / 59 / 50 % of cases.

**Conclusion:** A longer confirmation delay filters out false pivots: 5 bars works best, which is also TradingView's default. But the mean return is never positive. RSI divergence is kept only as a **confirmation feature**, not as a signal.

---

## 2. Candle colour is not a substitute for real volume delta

**Question:** Many charting tools estimate cumulative volume delta (CVD) from candle colour: a green candle counts as all buying. How close is that to the real aggressive buy − sell volume?

**Data:** Binance spot, last 30 days, hourly. Estimate from OKX lower-timeframe candles vs. real taker-buy volume (kline field 9: `delta = 2 × taker_buy − volume`).

**Result:**

| | Hourly correlation | Same direction | 30-day cumulative, real | 30-day cumulative, estimate |
|---|---|---|---|---|
| BTC / ETH | 0.51 – 0.65 | 66 – 70 % of hours | BTC −6,669 BTC (net selling) | BTC **+13,396 BTC** (net buying) |
| ETH | | | −158 k ETH | **+100 k ETH** |

**Conclusion:** The hourly errors accumulate. Over several days the estimate reported the **opposite** net direction. The terminal switched to real taker data, with a source tag on the chart. A second, independent sum of Binance data matched the implementation exactly (BTC 4h: −178.9 k BTC).

---

## 3. Spot vs. futures CVD divergence

**Question:** When price makes a new high or low but CVD does not, which market's CVD is the better warning: spot or perpetual futures?

**Data:** Binance taker data, 15 coins, 4h, ~1.4 years · outcome measured 10 bars after the new extreme.

| Divergence in | Reversal rate | Mean return | Price continued |
|---|---|---|---|
| Spot CVD | 53 % | +0.30 % | — |
| Futures CVD | 52 % | −0.12 % | — |
| Both | **55 %** | — | **32 %** (lowest) |
| Futures only | 49 % | **−0.60 %** | 41 % |

**Conclusion:** The differences are small, so this is a direction to follow, not proof. Spot CVD is the primary indicator. A divergence that shows up **only** on futures was the weakest case and should not be trusted alone.

---

## 4. The baseline strategies have no edge (yet)

**Question:** Do the rule-based strategies (Smart-Money-Concepts structure, harmonic patterns) beat chance?

**Method:** One year of BTC and ETH perpetuals on 1h, 4h and 1d. Each group is split by time: the first ~70 % for learning, the last ~30 % for testing. The strategy is compared with **random entries that use the same stop and target**. It must beat at least 80 % of them.

| | Trades | Win rate | Mean R | Profit factor |
|---|---|---|---|---|
| Learning (first 70 %) | 118 | 31 % | −0.03 | 0.97 |
| Test (last 30 %) | 59 | 25 % | −0.59 | 0.52 |
| **Total** | **177** | **29 %** | **−0.21** | — |

It beat 76 % of random entries; 80 % is required. **Verdict: no edge.**

**Conclusion:** The rules generate candidates but cannot tell good setups from bad ones. This is the starting point for **meta-labeling**: the strategy proposes, a model trained on human labels decides. See [`DATASET.md`](DATASET.md).

---

## 5. Outcome leakage in the labels, found and fixed

**Observation:** In an early audit of 28 reviewed trades, **every** trade labeled "should take" (5 of 5) turned out to be a winner. The reviewer had seen the outcome on the chart while labeling.

**Fix:**
- **Blind review:** the chart is masked after the trade's exit, recomputed for every timeframe.
- **Locked exam period:** setups on or after 1 July 2026 stay hidden until the final evaluation.
- **Separate labels:** whether the *setup* was valid is stored apart from what the *strategy did wrong*.
