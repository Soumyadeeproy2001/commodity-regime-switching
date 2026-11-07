# Regime-Switching Dynamic Portfolio Allocation on Commodity Futures

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/https://github.com/Soumyadeeproy2001/commodity-regime-switching/commodity-regime-switching/blob/main/Regime_switch_fixed.ipynb)

A Python research notebook that asks one question:

> **Can a portfolio beat a fixed Markowitz portfolio by detecting the market's regime (mood) and changing its commodity weights accordingly?**

**Short answer from this study: no.** After removing look-ahead bias and charging transaction costs, the static portfolio beat the regime-switching one both in-sample and out-of-sample. This repo documents the full, honest test, including why it likely failed.

---

## Data

- `Consolidated_cleaned.xlsx`: 6,688 daily rows with `Date` plus 7 commodity prices (Aluminum, Coffee, Copper, Corn, Crude Oil, Gold, Soybean), Jan 2000 – Jan 2026.
- The prices are **roll-adjusted continuous futures data**.
- Split: **in-sample** 2000-01-04 → 2015-08-13 (4,013 days) · **out-of-sample** 2015-08-14 → 2026-01-02 (2,674 days).
- Non-positive prices (e.g. WTI crude at –$37.63 in April 2020) are removed before taking logs and filled forward.
- **Data quality note:** a few days show an extreme one-day jump that reverses the next day, on the same dates across several commodities (e.g. 27–28 Nov 2008, when Copper moved +63% then –66%). The cause is not confirmed. They inflate kurtosis (up to 123) and may drive the very short Regime 1 and 3 spells. They have **not** been removed in the results below.

## Method

```
Prices → log & simple returns → 60/40 train/test split
   ├─ EDA: stats, Jarque-Bera, correlations, ADF
   ├─ GARCH(1,1): volatility clusters (ARCH-LM p < 0.0001)
   ├─ Static Markowitz: Min-Variance and Max-Sharpe (Ledoit-Wolf + SLSQP)
   ├─ Markov regime model: mean + variance switching, k = 4 by BIC
   ├─ Per-regime optimal weights (Max-Sharpe; Min-Var if all means ≤ 0)
   ├─ Dynamic weights = Σ P(regime_{t-1}) × regime weights   (no look-ahead)
   ├─ Backtest with 10 bps cost, stress tests, attribution
   └─ Out-of-sample validation, scorecard, overfitting check
```

**In plain words:** detect four market moods, build one portfolio per mood, and each day blend them using yesterday's mood probabilities. Then compare against a fixed portfolio.

### The four regimes found

| Regime | Daily volatility | Days | Avg spell | Personality |
|---|---|---|---|---|
| 0 | 0.66% | 1,724 | 11.8 days | Calm |
| 1 | 2.9% | 226 | 2.6 days | Stress (negative mean) |
| 2 | 1.1% | 1,990 | 10.8 days | Normal (positive mean) |
| 3 | 11.3% | 72 | 1.9 days | Extreme shock |

## Results (from the notebook's saved outputs)

### In-sample (2000 – Aug 2015)

| Metric | Static Min-Var | Static Max-Sharpe | Dynamic (gross) | Dynamic (net 10 bps) |
|---|---|---|---|---|
| CAGR | 10.6% | 12.9% | 4.9% | 0.8% |
| Volatility | 23.7% | 27.0% | 35.0% | 35.0% |
| Sharpe | 0.54 | **0.58** | 0.31 | 0.20 |
| Max drawdown | –38.6% | –41.8% | –66.6% | –73.7% |

Average daily one-way turnover of the dynamic strategy: **15.7%**, so costs matter.

### Out-of-sample (Aug 2015 – Jan 2026)

| Metric | Static | Dynamic |
|---|---|---|
| CAGR | 13.2% | 12.8% |
| Volatility | 19.2% | 27.4% |
| Sharpe | **0.74** | 0.58 |
| Max drawdown | –23.7% | –32.0% |

### Performance by period: crisis vs other periods

