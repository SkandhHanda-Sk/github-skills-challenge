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

## Event Processing Flow

When the logic flags an anomaly, it packages the incident into a structured event payload containing the timestamp, service name, metric values, and raw log message. 

The data flows through four core components:
*   **Event:** The data packet containing the problem details and the reason it was flagged.
*   **Producer:** Receives the packet and publishes it directly to the message channel.
*   **Topic:** An in-memory queue that temporarily holds events in sequence.
*   **Consumer:** Subscribes to the topic, pulls the data packets, and prints them out as the final pipeline result.

---

## Issues Found & Corrected

*   **Log Severity Mismatch:** The code was hardcoded to check for `WARNING` logs, missing the critical `ERROR` entries in our data. We updated the tracking logic to catch error-level logs and include them in the alert context.
*   **Broken Pipeline Connection:** The producer and consumer were accidentally instantiated with separate in-memory topic instances. As a result, messages were being published into a void while the consumer listened to an empty queue. We passed the same topic reference to both components so data flows continuously.

---

## Final Workflow Result

Running the complete tracking pipeline from the project root:
```bash
python src/aiops_pipeline.py
```

Produces the following successful run summary:
*   **Records processed:** 10
*   **Anomalies detected:** 2
*   **Events consumed:** 2

### Captured Incidents
1.  **10:05:00:** Triggered by an elevated API response time and an explicit `Payment service timeout` error message.
2.  **10:06:00:** Triggered by a critical combination of high latency, 94% CPU load, 91% memory usage, and a `Database connection timeout`.

This confirms that events now successfully travel end-to-end through the detector, producer, topic, and consumer.

---

## Limitations & Better Approaches

*   **Brittle Thresholds:** The script relies on hardcoded numeric limits. If normal service behavior or traffic levels change organically, these fixed numbers will trigger false alerts. A better approach is calculating a dynamic baseline from recent history using a moving average.
*   **Volatile Storage:** The simulated topic lives entirely inside the running script's memory. If the process stops or crashes, all pending alerts are instantly lost. A production system requires a persistent message broker (like Kafka or RabbitMQ) to guarantee data survival.

---

## How to Reproduce this Demonstration

1. Navigate to the project directory:
   ```bash
   cd /workspaces/github-skills-challenge
   ```

2. Verify raw anomaly tracking outputs:
   ```bash
   python -c "import json; from src.anomaly_detector import AnomalyDetector; data=json.load(open('data/service_data.json')); detector=AnomalyDetector(); [print(record['timestamp'], detector.detect(record)) for record in data]"
   ```

3. Run the complete end-to-end messaging pipeline:
   ```bash
   python src/aiops_pipeline.py
   ```
## Validation Results

We installed the required project packages and validated the entire setup using the test suite:
```bash
pip install -r requirements.txt
PYTHONPATH="/workspaces/github-skills-challenge:/workspaces/github-skills-challenge/src" python -m pytest -q
```

All tests passed successfully, confirming that:
*   Healthy records are ignored (no false alarms).
*   Abnormal records are correctly flagged.
*   Events pass properly from the producer, through the topic, and into the consumer.

We then executed the final pipeline:
```bash
python src/aiops_pipeline.py
```
The workflow successfully processed all **10 records**, isolated the **2 anomalies**, and consumed both events without dropping any data.
