# Phase 1 — Synthetic Data Generator

## What this phase does

Generates a realistic synthetic fleet dataset for a two-spool turbofan engine (CFM56-5B class). One row per flight cycle per engine. Every value traces back to a documented physical equation.

## Key equations implemented

| Equation | Formula | Source |
|---|---|---|
| ISA temperature correction | `theta = T_ambient_K / 288.15` | ICAO ISA standard |
| ISA pressure correction | `delta = P_ambient / 1013.25` | ICAO ISA standard |
| EGT correction (Kelvin) | `EGT_corrected = (EGT_raw_K / theta) - 273.15` | Standard EHM practice |
| N1/N2 correction | `N1_corrected = N1 / sqrt(theta)` | Standard EHM practice |
| Fuel flow correction | `FF_corrected = FF / (delta * sqrt(theta))` | Standard EHM practice |
| GPA influence coefficient | `delta_EGT = (HPT*12) + (HPC*8) + (LPT*4.5) + (fan*2)` | Urban (1975); Jaw & Mattingly (2009) |
| EGT margin | `margin = EGT_redline - (EGT_takeoff_new + delta_EGT * 1.1)` | Standard EHM definition |
| RUL | `RUL = (margin - margin_limit) / EGT_rate_per_cycle` | Derived |

## Audit 1 findings

Three bugs were caught by hand-verifying a sample row:

| ID | Issue | Fix |
|---|---|---|
| BUG1 | EGT_corrected computed in Celsius instead of Kelvin — off by ~75°C | `EGT_corrected = (EGT_raw_K / theta) - 273.15` |
| BUG2 | Degradation rate floor clamped at 0.001 — causing 5x too-fast degradation | Floor lowered to 0.00002 |
| CAL1 | Degradation rate 5x too aggressive (23°C per 1,000 EFC) | Recalibrated to ~6.3°C per 1,000 EFC |

## Output

`ehm_synthetic_fleet_v3.csv` — 35,000 rows, 20 engines, 36 columns.
