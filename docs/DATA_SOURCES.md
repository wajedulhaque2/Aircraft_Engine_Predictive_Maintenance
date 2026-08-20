# Data Sources

## Primary dataset

This project uses the NASA **C-MAPSS Turbofan Engine Degradation Simulation Data Set**, subset **FD001**.

Official source pages:

- NASA Prognostics Center of Excellence Data Set Repository: https://www.nasa.gov/intelligent-systems-division/discovery-and-systems-health/pcoe/pcoe-data-set-repository/
- NASA Open Data — CMAPSS Jet Engine Simulated Data: https://data.nasa.gov/dataset/cmapss-jet-engine-simulated-data

Data set citation:

> A. Saxena and K. Goebel (2008), *Turbofan Engine Degradation Simulation Data Set*, NASA Prognostics Data Repository, NASA Ames Research Center, Moffett Field, CA.

## FD001 scope

FD001 is the simplest of the classic C-MAPSS subsets and contains:

| Property | FD001 |
|---|---:|
| Training engines | 100 |
| Test engines | 100 |
| Operating conditions | 1 |
| Fault modes | 1 |
| Training behaviour | Complete run-to-failure trajectories |
| Test behaviour | Trajectories truncated before failure |
| Test target | Final Remaining Useful Life supplied separately |

The source rows contain:

- engine / unit identifier;
- operational cycle;
- three operating settings;
- 21 sensor measurements.

## Source files expected by the workbook

The Power Query layer expects the FD001 files from the NASA distribution:

```text
train_FD001.txt
test_FD001.txt
RUL_FD001.txt
```

The workbook does not require the source files merely to review the cached portfolio outputs. They are required only for a full Power Query refresh.

## Refresh configuration

The local source-folder path is controlled from the workbook `Inputs` sheet.

For a fresh local build:

1. Download the NASA C-MAPSS source archive.
2. Place the three FD001 files in a local project data folder.
3. Open the workbook in Microsoft 365 desktop Excel.
4. Update the Data Folder input to the local folder containing the FD001 files.
5. Select **Data → Refresh All**.
6. Confirm that the `Data_Quality` checks pass before using refreshed outputs.

## Workbook source layers

| Workbook layer | Purpose |
|---|---|
| `Raw_Train` | Power Query-loaded FD001 training observations |
| `Raw_Test` | Power Query-loaded FD001 test observations |
| `Raw_Test_RUL` | NASA final RUL targets for the 100 test engines |
| `PQ_Train` | Training observations enriched with maximum cycle, true RUL and Development / Validation group |
| `PQ_Test` | Test observations enriched with maximum observed cycle and latest-observation flag |

## Dataset interpretation

Training trajectories continue until simulated failure. Test trajectories stop before failure. The predictive task is therefore to estimate the number of remaining operational cycles at the final observed point of each unseen test engine.

FD001 represents one operating condition and one fault mode. This makes it appropriate for an interpretable Excel portfolio model, but it is not representative of the full complexity of real aircraft fleet health monitoring.

## Redistribution note

This repository documents the official NASA source and the files expected by the workbook rather than repackaging the raw source archive. Reviewers can obtain the dataset from NASA using the links above.
