# Workbook Architecture

The workbook is organised as a layered analytical model: reviewer-facing outputs first, then modelling / audit sheets, then query and raw-data layers.

## Recommended workbook navigation order

1. `Start_Here`
2. `Dashboard`
3. `Engine_Detail`
4. `Maintenance_Planner`
5. `Methodology`
6. `Inputs`
7. `Data_Quality`
8. `Reliability`
9. `Sensor_Profile`
10. `Engine_Summary`
11. `RUL_Model`
12. `Validation`
13. `Model_Data`
14. `PQ_Train`
15. `PQ_Test`
16. `Raw_Train`
17. `Raw_Test`
18. `Raw_Test_RUL`

## Analytical flow

```text
NASA FD001 source files
        |
        v
Power Query ingestion
        |
        +--> Raw_Train
        +--> Raw_Test
        +--> Raw_Test_RUL
        |
        v
Query enrichment
        |
        +--> PQ_Train
        |      - Max_Cycle
        |      - True_RUL
        |      - Model_Group
        |
        +--> PQ_Test
               - Max_Observed_Cycle
               - Is_Latest
        |
        v
Training / reliability layer
        |
        +--> Engine_Summary
        +--> Reliability
        +--> Sensor_Profile
        |
        v
Model_Data
        |
        +--> Health Score
        +--> Baseline predictions
        +--> Standardised sensors
        +--> Sensor model
        +--> Cycle² model
        +--> Final all-training features
        |
        v
RUL_Model + Validation
        |
        +--> model selection
        +--> held-out validation metrics
        +--> final NASA test metrics
        +--> Conservative RUL
        +--> maintenance status / priority
        |
        +----------------------+------------------+
        |                      |                  |
        v                      v                  v
Engine_Detail       Maintenance_Planner       Dashboard
                         |
                         v
                      Solver
```

## Sheet roles

| Sheet | Role |
|---|---|
| `Start_Here` | Reviewer entry point, workbook summary, navigation and refresh status |
| `Dashboard` | Executive reliability, model-performance and maintenance summary |
| `Engine_Detail` | Interactive individual-engine monitoring and trajectories |
| `Maintenance_Planner` | Top-risk fleet prioritisation and Solver scheduling |
| `Methodology` | Assumptions, definitions, modelling choices and limitations |
| `Inputs` | Data-folder path, maintenance thresholds, validation error allowance, window delays / capacities and Solver tie-break parameter |
| `Data_Quality` | Automated integrity and consistency checks |
| `Reliability` | Engine-life distribution, empirical survival and discrete hazard |
| `Sensor_Profile` | 21-sensor screening and final feature selection |
| `Engine_Summary` | One row per training engine with observed failure cycle and model group |
| `RUL_Model` | Baseline, development models, selected model and final all-training coefficients |
| `Validation` | Held-out validation, genuine NASA test results and current-fleet decision fields |
| `Model_Data` | Row-level training calculations and modelling features |
| `PQ_Train` | Power Query enriched training data |
| `PQ_Test` | Power Query enriched test data |
| `Raw_Train` | Query-loaded FD001 training source |
| `Raw_Test` | Query-loaded FD001 test source |
| `Raw_Test_RUL` | Query-loaded NASA test RUL target vector |

## Development / validation design

The 100 training engines are split by **engine trajectory**, not by individual rows:

```text
80 Development engines
20 Validation engines
```

Development data is used for:

- sensor screening;
- healthy / late-life Health Score reference values;
- sensor standardisation parameters during model development;
- regression fitting;
- model specification comparison.

Validation engines are held out from fitting and are used to compare the candidate RUL models and estimate the empirical absolute-error distribution.

Only after the model specification is locked is the chosen model refitted on all 100 training engines.

The NASA test RUL vector is then used once for the final unseen-engine evaluation.

## Power Pivot / Data Model

### Training model

```text
tblEngine[Unit_ID]
        1
        |
        |
        *
tblModelData[Unit_ID]
```

