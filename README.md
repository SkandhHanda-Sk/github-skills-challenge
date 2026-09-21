# GitHub Challenge

<img src="https://octodex.github.com/images/Professortocat_v2.png" align="right" height="200px" />

Hey there!

Your challenge is ready.
Follow the instructions provided for this challenge and complete the required tasks in this repository.

Make sure your work is committed and pushed to your repository before submission.

Good luck!


---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

## Project Overview

### Service Being Monitored
This project monitors a core application microservice that handles live traffic, tracking its health through real-time performance metrics (like CPU and latency) and system log traces.

### Operational Problem Being Addressed
Manual monitoring cannot keep up with cloud-scale applications. When intermittent issues occur—like sudden latency spikes or silent database failures—they often go unnoticed until they cause a full service outage. 

### Purpose of AIOps in This Assessment
AIOps bridges this gap by automating the operational lifecycle. Instead of human operators watching dashboards, the simulation automatically ingests messy telemetry data, identifies anomalies using pre-set logic, and instantly routes those findings as actionable events through a decoupled streaming pipeline for immediate triage.



## Log and Metric Analysis

The data in `data/service_data.json` comes from the `payment-service`.

The metric fields are:

- `response_time_ms`
- `cpu_percent`
- `memory_percent`

The log information is stored in:

- `log_level`
- `message`

The `service` field shows which service produced the record. The `timestamp`field shows when the observation was recorded. The timestamps use ISO 8601 format and increase by one minute, from `10:00` to `10:09` on`2026-09-20`.

Most observations appear to be normal. The response time is generally between 120 and 150 ms, CPU usage is between 42% and 50%, and memory usage is between 51% and 57%. These records have an `INFO` log level and say that the payment request was processed successfully.

The observation at `10:05` appears to be unusual. The response time increases to 610 ms, CPU usage reaches 75%, and memory usage reaches 70%. The record also contains an `ERROR` message saying `Payment service timeout`.

The observation at `10:06` is also unusual. The response time increases to 640 ms, CPU usage reaches 94%, and memory usage reaches 91%. The log reports a `Database connection timeout`.

These two records are different from the normal observations because they show higher resource usage, much slower response times, and error-level log messages.

## Anomaly Detection Findings

## Anomaly Detection Review

Our anomaly engine processed all 10 records and successfully flagged the 10:05 and 10:06 incidents with zero false positives. 

*   10:05 Capture: Flagged solely due to the response time jumping to 610 ms.
*   10:06 Capture: Flagged for a combined system spike—response time hit 640 ms, CPU maxed at 94%, and memory reached 91%.

### Gaps & Code Fixes
*   Missed Log Alerts: The detector initially missed the textual errors because it was mistakenly hardcoded to search for `WARNING` logs instead of critical `ERROR` strings. Updating the logic to intercept `ERROR` log messages significantly improves our alerting context.
*   Pipeline Limitation: Relying on fixed, hardcoded thresholds is brittle. If baseline service traffic changes naturally over time, static limits will cause a flood of false alerts. Switching to a rolling baseline (like dynamic Z-scores) would make the detection far more resilient.
