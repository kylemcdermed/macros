# MACROS

**MACROS** is an experimental quantitative intraday trading framework designed to combine market structure, statistical regime detection, historical pattern similarity, and machine-learning-based feature analysis.

The central research question is:

> Can an hourly structural imbalance, combined with market regime, historical top-of-hour analogues, volatility, volume, and trend characteristics, provide statistically useful information about subsequent intraday price direction and trade outcomes?

The system is currently under active research and development.

---

## Performance Research Objectives

MACROS is being developed against explicit **out-of-sample research targets** rather than optimizing solely for in-sample profitability.

The current performance objectives are:

| Metric                      |      Target |
| --------------------------- | ----------: |
| **Sharpe Ratio**            | **3.0–4.0** |
| **Minimum Target Hit Rate** |     **60%** |
| **Goal Hit Rate**           |     **70%** |
| **Holding Period**          |    Intraday |
| **Position at EOD**         |        Flat |
| **Overnight Exposure**      |        None |

The primary objective is to investigate whether the complete MACROS framework can produce a **3–4 Sharpe ratio** while maintaining a minimum directional/trade hit rate of approximately **60%**, with a stretch objective of approximately **70%**.

These figures represent **research targets, not historical or expected performance**. They must ultimately be demonstrated through chronological out-of-sample and walk-forward testing after accounting for transaction costs, fees, slippage, and realistic execution assumptions.

### Day-over-Day Stability

A secondary objective is **DoD (day-over-day) consistency**.

The research will therefore evaluate not only aggregate hit rate and Sharpe ratio, but also the stability of performance across individual trading days, market regimes, volatility environments, and time-of-day segments.

The goal is not simply to achieve a high aggregate backtest hit rate. MACROS should investigate whether its edge remains sufficiently stable as market conditions evolve.

```text
                    MACROS TARGETS
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
       SHARPE          HIT RATE       STABILITY
       3.0–4.0            │              DoD
                          │
                    ≥60% Target
                          │
                     70% Goal
```

---

## Core Trading Hypothesis

MACROS begins with a structural market event: a **three-candle Fair Value Gap (FVG) / price imbalance**.

A qualifying imbalance must form in association with a **Short-Term High (STH)** or **Short-Term Low (STL)**.

Conceptually:

* Bullish opportunities are associated with an imbalance forming around/beyond a relevant short-term low.
* Bearish opportunities are associated with an imbalance forming around/beyond a relevant short-term high.
* New qualifying imbalances are evaluated on an hourly basis.
* Valid hourly imbalances can be carried forward as potential entry zones until they are triggered, invalidated, expire, or the trading session ends.

The structural imbalance generates the **trade candidate**. The quantitative models provide context for deciding whether that candidate has favorable characteristics.

---

## Model Architecture

```text
                    MARKET DATA
                         │
                         ▼
              FVG + STH/STL ENGINE
                         │
                         ▼
                 TRADE CANDIDATE
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
    30-Minute HMM   Weighted kNN    Feature Engine
      Regimes       xx:50→xx:10          │
          │              │               │
          │              │       ┌───────┴────────┐
          │              │       │                │
          │              │   Volatility         Volume
          │              │       │                │
          │              │       └───────┬────────┘
          │              │               │
          │              │      Linear Regression
          │              │               │
          └──────────────┼───────────────┘
                         ▼
                       XGBoost
                         │
                         ▼
                   TRADE / PASS
                         │
                         ▼
                    RISK ENGINE
                         │
                 ┌───────┼───────┐
                 ▼       ▼       ▼
                TP      SL      EOD
```

---

## Hidden Markov Model — Market Regime

MACROS will investigate a **Hidden Markov Model (HMM)** for estimating latent market regimes using **30-minute observation windows**.

Potential initial states include:

$$
Z_t \in \{Bullish,\ Bearish,\ Range\}
$$

The HMM estimates probabilities such as:

$$
P(Z_t = Bullish \mid X_{1:t})
$$

and transition probabilities:

$$
P(Z_{t+1}\mid Z_t)
$$

The HMM is therefore intended primarily as a **market-regime/context layer**, rather than as the entry mechanism itself.

---

## Weighted kNN — Historical Analogue Model

Weighted k-Nearest Neighbors will investigate the behavior of historical markets surrounding each top-of-hour transition.

Each observation window covers:

$$
xx{:}50 \rightarrow xx{:}10
$$

For example:

```text
09:50 -------- 10:00 -------- 10:10
      PRE-HOUR       POST-HOUR
        10m              10m
```

The research question is:

> Given what the market has done from `xx:50 → xx:10`, what happened subsequently during historically similar observations?

Potential features include:

* Pre-hour return
* Post-hour return
* Pre-hour range
* Post-hour range
* Total range
* High extension
* Low extension
* Closing location
* Range expansion/contraction
* Directional efficiency
* Session location
* Volatility
* Volume
* Regression characteristics