| Period | Sharpe (static / dynamic) | Sortino (static / dynamic) | Max drawdown (static / dynamic) |
|---|---|---|---|
| Crisis: GFC 2008-09 | 0.53 / 0.20 | 0.75 / 0.21 | –38.6% / –53.5% |
| Crisis: commodity crash 2014-16 | 0.00 / –0.64 | 0.00 / –0.62 | –19.0% / –42.1% |
| Whole training period (2000 – Aug 2015) | 0.54 / 0.31 | 0.72 / 0.35 | –38.6% / –66.6% |
| Test period (Aug 2015 – Jan 2026) | 0.74 / 0.58 | 0.98 / 0.71 | –23.7% / –32.0% |

- **In crises**, the dynamic strategy lost more: drawdowns were 15 to 23 percentage points deeper than the static portfolio.
- **In the calmer, later years**, annual returns were almost equal (13.2% vs 12.8%), but dynamic carried more risk (27% vs 19% volatility).
- COVID 2020 falls inside the test window, so the in-sample stress test reports "no data" for it.

### Other findings

- **Attribution:** a simple equal-weight portfolio (Sharpe 0.56) beat every dynamic variant. The best dynamic variant was Mean+Volatility (0.36).
- **Regime prediction scorecard (OOS):** hit rate 55.0%, volatility accuracy 39.1%, overall 45.4% ("WEAK").
- **25-year vs 15+10 check:** dynamic Sharpe 0.46 vs 0.40, a gap of 0.06. Dynamic is weak in both versions, so a small gap shows consistency, not strength.

## Why did it likely not work?

1. **Tiny, short-lived regimes.** Regime 3 has 72 days and Regime 1 has 226, and they last about 2 days on average. Weights use *yesterday's* regime, so the regime has often ended by the time it is detected.
2. **Corner solutions.** Regime 1's "optimal" portfolio is 100% Coffee, a sign of overfitting to a small sample.
3. **High turnover** (15.7% per day) eats returns, even with probability blending.
4. **Estimation error.** Average returns are hard to estimate, and Max-Sharpe optimisation amplifies the errors. Published research finds that simple equal weighting is hard to beat for this reason (see References).
5. **Possible data artefacts.** One-day spike-and-reverse moves may be creating the 2-day "shock" regimes.
6. **Regimes drift over time**, so what was learned in 2000-2015 is less useful later.

## How to run

```bash
pip install -r requirements.txt
jupyter notebook Regime_switch_fixed.ipynb
```

Make sure `Consolidated_cleaned.xlsx` is in the same folder as the notebook. If you run it in Google Colab, upload the Excel file and use the path `/content/Consolidated_cleaned.xlsx`.

## Repo structure

```
├── Regime_switch_fixed.ipynb
├── Consolidated_cleaned.xlsx
├── requirements.txt
└── README.md
```

## Limitations

- Roll costs and slippage are not modelled; trading costs are a flat 10 bps.
- Holiday or bad-price spikes may remain in the data (see the data note above).
- The "25-year in-sample" run refits on all data, so it is optimistic by design.
- The scorecard weights (40/30/30) are an author design choice.
- Results come from one run of the notebook; re-running may shift numbers slightly.

## Next steps

- Clean or winsorise the one-day spikes and re-run.
- Test fewer, longer-lasting regimes (k = 2 or 3).
- Add weight caps (e.g. 40% per asset) and a turnover penalty.
- Shrink the regime portfolios toward equal weight.
- Use walk-forward (rolling) re-fitting.

## References

- DeMiguel, V., Garlappi, L., & Uppal, R. (2009). *Optimal Versus Naive Diversification: How Inefficient is the 1/N Portfolio Strategy?* Review of Financial Studies, 22(5), 1915–1953.
- Bulla, J., Mergner, S., Bulla, I., Sesboüé, A., & Chesneau, C. (2011). *Markov-switching Asset Allocation: Do Profitable Strategies Exist?*
- Shu, Y., Yu, C., & Mulvey, J. M. (2024). *Regime-Aware Asset Allocation: a Statistical Jump Model Approach.* arXiv:2402.05272.

## Disclaimer

Educational research only. Not investment advice. Past performance does not predict future results.
