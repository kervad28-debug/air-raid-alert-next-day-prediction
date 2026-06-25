# Next-Day Air-Raid Alert Prediction for Khmelnytska Oblast

## Project overview

This mini-project was created for Stage 2 of the KSE AI Agentic Summer School selection.

The project studies a daily time series of air-raid alert starts in Khmelnytska oblast, Ukraine. Its purpose is to test whether recent alert history and simple calendar information can help predict whether at least one air-raid alert will start on the following calendar day.

The final model is an interpretable Logistic Regression classifier. It is compared with simple baselines using chronological train, validation and test periods.

The project is educational and analytical. It must not be used as an operational warning system.

## Research question

Can recent air-raid alert history in Khmelnytska oblast predict whether at least one alert will start on the following calendar day?

For each day (t):

* `alert_started_today = 1` if at least one alert started on day (t);
* `alert_started_today = 0` otherwise;
* `alert_started_tomorrow` equals the occurrence value on day (t+1).

Calendar days are defined using the `Europe/Kyiv` timezone.

## Repository contents

```text
.
├── README.md
├── requirements.txt
├── .gitignore
├── notebooks/
│   └── air_raid_alert_analysis.ipynb
├── data/
│   ├── README.md
│   └── volunteer_data_en_snapshot_2026-06-25.csv
├── figures/
│   ├── monthly_alert_day_percentage.png
│   ├── rolling_30_day_alert_frequency.png
│   ├── validation_model_comparison.png
│   ├── final_test_model_comparison.png
│   ├── test_confusion_matrix_logistic_regression.png
│   └── test_confusion_matrix_persistence.png
├── outputs/
│   ├── final_test_metrics.csv
│   └── logistic_regression_pipeline.joblib
```

## Dataset and reproducibility

The source data comes from the public `volunteer_data_en.csv` file in the Ukrainian air-raid sirens dataset repository:

```text
https://raw.githubusercontent.com/Vadimkin/ukrainian-air-raid-sirens-dataset/main/datasets/volunteer_data_en.csv
```

Because the live CSV is updated over time, a fixed snapshot was saved for reproducibility.

Snapshot details:

* filename: `volunteer_data_en_snapshot_2026-06-25.csv`;
* extraction date: June 25, 2026;
* number of rows: 101,969;
* number of columns: 4;
* columns: `region`, `started_at`, `finished_at`, `naive`;
* SHA-256 checksum:

```text
0354ac29bf8086fa8ca0e51c5bd63041bdb4d8671da93dc189a40cf684f62a38
```

The checksum can be used to verify that the snapshot has not changed.

The raw timestamps are stored in UTC. They were converted to `Europe/Kyiv` before assigning events to calendar days.

## Selected region and observation window

Khmelnytska oblast was selected using explicit criteria:

* long and recent coverage;
* substantial numbers of positive and negative days;
* class balance close to 50%;
* enough alert records for lag and rolling features;
* no obvious region-specific coverage problem.

The fixed daily observation window is:

```text
2022-02-26 through 2026-06-24
```

This common full-day window avoids defining coverage only from the first and last alert observed in the region.

The resulting complete daily series contained:

* 1,580 calendar days;
* 801 days with at least one alert start;
* 779 days with no recorded alert start;
* no duplicate dates;
* no missing dates;
* no missing values.

After creating the following-day target, the last row was removed because its next-day outcome was outside the observation window. This produced 1,579 modelling rows before feature-history removal.

## Data-quality findings

The full snapshot contained:

* 0 failed datetime conversions;
* 0 exact duplicate rows;
* 0 negative durations;
* 5 zero-duration records;
* 5,023 records with `naive=True`;
* approximately 4.93% estimated endings.

All `naive=True` records had a duration of exactly 30 minutes.

The occurrence target uses alert start timestamps only. Therefore, records with estimated endings were retained for occurrence modelling.

For duration analysis, the transparent rule was to exclude:

* `naive=True` records;
* zero or negative durations;
* exact duplicates, if present.

Unusually long positive durations were not automatically removed because the available columns do not prove that they are invalid.

Duration was inspected for data-quality purposes, but duration was not used as the prediction target.

## Daily time-series construction

For Khmelnytska oblast:

1. `started_at` was converted from UTC to `Europe/Kyiv`.
2. The local calendar date was extracted.
3. Alert starts were counted for each date.
4. A complete date range was created.
5. Dates without a recorded alert start were assigned a count of zero.
6. A binary daily occurrence variable was created.
7. The next-day target was produced by shifting the daily occurrence variable by one day.

The sum of daily alert counts matched the 1,265 Khmelnytska alert records whose local start dates fell inside the fixed observation window.

## Features

The final frozen feature set contains 12 predictors.

### Current-day features