Closer historical observations receive greater influence:

$$
w_i=\frac{1}{d_i+\epsilon}
$$

Potential outputs include:

* \(P(Up)\)
* \(P(Down)\)
* Expected forward return
* Median forward return
* Neighbor dispersion
* Maximum Favorable Excursion (MFE)
* Maximum Adverse Excursion (MAE)

---

## XGBoost Feature Research

The initial core \(X\) feature families are:

### Volatility

* Realized volatility
* ATR
* Rolling standard deviation
* Range-normalized volatility
* Volatility expansion/contraction

### Volume

* Absolute volume
* Relative volume
* Rolling volume
* Volume ratios
* Volume associated with displacement/FVG formation

### Linear Regression

* Regression slope
* \(R^2\)
* Regression residual
* Standardized residual
* Distance from fitted trend
* Change in slope

Additional research may incorporate:

* FVG characteristics
* STH/STL characteristics
* HMM state probabilities
* kNN probabilities
* kNN expected returns
* kNN MFE/MAE
* Session location
* Time-of-day information

XGBoost and SHAP analysis will be used to investigate which features and feature interactions provide meaningful predictive information.

---

## Target Variable Research

The supervised-learning target \(Y\) remains an experimental design decision.

One primary candidate is:

$$
Y =
\begin{cases}
1,& \text{TP reached before SL}\\
0,& \text{SL reached before TP}
\end{cases}
$$

Alternative experiments may predict:

$$
Y=r_{forward}
$$

or:

$$
Y\in\{Up,Down,Flat\}
$$

These alternatives will be compared using chronological out-of-sample testing.

---

## Risk and Exit Research

Several approaches will be compared.

### Range + Volatility Barriers

Using the `xx:50 → xx:10` range:

$$
R=H-L
$$

potential barriers can be constructed using volatility:

$$
SL=L-k_{SL}\sigma
$$

$$
TP=H+k_{TP}\sigma
$$

### Entry-Centered Volatility Barriers

$$
SL=Entry-k_{SL}\sigma
$$

$$
TP=Entry+k_{TP}\sigma
$$

### Fixed Percentage Barriers

The original research hypothesis also includes fixed percentage barriers around entry, including approximately **±1.5%**.

### kNN-Conditioned MFE/MAE

Historical nearest neighbors may additionally provide empirical distributions of:

$$
MFE
$$

and:

$$
MAE
$$

which can potentially inform dynamically conditioned stop-loss and take-profit placement.

---

## Position Management

Every qualifying hourly imbalance may be carried forward as an active trade candidate.

Potential approaches include:

* One position maximum
* Multiple simultaneous hourly positions
* Pyramiding
* Confidence-weighted target exposure

The initial research implementation will favor **one open position at a time** to isolate the underlying signal's behavior.

---

## End-of-Day Risk Rule

MACROS is an intraday framework.

Every trade must terminate through:

$$
Exit\in\{TP,\ SL,\ EOD\}
$$

Any remaining open position is flattened before the market close.

No intentional overnight exposure is maintained.

---

## Validation

MACROS will use chronological **walk-forward validation** rather than relying on random train/test splits.

Evaluation will include:

* Sharpe ratio
* Hit rate
* Expectancy
* Profit factor
* Maximum drawdown
* Average winner / loser
* MFE / MAE
* Probability calibration
* Performance by hour
* Performance by HMM regime
* Performance by volatility regime
* Performance by kNN confidence
* Performance by XGBoost confidence
* **Day-over-day performance stability**

All features, normalization, regime estimation, neighbor selection, and model predictions must use only information available at the decision timestamp.

---

## Ablation Testing

Each component must demonstrate incremental out-of-sample value.

```text
FVG
 ↓
FVG + STH/STL
 ↓
+ HMM
 ↓
+ weighted kNN
 ↓
+ XGBoost
 ↓
FULL MACROS
```

The objective is not to maximize model complexity. The objective is to determine whether each additional component produces measurable improvements in **risk-adjusted out-of-sample performance**.

---

## Research Targets

```text
Sharpe Ratio
    │
    └────── TARGET: 3.0–4.0

Hit Rate
    │
    ├────── MINIMUM TARGET: ~60%
    │
    └────── STRETCH GOAL:   ~70%

Consistency
    │
    └────── Day-over-Day stability

Exposure
    │
    └────── Intraday only / Flat EOD
```

These are **objectives for the research process and are not claims of achieved performance**.

---

## Status

🚧 **Active Quantitative Research / Work in Progress**

MACROS currently represents a working research hypothesis. Model architecture, feature definitions, targets, risk parameters, and execution rules remain subject to empirical testing.

The project will prioritize **out-of-sample robustness, DoD consistency, statistical significance, realistic execution assumptions, and reproducibility** over optimized in-sample performance.
