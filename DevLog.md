# Quantitative Strategy Pipeline: ML + Momentum Ensemble (5% Guardrail)


## NOTE
This document details the general plan of version 1.  Subject to change

## Executive Summary
This document outlines the architecture and execution of an institutional-grade, long-only quantitative trading strategy. The pipeline trades the 11 S&P 500 sector ETFs by blending a non-linear Machine Learning alpha model (LightGBM) with a traditional Trailing Momentum factor. The strategy strictly adheres to a **5% maximum active weight constraint** against a mathematically reconstructed Synthetic S&P 500 benchmark.

---

## Process Flow & Methodology

### 1. Data Ingestion & Leak-Proof Feature Engineering
* **Data Source:** Monthly historical ETF prices and macroeconomic indicators are loaded from a compressed dataset (`ETF_ML_Monthly_Dataset.gz`). Proxy tickers (IYR, VOX) are mapped to standard SPDR sector tickers (XLRE, XLC) to ensure a complete 20-year history.
* **Micro-Features:** The engine calculates raw trailing returns (1M, 3M, 6M) and trailing volatility (3M).
* **Leak Prevention:** All calculated features are explicitly shifted forward by one month. This guarantees that when the model predicts January's returns, it only has access to data ending in December, completely eliminating look-ahead bias.
* **Cross-Sectional Ranking:** Raw features are converted into cross-sectional percentile ranks (0.0 to 1.0) to normalize the data across differing market regimes.

### 2. Alpha Generation (LightGBM)
* **Target Definition:** The target variable is the percentile rank of the next month's return (identifying which sectors will relatively outperform). 
* **Polarized Training:** To reduce market noise, the model drops the middle 40% of returns and trains strictly on the extremes (the Top 30% and Bottom 30% of performers).
* **Out-of-Sample Walk-Forward Validation:** The pipeline utilizes a `TimeSeriesSplit` with a strict **10-Year Rolling Window** (1,320 rows). The model trains on a decade of data, predicts the next unknown block, and rolls forward. This generates a 20-year history of 100% Out-of-Sample (OOS) predictions.

### 3. Benchmark Construction (Synthetic SPY)
* **The Problem:** The actual S&P 500 index experiences "Beta Drag" as specific sectors (like Tech) grow massively over time, skewing historical performance comparisons.
* **The Solution:** The pipeline uses an **EWMA (Exponentially Weighted Moving Average) Optimizer** to mathematically reconstruct the S&P 500. It reverse-drifts the exact current index weights back through 20 years of history.
* **Result:** This produces an anchored, pure "Synthetic SPY" baseline with a low annualized tracking error (~1.51%), ensuring the ML strategy's alpha is measured accurately.

### 4. The Factor Ensemble (80/20 Blend)
The pipeline does not rely solely on the ML model. It utilizes a blended conviction score to rank the sectors:
* **80% Machine Learning:** The non-linear LightGBM probability that the sector will outperform.
* **20% 6-Month Momentum:** A cross-sectional percentile rank of the sector's trailing 6-month return.
* **Strategic Value:** The 20% momentum factor acts as a quantitative "circuit breaker," preventing the ML model from fighting massive, established cap-weighted trends (avoiding value traps).

### 5. Risk-Constrained Optimization (Rank and Fill)
Once the sectors are ranked by their blended conviction score, the optimizer allocates capital:
* **Baseline Assignment:** Every sector starts at its actual S&P 500 market-cap weight.
* **Top-Down Allocation:** The optimizer takes capital from the worst-ranked sectors and gives it to the best-ranked sectors.
* **The Guardrail:** No sector is allowed to deviate more than **+/- 5.0%** from its baseline SPY weight. This ensures strict adherence to institutional tracking error mandates.

### 6. Sensitivity Analysis & Validation
The pipeline stress-tests the 5% guardrail by running backtests across varying constraints (from 2% to 100% absolute freedom).
* **Finding:** Giving the model total freedom (100% guardrail) destroys the Information Ratio by exponentially increasing the Tracking Error. 
* **Optimization:** The mathematical peak efficiency of the 80/20 ensemble model occurs exactly at the 5% guardrail limit, proving the fund mandate aligns perfectly with the strategy's optimal risk profile (yielding an Information Ratio of ~0.29).

### 7. Production Execution
In the final stage, the pipeline shifts from historical backtesting to live production mode:
1.  Isolates the most recent month's macroeconomic and price data.
2.  Passes it through the frozen LightGBM model and Momentum calculator to get live 80/20 scores.
3.  Runs the 5% constrained optimizer.
4.  Outputs a detailed **Live Portfolio Allocation Plan**, specifying exactly which sectors to overweight, which to underweight, and the precise target portfolio weights for execution in the upcoming month.