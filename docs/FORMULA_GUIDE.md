# Formula, DAX & Power Query Guide

This guide documents the most important logic used in the Aircraft Engine Health Monitoring & Predictive Maintenance workbook.

## 1. Core training labels

### Training true RUL

For each training observation:

```text
True RUL = Engine Maximum Cycle − Current Cycle
```

In the query / model layer this is equivalent to:

```excel
=Max_Cycle-Cycle
```

Each training engine is assigned wholly to either Development or Validation so observations from the same trajectory cannot appear in both model fitting and validation.

### Latest test observation

For the NASA test fleet:

```text
Is_Latest = Current Cycle = Engine Maximum Observed Cycle
```

Only the final observed row of each test engine is used for the genuine 100-engine NASA test evaluation.

---

# 2. Sensor screening

All 21 sensor channels are profiled before modelling. Screening is performed on Development engines only.

The final selected sensors are:

```text
Sensor_11
Sensor_04
Sensor_12
Sensor_07
Sensor_15
```

The screening layer considers:

- variability / near-constant behaviour;
- correlation with operational cycle;
- correlation with true RUL;
- healthy-to-late-life mean shift;
- standardised effect size.

The purpose is to select sensors systematically rather than choosing visually attractive trajectories after seeing the final test set.

---

# 3. Health Score

For each selected sensor, a degradation fraction is calculated from Development-fleet healthy and late-life reference values.

Conceptually:

```text
Sensor Degradation =
    CLAMP(
        (Current Sensor Value − Healthy Reference)
        / (Late-Life Reference − Healthy Reference),
        0,
        1
    )
```

The Excel implementation follows the pattern:

```excel
=MAX(0,MIN(1,(CurrentValue-HealthyReference)/(LateLifeReference-HealthyReference)))
```

Five selected sensors are equally weighted:

```text
Health Score = 100 × (1 − AVERAGE(selected-sensor degradation))
```

Interpretation:

- 100 = healthy-like relative to Development references;
- 0 = late-life-like relative to Development references.

Health Score is a project-defined analytical indicator, not a certified engine-health metric.

---

# 4. Naive RUL baseline

The baseline uses the Development-fleet mean observed failure cycle:

```text
Baseline Predicted RUL = MAX(0, Development Mean Failure Cycle − Current Cycle)
```

The workbook Development mean observed failure cycle is approximately:

```text
208.2 cycles
```

The baseline establishes whether the sensor model adds predictive value beyond engine age alone.

---

# 5. Sensor standardisation

For regression, each selected sensor is converted to a z-score:

```text
Z(sensor) = (Current Value − Training Mean) / Training Standard Deviation
```

During model development the mean and standard deviation are estimated from Development data only.

After model selection is locked, the final model is refitted using means and standard deviations from all 100 training engines.

---

# 6. RUL regression progression

## Model 1

```text
RUL ~ Cycle
    + Z(Sensor_11)
    + Z(Sensor_04)
    + Z(Sensor_12)
    + Z(Sensor_07)
    + Z(Sensor_15)
```

## Selected model

```text
RUL ~ Cycle
    + Cycle²
    + Z(Sensor_11)
    + Z(Sensor_04)
    + Z(Sensor_12)
    + Z(Sensor_07)
    + Z(Sensor_15)
```

Cycle² is retained because it improves held-out validation performance relative to the simpler sensor model.

The Excel model is estimated with `LINEST`.

---

# 7. Final all-training model coefficients

After the specification is locked, the model is refitted on all 100 training engines.

Approximate final coefficients:

| Term | Coefficient |
|---|---:|
| Intercept | 183.728957 |
| Cycle | -1.097564 |
| Cycle² | 0.00262324 |
| Z(Sensor_11) | -9.846685 |
| Z(Sensor_04) | -7.577748 |
| Z(Sensor_12) | 5.259866 |
| Z(Sensor_07) | 5.077983 |
| Z(Sensor_15) | -5.831868 |

