# Synthetic Engine Health Monitoring (EHM) Analytics

A six-phase analytics project built on synthetic turbofan fleet data — covering data generation, exploratory analysis, anomaly detection, and remaining useful life (RUL) prediction.

Built to demonstrate domain understanding of aerospace prognostics, not just Python skills.

---

## What is Engine Health Monitoring?

Gas turbine engines degrade over time. The high-pressure turbine (HPT) blades erode, compressor efficiency drops, and fuel consumption rises. Left unmanaged, this degradation eventually causes the exhaust gas temperature (EGT) to exceed its certified limit — forcing an unscheduled engine removal.

EHM is the discipline of tracking this degradation in real time using on-wing sensor data, so that maintenance can be planned before limits are reached rather than after. The primary metric is **EGT margin** — the gap between the current takeoff EGT and the certified redline. As an engine degrades, that gap narrows. When it reaches a minimum threshold, the engine goes to the shop.

The goal of modern EHM is to predict *when* the margin will reach that threshold — the **Remaining Useful Life (RUL)** — accurately enough to optimise maintenance scheduling across an entire fleet.

---

## Project Structure

```
ehm-analytics/
│
├── data/
│   └── ehm_synthetic_fleet_v3.csv        # 35,000 rows, 20 engines, 36 columns
│
├── docs/
│   ├── EHM_Modelling_Assumptions.pdf     # Seven modelling assumptions with prepared answers
│   ├── EHM_Project_Scope.pdf             # Six-phase project roadmap
│   └── EHM_Project_Handover_v2.pdf       # Full project continuity document
│
├── phase1_data_generator/
│   ├── EHM_Phase1_v3_Rebuild.ipynb       # Synthetic fleet data generator
│   └── README.md
│
├── phase2_eda/
│   ├── EHM_Phase2_EDA_Beginner.ipynb     # Fleet health exploratory analysis
│   └── README.md
│
├── phase3_anomaly_detection/
│   ├── EHM_Phase3_AnomalyDetection.ipynb # CUSUM + Isolation Forest
│   └── README.md
│
├── phase4_rul_prediction/
│   ├── EHM_Phase4_RUL_Prediction.ipynb   # XGBoost + LSTM RUL models + SHAP
│   └── README.md
│
├── requirements.txt
└── README.md                             # This file
```

---

## The Dataset

The dataset is synthetic, generated from first principles using Gas Path Analysis (GPA) — the same mathematical framework used in real EHM systems.

| Attribute | Value |
|---|---|
| Engines | 20 (ENG-001 to ENG-020) |
| Total rows | 35,000 (~1,750 cycles per engine on average) |
| Columns | 36 |
| Engine type | Two-spool turbofan (CFM56-5B class) |
| RAG breakdown | 10 Green, 8 Amber, 2 Red |
| RUL range | ~3,700 to ~64,000 cycles |
| Degradation rate | ~6.3°C EGT margin loss per 1,000 EFC (real CFM56 range: 3–7°C) |
| Faulty sensors | ENG-006 (EGT), ENG-011 (EGT), ENG-014 (vibration) |

Every value in the dataset traces back to a documented physical equation. The modelling assumptions document in `/docs` records every simplification made, why it was made, and what effect it has on the analysis.

---

## Phase Summary

### Phase 1 — Synthetic Data Generator

Builds the fleet dataset from scratch using:
- ISA (International Standard Atmosphere) corrections for temperature and pressure
- Gas Path Analysis influence coefficients (HPT: 12°C/%, HPC: 8°C/%, LPT: 4.5°C/%, Fan: 2°C/%)
- Stochastic degradation trajectories with compressor wash recovery events
- Injected sensor faults and step-change damage events

Three bugs were caught and fixed during Audit 1 — including a Celsius/Kelvin error in the EGT correction formula that was producing temperatures 75°C too high.

### Phase 2 — Exploratory Data Analysis

Five analytical questions answered from a maintenance controller's perspective:
- Which engines show out-of-bounds behaviour?
- Where does each engine sit on the RAG (Red/Amber/Green) scale?
- How many cycles remain for Amber and Red engines?
- Are sensors working correctly?
- Which route type degrades engines fastest?

### Phase 3 — Anomaly Detection

Two anomaly types detected using two methods:

| Method | Target | Result |
|---|---|---|
| CUSUM | Sudden step-change damage events | Detects sustained EGT margin shifts with advance warning before the event |
| Isolation Forest | Multivariate outlier rows | Useful for ranking engine health; low precision against row-level labels (expected and explained) |
| Z-score spike rate | Sensor fault anomalies | Correctly identifies all three faulty sensor engines |

