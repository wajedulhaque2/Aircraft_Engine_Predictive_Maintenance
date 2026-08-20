# Portfolio Workbook

[`Aircraft_Engine_Predictive_Maintenance_Portfolio.xlsx`](Aircraft_Engine_Predictive_Maintenance_Portfolio.xlsx) is the cached reviewer copy of the final Excel model.

## Reviewer requirements

The workbook is designed for **Microsoft 365 desktop Excel** and uses:

- Power Query;
- Power Pivot / Data Model;
- DAX measures;
- dynamic-array formulas including `LET`, `FILTER`, `HSTACK` and `TAKE`;
- Solver for maintenance-window optimisation.

The cached analytical outputs can be reviewed without refreshing the NASA source data.

A full refresh requires the local NASA C-MAPSS FD001 source files and an updated Data Folder path on the workbook `Inputs` sheet. Solver is only required if the maintenance allocation is re-optimised.

For a quick review before opening the workbook, see the [3-page project summary](../docs/Aircraft_Engine_Project_Summary.pdf).