* `alert_started_today`
* `alert_count_today`

These are known by the end of day (t).

### Lag features

* `alert_started_lag_1`
* `alert_started_lag_2`
* `alert_started_lag_7`
* `alert_count_lag_1`

These use only days before (t).

### Rolling features

* `alert_frequency_7d`, calculated over (t-6) through (t);
* `alert_frequency_30d`, calculated over (t-29) through (t);
* `alert_count_sum_7d`, calculated over (t-6) through (t).

The rolling windows include day (t), which is valid because the prediction is assumed to be made after day (t) is complete. They never include day (t+1).

### Recency feature

* `days_since_last_alert`

A value of:

* 0 means an alert occurred on day (t);
* 1 means the latest alert day was (t-1);
* 2 means it was (t-2), and so on.

The feature was calculated by carrying the most recent past alert date forward. It does not use future information.

### Calendar features

* `tomorrow_day_of_week`
* `tomorrow_month`

These describe the day being predicted and are known in advance.

The first 29 modelling rows were removed because they lacked a complete 30-day rolling history.

The final feature dataset contained 1,550 observations.

## Model and preprocessing

The final model is Logistic Regression.

Numerical features were standardized using `StandardScaler`.

The two calendar variables were treated as categorical variables using `OneHotEncoder`.

Preprocessing and Logistic Regression were combined in one scikit-learn `Pipeline`. This ensured that scaling values, encoded categories and model coefficients were learned only from the fitting period.

Frozen model settings:

```text
LogisticRegression(
    random_state=42,
    max_iter=1000
)
```

The classification threshold was fixed at 0.5.

No hyperparameter tuning, threshold adjustment, feature removal or alternative model testing was performed after validation.

## Time-series validation

Random shuffling was not used.

The fixed chronological periods were:

* training: through 2025-03-06;
* validation: 2025-03-07 through 2025-10-29;
* test: 2025-10-30 through 2026-06-23.

The training period was used to fit preprocessing and model parameters.

The validation period was used to evaluate the frozen feature set and Logistic Regression configuration.

After the modelling decisions were frozen, the model was refitted using the combined training and validation periods and evaluated once on the final test period.

## Baselines

### Majority-class baseline

This baseline predicts the most common class from the training period for every future observation.

### Historical-frequency baseline

This baseline assigns the training-period positive frequency as the same probability for every future observation.

Because the probability is constant, it has no ranking ability and produces ROC-AUC of 0.5 when both classes are present.

### Persistence baseline

The persistence rule is:

```text
prediction for tomorrow = alert occurrence today
```

This is a simple time-series baseline that tests whether the next day tends to resemble the current day.

Persistence was the main benchmark for the final model.

## Distribution shift

The positive-target rate changed over time:

* training positive rate: 56.02%;
* validation positive rate: 41.35%.

This indicates temporal distribution shift. The probability of an alert day was not constant across the observation period.

The changing class balance means that a model fitted on earlier data may overpredict alerts during later periods, even when its implementation is technically correct.

## Validation results

Validation results for Logistic Regression:

| Metric            | Logistic Regression | Persistence |
| ----------------- | ------------------: | ----------: |
| Accuracy          |              0.5612 |      0.5359 |
| Balanced accuracy |              0.5356 |      0.5215 |
| Precision         |              0.4634 |      0.4388 |
| Recall            |              0.3878 |      0.4388 |
| F1                |              0.4222 |      0.4388 |
| ROC-AUC           |              0.5350 |    Not used |

Logistic Regression improved validation accuracy, balanced accuracy and precision, but persistence achieved higher recall and F1.

## Final test results

The final test period was evaluated once after freezing the model, features, preprocessing and threshold.

| Metric            | Logistic Regression |  Persistence |
| ----------------- | ------------------: | -----------: |
| Accuracy          |              0.5907 |       0.5654 |
| Balanced accuracy |              0.5073 |       0.5239 |
| Precision         |              0.3654 |       0.3810 |
| Recall            |              0.2289 |       0.3855 |
| F1                |              0.2815 |       0.3832 |
| ROC-AUC           |              0.4979 | Not reported |

Logistic Regression confusion matrix:

```text
[[121, 33],
 [ 64, 19]]
```

Persistence confusion matrix:

```text
[[102, 52],
 [ 51, 32]]
```

Logistic Regression achieved higher raw accuracy because it correctly classified more no-alert days.

However, persistence performed better on:

* balanced accuracy;
* precision;
* recall;
* F1.

The Logistic Regression ROC-AUC of 0.4979 indicates that its probability ranking was effectively no better than random during the final test period.

The main conclusion is therefore that the Logistic Regression model did not provide a practically strong improvement over the simple persistence rule.

## Figure interpretations