Key insight: CUSUM is the statistically optimal algorithm for detecting sustained mean shifts — matching exactly the physical nature of gas turbine degradation. Foundation: Page (1954).

### Phase 4 — RUL Prediction

Two models built and compared across five analytical questions:

**Q1 — Accuracy:** XGBoost outperformed LSTM on this dataset (RMSE 9,020 vs 20,809 cycles). The explicit degradation features (HPT_degradation, EGT_margin) give XGBoost a direct signal that LSTM must learn from sequences — an advantage on a dataset of this size.

**Q2 — Post-IDP window:** XGBoost was more accurate than LSTM in the 200 cycles immediately after the CUSUM alarm fired on all four test engines. LSTM's temporal memory did not provide an advantage here — most likely a dataset size effect.

**Q3 — Fleet SHAP:** HPT_degradation dominated predictions with a mean SHAP impact of 8,918 cycles — six times higher than the next feature. This matches exactly what the GPA influence coefficients predict: HPT operates at the highest temperature and contributes most to EGT margin erosion. The model learned the correct physics.

**Q4 — Per-engine SHAP:** Each engine's top SHAP driver was HPT_degradation. The secondary drivers varied (SFC, LPT, HPC) — providing a per-engine fault signature that a maintenance controller could use to direct inspection effort.

**Q5 — Initial condition effect:** Near-zero correlation (r = 0.018) between starting EGT margin and degradation rate. In this dataset, the two are independent by construction. Whether this holds in real fleets — where route type, operating environment, and build quality interact — is an open question.

Three design decisions distinguished this from a standard RUL implementation:
1. **Engine-specific IDP** — piecewise RUL cap using the CUSUM alarm cycle rather than a fixed value (improvement on Saxena 2008)
2. **post_IDP feature** — explicit binary flag bridging Phase 3 CUSUM output into Phase 4 features
3. **Baseline normalisation** — per-engine z-score against each engine's own first-200-cycle baseline, removing manufacturing variation from the features

---

## Key References

| Paper | Why it matters |
|---|---|
| Page, E.S. (1954). Continuous Inspection Schemes. *Biometrika*, 41(1/2), 100–115. | CUSUM theoretical foundation — proves optimality for mean shift detection |
| Saxena, A. et al. (2008). Damage Propagation Modeling for Aircraft Engine Run-to-Failure Simulation. *PHM Conference*. | C-MAPSS dataset paper — the standard RUL benchmark in the field |
| Heimes, F.O. (2008). Recurrent Neural Networks for Remaining Useful Life Estimation. *PHM Conference*. | First application of LSTM to turbofan RUL on C-MAPSS |
| Jaw & Mattingly (2009). Aircraft Engine Controls. AIAA. | GPA influence coefficient framework |
| Lundberg & Lee (2017). A Unified Approach to Interpreting Model Predictions. *NeurIPS*. | TreeSHAP — exact feature importance for tree models |

---

## Limitations

This project uses synthetic data. Several known simplifications affect how directly the results transfer to real EHM:

- **No OAT scatter in EGT margin** — real data has ±3–5°C residual scatter after ISA correction, making anomaly detection harder than it appears here
- **Run-to-threshold, not run-to-failure** — RUL is defined as cycles to the EGT redline, not to actual engine failure; C-MAPSS uses run-to-failure
- **Single operating condition** — no altitude or Mach variation beyond OAT; C-MAPSS has six discrete operating conditions
- **Uniform degradation mode** — all engines degrade via the same HPT/HPC/LPT/fan mechanism; real fleets show more diverse failure signatures

Full documentation of assumptions and their Phase 4 impact: `/docs/EHM_Modelling_Assumptions.pdf`

---

## Requirements

```
pandas
numpy
matplotlib
seaborn
scikit-learn
xgboost
shap
tensorflow
```

Full pinned versions: `requirements.txt`

---

## Running the Notebooks

All notebooks are designed for **Google Colab**. Open each notebook in Colab, upload `ehm_synthetic_fleet_v3.csv` when prompted, and run cells top to bottom.

Phases must be run in order (1 → 2 → 3 → 4) as each phase builds on the dataset and findings of the previous one.

---

*Built by Shameeh (Shami) Rahman | April 2026*
*MSc Business Analytics, Warwick Business School*
