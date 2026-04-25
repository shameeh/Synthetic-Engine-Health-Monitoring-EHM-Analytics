# Phase 3 — Anomaly Detection

<img width="2752" height="1536" alt="unnamed (4)" src="https://github.com/user-attachments/assets/542ccd7e-f647-4cf0-aea0-d1105c1e43b7" />


## What this phase does

Detects two distinct types of anomaly using three methods. The framing — two anomaly types, not one — came from understanding the physics before writing any code.

## Two anomaly types

**Type A — Sudden damage events**
Step-change in HPT efficiency caused by foreign object damage or compressor surge. Shows up as an instant drop in EGT margin. Flagged in `is_anomaly` and `maintenance_event` columns (113 rows across the fleet).

**Type B — Sensor fault anomalies**
Elevated spike rates on ENG-006 and ENG-011 (EGT sensor) and ENG-014 (vibration_1). Flagged in `has_faulty_sensor` and `faulty_sensor` columns.

## Four questions answered

| # | Question | Method |
|---|---|---|
| Q1 | Can CUSUM detect sudden damage events before EGT margin hits the red limit? | CUSUM on EGT_margin (k=0.5, h=5.0, baseline=200 cycles) |
| Q2 | Can Isolation Forest identify anomalous rows using multiple parameters? | sklearn IsolationForest, contamination=0.003, 6 engineered features |
| Q3 | Which engines have elevated sensor spike rates? | Rolling z-score, \|z\|>3 on EGT_margin and vibration_1 |
| Q4 | How do CUSUM and Isolation Forest compare against ground truth? | Precision and recall against is_anomaly labels |

## Why CUSUM

CUSUM (Page 1954) is the statistically optimal algorithm for detecting a sustained shift in process mean using the minimum number of observations. This matches exactly the physical nature of gas turbine degradation — a slow persistent drift rather than a sudden threshold exceedance. It requires sustained deviation before alarming, making it far less prone to false alarms than a simple threshold.

## Audit 3 findings

All three checks verified by hand. CUSUM calculation on ENG-004 at cycle 6251 built in Excel from scratch — result S = 5.4247 matches notebook to 4 decimal places.

Key finding: z-score spike method cannot reliably separate sensor faults from fast-degrading engines. ENG-014 vibration spike rate (0.50%) was barely distinguishable from ENG-001 (0.40%). Documented as a known limitation — a production system would apply z-score to ISA-corrected detrended parameters.

## Bridge to Phase 4

The CUSUM alarm cycle per engine becomes the **Initial Deterioration Point (IDP)** in Phase 4. This creates a direct, traceable link between anomaly detection and RUL prediction — CUSUM output becomes a Phase 4 input feature.
