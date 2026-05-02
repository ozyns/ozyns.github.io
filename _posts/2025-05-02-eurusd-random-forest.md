---
title: "EUR/USD Forecasting with Random Forest — A Machine Learning Approach"
date: 2025-05-02 00:00:00 +0000
categories: [Machine Learning, Finance]
tags: [python, random-forest, forex, time-series, sklearn, yfinance]
math: true
---

> **TL;DR** — Raw price data alone isn't useful. We engineered 36 features (returns, volatility, RSI, lag windows), used a proper time-based split to avoid leakage, and trained a Random Forest reaching R² = 0.95 and 74.76% directional accuracy. Built by a team of three, each handling a key part: data, modeling, and evaluation.

---

## Motivation

Forex prediction is one of the classic hard problems in ML — the signal-to-noise
ratio is brutal, markets are non-stationary, and naive models overfit almost
immediately. I chose EUR/USD as my target for this project because it's the
most liquid currency pair in the world, which means cleaner data and fewer
anomalous spikes.

The goal was not to build a trading bot, but to practice the full supervised
learning pipeline: data collection → feature engineering → model selection →
evaluation → deployment.

---

## Architecture Overview

The project is split into three clean classes, each owned by a different
"role" in the team:

```
EnhancedDataEngineer      →  collect_data()  +  create_enhanced_features()
OptimizedModelDeveloper   →  prepare_data()  +  train_model()  +  evaluate_model()
ModelEvaluator            →  create_plots()
```

This separation made it easy to swap the model later (e.g. replace Random
Forest with Gradient Boosting) without touching the data layer.

---

## Data Collection

Data is pulled live from Yahoo Finance using `yfinance`:

```python
eurusd = yf.download('EURUSD=X', start=start_date, end=end_date, progress=False)
```

Two additional correlated assets are also fetched as exogenous features:

| Symbol | Asset  | Rationale                              |
|--------|--------|----------------------------------------|
| GC=F   | Gold   | Safe-haven flows correlate with EUR    |
| CL=F   | Oil    | Petrodollar dynamics affect USD supply |

If any download fails, the pipeline falls back to synthetic data so training
is never blocked.

---

## Feature Engineering

This was the most impactful step. Raw close prices are nearly useless for a
tree model — what matters is *change*, *momentum*, and *context*.

### Features created (30+ total)

| Category         | Features                                              |
|-----------------|-------------------------------------------------------|
| Returns          | `return_1d`, `return_3d`, `return_7d`                |
| Volatility       | rolling std over 7 and 14 days                       |
| Moving averages  | MA(7), MA(21), MA(50) + crossover ratios             |
| Momentum / RSI   | RSI(14) + overbought/oversold signal                 |
| Lag features     | price and return lags at 1, 2, 3, 5, 7, 14 days     |
| Rolling stats    | 7-day high, low, range                               |
| Temporal         | day of week, month, quarter                          |
| Exogenous        | Gold/Oil returns and 7-day MAs                       |

The RSI is computed manually to avoid any library dependency:

$$RSI = 100 - \frac{100}{1 + RS}, \quad RS = \frac{\text{avg gain}}{\text{avg loss}}$$

---

## Model

```python
RandomForestRegressor(
    n_estimators    = 200,
    max_depth       = 15,
    min_samples_split = 10,
    min_samples_leaf  = 4,
    max_features    = 'sqrt',
    bootstrap       = True,
    n_jobs          = -1,
    random_state    = 42
)
```

**Why Random Forest over a simple regression?**

- Handles non-linear interactions between features (e.g. RSI × volatility)
- Robust to outliers and missing values
- Built-in feature importance for interpretability
- No need to scale features

The train/test split is **chronological** (no shuffle) to avoid look-ahead
bias — the last 20% of dates form the test set.

---

## Results

<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>

<div id="chart-container" style="position:relative; height:320px; margin:2rem 0;">
  <canvas id="predChart"></canvas>
  <p id="chart-loading" style="text-align:center; color:#888; padding-top:4rem;">Loading chart…</p>
</div>

