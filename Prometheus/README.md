# Prometheus V2.2.4 – Changelog

![version](https://img.shields.io/badge/version-2.2.4-blue)
![status](https://img.shields.io/badge/status-stable-green)
![build](https://img.shields.io/badge/build-patch--4-lightgrey)
![language](https://img.shields.io/badge/language-Bash-blue)
![modulation](https://img.shields.io/badge/modulation-supported-success)
![fan%20ratio](https://img.shields.io/badge/fan_ratio-dynamic-success)
![flame%20quality](https://img.shields.io/badge/flame_quality-validated-success)
![autocalibration](https://img.shields.io/badge/calibration-interactive%20+%20adaptive-orange)

---

## Overview

Prometheus V2.2.4 introduces a major refinement in gas estimation logic by incorporating **adaptive correction multipliers** based on **fan speed ratio** and **flame sensor quality**. This version increases precision and resilience under real-world data volatility, and maintains full compatibility with prior `.env` configurations and success log formats.

---

## What's New in 2.2.4

| Feature                        | Description                                                                 |
|-------------------------------|-----------------------------------------------------------------------------|
| **Fan Ratio Adjustment**       | Calculates the normalized fan speed across the cycle (min–max interpolation) |
| **Flame Quality Index**       | Evaluates signal noise and strength from `FlameSensingASIC` for dynamic adjustment |
| **Dual-Multiplier Correction**| Multiplies modulation-based gas estimation by combined fan/flame coefficient |
| **Fallback-Ready**            | In absence of real sensor values, fallback thresholds are used automatically |
| **Expanded JSON Output**      | Output includes both raw and corrected gas, fan/flame stats, and multipliers |

---

## Calculation Pipeline

Gas estimation in this version follows a multi-stage process:

1. **Base Estimate** – Derived from modulation average and cycle duration  
2. **Fan Ratio Calculation** – Normalized position of `FanSpeed` within min/max  
3. **Flame Quality Adjustment** – Based on ASIC reading trend (good if <40, bad if >80)  
4. **Dynamic Multiplier** – `1 + (fan_ratio * fan_coeff) - (flame_quality * flame_coeff)`  
5. **Physiological Loss Correction** – Adds percentage defined in `.env`  
6. **Final Value** – Total consumption in m³, appended to JSON log and notification

---

## JSON Output Example

```json
{
  "start": 1,
  "timestamp_start": "2025-05-20 09:14:00",
  "end": 0,
  "timestamp_end": "2025-05-20 09:17:12",
  "total_time": 192,
  "modulation_avg": "52.3",
  "fan_avg": "4987",
  "fan_min": "4840",
  "fan_max": "5160",
  "flame_avg": "61.2",
  "fan_ratio": "0.735",
  "flame_quality": "0.472",
  "fan_ratio_multiplier": "0.1",
  "flame_quality_multiplier": "0.1",
  "correction_percent": "10",
  "gas_modulation_m3": "0.14321",
  "final_consumption_m3": "0.15179",
  "total_consumption_m3": "0.16697"
}
```

> **Note:** If values like `fan_avg` or `flame_avg` are missing in the JSON, it means **fallback values were automatically applied**. These are still visible in the debug log for full traceability.

---

## Improvements Over Previous Versions

| Area                     | v2.2.3                          | v2.2.4 (current)                                    |
|--------------------------|----------------------------------|----------------------------------------------------|
| Fan/Flame correction     | Not present                     | Dual multiplier logic                              |
| Fallback support         | Partial (modulation only)       | Full fallback for modulation, fan, flame           |
| JSON output              | Basic modulation + gas          | Expanded with full telemetry and correction factors |
| Log structure            | Simple                          | Verbose with DEBUG sections and breakdowns         |
| Telegram notifications   | Basic per cycle                 | Full-cycle summary with reliability estimation     |

---

## Fallback Handling

| Sensor                 | Fallback Behavior                             |
|------------------------|------------------------------------------------|
| `ModulationTempDesired`| Uses default modulation (e.g. 60%)             |
| `FanSpeed`             | Uses preset min/avg/max defined in `.env`     |
| `FlameSensingASIC`     | Uses neutral value (e.g. 60) and penalty 0.5   |

All fallback applications are logged explicitly in `debug_prometheus_v2.log`.

---

## Logging Enhancements

Each completed cycle now generates:

- **One JSON object** appended to `prometheus_success_v2.log`
- **One structured line** in `debug_prometheus_v2.log` with intermediate steps:
  - Raw modulation power
  - Useful power (after efficiency)
  - MJ/h value
  - Gas before correction
  - Correction amount
  - Final consumption (m³)

---

## Reliability and Accuracy Notes

| Condition                        | Expected Accuracy Deviation |
|----------------------------------|-----------------------------|
| Full data (mod, fan, flame)      | ±1.5%                       |
| Missing fan or flame             | ±3–5% (uses fallback)       |
| No modulation collected          | No estimation for that cycle|

Estimates converge over time thanks to persistent correction mechanisms (`gas_correction.sh`).

---

## Compatibility

- Fully compatible with V2.0 and V2.1 `.env` and `.log` structures  
- Correction script continues to read from both `prometheus_success.log` and `prometheus_success_v2.log`  
- No change in `.env` required unless adjusting fallback values

---

## Summary

Version 2.2.4 represents a meaningful step forward in gas monitoring precision by introducing **adaptive logic** that reacts to actual fan and flame conditions. While preserving the lightweight Bash-only architecture, it allows for smarter estimation with better fault recovery. Logs remain clean, JSON-ready, and integrate seamlessly with Telegram and calibration tools.

> Prometheus V2.2.4 is designed for long-term, low-maintenance, real-world deployments with enhanced precision and robustness.