### Monthly percentage of alert days

The percentage of days with at least one recorded alert start varies across months. This shows that the frequency of alert days was not constant over the observation period. The figure is descriptive and does not explain why the changes occurred.

### 30-day rolling alert frequency

The rolling series contains periods with relatively high and low recent alert frequency. This supports the observation that the data distribution changed over time. Each point uses only the current day and the preceding 29 days.

### Validation model comparison

On validation data, Logistic Regression achieved higher accuracy, balanced accuracy and precision than persistence. Persistence achieved higher recall and F1, indicating a different balance between missed alerts and false positive predictions.

### Final test model comparison

On the final test period, Logistic Regression achieved higher raw accuracy, but persistence performed better on the class-sensitive metrics. The higher Logistic Regression accuracy did not represent consistently better prediction of both classes.

### Logistic Regression confusion matrix

Logistic Regression correctly classified many no-alert days but missed a substantial number of actual alert days. This is consistent with its low recall during the final period.

### Persistence confusion matrix

Persistence identified more actual alert days than Logistic Regression but also produced more false positives. Its stronger balanced accuracy and F1 indicate more even performance between the two classes.

## Test-set disclosure

During initial split verification, the class distribution of the held-out test period was accidentally printed.

This counts as limited test-set inspection because it revealed information about future target balance.

However:

* no test predictions were calculated at that stage;
* no test performance metrics were calculated;
* no feature selection used test performance;
* no hyperparameter tuning used test performance;
* no threshold selection used test performance;
* no model selection used test performance.

After the issue was identified, the predefined test period was kept unchanged and was not used again until the single final evaluation.

This limitation is disclosed to keep the evaluation process transparent.

## Limitations

1. The dataset is volunteer-collected and may contain missed or delayed records.
2. Absence of a record is treated as no recorded alert start, but this cannot prove that every zero is a true absence.
3. The project predicts alert declarations, not attacks, damage or physical danger.
4. Daily aggregation removes exact timing, within-day intensity and alert duration.
5. Days with one alert and days with several alerts share the same binary target.
6. The model uses only historical alert and calendar information.
7. Important external variables are absent, including military, geographic, weather, intelligence and political information.
8. Relationships in the series changed over time.
9. Logistic Regression assumes a relatively stable relationship between predictors and the target, which may not hold.
10. The final test ROC-AUC indicates very weak probability ranking.
11. The test target distribution was accidentally viewed before final evaluation, although it was not used for model development.
12. Results for Khmelnytska oblast should not be generalized automatically to other regions.

## Responsible-use disclaimer

This project is an educational time-series exercise.

It is not an emergency-warning product and must not be used to make personal safety decisions.

Official air-raid alert systems and instructions from Ukrainian authorities should always be followed. A model based on historical data cannot reliably predict future attacks or replace official warning infrastructure.

## Running the notebook

### Google Colab

1. Open `notebooks/air_raid_alert_analysis.ipynb` in Google Colab.
2. Upload `data/volunteer_data_en_snapshot_2026-06-25.csv` when requested, or mount Google Drive.
3. Make sure the notebook uses the fixed snapshot rather than downloading the live CSV.
4. Run the notebook cells from top to bottom.
5. Confirm that the printed SHA-256 checksum matches:

```text
0354ac29bf8086fa8ca0e51c5bd63041bdb4d8671da93dc189a40cf684f62a38
```

6. Generated figures should be saved in `figures/`.
7. Final metrics and the fitted pipeline should be saved in `outputs/`.

### Local Python environment

Create and activate a virtual environment:

```bash
python -m venv .venv
```

On macOS or Linux:

```bash
source .venv/bin/activate
```

On Windows:

```bash
.venv\Scripts\activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

Start Jupyter:

```bash
jupyter notebook
```

Open the notebook and run cells in order.

## AI-assisted development

This project was developed with AI assistance.

The AI was used to:

* identify and compare public dataset options;
* explain Python and pandas operations;
* design validation checks;
* identify potential leakage;
* construct baseline evaluation;
* explain modelling decisions;
* assist with documentation.

The workflow remained interactive. Code was run in Google Colab, actual outputs were pasted back for verification, and assumptions were corrected when they did not match the observed data.

The complete AI conversation log is submitted separately through the Stage 2 submission form.

## Main conclusion

The project demonstrates that a machine-learning model can achieve higher raw accuracy without being better across both target classes.

Logistic Regression reached test accuracy of 0.5907, but persistence achieved stronger balanced accuracy, precision, recall and F1.

The final ROC-AUC of 0.4979 indicates that the model probabilities did not provide useful ranking during the test period.

For this dataset and evaluation period, recent alert persistence was a stronger and more reliable benchmark than the fitted Logistic Regression model.