`tblEngine` is the one-row-per-training-engine dimension / summary table.

`tblModelData` contains row-level training observations and calculated model features.

This relationship supports engine counts, life-distribution measures and training-model PivotTables without duplicating engine-level metrics.

### Current NASA test fleet

`tblCurrentFleet` is intentionally **disconnected** from `tblEngine`.

This is an important design choice: test Engine 1 is a different simulated engine from training Engine 1. Relating the two tables by the numeric Unit_ID would create a false relationship.

Current-fleet DAX measures therefore operate directly on `tblCurrentFleet`.

## Power Pivot measures

The model contains measures for:

- engine count;
- Development and Validation engine counts;
- average observed life;
- median observed life;
- P10 and P90 observed life;
- average training Health Score;
- current engine count;
- average current predicted RUL;
- average Conservative RUL;
- Critical / Warning / Normal engine counts;
- average current Health Score.

Backend PivotTables on the Dashboard sheet are used to validate these measures and provide support for visible KPI cells.

## Reliability layer

`Reliability` converts the one-row-per-engine failure-cycle summary into an empirical survival table.

For each cycle the workbook calculates:

```text
At Risk
Failures
Empirical Survival
Discrete Hazard
```

Cached training-fleet life statistics:

```text
Mean    206.31 cycles
Median  199 cycles
P10     154.9 cycles
P90     275.1 cycles
Min     128 cycles
Max     362 cycles
```

## Sensor layer

`Sensor_Profile` evaluates all 21 channels and locks the five selected sensors:

```text
Sensor_11
Sensor_04
Sensor_12
Sensor_07
Sensor_15
```

The selection is formula-driven and based on Development data only.

## Model layer

`Model_Data` contains the row-level calculations needed for modelling:

- selected-sensor degradation fractions;
- Health Score;
- baseline RUL prediction;
- Development-standardised sensor features;
- sensor-model prediction and residual fields;
- Cycle² feature and model prediction;
- final all-training standardised features.

`RUL_Model` stores the model parameters, validation comparison and final all-training coefficients.

`Validation` contains both:

- held-out Validation-engine rows used for model comparison;
- final latest-observation rows for the 100 unseen NASA test engines.

## Maintenance decision-support layer

Each current test engine receives:

```text
Predicted RUL
Conservative RUL
Maintenance Status
Priority Rank
Health Score
```

The 20 highest-priority engines feed `Maintenance_Planner`.

Solver assigns those engines across four windows with delays of 0, 15, 30 and 45 cycles and capacity of five engines per window.

The Solver decision matrix is grouped / collapsible in the presentation build so reviewers can inspect the optimisation mechanics without permanently cluttering the sheet.

## Reviewer-facing outputs

### Dashboard

Answers the five main portfolio questions:

1. What does fleet life reliability look like?
2. How well did the locked RUL model perform on unseen engines?
3. How many engines currently require attention?
4. Which engines are highest priority?
5. How is constrained maintenance capacity allocated?

### Engine Detail

Provides engine-level inspection with:

- current cycle;
- predicted RUL;
- Conservative RUL;
- Health Score;
- maintenance status;
- priority rank;
- selected-sensor degradation trajectories;
- Health Score trajectory;
- actual-vs-predicted RUL trajectory for evaluation.

### Maintenance Planner

Shows the link from model output to operational decision support:

```text
Current fleet risk
→ ranked Top 20
→ urgency weights
→ binary Solver assignment
→ maintenance-window schedule
```

## Auditability principles

1. Raw and query output sheets are kept separate from presentation sheets.
2. Model assumptions are exposed on `Inputs` rather than buried inside formulas.
3. The model-development split is engine-level to prevent trajectory leakage.
4. Test engine IDs are not related to training engine IDs in the Data Model.
5. The empirical uncertainty allowance comes from held-out validation errors.
6. Maintenance thresholds and capacities are documented as portfolio assumptions.
7. Backend tables remain available for formula and Pivot-table audit even when hidden or grouped in the presentation view.
