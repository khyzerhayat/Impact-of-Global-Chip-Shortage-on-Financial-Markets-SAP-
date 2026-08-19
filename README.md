# Impact of the Global Chip Shortage on Financial Markets

A multi-method analysis of how the 2020–2022 semiconductor shortage affected sector-level market performance, using eleven years of daily equity data and a mix of SQL, Python, and R.

**BAIM635 Capstone Project**

---

## Overview

The semiconductor shortage that began in early 2020 disrupted industries far beyond chip manufacturing itself. This project asks a measurable version of that question: **did the shortage produce a detectable structural change in how semiconductor-linked equities behaved relative to the broader market — and did that change persist once the acute shortage eased?**

To answer it, the 2015–2025 window is divided into three regimes:

| Phase | Period | Characterization |
|-------|--------|------------------|
| Phase 1 | 2015–2019 | Pre-shortage; stable semiconductor supply |
| Phase 2 | 2020–2022 | COVID shutdowns and the global chip shortage |
| Phase 3 | 2023–2025 | Post-acute recovery; tariff and export-control tensions |

Each phase is analyzed independently and then compared, so that shifts in volatility, correlation, and clustering behavior can be attributed to a specific regime rather than to the decade as a whole.

## Universe

Nine tickers, grouped by exposure type.

**Directly impacted sectors** — firms whose operations depend on physical chip supply:

- `AAPL` — Consumer electronics
- `INTC` — Semiconductor manufacturing
- `GM` — Automotive
- `AVGO` — Telecommunications / semiconductor design

**Emerging technology sectors** — firms exposed to downstream demand shocks rather than direct supply constraints:

- `NVDA` — GPU / AI
- `QCOM` — Wireless / 5G
- `FSLR` — Clean energy

**Benchmarks:**

- `SOXX` — iShares Semiconductor ETF
- `SPY` — S&P 500 ETF

This split is the analytical backbone of the project: it separates *primary supply-chain exposure* from *downstream technological exposure*, which turn out to behave very differently under stress.

## Data Pipeline

Source data is minute-level OHLCV bars from Polygon, stored in a `polygon_1m_bars` schema in PostgreSQL.

The extraction query builds a daily close series from scratch rather than relying on a vendor daily feed:

1. **Generate a weekday calendar** across 2015-01-01 → 2025-11-24 using `generate_series`, filtered to `ISODOW` 1–5.
2. **Exclude NYSE holidays** via an explicit union of every market holiday from 2015 through 2025.
3. **Extract the daily close** for each ticker with a correlated subquery selecting the last 1-minute bar between 09:30 and 16:00.
4. **Pivot into a wide table** — one row per trading day, one column per ticker — materialized as `merged_stocks_prices` and exported to `chips_data.csv`.

Final dataset: **2,743 trading days × 9 tickers**, 2015-01-02 through 2025-11-24.

## Methods

| Analysis | Tool | Purpose |
|----------|------|---------|
| Returns comparison | SAP Predictive Analytics | Track sector performance across phases |
| Volatility decomposition | Python | Measure phase-wise sensitivity to supply-chain stress |
| Correlation matrices | Python | Detect co-movement and decoupling between regimes |
| K-means clustering | R | Group tickers by risk/return behavior per phase |
| Association rule mining | R | Identify SOXX-linked co-movement patterns |
| Chow structural break test | Python | Test for regime shifts at 2020-01-01 and 2023-01-01 |

Volatility is measured as the standard deviation of daily returns within each phase. Clustering operates on the (mean return, standard deviation) pair per ticker per phase, so a stock's cluster membership can be tracked as it migrates across regimes.

## Key Findings

**1. The 2020 break is statistically real; the 2023 one is not.**

| Break date | F-statistic | p-value | Result |
|------------|-------------|---------|--------|
| 2020-01-01 | 4.395 | 0.0124 | Significant structural break |
| 2023-01-01 | 0.701 | 0.4963 | No break detected |

The SOXX–SPY return relationship changed fundamentally at the onset of the shortage. Tariff and export-control tensions after 2023, despite dominating headlines, did not alter that relationship in a statistically detectable way.

**2. Semiconductor volatility rose sharply and did not fully revert.** SOXX volatility roughly doubled from Phase 1 into the shortage period, while SPY's increase was far smaller — the disruption was sector-specific, not a general market phenomenon.

**3. Exposure type predicted the shape of the shock, not just its size.** Directly impacted firms (AAPL, GM) saw volatility peak *during* the shortage and then normalize, consistent with acute operational constraints that resolved. Firms facing chronic competitive and geopolitical pressure (INTC) show a steadier upward climb in risk across all three phases instead of a single spike.

**4. Correlation structure fragmented and then partially reformed.** Phase 1 showed high baseline synchronization across the universe. Phase 2 broke it — AAPL's correlation with SOXX collapsed as the market priced company-specific supply exposure rather than sector trend. By Phase 3, AAPL had re-synchronized with SPY while NVDA remained largely uncorrelated with the semiconductor index, reflecting how idiosyncratic the AI growth story became.

**5. Cluster membership migrates in an interpretable way.** AAPL moves from the stable blue-chip cluster in Phase 1 into the highest-risk cluster in Phase 2, then back to the stable core in Phase 3 — a clean quantitative trace of a supply-chain scare arriving and resolving.

## Repository Contents

```
├── README.md
├── chips_data.csv                  # Extracted daily close prices, 2015–2025
├── Global_Chip_Shortage.docx       # Full written report
└── Global_Chip_Shortage.pptx       # Presentation deck
```

## Known Limitations

**Prices are unadjusted for corporate actions.** The Polygon close prices used here are raw, so stock splits inside the sample window appear as single-day price collapses. Five splits are affected:

| Ticker | Date | Split |
|--------|------|-------|
| AAPL | 2020-08-31 | 4:1 |
| NVDA | 2021-07-20 | 4:1 |
| SOXX | 2024-03-07 | 3:1 |
| NVDA | 2024-06-10 | 10:1 |
| AVGO | 2024-07-15 | 10:1 |

These inflate phase-level volatility estimates for the affected tickers and depress their mean returns, which in turn shifts cluster assignments and correlation values in Phases 2 and 3. The Chow test conclusions are robust to this, but the volatility and clustering figures for AAPL, NVDA, AVGO, and SOXX should be read with the caveat in mind. A split-adjustment layer is the primary planned revision.

**Missing data.** `AAPL` has a 43-day extraction gap from 2024-05-21 to 2024-07-23. Two full-market closures — 2018-12-05 (Bush state funeral) and 2025-01-09 (Carter state funeral) — are absent from the holiday exclusion list and appear as null rows.

**Ticker scope.** `TXN` is selected in the extraction query but does not appear in the exported dataset.

## Tech Stack

`PostgreSQL` · `Python` (pandas, statsmodels) · `R` (k-means, arules) · `SAP Predictive Analytics`
