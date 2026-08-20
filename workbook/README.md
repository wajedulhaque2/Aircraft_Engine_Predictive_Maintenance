# Portfolio Workbook

## Reviewer requirements

The workbook is designed for **Microsoft 365 desktop Excel** and uses:

- Power Query;
- Power Pivot / Data Model;
- DAX measures;
- dynamic-array formulas such as `LET`, `FILTER`, `HSTACK` and `TAKE`;
- Solver for maintenance-window optimisation.

The cached analytical outputs can be reviewed without refreshing the NASA source data.

A full refresh requires the local FD001 source files and an updated Data Folder path on the workbook `Inputs` sheet.

## Before publishing a new workbook version

1. Confirm `Data_Quality` is fully passing.
2. Confirm the final NASA test KPIs on `Dashboard` and `Validation` agree.
3. Confirm Solver status is `PASS` and each maintenance window is within capacity.
4. Confirm the workbook opens without external-link or repair warnings.
5. Use Excel **File → Info → Check for Issues → Inspect Document** and remove unnecessary document properties / personal information from the public copy.
6. Ensure no personal local folder path is visible on `Inputs` or embedded in the public workbook copy.
7. Save the public file as `Aircraft_Engine_Predictive_Maintenance_Portfolio.xlsx`.