<script>
(async () => {
  try {
    const res  = await fetch('/assets/data/eurusd/predictions.json');
    const data = await res.json();
    document.getElementById('chart-loading').style.display = 'none';
    const ctx = document.getElementById('predChart').getContext('2d');
    new Chart(ctx, {
      type: 'line',
      data: {
        labels: data.dates,
        datasets: [
          {
            label: 'Actual',
            data: data.actual,
            borderColor: '#58a6ff',
            backgroundColor: 'transparent',
            borderWidth: 2,
            pointRadius: 0,
            tension: 0.3,
          },
          {
            label: 'Predicted',
            data: data.predicted,
            borderColor: '#f78166',
            backgroundColor: 'transparent',
            borderWidth: 2,
            pointRadius: 0,
            tension: 0.3,
            borderDash: [4, 3],
          },
        ],
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        interaction: { mode: 'index', intersect: false },
        plugins: {
          legend: { labels: { color: '#c9d1d9' } },
          tooltip: {
            callbacks: {
              label: ctx => `${ctx.dataset.label}: ${ctx.parsed.y.toFixed(5)}`,
            },
          },
        },
        scales: {
          x: {
            ticks: { color: '#8b949e', maxTicksLimit: 8, maxRotation: 30 },
            grid: { color: 'rgba(48,54,61,0.6)' },
          },
          y: {
            ticks: { color: '#8b949e' },
            grid: { color: 'rgba(48,54,61,0.6)' },
          },
        },
      },
    });
  } catch(e) {
    document.getElementById('chart-loading').textContent = 'Chart failed to load.';
  }
})();
</script>

<div id="feature-chart-container" style="position:relative; height:340px; margin:2rem 0;">
  <canvas id="featureChart"></canvas>
</div>

<script>
(async () => {
  const res  = await fetch('/assets/data/eurusd/feature_importance.json');
  const data = await res.json();
  const ctx  = document.getElementById('featureChart').getContext('2d');
  new Chart(ctx, {
    type: 'bar',
    data: {
      labels: data.features,
      datasets: [{
        label: 'Importance',
        data: data.importances,
        backgroundColor: 'rgba(88, 166, 255, 0.7)',
        borderColor: '#58a6ff',
        borderWidth: 1,
      }]
    },
    options: {
      indexAxis: 'y',
      responsive: true,
      maintainAspectRatio: false,
      plugins: { legend: { display: false } },
      scales: {
        x: { ticks: { color: '#8b949e' }, grid: { color: 'rgba(48,54,61,0.6)' } },
        y: { ticks: { color: '#c9d1d9', font: { size: 11 } }, grid: { color: 'rgba(48,54,61,0.6)' } }
      }
    }
  });
})();
</script>

### Metrics

<div id="metrics-container">
  <p style="color:#888;">Loading metrics…</p>
</div>

<script>
(async () => {
  const res = await fetch('/assets/data/eurusd/metrics.json');
  const m   = await res.json();

  document.getElementById('metrics-container').innerHTML = `
    <table>
      <thead>
        <tr><th>Metric</th><th>Value</th><th>Notes</th></tr>
      </thead>
      <tbody>
        <tr><td>R²</td><td><strong>${m.r2}</strong></td><td>Variance explained</td></tr>
        <tr><td>RMSE</td><td>${m.rmse}</td><td>Root mean squared error (in rate units)</td></tr>
        <tr><td>MAE</td><td>${m.mae}</td><td>Mean absolute error</td></tr>
        <tr><td>MAPE</td><td>${m.mape}%</td><td>Mean absolute percentage error</td></tr>
        <tr><td>Directional accuracy</td><td>${m.directional_accuracy}%</td><td>% of up/down moves correctly predicted</td></tr>
        <tr><td>Train samples</td><td>${m.train_samples}</td><td></td></tr>
        <tr><td>Test samples</td><td>${m.test_samples}</td><td></td></tr>
        <tr><td>Features</td><td>${m.total_features}</td><td></td></tr>
        <tr><td>Model</td><td>${m.model}</td><td>n_estimators=${m.n_estimators}, max_depth=${m.max_depth}</td></tr>
        <tr><td>Generated</td><td colspan="2">${m.generated_at}</td></tr>
      </tbody>
    </table>
  `;
})();
</script>

---

## Evaluation Charts

![Evaluation report — 4-panel figure](/assets/img/evaluation_report.png)
_From top-left: actual vs predicted time series · scatter with R² · error
distribution · top-12 feature importances_

The error distribution is centred very close to zero with no heavy tail on
either side — a good sign that the model is not systematically biased.

The most important features are (unsurprisingly) recent lag values and
short-term moving average crossovers. The RSI signal and volatility features
contribute meaningful lift on top.

---

## Limitations

- **Look-ahead risk** — features like `MA_50` require 50 days of history, so
  the model cannot be used in a true real-time setting without a warm-up window.
- **Non-stationarity** — the model is retrained periodically; old parameters
  decay as market regimes shift.
- **No macro features** — interest rate differentials (ECB vs Fed), CPI prints,
  and geopolitical events are not captured.
- **This is not a trading signal** — high R² on price levels is expected
  because prices are autocorrelated. Directional accuracy is the more honest
  measure of predictive skill.

---

*Source code: [github.com/ozyns/EUR-USD-Forecasting-with-Random-Forest](https://github.com/ozyns/EUR-USD-Forecasting-with-Random-Forest)*
