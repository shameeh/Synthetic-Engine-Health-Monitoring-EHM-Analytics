# Phase 4 — Remaining Useful Life (RUL) Prediction

## What this phase does

Builds two RUL regression models (XGBoost and LSTM) and answers five analytical questions — covering accuracy, model comparison, feature importance, per-engine diagnostic triage, and initial condition effects.

## Three design decisions

These decisions distinguish this from a standard RUL implementation:

**Decision 1 — Engine-specific piecewise RUL cap**
Saxena (2008) introduced capping RUL at a fixed maximum during the healthy period because sensors cannot distinguish between very high RUL values. This project improves on that: instead of a fixed cap, the CUSUM alarm cycle from Phase 3 is used as the cap point — physically motivated and different for every engine.

**Decision 2 — post_IDP binary feature**
A binary column (0 before CUSUM alarm, 1 after) tells the model explicitly that degradation has started. This bridges Phase 3 output directly into Phase 4 features.

**Decision 3 — Baseline normalisation per engine**
Raw sensors normalised against each engine's own first-200-cycle baseline rather than fleet-wide. Removes manufacturing variation from the features — the model sees degradation drift, not absolute levels.

## Five questions answered

| # | Question | Key finding |
|---|---|---|
| Q1 | How accurately do XGBoost and LSTM predict RUL? | XGBoost RMSE 9,020 cycles vs LSTM 20,809 cycles. XGBoost wins clearly. |
| Q2 | Does LSTM outperform XGBoost in the 200 cycles after IDP? | XGBoost more accurate on all four test engines. Temporal memory did not pay off at this dataset size. |
| Q3 | Which features drive RUL predictions fleet-wide? | HPT_degradation dominates (mean SHAP = 8,918 cycles) — six times higher than the next feature. Matches GPA physics exactly. |
| Q4 | Which features drive each engine's prediction individually? | HPT_degradation is the top driver for all test engines. Secondary drivers vary (SFC, LPT, HPC) — providing a per-engine fault signature for maintenance triage. |
| Q5 | Do engines with lower starting EGT margin degrade faster? | Near-zero correlation (r = 0.018). Starting condition and degradation rate are independent in this dataset. |

## Why XGBoost outperformed LSTM

The dataset contains explicit degradation features (HPT_degradation, EGT_margin, SFC_degradation_pct) that directly encode the physics. XGBoost reads these as tabular features — it has the answer almost directly. LSTM must learn the same information from sequences of normalised sensor readings across 50-cycle windows, which requires more data and more training time than a ~1,750-cycle-per-engine synthetic dataset provides. In the C-MAPSS literature, LSTM's advantage appears with hundreds of engines and anonymised sensors — conditions this dataset does not replicate.

## SHAP physics check

The top 5 features by mean absolute SHAP value:

| Feature | Mean \|SHAP\| (cycles) |
|---|---|
| HPT_degradation | 8,918 |
| SFC_degradation_pct | 1,570 |
| LPT_degradation | 1,456 |
| HPC_degradation | 1,104 |
| fan_degradation | 580 |

This ordering matches the GPA influence coefficients exactly: HPT (IC=12) > HPC (IC=8) > LPT (IC=4.5) > Fan (IC=2). The model learned the correct physical relationships from the data.

## Known limitations

- Clean synthetic data overstates accuracy — no OAT scatter in EGT margin (see Assumption A-02)
- Run-to-threshold, not run-to-failure — different RUL definition than C-MAPSS
- LSTM would likely perform better with more engines or on C-MAPSS directly
- SHAP computed on XGBoost only (TreeSHAP — exact); LSTM SHAP (DeepSHAP) was not implemented
