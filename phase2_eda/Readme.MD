# Phase 2 — Exploratory Data Analysis

## What this phase does

Answers five analytical questions about the synthetic fleet from a maintenance controller's perspective. The questions came from the analyst, not from the code.

## Five questions answered

| # | Question | Method |
|---|---|---|
| Q1 | Which engines show out-of-bounds behaviour? | EGT margin trend lines, anomaly heatmap |
| Q2 | Where does each engine sit on the RAG scale? | Red/Amber/Green classification by margin, RUL, and degradation rate |
| Q3 | How many cycles remain for Amber and Red engines? | Linear regression on smoothed trend, projected to redline crossing |
| Q4 | Are sensors working correctly? | Dropout rate per sensor, outlier rate per engine (z-score > 3) |
| Q5 | Which route type degrades engines fastest? | Degradation rate per route, normalised trajectory comparison |

## Audit 2 findings

- All five analytical equations verified by hand on ENG-004 and ENG-017
- Discovered 100 rows with NaN in the raw EGT column — sensor dropout events, not a bug. Flagged as `EGT_dropout` column in Phase 4.
- Confirmed EGT_corrected formula: `(EGT_raw_K / theta) - 273.15` — verified on ENG-001 cycle 15625 (result: 692.2°C, matches CSV to rounding)