The final prediction equation is therefore approximately:

```text
Predicted RUL = 183.728957
              − 1.097564 × Cycle
              + 0.00262324 × Cycle²
              − 9.846685 × Z(Sensor_11)
              − 7.577748 × Z(Sensor_04)
              + 5.259866 × Z(Sensor_12)
              + 5.077983 × Z(Sensor_07)
              − 5.831868 × Z(Sensor_15)
```

Negative final predictions are clamped where required for decision-support use.

---

# 8. Model evaluation metrics

For prediction error:

```text
Error = Predicted RUL − Actual RUL
Absolute Error = ABS(Error)
Squared Error = Error²
```

### MAE

```excel
=AVERAGE(Absolute_Error_Range)
```

### RMSE

```excel
=SQRT(AVERAGE(Squared_Error_Range))
```

### Bias

```excel
=AVERAGE(Error_Range)
```

Positive bias means the model is optimistic on average and predicts more remaining life than is actually observed.

### Predictive R²

```text
Predictive R² = 1 − SSE / SST
```

Excel pattern:

```excel
=1-SUM(Squared_Error_Range)/DEVSQ(Actual_RUL_Range)
```

## Held-out validation result for selected Cycle² model

```text
MAE            24.94 cycles
RMSE           31.88 cycles
Bias            9.76 cycles
Predictive R²   0.746
```

## Genuine NASA final test result

```text
MAE            22.85 cycles
RMSE           28.44 cycles
Bias           +9.71 cycles
Predictive R²   0.532
Test engines      100
```

---

# 9. Empirical prediction uncertainty

The workbook calculates the 90th percentile of held-out validation absolute error:

```excel
=PERCENTILE.INC(Validation_Absolute_Error_Range,0.9)
```

Cached portfolio value:

```text
55.90146 cycles
```

This is described as a **90th-percentile validation error band**. It is not a statistical confidence interval.

### Conservative RUL

```excel
=MAX(0,Predicted_RUL-P90_Validation_Error)
```

This deliberately turns model uncertainty into a more cautious maintenance-planning variable.

---

# 10. Maintenance classification

Portfolio thresholds are controlled from `Inputs`.

```text
Critical threshold = 30 cycles
Warning threshold  = 60 cycles
```

Classification logic:

```excel
=IF(Conservative_RUL<=Critical_Threshold,
    "Critical",
    IF(Conservative_RUL<=Warning_Threshold,
       "Warning",
       "Normal"))
```

The thresholds are modelling assumptions for demonstration and are not approved aviation maintenance limits.

---

# 11. Priority ranking

Many engines can have Conservative RUL = 0, so ranking requires deterministic tie-breaking.

The workbook ranks by:

1. lower Conservative RUL;
2. lower Predicted RUL;
3. lower Engine ID.

Excel pattern:

```excel
=1
 +COUNTIF(Conservative_RUL_Range,"<"&Current_Conservative_RUL)
 +COUNTIFS(Conservative_RUL_Range,Current_Conservative_RUL,
           Predicted_RUL_Range,"<"&Current_Predicted_RUL)
 +COUNTIFS(Conservative_RUL_Range,Current_Conservative_RUL,
           Predicted_RUL_Range,Current_Predicted_RUL,
           Engine_ID_Range,"<"&Current_Engine_ID)
```

This produces unique ranks from 1 to 100.

---

# 12. Solver urgency weight

The Top 20 priority engines receive an urgency weight:

```excel
=1/(Conservative_RUL+1)+TieBreakWeight/(Predicted_RUL+1)
```

where:

```text
TieBreakWeight = 0.001
```

The small second term preserves the primary Conservative-RUL ordering while differentiating engines tied at the same conservative value.

---

# 13. Solver objective and constraints

Binary decision variables assign each of the Top 20 engines to maintenance windows A–D.

For each engine:

```text
Assigned Total = A + B + C + D = 1
```

Scheduled delay:

