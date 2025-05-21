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

---

# Prometheus V2.0 – Gas Monitoring System

![version](https://img.shields.io/badge/version-2.0-blue)
![status](https://img.shields.io/badge/status-beta-orange)
![license](https://img.shields.io/badge/license-CC--BY--NC%204.0+-lightgrey)
![security scan](https://img.shields.io/badge/security-scanned%20with%20leakscan.py-green)
![language](https://img.shields.io/badge/language-Bash-blue)

> Real-time gas usage estimation for Vaillant eBUS boilers using modulation and flame state analysis.  
> Designed for long-term unattended monitoring, fine-tuning, and self-correction.

> **Why “Prometheus”?**
> 
> After exhausting every alternative — from contacting the gas supplier, attempting to access official meter data, exploring certified sensors and industrial devices, to testing multiple estimation algorithms and statistical models — this approach emerged as the most scientifically grounded and technically feasible solution available without invasive or costly installations.
> 
> The name "Prometheus" is inspired by the myth of Prometheus, who defied the gods to bring fire to mankind. In the same spirit, this project attempts to illuminate what has long been kept deliberately opaque: accurate, real-time gas consumption.  
> 
> Manufacturers, thermostats, and even gas providers often keep this data locked away, fragmented, or proprietary — making it difficult or impossible for users to monitor their real usage. **This project challenges that by reclaiming visibility and control.**

---

<details>
<summary><strong>Project Origins – Context and Technical Motivation</strong></summary>

This project originates from the limitations encountered when trying to retrieve reliable, real-time gas consumption data from **Vaillant eBUS boilers**, even when using advanced integrations like **ebusd** and **Home Assistant**.

### Initial Framework

The foundation of this system is inspired by the excellent work of:

- [john30/ebusd](https://github.com/john30/ebusd) – the core tool for accessing boiler registers over eBUS.
- [john30/ebusd-esp32](https://github.com/john30/ebusd-esp32) – a DIY ESP32-based eBUS WiFi interface.

Using these tools, we built a **custom control unit** and began testing readings directly from the eBUS line, avoiding proprietary or cloud-dependent gateways.

---

### Limitations of Standard Integrations

Despite enabling MQTT autodiscovery in Home Assistant, we encountered several roadblocks:

- Some key registers (e.g., `Flame`, `Gasvalve`) were **not available** or always returned cached/stale values.
- Queries via `ebusctl read` without `-f` flag would frequently fail or return **incorrect data**.
- MQTT-sourced values were **delayed or approximated**, unsuitable for real-time gas estimation.

These shortcomings rendered traditional integrations **unusable for accurate logging** or scientific monitoring.

---

### Real-Time Access Strategy

We identified that many of the boiler's critical values were indeed accessible — but only via **forced command-line calls**, such as:

```bash
docker exec -it <ebusd_container> ebusctl read -f Flame
```

> Without the `-f` flag, most values were either cached or wrong.

By directly querying with SSH into the Home Assistant host and executing the required commands, we regained full control over **live data**.

---

### Verified Command Examples

```bash
# Real-time flame state (only reliable method)
docker exec -it <container> ebusctl read -f Flame

# Optional: gas valve status
docker exec -it <container> ebusctl read -f Gasvalve

# Discover available registers
docker exec -it <container> ebusctl find | grep -i flame
docker exec -it <container> ebusctl find | grep -i gas
```

Results were then tested against boiler behavior to ensure accuracy.

---

### Diagnostic Journey

A multi-step diagnostic confirmed that most `bai` registers were inaccessible using regular discovery:

```bash
# Step 1 – Read without -f: failed
ebusctl read Flame  → always returned "off"

# Step 2 – Search with `find -f on`: no result

# Step 3 – grep bai circuit: mostly "no data stored"

# Final working method:
ebusctl read -f Flame  → returned correct state
```

Thus, we built a script-based pipeline to:

- Read **ModulationTempDesired** and **Flame** every few seconds.
- Detect **flame cycles** and calculate **gas consumption** per event.
- Log and notify via Telegram with full JSON output.

---

### SSH Setup (Bidirectional)

To achieve this, we created a **bidirectional SSH link** between the Raspberry Pi (running the script) and the Home Assistant host (running ebusd):

#### From Home Assistant → Raspberry Pi
```bash
cat ~/.ssh/id_rsa.pub | ssh pi@RASPBERRY_IP \
"mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"
```

#### From Raspberry Pi → Home Assistant
```bash
cat ~/.ssh/id_rsa.pub | ssh homeassistant@HOMEASSISTANT_IP \
"mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"
```

SSH keys were exchanged with appropriate `chmod` protections.

---

### Resulting Architecture

- **All logic** runs on the Raspberry Pi, avoiding unnecessary load on Home Assistant.
- **All reads** are forced and real-time, independent from MQTT cache.
- **All logging** is local, permanent, and non-cloud-dependent.
- **All cycle data** is stored as structured logs and pushed optionally to Google Sheets.

---

### Why “Prometheus”?

After months of testing, vendor refusals, inaccessible APIs, cloud limitations, and failed attempts to access the official gas meter readings, this was the **only viable and truly scientific method** to estimate gas consumption without invasive hardware.

Like **Prometheus in Greek mythology**, who brought fire (knowledge) to humans despite divine restrictions, this script is a symbolic act of **technological defiance and data liberation**.

</details>

---

## Overview

Prometheus V2 is a complete rewrite and enhancement of the original Prometheus V1 gas monitoring script.  
It runs continuously in background, logs individual heating cycles with estimated gas usage in cubic meters, and optionally sends notifications via Telegram.

The system reads boiler telemetry from `ebusd` (running in Docker on Home Assistant or any Linux machine), using a direct SSH command interface for maximum reliability.

---

## Key Features

- **Real-time estimation** of gas usage per ignition cycle (based on `ModulationTempDesired` and `Flame`)
- **Continuous operation** via infinite loop and background start on boot
- **High reliability** with:
  - Alternated polling (2s loop)
  - Telegram alerts
  - JSON logs for each cycle
- **Self-correction module**: compare real gas meter readings and auto-adjust estimation accuracy
- **Reminder system** for weekly gas meter check via Telegram

---

## Why Prometheus V2?

Compared to V1:

- All settings are moved into external `.env` config files
- Modular design: main script, watchdog, cleanup, correction
- Mathematical estimation based on real thermodynamic formula:
  
  ```
  gas_m³ = ((MAX_POWER * modulation_avg / 100) * efficiency * 0.0036 / PCI) * (duration / 3600)
  ```

- Real data-driven loop with fail-safe checks, time boundaries, and log isolation
- No hardcoded paths or tokens – all secrets are in `.env` files (example templates provided)

---

## Repository Structure

| File                           | Description                                                |
|--------------------------------|------------------------------------------------------------|
| `prometheus.sh`                | Main monitoring script (continuous loop)                  |
| `run_prometheus.sh`            | Watchdog script (checks if Prometheus is running)         |
| `clean_prometheus_debug.sh`    | Weekly cleaner for debug log                              |
| `gas_correction.sh`            | Interactive tool to calibrate estimated gas vs real data  |
| `correction_reminder.sh`       | Sends a Telegram reminder after a configurable number of days |
| `.env.prometheus.example`      | Environment config (boiler parameters + tokens)           |
| `.env.correction.example`      | Config for gas correction and reminder interval           |

---

## Log Files

| File                      | Description                                |
|---------------------------|--------------------------------------------|
| `debug_prometheus.log`    | All modulation/flame readings and errors   |
| `prometheus_success.log`  | JSON logs of complete heating cycles       |
| `prometheus_errors.log`   | Failed SSH commands or calc exceptions     |

---

## Telegram Integration

The system uses a Telegram bot to notify:

- Each successful cycle (with duration and estimated gas)
- Correction suggestions (e.g. after comparing real vs estimated)
- Weekly reminders to check and input gas readings

---

## Correction Mechanism

Prometheus V2 includes a **calibration system**:

1. You input two manual gas readings and their timestamps.
2. It compares them to values logged by Prometheus V1 and V2.
3. Calculates % errors and suggests a new correction factor.
4. If accepted, the `.env.correction` file is updated automatically.

---

<details>
<summary>Legacy – Prometheus V1 Description (click to expand)</summary>

### Prometheus V1 – Legacy Recap

Prometheus V1 was a monolithic script, designed to:

- Run continuously via `@reboot` in crontab
- Alternate every 2 seconds between:
  - `ModulationTempDesired`
  - `Flame`
- Detect ignition start/end and calculate duration
- Estimate gas usage using fixed formula
- Send Telegram alerts and log each cycle

#### Limitations:

- All parameters were embedded in the script
- No modularity, no `.env` configuration
- No correction logic or adaptation to real-world readings
- Poor isolation between code and configuration
- No control on reminders or input validation

V2 aims to **address all these limitations**.

</details>

---

## Final Notes

- System designed for long-term autonomous operation
- Compatible with any system that can SSH into Home Assistant
- Script logic tested for gas estimation accuracy (±5%)
- Correction system reduces drift over time via real data
- Fully bash-native: no Python, no dependencies outside `bc`, `ssh`, `curl`

> Make sure to never commit actual tokens or paths. Always use the `.example` files and add `.env.*` to `.gitignore`.

---

# Block 2 – Installation and Startup

## 1. Installation Steps

To install and activate Prometheus V2, follow these steps:

### a) Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/prometheusV2.git
cd prometheusV2
```

> Replace `YOUR_USERNAME` with your actual GitHub username.

---

### b) Create Configuration Files

Copy and edit the provided example `.env` files:

```bash
cp .env.prometheus.example .env.prometheus
cp .env.correction.example .env.correction
```

Edit them with your actual parameters:

```bash
nano .env.prometheus
nano .env.correction
```

Required values include:

- **Boiler power and efficiency**
- **Gas lower heating value**
- **Telegram bot token and chat ID**
- **Correction percentage (default: 10%)**
- **Reminder interval (e.g. every 7 days)**

---

### c) Grant Execute Permissions

Ensure all scripts are executable:

```bash
chmod +x prometheus.sh
chmod +x run_prometheus.sh
chmod +x clean_prometheus_debug.sh
chmod +x gas_correction.sh
chmod +x correction_reminder.sh
```

---

## 2. Auto-start via Crontab

Edit the crontab:

```bash
crontab -e
```

And add the following jobs:

```cron
@reboot nohup bash /path/to/prometheus.sh >> /path/to/debug_prometheus.log 2>&1 &
* * * * * /path/to/run_prometheus.sh
0 4 * * 1 /path/to/clean_prometheus_debug.sh
10 10 * * * /path/to/correction_reminder.sh
20 20 * * * /path/to/correction_reminder.sh
```

> Replace `/path/to/` with the real path to your scripts.  
> Reminder is triggered at 10:10 and 20:20 once a day if threshold is passed.

---

## 3. Manual Startup (for testing)

You can launch Prometheus manually for testing:

```bash
nohup bash prometheus.sh >> debug_prometheus.log 2>&1 &
```

To stop:

```bash
pkill -f prometheus.sh
```

To follow the log:

```bash
tail -f debug_prometheus.log
```

---

## 4. File Tree Summary

```bash
prometheusV2/
├── prometheus.sh                 # Main monitoring script
├── run_prometheus.sh             # Watchdog
├── clean_prometheus_debug.sh     # Weekly cleaner
├── gas_correction.sh             # Calibration tool
├── correction_reminder.sh        # Notification reminder
├── .env.prometheus               # Main configuration
├── .env.correction               # Correction config
├── .env.prometheus.example       # Example template
├── .env.correction.example       # Example template
├── debug_prometheus.log          # Debug output
├── prometheus_success.log        # Valid cycles
├── prometheus_errors.log         # Errors and failed commands
```

---

> ✅ **Once installed, Prometheus V2 will run automatically on every boot, monitor heating activity, and continuously improve over time.**

---

# Block 3 – Gas Calculation and Cycle Logic

## 1. Functional Overview

Prometheus V2 continuously monitors your boiler’s operation and estimates the amount of gas consumed per heating cycle. The logic relies on two core values:

- **Modulation percentage** (`ModulationTempDesired`)
- **Flame state** (`Flame`)

The script alternates every 2 seconds:
- One cycle checks the modulation.
- The next checks the flame state.

This results in:
- One modulation reading every 4 seconds.
- One flame reading every 4 seconds.

---

## 2. Cycle Detection Logic

| Event                          | Action                                                  |
|-------------------------------|----------------------------------------------------------|
| Flame changes from OFF to ON  | Record `start_time`, initialize `MODULATION_VALUES`     |
| Flame remains ON              | Append modulation values to array                       |
| Flame changes from ON to OFF  | Record `end_time`, calculate gas, save event            |

The script logs valid cycles when:
- At least one modulation value was collected.
- The cycle lasted more than 0 seconds.

---

## 3. Gas Estimation Formula

The estimated gas (in m³) for each cycle is computed as:

```bash
gas_m3 = ((MAX_POWER * MODULATION_PERCENT / 100) * EFFICIENCY * 0.0036 / PCI) * (DURATION_SEC / 3600)
```

### Parameter Definitions:

| Variable            | Description                               | Example        |
|---------------------|-------------------------------------------|----------------|
| `MAX_POWER`         | Max boiler power in watts                 | 24000 W        |
| `MODULATION_PERCENT`| Average modulation value over the cycle   | e.g. 45.6%     |
| `EFFICIENCY`        | Boiler efficiency (decimal)               | 0.99           |
| `0.0036`            | Conversion from W to MJ/h                 | fixed          |
| `PCI`               | Lower heating value of the gas (MJ/m³)    | 34.7           |
| `DURATION_SEC`      | Length of the cycle in seconds            | e.g. 180 sec   |

> Example:  
> 24000 W * 0.456 * 0.99 * 0.0036 / 34.7 * 180 / 3600  
> ≈ **0.152 m³**

---

## 4. Logging Structure

Each valid cycle is recorded in `prometheus_success.log` in JSON format:

```json
{
  "start": 1,
  "timestamp_start": "2025-05-18 09:13:21",
  "end": 0,
  "timestamp_end": "2025-05-18 09:17:02",
  "duration_sec": 221,
  "modulation_avg": "43.25",
  "gas_consumed": "0.164231 m³"
}
```

---

## 5. Telegram Notification

When a cycle is completed and gas is calculated, a Telegram alert is sent:

```
{ "start": 1, "timestamp_start": "…", "end": 0, "timestamp_end": "…", … }
```

> This ensures you’re notified in real time about each flame cycle and the estimated gas used.

---

## 6. Resilience Features

The script includes:

- **SIGTERM and SIGQUIT handling**  
  Ensures clean shutdown or forced exit if needed.

- **Retry logic for failed SSH/ebusctl commands**  
  Automatically retries up to 3 times on errors.

- **Watchdog & auto-restart**  
  Ensures script recovery after crashes or interruptions.

---

## 7. Summary of Cycle Evaluation

| Flame Event         | Action Taken                          |
|---------------------|----------------------------------------|
| `off → on`          | Save start timestamp, clear modulation |
| `on → on`           | Append modulation if valid             |
| `on → off`          | Save end timestamp, compute gas        |
| Modulation missing  | Abort cycle                            |
| Duration = 0 sec    | Abort cycle                            |

---

> ✅ This design balances simplicity and accuracy, making Prometheus V2 ideal for tracking your boiler’s behavior with minimal overhead and high reliability.

---

# Block 4 – Adaptive Correction and Self-Calibration Tools

## 1. Objective

Prometheus V2 introduces **interactive gas correction** capabilities to fine-tune estimates based on **real-world gas meter readings**. This ensures better alignment between calculated and actual gas usage over time.

The correction system is made up of:

- A **manual calibration script** (`gas_correction.sh`)
- A **reminder script** (`correction_reminder.sh`)
- A dedicated **environment config file** (`.env.correction`)

---

## 2. Calibration Workflow

### Step-by-step logic:

1. **First reading**  
   The user enters:
   - The gas meter value (e.g., 1755.397)
   - Date and time of the reading

2. **Waiting period**  
   The system stores the reading and waits a configurable number of days (default: 7).

3. **Second reading**  
   The user enters:
   - New gas meter value (e.g., 1755.930)
   - Date and time of the new reading

4. **Evaluation**  
   The script:
   - Calculates actual gas consumed
   - Analyzes `prometheus_success.log` and `prometheus_success_v2.log` to sum up:
     - Estimated gas by V1
     - Estimated gas by V2
   - Compares both to the real usage
   - Computes the error rate

5. **Suggested correction**  
   The script proposes a new correction percentage for V2, asking:

   ```
   Current precision: 90.4%
   Suggested correction: +6.25%
   Expected new precision: 97.3%
   → Do you want to apply this correction? [Y/n]
   ```

6. **Update**  
   If confirmed, it updates `.env.correction` with the new `CORRECTION_PERCENTAGE`.

---

## 3. Correction Configuration File

File: `.env.correction`

```bash
# Current correction percentage applied to V2
CORRECTION_PERCENTAGE=10

# Number of days to wait between two manual gas readings
CORRECTION_INTERVAL_DAYS=7
```

> This allows the calibration system to be persistent and user-friendly, editable from outside the code.

---

## 4. Reminder System

A separate script (`correction_reminder.sh`) is scheduled via `cron`:

- It checks if the time interval since the **last recorded correction** exceeds the threshold (`CORRECTION_INTERVAL_DAYS`)
- If so, it sends **two Telegram reminders daily** (at 10:00 and 20:00) prompting the user to perform a new gas reading

### Example message:

```
Reminder: Please take a new gas meter reading.
Last recorded correction: 2025-05-11 10:00
It has been 7 days since your last calibration.
```

---

## 5. Safety Checks and UX

The correction tool includes:

| Validation Type       | Behavior                                                      |
|------------------------|---------------------------------------------------------------|
| Missing reading         | Asks for new input                                            |
| Invalid format          | Suggests corrected format and confirms it with user          |
| Timestamp logic         | Ensures that the second reading is after the first           |
| Invalid gas delta       | Avoids division by zero or nonsensical corrections           |
| Precision under 90%     | Warns if estimation is still too inaccurate                  |
| Double confirmation     | Before applying permanent correction to environment settings |

---

## 6. Example Log Output

A comparison report will be shown:

```text
=== COMPARISON ===
Period: 2025-05-11 10:00 → 2025-05-18 10:01
Actual gas usage:      0.533 m³
Estimated by V1:       0.491 m³ (error ≈ 7.88%)
Estimated by V2:       0.572 m³ (error ≈ 7.32%)

Previous correction:   +10%
Suggested correction:  +6.25%
Estimated new precision: 97.3%

→ Apply this correction now? [Y/n]
```

---

## 7. Benefits of the Correction System

| Feature                    | Benefit                                                      |
|---------------------------|---------------------------------------------------------------|
| Manual user-driven input  | Maximum flexibility and control                               |
| Adaptive error reduction  | Keeps estimates close to real usage                          |
| Persistent environment    | Maintains config across restarts and updates                 |
| Lightweight implementation| Pure Bash, no external libraries                             |

---

> This correction loop allows Prometheus to evolve from a static estimator to a **self-calibrating system**, combining the precision of real-world measurements with the autonomy of automated logging.

---

# Block 5 – Logging, Security and Maintenance

## 1. Logging Strategy

Prometheus V2 maintains three key log files:

| File                      | Description                                                       |
|---------------------------|--------------------------------------------------------------------|
| `debug_prometheus.log`    | Real-time operational log: modulation, flame status, events        |
| `prometheus_success.log`  | Valid heating cycles with estimated gas usage                     |
| `prometheus_errors.log`   | Errors, command failures, SSH issues                              |

Each log is updated continuously, enabling full traceability.

> The log format is human-readable and JSON-compatible, enabling further automation or export.

---

## 2. Log File Examples

### `debug_prometheus.log`

```
Sun 18 May 13:25:01 CEST 2025 - Prometheus started
Sun 18 May 13:25:04 CEST 2025 - Flame: on
Sun 18 May 13:25:06 CEST 2025 - Modulation: 42.0
...
```

### `prometheus_success.log`

```json
{
  "start": 1,
  "timestamp_start": "2025-05-18 13:25:01",
  "end": 0,
  "timestamp_end": "2025-05-18 13:27:14",
  "duration_sec": 133,
  "modulation_avg": "39.0",
  "gas_consumed": "0.074300 m³"
}
```

### `prometheus_errors.log`

```
Sun 18 May 14:01:04 CEST 2025 - Attempt 1 failed: ssh root@192.168.1.101 ...
Sun 18 May 14:01:06 CEST 2025 - Attempt 2 failed ...
...
```

---

## 3. File Rotation and Cleanup

To avoid disk overflow:

- `debug_prometheus.log` is **emptied weekly** by `clean_prometheus_debug.sh`
- `prometheus_success.log` can be periodically archived
- `prometheus_errors.log` can be monitored for recurring issues

Recommended CRON entries:

```cron
0 4 * * 1 /your/path/system-scripts/clean_prometheus_debug.sh
```

Cleanup script:

```bash
#!/bin/bash
# Clean Prometheus debug log (safe empty)
: > /your/path/ML_scripts/debug_prometheus.log
```

---

## 4. Security Considerations

### Telegram Integration

Ensure the following:

- `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` are stored in `.env.prometheus`
- Never commit `.env.prometheus` to version control
- Redact logs and variables before sharing the code

Example `.env.prometheus.example`:

```bash
TELEGRAM_BOT_TOKEN="your_bot_token_here"
TELEGRAM_CHAT_ID="your_chat_id_here"
```

> Use `.gitignore` to exclude all sensitive `.env` files.

---

## 5. Self-Healing Capabilities

Prometheus V2 can:

- **Detect if eBUSd is inactive**
  - Uses SSH and Docker commands to query container state
  - If stopped, automatically attempts restart

- **Handle signal interrupts**
  - SIGTERM and SIGQUIT are caught and gracefully handled
  - Ensures safe shutdown and cleanup

- **Recover from zombie status**
  - Watchdog script (`run_prometheus.sh`) monitors log freshness
  - If inactive > 5 minutes, the script is force-restarted

> These checks ensure **99.9% uptime** even in low-power or headless environments.

---

## 6. Maintenance Recommendations

| Action                             | Frequency  | Command or File                               |
|------------------------------------|------------|-----------------------------------------------|
| Check debug log activity           | Weekly     | `tail -f debug_prometheus.log`                |
| Archive success log                | Monthly    | `cp prometheus_success.log backup_YYYYMM.log` |
| Review error logs                  | Weekly     | `less prometheus_errors.log`                  |
| Backup environment files           | Monthly    | `.env.prometheus`, `.env.correction`          |
| Check calibration logs             | Monthly    | `gas_correction.sh` history                   |

---

## 7. Optional Hardening Ideas

- **Use `systemd`** instead of `cron` for guaranteed service supervision
- **Encrypt logs** or restrict access with user permissions
- **Rotate logs automatically** with `logrotate`
- **Push critical errors to Telegram or email**
- **Monitor SSH failures or stale connections**

---

## 8. Conclusion

Prometheus V2 logging and maintenance design provides:

- Transparency: all data is recorded and auditable
- Safety: logs are pruned, sensitive info is excluded
- Resilience: watchdogs and traps prevent system lockups
- Extensibility: all logs are plain-text and JSON-compatible

> All scripts are built in **pure Bash**, making the system portable, efficient, and fully controllable.

---

# Block 6 – Adaptive Calibration via `gas_correction.sh`

## 1. Objective

The `gas_correction.sh` script introduces **interactive adaptive calibration** for Prometheus.

Its purpose is to:

- Compare actual gas meter readings (manually entered)
- Evaluate the error of `prometheus.sh` estimations (v1 and v2)
- Suggest an updated correction factor for `prometheus_v2`
- Apply the new factor (if confirmed)
- Improve precision over time

---

## 2. Operational Concept

Calibration works in **two steps**:

### STEP 1 – Initial Reading

- You manually insert:
  - **Gas meter value** (e.g. `1755.397`)
  - **Timestamp** of the reading (e.g. `2025-05-17 11:38`)
- The script records the values in the correction file (`.env.correction`).
- It waits a configurable number of days (default = 7) before prompting again.

> This step is stored and persistent. It is not overwritten until a second reading is confirmed.

### STEP 2 – Follow-Up Reading

- You run the script again and input:
  - New gas meter value (e.g. `1755.930`)
  - Timestamp of the new reading
- The script:
  - Calculates the **real gas usage** in the interval
  - Parses both `prometheus_success.log` and `prometheus_success_v2.log`
  - Computes:
    - Estimated gas usage from v1 and v2
    - Their % error compared to real data
  - Suggests a new correction % for v2 (e.g. from `+10%` to `+3.4%`)
  - Shows estimated new accuracy improvement
  - Asks if you want to **apply this correction**

---

## 3. Example Output

```
=== COMPARISON ===
Period: Sat 17 May 11:38:00 → Sun 18 May 12:05:00
Real consumption: 0.533 m³

Prometheus V1: 0.423 m³ (error ≈ 20.6%)
Prometheus V2: 0.479 m³ (error ≈ 10.1%)

Current correction: +10%
Suggested correction: +3.4%
Previous accuracy: ~89.9%
New predicted accuracy: ~96.6%

→ Apply this correction? [Y/n]:
```

If confirmed:

- The `.env.correction` file is updated with the new correction percentage.
- Prometheus v2 uses this correction from the next run.

---

## 4. Technical Logic

### Data Source

- V1: `prometheus_success.log`
- V2: `prometheus_success_v2.log`
- Both logs must have entries matching the interval defined by `timestamp1` and `timestamp2`

### Formula

```bash
new_correction = (real_gas / estimated_v2 - 1) * 100
```

This corrects the v2 logic with minimal deviation.

### Accuracy Estimation

Accuracy is shown as:

```bash
accuracy = 100 - abs((real - estimated) / real * 100)
```

---

## 5. Storage File: `.env.correction`

Stored variables include:

```bash
GAS_CORRECTION_PERCENTAGE=10
GAS_READING_1=1755.397
GAS_TIMESTAMP_1="2025-05-17 11:38"
GAS_READING_2=1755.930
GAS_TIMESTAMP_2="2025-05-18 12:05"
GAS_CORRECTION_REMINDER_DAYS=7
```

You may edit this file manually if needed.  
It is also used by the `correction_reminder.sh` script.

---

## 6. Smart Input Validation

The script includes validation for:

- Date/time format (`YYYY-MM-DD HH:MM`)
- Float numbers (e.g., `1755.397`)
- Reading 2 must be higher than Reading 1
- Date 2 must be after Date 1

If invalid, it suggests corrected formats or asks again.

---

## 7. Fully Interactive Mode

Every prompt is user-friendly and confirms steps:

- `"Confirm date? [Y/n]"`
- `"Do you want to apply this correction?"`
- `"Correction applied successfully!"`

All changes are printed and logged for auditability.

---

## 8. Summary

| Feature                      | Status      |
|-----------------------------|-------------|
| Interactive first/second reading | ✅ Yes |
| Automatic comparison with logs   | ✅ Yes |
| Suggest correction for v2        | ✅ Yes |
| Writes to `.env.correction`      | ✅ Yes |
| Fully bash-based                 | ✅ Yes |
| Safe, isolated, portable         | ✅ Yes |

---

> The `gas_correction.sh` script enhances reliability by introducing data-backed tuning.  
> With a few measurements, Prometheus V2 converges to <3% error margin in most scenarios.

---

## Dual Gas Estimation Logic – Why Prometheus v2 Is More Accurate

One of the key innovations introduced in **Prometheus v2** is the use of a **dual-estimation method** for calculating gas consumption, instead of relying solely on modulation values as in the previous version.

This new approach applies **two different formulas** in parallel and computes the **average** of the two results. This hybrid method increases resilience and smooths out errors caused by sensor noise or sampling misalignment.

---

### Formula 1 – Modulation-Based Estimation

This is the **traditional method** used in Prometheus v1 and is still used in v2.

```bash
gas_m3_mod = ((MAX_POWER * avg_modulation / 100) * BOILER_EFFICIENCY * 0.0036 / GAS_LOWER_HEATING_VALUE) * (duration_sec / 3600)
```

- `avg_modulation` → Average of all modulation values recorded while the flame is ON.
- This method is effective **only if modulation is sampled regularly and frequently**, and fails if no valid modulation is recorded (e.g., due to brief heating cycles).

---

### Formula 2 – Duration-Based Estimation

This new method assumes that, when the boiler flame is active, it is modulating at **100% power** unless proven otherwise.

```bash
gas_m3_duration = ((MAX_POWER * BOILER_EFFICIENCY) * 0.0036 / GAS_LOWER_HEATING_VALUE) * (duration_sec / 3600)
```

- This method becomes useful when **no valid modulation samples** are available (e.g., extremely short cycles).
- It acts as a **fallback** and ensures a gas estimation is always provided, even under sampling failure conditions.

---

### Why Combine Them?

Prometheus v2 computes both `gas_m3_mod` and `gas_m3_duration` in real time:

- If both values are valid:  
  → It computes the **average** between the two as the final result.
  
- If only one value is valid (e.g., modulation failed):  
  → It uses the valid one.
  
- If none are valid (e.g., corrupted cycle):  
  → No gas is logged for that cycle.

This system allows Prometheus to **self-correct** and **minimize bias** introduced by:
- Missing modulation samples
- Sensor desynchronization
- Short or incomplete flame activations

---

### Advantages Over v1

| Feature                         | v1                        | v2                              |
|----------------------------------|-----------------------------|-----------------------------------|
| Estimation method               | Modulation only           | Modulation + Duration fallback |
| Handles short flame cycles      | No                        | Yes                            |
| Tolerates modulation loss       | No                        | Yes                            |
| Error smoothing                 | None                      | Averaged dual-source logic     |
| Adaptive correction (correction.sh) | No                     | Yes                            |

---

> This dual-estimation logic is the main reason why Prometheus v2 achieves a **higher predicted precision**, especially when combined with the gas correction system (`gas_correction.sh`).

---

# Block 7 – Watchdog, Protections, and Fallback

## Objective

Ensure that `prometheus.sh` is **always running**, using:

- **Automatic startup on boot** (`@reboot` in crontab)
- **Watchdog script** executed every 1 minute (`run_prometheus.sh`)
- **Zombie detection**: if the debug log has not been updated in more than 5 minutes, the script is considered stuck and restarted
- **Weekly cleanup** of the debug log to prevent disk saturation

---

## Watchdog Script – `run_prometheus.sh`

```bash
#!/bin/bash

LOG_FILE="$HOME/prometheus/debug_prometheus.log"
SCRIPT_PATH="$HOME/prometheus/prometheus.sh"

# Check if the log has been inactive for more than 5 minutes
if [ -f "$LOG_FILE" ]; then
    last_update=$(stat -c %Y "$LOG_FILE")
    now=$(date +%s)
    diff=$((now - last_update))
    if [ "$diff" -gt 300 ]; then
        echo "$(date) - Log inactive for over 5 minutes. Restarting Prometheus." >> "$LOG_FILE"
        pkill -f "$SCRIPT_PATH"
        sleep 2
    fi
fi

# Start Prometheus if not already running
if ! pgrep -f "$SCRIPT_PATH" > /dev/null; then
    echo "$(date) - Prometheus not running. Launching..." >> "$LOG_FILE"
    nohup bash "$SCRIPT_PATH" >> "$LOG_FILE" 2>&1 &
fi
```

---

## Automatic Startup – CRON Configuration

```cron
# Start Prometheus on boot
@reboot nohup bash $HOME/prometheus/prometheus.sh >> $HOME/prometheus/debug_prometheus.log 2>&1 &

# Watchdog every minute
* * * * * bash $HOME/prometheus/run_prometheus.sh

# Weekly debug log cleanup
0 4 * * 1 bash $HOME/prometheus/clean_prometheus_debug.sh
```

---

## Log Cleanup Script – `clean_prometheus_debug.sh`

```bash
#!/bin/bash
# Empties the debug log
: > "$HOME/prometheus/debug_prometheus.log"
```

---

## Managed Scenarios

| Situation                         | Automatic Action                              |
|----------------------------------|-----------------------------------------------|
| Script not running               | Watchdog starts it                             |
| Script zombie (inactive log)     | Watchdog kills and restarts it                |
| System reboot                    | Script restarts via `@reboot` cron            |
| Log growing excessively          | Weekly truncation via `clean_prometheus_debug.sh` |

---

## Optional Future Expansions

- Telegram notification when watchdog restarts the script
- Dedicated error log (`prometheus_errors.log`)
- Replace CRON with a systemd service (for non-RPi platforms)

> **Security Note**: never include absolute paths, usernames, or tokens in the repository. Use `.env` files and `.gitignore` to isolate and protect sensitive data.

---

# Block 8 – Preventive Maintenance and Reliability Guidelines

## Maintenance Overview

To guarantee long-term reliability and minimize downtime or inaccuracies in the `prometheus.sh` monitoring system, a set of **preventive maintenance tasks** is recommended. These actions are mostly automated but can be reviewed or reinforced manually as needed.

---

## 1. Automated Weekly Log Cleanup

To prevent disk usage from uncontrolled log growth, the following is scheduled:

- Script: `clean_prometheus_debug.sh`
- Schedule: Every **Monday at 04:00**
- Action: Empties the `debug_prometheus.log` file without deleting it.

```bash
#!/bin/bash
: > "$HOME/prometheus/debug_prometheus.log"
```

**CRON entry:**

```cron
0 4 * * 1 bash $HOME/prometheus/clean_prometheus_debug.sh
```

---

## 2. Minute-Based Watchdog Health Check

Script: `run_prometheus.sh`  
Purpose: Verifies that `prometheus.sh` is running and actively updating the debug log.

- If the script is **not running**, it starts it.
- If the log has **not been updated in over 5 minutes**, it forcefully restarts the script.

This guarantees **resilience even in case of crashes, zombie states, or freezes**.

---

## 3. Automatic Startup on Reboot

To ensure `prometheus.sh` runs automatically after system reboot:

**CRON entry:**

```cron
@reboot nohup bash $HOME/prometheus/prometheus.sh >> $HOME/prometheus/debug_prometheus.log 2>&1 &
```

This avoids any manual intervention after system restarts or power failures.

---

## 4. Manual Log Review (Optional)

Occasional manual inspections are advised:

- **Every 1–2 months:**
  - Check `debug_prometheus.log` for error patterns or irregularities.
  - Review `prometheus_success.log` to confirm expected frequency of heating cycles.
  - Verify `prometheus_errors.log` to catch repeated failures in SSH or sensor reads.

---

## 5. Recommended Backup Targets

To ensure continuity, back up the following periodically:

| File/Directory                  | Purpose                                  |
|--------------------------------|------------------------------------------|
| `prometheus.sh`                | Main script                              |
| `run_prometheus.sh`            | Watchdog restart script                  |
| `clean_prometheus_debug.sh`    | Log truncation utility                   |
| `.env.prometheus`              | Configuration file with boiler specs     |
| `prometheus_success.log`       | Logged gas cycles (valuable data)        |
| `debug_prometheus.log`         | May help trace errors if problems arise  |
| `prometheus_errors.log`        | Troubleshooting failed command history   |
| Output of `crontab -l`         | In case of OS reinstall or SD card loss  |

Use version control or cloud backup tools with caution, avoiding sensitive data exposure.

---

## 6. Maintenance Recap Table

| Task                               | Frequency     | Automated | Notes                                  |
|------------------------------------|---------------|-----------|----------------------------------------|
| `debug_prometheus.log` cleanup     | Weekly (Mon)  | Yes       | Truncated every Monday at 4 AM         |
| `prometheus.sh` status check       | Every minute  | Yes       | Watchdog restarts if not found         |
| Log activity verification          | Every minute  | Yes       | Restarted if inactive >5 minutes       |
| Reboot recovery                    | On reboot     | Yes       | Cron `@reboot` line triggers relaunch  |
| Manual inspection of logs          | Monthly       | No        | Look for anomalies in all log files    |
| Manual backup                      | Monthly       | No        | Export `.log` files and scripts safely |

---

## 7. Long-Term Resilience Strategy

These layers of protection ensure that the monitoring stack:

- Recovers automatically from most software or runtime failures.
- Remains **always-on** with no user intervention.
- Can be **debugged or restored quickly** using logs and backups.
- Can be adapted for **different hardware environments** (e.g., Raspberry Pi, server, NAS).

**Recommendation:** schedule periodic checks in your calendar or monitoring dashboard (e.g., Uptime Kuma, Grafana, or Home Assistant notifications) to ensure total visibility.

---

> **Reminder**: Keep `.env` files and sensitive scripts (e.g., containing credentials or internal IPs) out of version control using `.gitignore`. Always scan files before publishing.

---

# Block 9 – Accuracy in Detecting Short Heating Cycles

## Purpose

This section documents the **limits and capabilities** of `prometheus.sh` in detecting short-duration heating cycles based on the reading interval and the behavior of the boiler. Understanding this helps assess the **minimum cycle length** that can be reliably logged and how modulation values influence gas estimation accuracy.

---

## 1. Sampling Strategy

The script alternates between two key readings:

| Cycle     | Parameter                  | Frequency |
|-----------|----------------------------|-----------|
| Even      | `ModulationTempDesired`    | Every 4s  |
| Odd       | `Flame`                    | Every 4s  |

Each loop iteration sleeps for 2 seconds, with parameters alternating per cycle.

### Implication:

- **Modulation** is sampled every **4 seconds**
- **Flame** is sampled every **4 seconds**, offset from Modulation

This design **reduces system load** while maintaining near real-time observation.

---

## 2. Minimum Cycle Detection Requirements

For a heating cycle to be **valid** and properly logged, these must occur:

- **Flame changes from `off` to `on`** → `start_time` is set.
- **At least one `Modulation` value** is collected while Flame is `on`.
- **Flame changes from `on` to `off`** → `end_time` is set and gas is computed.

This means at least **two full polling cycles** are required to fully detect a cycle.

---

## 3. Practical Detection Thresholds

| Cycle Duration (seconds) | Detection Probability | Explanation                                  |
|--------------------------|------------------------|----------------------------------------------|
| 0–3                      | Impossible             | Too fast, no guaranteed Flame/Modulation read |
| 4–6                      | Low                    | May catch `Flame` or `Modulation`, not both  |
| 7–9                      | Medium                 | Possible if timings align                    |
| 10–12                    | High                   | At least one Modulation & one Flame reading  |
| >12                      | Very High              | Multiple readings ensure accuracy            |

---

## 4. Accuracy Considerations

### Gas estimation formula:

```bash
gas_m3 = ((MAX_POWER * modulation_percent / 100) * efficiency * 0.0036 / PCI) * (duration_sec / 3600)
```

**Smaller durations** and **fewer modulation points** can cause:

- Underestimation or overestimation due to low resolution.
- Higher **relative error** if modulation varies within short periods.
- Instability in average calculation when only one value is collected.

> For cycles shorter than ~8 seconds, the modulation average may be **statistically irrelevant**.

---

## 5. Suggested Minimum Duration

To ensure accurate results, it's recommended to consider cycles **≥10 seconds** as statistically valid. Shorter cycles might be **skipped**, **logged with low accuracy**, or **ignored**.

> If absolute micro-cycle tracking is needed, consider:
> - Reducing `sleep` to 1s (at CPU/log cost)
> - Implementing faster sampling in a compiled language (e.g., Go, C)

---

## 6. Confirmation from Field Tests

Field data confirms:

- **Cycles >10s** are consistently tracked.
- **Cycles <6s** are almost always missed or skipped.
- Detected short cycles often show **near-zero gas** due to insufficient modulation samples.

---

## 7. Optional Enhancements (Future Work)

| Enhancement                   | Status   | Notes                                   |
|-------------------------------|----------|-----------------------------------------|
| Faster sampling loop          | Planned  | Needs performance evaluation            |
| Sub-second polling            | Unstable | `bc` and `ssh` delay may bottleneck     |
| Hybrid Python integration     | TBD      | Possible faster parsing and precision   |

---

## Summary

- `prometheus.sh` balances reliability and load by sampling every 2 seconds.
- The architecture reliably detects cycles **≥10s**.
- Very short cycles (<6s) cannot be consistently captured.
- Logging and gas estimates are **most accurate** in longer heating phases.
```

---

# Block 10 – Final Summary and Changelog

---

## Summary of Improvements in Version 2 (Compared to V1)

| Feature                                | V1 Status                         | V2 Status                            | Improvement Description                                    |
|----------------------------------------|-----------------------------------|--------------------------------------|------------------------------------------------------------|
| Gas Estimation Accuracy                | Static parameters only            | Dynamic correction based on real data | Correction logic added via interactive `gas_correction.sh` |
| Log Management                         | Only weekly cleanup               | Structured, includes validation steps | Better control and separation of logs                      |
| Modulation Sampling                    | Present                           | Unchanged (every 4s)                  | Core logic preserved with validation                       |
| SSH Command Integration                | Inline                            | Isolated in `try_command()`          | Retry logic with full logging on failures                  |
| Telegram Notifications                 | Present                           | Improved logging and message tracking | Unified with correction feedback                           |
| Correction System                      | Not present                       | Yes (`gas_correction.sh`, `.env.correction`) | Allows tuning estimation model                             |
| Reminder System                        | Not present                       | Yes (`correction_reminder.sh`)       | Notifies user after fixed days for new gas reading         |
| Configuration Files                    | Hardcoded in script               | Externalized `.env` files             | Safer and more maintainable                                |
| Logs Format                            | JSON-like, Italian                | JSON-like, English                   | Standardized for integration                               |
| Error Handling                         | Minimal                           | Centralized + retries                | Auto-recovery from failures                                |
| Security Review                        | Manual only                       | Verified with `leakscan.py`          | Ensures credentials are not exposed                        |
| Documentation                          | Partial README                    | Full modular Markdown                | Complete GitHub-ready structure                           |

---

## Changelog

### v1.0 (Legacy)

- Basic script structure with modulation/flame read every 2 seconds.
- Simple logging mechanism with basic Telegram alerts.
- No support for correction or reconfiguration.

### v2.0 (Current)

- Full environment configuration split into `.env` files.
- Rewritten logging and error system.
- Modular function blocks.
- Gas correction script with persistent storage.
- Telegram reminder system for user-driven recalibration.
- Markdown documentation reorganized and translated to English.
- Verified using `leakscan.py` to ensure safety before publication.

---

## Known Limitations

- The system is based on external SSH commands: if network access or Docker container name changes, the script must be updated.
- Detection of very short heating cycles (<6s) may still be missed due to polling frequency.
- Telegram token must be handled with care. Do not publish real credentials.

---

## Future Ideas

- Optional SQLite log backend for advanced querying.
- Automatic chart generation from success logs.
- Integration with Home Assistant via REST endpoint.
- Dynamic polling interval based on flame frequency.

---

## Final Notes

The `prometheusV2` system is designed to be robust, transparent, and fully automatable. By applying regular gas readings and minor human interaction, it can continuously improve its gas usage estimation with an error rate below 2–3%, making it an efficient alternative to direct gas flow metering in non-commercial home environments.

> Always test new versions on test environments before deploying to production.

---

## Legal Notice and Disclaimer

This software is provided **"as is"** under the terms of the license described in `license.md`. The author **assumes no responsibility** for the use of this script outside personal experimentation or educational purposes. Any use of this project in **production**, **commercial**, or **safety-critical environments** is **strongly discouraged**.

Despite efforts to ensure accurate mathematical logic and estimation models, **this script is not intended to replace certified gas metering systems**. It does **not provide legally valid consumption data**, and **should not be used for billing or regulatory purposes**.

> **Important:** Accurate gas consumption estimation at 100% precision is **physically impossible** without a certified flow meter installed directly on the gas line. This is due to micro-losses and tolerances that are **inherent even in official metering systems** and **accounted for by gas providers themselves**. In many countries, utilities estimate average distribution losses of **1% to 2%** as part of the delivery process.

### Development and Contribution

This project is still under **active testing**. The current `beta` badge reflects its experimental nature. Contributions are welcome if:
- they improve **calculation logic**
- add **hardware compatibility**
- enhance **reporting or integration**

> Pull requests and forks are encouraged, provided they maintain compatibility with the core architecture.

### Security Notice

This script and all published files have been scanned with [`leakscan.py`](./leakscan.py) to ensure the absence of API tokens, credentials, or other sensitive data.

'''

---

# Prometheus Monitoring System – Project Overview

This project was inspired by the excellent work of [john30/ebusd](https://github.com/john30/ebusd) and [john30/ebusd-esp32](https://github.com/john30/ebusd-esp32), particularly the integration of an ESP32-based eBUS WiFi module connected directly to the boiler. We took this as a foundation and extended it by building a dedicated control unit (described in other sections of this repository) to enhance data monitoring and reliability.

While Home Assistant supports MQTT autodiscovery and `ebusd` integration, we observed critical limitations:
- The autodiscovery did not expose all boiler parameters.
- Some registers (e.g., `Flame`, `Gasvalve`) were either not shown or only returned stale cached values.
- Commands that should return real-time data appeared inactive or unresponsive through the standard interface.

We solved this by identifying the correct configuration file for our boiler (`bai.308523.inc`) and accessing the values directly using forced command-line calls to `ebusctl`, bypassing autodiscovery where necessary.

This hybrid approach allows us to:
- Use Home Assistant as the automation platform.
- Extract precise boiler data (especially for gas/flame cycles) via shell commands.
- Log events externally, enabling accurate tracking of gas consumption, flame activity, and system behavior.

---

## Verified eBUSD Commands for Flame Status

```bash
# 1. Real-time Flame state (bypasses cache)
docker exec -it <ebusd_container_name> ebusctl read -f Flame

# Returns:
# → "on"  = boiler is burning gas
# → "off" = boiler is idle

# 2. Repeat while hot water or heating is active
docker exec -it <ebusd_container_name> ebusctl read -f Flame

# 3. (Optional) Check Gasvalve status
docker exec -it <ebusd_container_name> ebusctl read -f Gasvalve

# 4. List registers containing 'flame' or 'gas'
docker exec -it <ebusd_container_name> ebusctl find | grep -i flame
docker exec -it <ebusd_container_name> ebusctl find | grep -i gas
```

---

## Diagnostic Recap – Flame Register Troubleshooting

```bash
# First attempt: failed (no -f flag)
docker exec -it <ebusd_container_name> ebusctl read Flame

# Result: always "off", even during combustion

# Second attempt: list active commands
docker exec -it <ebusd_container_name> ebusctl find -f on

# Result: Flame and Gasvalve not listed

# Third attempt: check BAI circuit
docker exec -it <ebusd_container_name> ebusctl find | grep bai

# Result: mostly "no data stored" → suspected circuit lock

# Final working command:
docker exec -it <ebusd_container_name> ebusctl read -f Flame

# ✅ Result: returns "on" during combustion, "off" at rest
# → Confirmed working and reliable
```

**Conclusion:** The only reliable way to read the `Flame` state is with:

```bash
docker exec -it <ebusd_container_name> ebusctl read -f Flame
```

Avoid using the command **without `-f`**, as it may return outdated or cached values.

---

## Bidirectional SSH Link – Raspberry Pi ↔ Home Assistant

### 1. SSH Key Created on Home Assistant
- User: `homeassistant_user`  
- IP: `HOME_ASSISTANT_IP`  
- Key path: `/-redacted-/-redacted-/id_rsa.pub`

> NOTE: standard path shown for educational purposes – never expose real keys in public repositories.

### 2. Key Copied to Raspberry Pi (`raspberry_user@RASPBERRY_IP`)
```bash
cat ~/-redacted-/id_rsa.pub | ssh raspberry_user@RASPBERRY_IP \
  "mkdir -p ~/-redacted- && cat >> ~/-redacted-/authorized_keys && chmod 700 ~/-redacted- && chmod 600 ~/-redacted-/authorized_keys"
```

Now Home Assistant can access the Raspberry Pi without a password:
```bash
ssh raspberry_user@RASPBERRY_IP
```

### 3. Raspberry Access to Home Assistant (`homeassistant_user@HOME_ASSISTANT_IP`)
```bash
cat ~/-redacted-/id_rsa.pub | ssh homeassistant_user@HOME_ASSISTANT_IP \
  "mkdir -p ~/-redacted- && cat >> ~/-redacted-/authorized_keys && chmod 700 ~/-redacted- && chmod 600 ~/-redacted-/authorized_keys"
```

Now Raspberry Pi can access Home Assistant with:
```bash
ssh homeassistant_user@HOME_ASSISTANT_IP
```

### 4. Bidirectional Access Verified
Both directions confirmed working without passwords.

### 5. Remote Commands from Raspberry Pi
Example – Read boiler flame state:
```bash
ssh homeassistant_user@HOME_ASSISTANT_IP \
  "docker exec -i <ebusd_container_name> ebusctl read -f Flame"
```

**Confirmed:** Real-time `on` / `off` response.

---

## Future Use

This SSH link allows the Raspberry Pi to:
- Query Home Assistant remotely
- Extract real-time data from `ebusd`
- Log everything locally without relying on MQTT or Home Assistant sensors

---

## Block 1 – System Description

The **Prometheus** system is an advanced Bash script for the automatic monitoring of estimated natural gas consumption on **Vaillant eBUS** boilers, with analysis of **modulation** and **flame** status. It is designed to:

- Start automatically upon system boot.
- Execute a continuous and self-healing cycle.
- Estimate gas consumption in **m³** for each flame ignition.
- Write each event to a `.log` file and send a real-time **Telegram alert**.
- Operate 24/7 even in case of reboots, crashes, or temporary issues.

### Main Features

- **Automatic startup** via `@reboot` in `crontab`.
- **Watchdog** every 60s: detects zombie or missing processes.
- **Weekly cleanup** of the debug log.
- **Simplified self-diagnosis** through log analysis.
- **Alternating data cycle**: reads `Modulation` and `Flame` every 2s sequentially.

### Requirements

- **Operating system**: Debian-based (e.g., Raspberry Pi OS)
- **Active eBUS interface**
- **Active SSH connection** to the system with `ebusd` Docker
- **Access to crontab and script execution permissions**
- Valid Telegram bot token and chat ID

> ⚠️ Note: Be careful not to publish actual Telegram tokens or chat IDs in public repositories.

### Involved Components

| Component                  | Description                                                                 |
|----------------------------|-----------------------------------------------------------------------------|
| `prometheus.sh`            | Main script, performs continuous monitoring and calculation logic         |
| `run_prometheus.sh`        | Watchdog: verifies that `prometheus.sh` is active, restarts it if necessary |
| `debug_prometheus.log`     | Debug log with all events and readings                                 |
| `prometheus_success.log`   | Log of cycles with valid calculations and sent notifications                        |
| `crontab`                  | Manages automatic startup, watchdog, and weekly cleanup                   |
| `clean_prometheus_debug.sh`| Weekly empties the debug log to prevent disk filling        |



# Block 2 – Data Flow and Internal Operation

## 1. Operational Cycle

The core of the `prometheus.sh` script is an infinite `while` loop alternating every 2 seconds between two main operations:

- **Reading modulation** (`ModulationTempDesired`)
- **Reading flame status** (`Flame`)

This structure allows efficient staggered data collection, avoiding overlaps or simultaneous reads.

### Sequential Cycle

| Seconds | Operation          |
|---------|--------------------|
| 0       | Read `Modulation`  |
| 2       | Read `Flame`       |
| 4       | Read `Modulation`  |
| 6       | Read `Flame`       |
| ...     | ...                |

## 2. State Handling

### Ignition

When the flame changes from `off` to `on`, the script:

- Records the **start timestamp**
- Clears the `MODULATION_VALUES` array
- Sets `flame_on=true`

### During Ignition

- On each even-numbered cycle, if `flame_on=true`, the current modulation is added to the array.
- The array will be used to calculate the **average modulation** later.

### Shutdown

When the flame switches back to `off`, the script:

- Records the **end timestamp**
- Calculates the **total duration**
- Calculates the **average modulation**
- Estimates **gas consumption (m³)** using the `calculate_gas_from_modulation` function
- Logs everything into `prometheus_success.log` and sends a Telegram message

## 3. Gas Calculation Formula

```bash
gas_m3 = ((MAX_POWER * modulation_percent / 100) * efficiency * 0.0036 / PCI) * (duration_sec / 3600)
```

**Parameters:**

- `MAX_POWER` = 24000 W  
- `EFFICIENCY` = 0.99  
- `WATT_TO_MJH` = 0.0036  
- `PCI (Lower Heating Value of Gas)` = 34.7 MJ/m³

> ⚠️ The values shown for `MAX_POWER`, `EFFICIENCY`, and `PCI` are examples. Always refer to your boiler's official technical documentation to verify the correct parameters for your specific model.

## 4. Telegram Notifications

Each completed event includes:

- Start and end timestamp  
- Total time in seconds  
- Estimated gas usage in cubic meters

> ⚠️ Ensure your Telegram bot token and chat ID are kept private and never committed to public repositories.
---

# Block 3 – Watchdog and Automatic Protection

## 1. Automatic Startup on Boot

The script is automatically launched at system boot using `@reboot` in the `crontab`:

```cron
@reboot nohup bash /-redacted-/youruser/ML_scripts/prometheus.sh >> /-redacted-/youruser/ML_scripts/debug_prometheus.log 2>&1 &
```

This ensures that every time the Raspberry Pi is powered on or rebooted, the script starts automatically.

## 2. Minute-Based Watchdog

The script `run_prometheus.sh` checks every minute whether `prometheus.sh` is running. If it’s not found among active processes, it will be restarted:

```bash
#!/bin/bash

# Watchdog Prometheus – executed every minute via cron
if ! pgrep -f "/-redacted-/youruser/ML_scripts/prometheus.sh" > /dev/null; then
    echo "$(date) - Prometheus not running, restarting" >> /-redacted-/youruser/ML_scripts/debug_prometheus.log
    nohup bash /-redacted-/youruser/ML_scripts/prometheus.sh >> /-redacted-/youruser/ML_scripts/debug_prometheus.log 2>&1 &
fi

# Check: if the log hasn't changed for over 5 minutes, the script may be zombie
LOG_FILE="/-redacted-/youruser/ML_scripts/debug_prometheus.log"
if [ -f "$LOG_FILE" ]; then
    last_mod=$(stat -c %Y "$LOG_FILE")
    now=$(date +%s)
    diff=$((now - last_mod))
    if [ "$diff" -gt 300 ]; then
        echo "$(date) - WARNING: log inactive for $diff seconds. Forcing restart." >> "$LOG_FILE"
        pkill -f "/-redacted-/youruser/ML_scripts/prometheus.sh"
        nohup bash /-redacted-/youruser/ML_scripts/prometheus.sh >> "$LOG_FILE" 2>&1 &
    fi
fi
```

This script is scheduled in `cron` every minute:

```cron
* * * * * /-redacted-/youruser/system-scripts/run_prometheus.sh
```

## 3. Weekly Log Cleanup

To prevent the debug log from growing indefinitely, a weekly cleanup script empties it (without deleting the file):

```bash
#!/bin/bash
# clean_prometheus_debug.sh
: > /-redacted-/youruser/ML_scripts/debug_prometheus.log
```

This is scheduled via `cron`:

```cron
0 4 * * 1 /-redacted-/youruser/system-scripts/clean_prometheus_debug.sh
```

## 4. Signal Handling

The `prometheus.sh` script handles SIGINT, SIGTERM, and SIGQUIT signals:

- `SIGINT` / `SIGTERM` → clean shutdown  
- `SIGQUIT` → forced kill and exit

```bash
trap handle_sigterm SIGINT SIGTERM
trap handle_sigquit SIGQUIT
```

## 5. Additional Notes

- If `Flame` remains always off (e.g. during summer), the script stays idle but ready to resume action.
- If a silent failure occurs (e.g. blocked log), the watchdog will detect and restart it.

---

# Block 4 – Maintenance and Diagnostics

## 1. Log Files Used

- **debug_prometheus.log**  
  - Path: `/-redacted-/youruser/ML_scripts/debug_prometheus.log`  
  - Writes approximately every 2 seconds  
  - Contains timestamps, Flame status, Modulation readings, startup messages, and detected issues  
  - Emptied weekly via `clean_prometheus_debug.sh`  

- **prometheus_success.log**  
  - Path: `/-redacted-/youruser/ML_scripts/prometheus_success.log`  
  - Records only successfully completed cycles  
  - Each entry is in JSON format with `timestamp_start`, `timestamp_end`, `total_time`, and `gas_consumption`  

- **prometheus_errors.log**  
  - Path: `/-redacted-/youruser/ML_scripts/prometheus_errors.log`  
  - Logs SSH command failures and related issues  

## 2. Recommended Manual Diagnostics

To check if the script is running:

```bash
pgrep -f prometheus.sh
```

To monitor activity in real time:

```bash
tail -f /-redacted-/youruser/ML_scripts/debug_prometheus.log
```

To view recent errors:

```bash
tail /-redacted-/youruser/ML_scripts/prometheus_errors.log
```

## 3. Manual Reset and Maintenance

In case of issues:

1. Manually kill the script:

```bash
pkill -f prometheus.sh
```

2. Restart manually:

```bash
nohup bash /-redacted-/youruser/ML_scripts/prometheus.sh >> /-redacted-/youruser/ML_scripts/debug_prometheus.log 2>&1 &
```

3. Optional debug log cleanup:

```bash
: > /-redacted-/youruser/ML_scripts/debug_prometheus.log
```

## 4. Recommended Backup

It is advisable to periodically back up the following:

- `prometheus_success.log`  
- `prometheus.sh` and `run_prometheus.sh`  
- Any `*.sh` scripts and `crontab -l` output  

## 5. Monitoring Frequency and Zombie Risk

- `run_prometheus.sh` performs a check every 60 seconds  
- Considered stable if the log updates every 2–4 seconds  
- If the log has not updated in more than 300 seconds, it is presumed to be stuck and will be restarted automatically  

## 6. Security

- The script does not access the internet except for Telegram notifications  
- All critical commands are logged  
- SSH credentials are assumed to be managed securely via key-based or direct access

> ⚠️ Ensure your Telegram bot token and SSH keys are not exposed in logs or public repositories.

---

# Block 5 – Full CRON Configuration

## 1. Active CRON Jobs

The current output of `crontab -l` is:

```cron
* * * * * /list1
0 3 * * * /list2
*/5 * * * * /list3
0 4 * * * /list4
0 9 */2 * * /list5
0 3 1 * * /list6
@reboot nohup bash /-redacted-/youruser/ML_scripts/prometheus.sh >> /-redacted-/youruser/ML_scripts/debug_prometheus.log 2>&1 &
* * * * * /-redacted-/youruser/system-scripts/run_prometheus.sh
0 4 * * 1 /-redacted-/youruser/system-scripts/clean_prometheus_debug.sh
```

> NOTE: This CRON schedule is for documentation purposes only. Replace `/-redacted-/youruser/` with your actual user path, and avoid exposing any sensitive scripts or credentials in scheduled jobs.

## 2. Entries Relevant to Prometheus

- `@reboot ... prometheus.sh`  
  Launches the Prometheus script automatically at system startup.  
  Uses `nohup` to survive shell termination.

- `* * * * * run_prometheus.sh`  
  Watchdog that checks every minute if `prometheus.sh` is active.  
  If not found, it restarts it using `nohup`.

- `0 4 * * 1 clean_prometheus_debug.sh`  
  Empties `debug_prometheus.log` every Monday at 04:00.  
  Helps avoid disk saturation due to excessive logging.

## 3. Useful Commands

To edit the active crontab:

```bash
crontab -e
```

To check current CRON jobs:

```bash
crontab -l
```

To manually trigger the watchdog:

```bash
bash /-redacted-/youruser/system-scripts/run_prometheus.sh
```

## 4. `clean_prometheus_debug.sh` Script (Full Content)

```bash
#!/bin/bash
# Empties the debug log without deleting the file
: > /-redacted-/youruser/ML_scripts/debug_prometheus.log
```

Make sure the file is executable:

```bash
chmod +x /-redacted-/youruser/system-scripts/clean_prometheus_debug.sh
```
---

# Block 6 – Architecture and Operation of the `prometheus.sh` Script

## 1. General Structure

The `prometheus.sh` script is organized into three functional blocks:

- **Block 1 – Initialization and Setup**
  - Defines global variables and customizable parameters.
  - Includes utility functions such as `try_command`, `is_number`, `send_telegram_notification`, etc.
  - Sets up logs and initializes state variables.

- **Block 2 – Calculation Functions**
  - `calculate_gas_from_modulation`: calculates the estimated gas consumption (m³) based on the average modulation and cycle duration.
  - `log_event`: stores event data in `prometheus_success.log` and sends a Telegram notification.

- **Block 3 – Cyclical Monitoring**
  - The `monitor_heating_cycle` function:
    - Alternates every 2 seconds between reading the Flame state and the Modulation value.
    - Computes gas usage only if at least one valid value has been collected and the flame turns off.
  - Includes signal handling (`SIGTERM`, `SIGQUIT`) and automatic restart of eBUSd if not active.
  - Runs automatically on system boot and in background via `nohup`.

## 2. Detection Logic

- When **Flame changes from OFF to ON**, the script records the `start_time` and starts collecting `ModulationTempDesired` values.
- When **Flame changes back to OFF**, it computes the duration and gas consumption **only if**:
  - The duration is greater than 0 seconds.
  - At least one valid modulation value was collected.
- Results are saved in the success log and sent via Telegram.

## 3. Read Frequency

- The script alternates between reading **Modulation** and **Flame** every 2 seconds.
- This results in:
  - One Modulation read every 4 seconds.
  - One Flame read every 4 seconds.
- The balanced rhythm ensures synchronized data collection without parallel processes.

## 4. Advantages of This Architecture

- **Reliability**: No parallel subprocesses → reduced risk of zombie or locked states.
- **Clarity**: Continuous detailed logging every 2 seconds.
- **Modularity**: Each function is clearly separated, testable, and documentable.
- **Resilience**: Watchdog and CRON protections guarantee recovery after faults or errors.

## 5. Expected Output

- `debug_prometheus.log`:
  - Logs every step (start, errors, flame on/off, modulation values).
- `prometheus_success.log`:
  - Stores JSON-formatted events with timestamps, duration, and estimated gas.
- Real-time Telegram notification for each completed cycle.

> ⚠️ Ensure that your Telegram bot token and chat ID, if configured in the script, are not published in the code or logs.

---

# Block 7 – Watchdog, Protections and Fallback

## Objective

Ensure that `prometheus.sh` is **always running**, using:

- Automatic startup on boot (`@reboot`)
- Watchdog every 1 minute (`run_prometheus.sh`)
- Detection of zombie/stuck script (log inactive for over 5 minutes)
- Weekly log cleanup to avoid disk saturation

## Watchdog Script – `/-redacted-/youruser/system-scripts/run_prometheus.sh`

```bash
#!/bin/bash

LOG_PATH="/-redacted-/youruser/ML_scripts/debug_prometheus.log"
SCRIPT_PATH="/-redacted-/youruser/ML_scripts/prometheus.sh"

# Check if the log has been inactive for more than 5 minutes
if [ -f "$LOG_PATH" ]; then
    last_update=$(stat -c %Y "$LOG_PATH")
    now=$(date +%s)
    diff=$((now - last_update))
    if [ "$diff" -gt 300 ]; then
        echo "$(date) - Log inactive for over 5 minutes, restarting Prometheus" >> "$LOG_PATH"
        pkill -f "$SCRIPT_PATH"
        sleep 2
    fi
fi

# Start if not already running
if ! pgrep -f "$SCRIPT_PATH" > /dev/null; then
    echo "$(date) - Prometheus not running, starting it" >> "$LOG_PATH"
    nohup bash "$SCRIPT_PATH" >> "$LOG_PATH" 2>&1 &
fi
```

## Associated CRON Jobs

```cron
# Watchdog every minute
* * * * * /-redacted-/youruser/system-scripts/run_prometheus.sh

# Startup on system boot
@reboot nohup bash /-redacted-/youruser/ML_scripts/prometheus.sh >> /-redacted-/youruser/ML_scripts/debug_prometheus.log 2>&1 &
```

## Weekly Log Cleanup

### Script `/-redacted-/youruser/system-scripts/clean_prometheus_debug.sh`

```bash
#!/bin/bash
: > /-redacted-/youruser/ML_scripts/debug_prometheus.log
```

### CRON

```cron
# Cleanup every Monday at 04:00
0 4 * * 1 /-redacted-/youruser/system-scripts/clean_prometheus_debug.sh
```

## Managed Scenarios

| Critical Case                    | Automatic Action                          |
|----------------------------------|-------------------------------------------|
| Script not running               | Watchdog starts it                        |
| Zombie script (inactive log)     | Forced restart using `pkill`              |
| System reboot                    | Automatic launch via `@reboot`            |
| Log file growing too large       | Weekly cleanup with empty file            |

## Optional Future Expansions

- Telegram notification upon forced restart  
- Separate error logging (`prometheus_errors.log`)  
- Systemd-based version instead of cron

> ⚠️ Reminder: if implementing Telegram notifications, make sure your bot token and chat ID are never exposed in public repositories.

---


## 8. Preventive Maintenance Requirements

To ensure continuous operation of the `prometheus.sh` script, the following preventive maintenance actions are recommended:

- **Weekly log check** (`debug_prometheus.log`, automated): the log is emptied every Monday at 04:00 to prevent disk space issues.
- **Automatic monitoring every minute**: the `run_prometheus.sh` script verifies that `prometheus.sh` is running and restarts it if necessary.
- **Automatic startup on every reboot**: handled via `@reboot` in the crontab.
- **Recommended manual maintenance**:
  - Every 1–2 months, check the size of `prometheus_success.log`.
  - Check `prometheus_errors.log` for recurring errors or anomalies.

These measures help ensure long-term reliability and reduce the risk of undetected failures.

---

## 9. Accuracy in Detecting Short Heating Cycles

The `prometheus.sh` script is designed to automatically detect flame ignition cycles, including short ones. However, due to alternating reads between `ModulationTempDesired` and `Flame`, there are practical limits to the minimum detection resolution.

### Sampling Frequency

- The script performs **one reading every 2 seconds**, alternating:
  - `ModulationTempDesired`
  - `Flame`
- Therefore, each parameter is read **once every 4 seconds**.

### Behavior with Micro-Cycles

| Cycle Duration (sec) | Detection Probability | Notes                                      |
|----------------------|------------------------|--------------------------------------------|
| < 4 sec              | **Very low**           | May be completely missed                   |
| 5–7 sec              | **Low**                | Detectable only if perfectly synchronized  |
| 8–12 sec             | **Medium–High**        | At least one valid read for both values    |
| > 12 sec             | **High**               | Stable and reliable detection              |

### Realistic Minimum Threshold

To consider a cycle valid for consumption calculation, the following are required:

- **At least one `Flame = on` reading**
- **At least one valid `Modulation` reading while the flame is on**
- **A following `Flame = off` reading**

This implies a **realistic minimum duration of ~8 seconds** to ensure statistical reliability and valid modulation averaging.

### Final Notes

- Cycles lasting **10–12 seconds** are consistently detected, as confirmed by recent testing.
- Cycles shorter than **6–8 seconds** may be completely missed.
- Reducing the `sleep` interval to 1 second is technically possible, but would significantly increase system load and log verbosity.
---

---

**Security Notice**  
This file has been scanned using [`leakscan.py`](./leakscan.py), a custom security tool designed to detect potential secrets, credentials, and high-risk patterns in text files.  
For more information or to contribute, refer to the script's README in this repository.

---
