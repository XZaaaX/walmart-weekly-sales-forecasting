# Walmart Weekly Sales Forecasting

End-to-end time-series forecasting pipeline for weekly retail sales at the
Store x Department level, built using the [Walmart Recruiting - Store Sales
Forecasting](https://www.kaggle.com/c/walmart-recruiting-store-sales-forecasting)
Kaggle dataset.

## Business Problem

Retailers need reliable demand forecasts to support inventory planning,
staffing, and promotional planning. This project forecasts weekly sales for
each Store x Department combination over a 39-week future horizon using
historical sales, store characteristics, holiday indicators, markdown
activity, and economic variables.

## Objective

Build a leakage-safe forecasting pipeline that:

- Predicts `Weekly_Sales` for an unseen 39-week period
- Evaluates models using chronological validation
- Selects the final model using validation MAE, keeping holdout blind
- Simulates real multi-step forecasting through recursive forecasting
- Translates forecast results into store- and department-level business insights

## Key Results

- **Final model:** XGBoost (tuned)
- **Validation MAE:** 1,417.5
- **Holdout MAE:** 1,272.1
- **Holdout WAPE:** 8.0%
- **Recursive 17-week MAE:** 1,655.4
- **Recursive error increase:** +30.1%
- **Production forecast horizon:** 39 weeks

## Dataset

| File | Rows | Description |
|---|---:|---|
| `train.csv` | 421,570 | Weekly sales by Store, Department, and Date |
| `test.csv` | 115,064 | Future dates requiring forecasts |
| `features.csv` | 8,190 | Economic, holiday, and markdown features |
| `stores.csv` | 45 | Store type and size information |

45 stores and 81 departments at a weekly grain. Historical sales cover
2010-02-05 to 2012-10-26; the forecast horizon covers 2012-11-02 to
2013-07-26.

## Methodology

### Data Preparation

- Parsed and validated date information
- Integrated sales, store, holiday, markdown, and economic data
- Checked duplicates, missing values, and data consistency
- Forward-filled time-varying economic variables (CPI, Unemployment) per
  store, in chronological order, to avoid leaking future values backward

### Feature Engineering

Features are constructed at the `Store x Department` level.

- Calendar features: Year, Month, WeekOfYear, Quarter, DayOfWeek
- Lag features: 1, 2, 4, 8, 12, and 52 weeks
- Rolling statistics: 4-, 8-, and 12-week means; 4- and 12-week standard deviations

Rolling features are calculated after shifting the target by one period, so
no future information enters the feature set.

## Validation Strategy

The project uses strictly chronological evaluation:

```text
Historical Data
      |
      +-- Fit
      +-- Validation
      +-- Holdout
```

Hyperparameter tuning uses date-based expanding-window cross-validation
rather than random or shuffled splits.

The model with the lowest **validation** MAE is selected. The holdout set is
reserved for a blind final evaluation and does not influence model
selection. Once selected, the model is **retrained on the full historical
data** (fit + validation + holdout, 261,083 rows through 2012-10-26, same
hyperparameters - not re-tuned) before producing the actual production
forecast, so the deployed model isn't left months behind the most recent
actuals while the reported metrics stay tied to the blind evaluation.

## Model Development

Four models were compared:

- Linear Regression
- Random Forest
- XGBoost
- LightGBM

Random Forest, XGBoost, and LightGBM were tuned using `RandomizedSearchCV`
with MAE-based scoring.

## Model Performance

| Model | Validation MAE | Holdout MAE | Holdout RMSE | Holdout WAPE | Holdout R2 |
|---|---:|---:|---:|---:|---:|
| **XGBoost** | **1,417.5** | 1,272.1 | 2,672.9 | 8.0% | 0.985 |
| Random Forest | 1,421.2 | **1,248.2** | **2,611.5** | **7.8%** | **0.986** |
| LightGBM | 1,449.1 | 1,292.6 | 2,718.4 | 8.1% | 0.985 |
| Linear Regression | 1,659.0 | 1,656.1 | 3,007.3 | 10.4% | 0.981 |

XGBoost was selected as the final model based on the lowest validation MAE.
Random Forest achieved slightly lower error on the holdout set, but the
holdout was not used for model selection.

## Recursive Forecasting

The 39-week test horizon is forecast recursively. Each prediction is
appended to the corresponding Store x Department history and becomes an
input for the next week's lag and rolling features - the same mechanism a
real production forecast needs, since future actuals don't exist yet.

```text
Historical Actual Sales
        |
Generate features for week T
        |
Predict week T
        |
Append prediction to history
        |
Generate features for week T+1
        |
Predict week T+1
        |
Continue until the forecast horizon is complete
```

A recursive backtest was also run over a 17-week historical holdout period,
where actuals exist, to measure error accumulation:

| Evaluation | MAE |
|---|---:|
| One-step (holdout) | 1,272.1 |
| Recursive, 17 weeks (holdout) | 1,655.4 |
| **Increase** | **+30.1%** |

This illustrates error accumulation when earlier predictions become inputs
for later forecasts. The 17-week finding is reported as-is and is not
extrapolated to the full 39-week production horizon, which was not directly
tested.

## Business Insights

- Store 4 has the highest total forecasted sales; Store 20 has the highest
  historical average weekly sales - forecast and historical rankings don't
  fully agree.
- Department 92 leads both in total forecast and historical average.
- Holiday weeks average ~7.1% higher sales than non-holiday weeks
  (association, not a causal claim).
- Recursive forecasting produces measurably higher error than one-step
  evaluation, motivating periodic monitoring and reassessment as new
  actuals become available.

## Limitations

- Hyperparameter search was intentionally limited for computational
  efficiency, so selected configurations may not be globally optimal.
- ~1.30% of Store-Department row pairs have a date gap other than 7 days
  (e.g. a department opening/closing mid-period), so a small fraction of
  lag features represent "the previous observation" rather than exactly
  one calendar week prior - documented rather than fixed with a full
  calendar reindex, given the small scale.
- Holdout MAE reflects one-step evaluation; recursive (multi-step) error is
  higher, as shown above.
- Forecasts and feature importances are associative, not causal estimates.

## Dashboard

A Power BI dashboard is planned as a business-facing companion to this
notebook, covering forecast trends, store/department breakdowns, and model
performance.

## Tech Stack

Python | Pandas | NumPy | Scikit-learn | XGBoost | LightGBM | SHAP |
Matplotlib | Seaborn | Jupyter

## Repository Structure

```text
walmart-weekly-sales-forecasting/
|
+-- README.md
+-- .gitignore
+-- requirements.txt
+-- walmart_sales_forecasting.ipynb
```

## Data Source

Raw CSV files are not committed to this repository. The dataset is
available through the Walmart Recruiting - Store Sales Forecasting
competition on Kaggle. Download `train.csv`, `test.csv`, `features.csv`,
and `stores.csv`, place them in a local `data/` folder next to the
notebook (not tracked by git - see `.gitignore`), before running the
notebook.