```text
Scheduled Delay =
    A × 0
  + B × 15
  + C × 30
  + D × 45
```

Weighted delay:

```text
Weighted Delay = Urgency Weight × Scheduled Delay
```

Solver objective:

```text
Minimise SUM(Weighted Delay)
```

Subject to:

- each decision variable is binary;
- every selected engine is assigned exactly once;
- each maintenance window contains no more than five engines.

The cached portfolio solution assigns five engines to each window and returns `PASS` for the schedule-status control.

---

# 14. Current-fleet Power Pivot measures

The NASA current test fleet is stored in a disconnected Data Model table so test engine IDs are never related to training engine IDs.

### Current Engine Count

```DAX
Current Engine Count :=
DISTINCTCOUNT(tblCurrentFleet[Unit_ID])
```

### Average Current RUL

```DAX
Average Current RUL :=
AVERAGE(tblCurrentFleet[Predicted_RUL])
```

### Average Conservative RUL

```DAX
Average Conservative RUL :=
AVERAGE(tblCurrentFleet[Conservative_RUL])
```

### Critical Engine Count

```DAX
Critical Engine Count :=
CALCULATE(
    [Current Engine Count],
    tblCurrentFleet[Maintenance_Status] = "Critical"
)
```

The Warning and Normal measures use the same pattern.

### Average Current Health Score

```DAX
Average Current Health Score :=
AVERAGE(tblCurrentFleet[Health_Score])
```

---

# 15. Training-model Power Pivot measures

### Engine Count

```DAX
Engine Count :=
DISTINCTCOUNT(tblEngine[Unit_ID])
```

### Average Observed Life

```DAX
Average Observed Life :=
AVERAGE(tblEngine[Observed_Failure_Cycle])
```

### Median Observed Life

```DAX
Median Observed Life :=
MEDIAN(tblEngine[Observed_Failure_Cycle])
```

### P10 / P90 observed life

```DAX
P10 Observed Life :=
PERCENTILEX.INC(
    tblEngine,
    tblEngine[Observed_Failure_Cycle],
    0.1
)
```

```DAX
P90 Observed Life :=
PERCENTILEX.INC(
    tblEngine,
    tblEngine[Observed_Failure_Cycle],
    0.9
)
```

### Development / Validation counts

```DAX
Development Engine Count :=
CALCULATE(
    [Engine Count],
    tblEngine[Model_Group] = "Development"
)
```

```DAX
Validation Engine Count :=
CALCULATE(
    [Engine Count],
    tblEngine[Model_Group] = "Validation"
)
```

---

# 16. Important Power Query logic

Power Query performs the ingestion and structural transformations before the analytical formulas are applied.

Key transformations include:

```text
Training:
source rows
→ typed columns
→ engine maximum cycle
→ true row-level RUL
→ Development / Validation engine group

Test:
source rows
→ typed columns
→ maximum observed cycle by engine
→ Is_Latest flag
```

The workbook also uses a small refresh-time query based on:

```powerquery
DateTime.LocalNow()
```

so the `Start_Here` / Dashboard refresh timestamp changes when `Refresh All` is run.

---

# 17. Model design principles

1. **Split by engine, not row.** This prevents trajectory leakage between fitting and validation.
2. **Screen sensors before final test evaluation.** The NASA test RUL targets are not used to choose sensors or model specification.
3. **Use a baseline first.** Sensor regression must demonstrate value beyond engine age alone.
4. **Increase complexity only when validation improves.** Cycle² is retained because the held-out metrics improve.
5. **Lock before final test.** The final NASA test is treated as a genuine unseen evaluation.
6. **Keep uncertainty visible.** Conservative RUL is derived from held-out validation error rather than hidden behind a point estimate.
7. **Keep maintenance assumptions explicit.** Thresholds, capacities and tie-break weights are parameters, not aviation rules.
8. **Keep test and training engine identities separate.** The current test fleet is intentionally disconnected in the Data Model.
