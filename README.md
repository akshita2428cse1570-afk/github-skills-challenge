# GitHub Challenge

<img src="https://octodex.github.com/images/Professortocat_v2.png" align="right" height="200px" />

Hey there!

Your challenge is ready.
Follow the instructions provided for this challenge and complete the required tasks in this repository.

Make sure your work is committed and pushed to your repository before submission.

Good luck!


---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

# AIOps Monitoring & Event Processing

## Scenario
The monitored service is `payment-service`, which produces operational metrics and logs. The operational problem is spotting abnormal behaviour (slow responses, resource pressure, error logs) quickly. In this assessment, AIOps means detecting these issues automatically and processing them as events through a simulated producer, topic and consumer flow (a lightweight Python simulation, not real Kafka or Airflow).

## Repository Structure
| Stage | File |
|---|---|
| Operational data (metrics and logs) | `data/service_data.json` |
| Anomaly detection | `src/anomaly_detector.py` |
| Event production | `src/event_producer.py` |
| Event topic | `src/event_topic.py` |
| Event consumption | `src/event_consumer.py` |
| Final AIOps processing | `src/aiops_pipeline.py` |
| Validation | `tests/test_aiops_pipeline.py` |

## Operational Data
`data/service_data.json` holds 10 records, one per minute from 10:00 to 10:09 on 2026-09-20.
- Metrics: `response_time_ms`, `cpu_percent`, `memory_percent`
- Log fields: `log_level`, `message`
- Timestamps: ISO 8601, one-minute spacing, copied into each event
- Identifier: `service`

## Observations
- Normal (10:00-10:04 and 10:07-10:09): 120-150 ms response time, CPU 42-50%, memory 51-57%, INFO logs.
- Unusual:
  - 10:05: 610 ms, CPU 75%, memory 70%, ERROR "Payment service timeout"
  - 10:06: 640 ms, CPU 94%, memory 91%, ERROR "Database connection timeout"

This looks like an incident where a database problem caused timeouts and resource pressure.

## Anomaly Detection Findings
`src/anomaly_detector.py` flags a record when any of these hold: response time > 500 ms, CPU > 80%, memory > 80%, or log level is ERROR/CRITICAL.

| Timestamp | Reasons |
|---|---|
| 10:05 | High response time, Error log detected |
| 10:06 | High response time, High CPU utilization, High memory utilization, Error log detected |

No normal record was flagged and no expected anomaly was missed. Each event carries its reasons, so it is clear why it was flagged.

## Event Flow
Operational Data -> AnomalyDetector -> Event -> EventProducer -> EventTopic -> EventConsumer -> AIOps Output (`src/aiops_pipeline.py`)

- **Event/message:** a dict with `timestamp`, `service`, `type` ("ANOMALY"), `reasons` and the `source` record.
- **Producer:** publishes each anomaly event to the topic.
- **Topic:** an in-memory list that holds events. The topic name is only a label, so the producer and consumer must share the same topic object.
- **Consumer:** reads the events from the topic.
- **AIOps output:** the pipeline prints the consumed events.

## Final Execution Result
Command: `python src/aiops_pipeline.py`

```
Records processed: 10
Anomalies detected: 2
Events consumed: 2

Service: payment-service
Timestamp: 2026-09-20T10:05:00
Type: ANOMALY
Reasons: High response time, Error log detected

Service: payment-service
Timestamp: 2026-09-20T10:06:00
Type: ANOMALY
Reasons: High response time, High CPU utilization, High memory utilization, Error log detected
```

Validation: `python -m pytest -v` -> 8 passed (4 in `tests/calculations_test.py` and 4 in `tests/test_aiops_pipeline.py`).

## Issues Identified and Corrected
1. **`src/aiops_pipeline.py`:** the producer published to "service-events" while the consumer read from a separate "anomaly-events" topic, so 0 events were consumed. Fixed by making both use the same topic object. Verified: events consumed went from 0 to 2.
2. **`src/anomaly_detector.py`:** the log check looked for "WARNING", but the data only contains INFO and ERROR, so error logs were never flagged. Fixed to check ERROR/CRITICAL. Verified: "Error log detected" now appears on both events.
3. **Test configuration:** pytest failed with `ModuleNotFoundError: No module named 'src'`. Fixed by adding `pytest.ini` with `pythonpath = . src`. Verified: all 8 tests pass.

## Limitation / Possible Improvement
Thresholds are static. At 10:05, CPU (75%) and memory (70%) are far above the baseline (about 42-50% and 51-57%) but under the 80% limits, so only response time and the log flagged that record. A rolling baseline or z-score per service would catch this. The detector also ignores the log `message` text, and 10 records is too small a sample to tune thresholds.

## How to Reproduce
1. Fork the repository and open it in a GitHub Codespace.
2. Install dependencies: `pip install -r requirements.txt`
3. Run the pipeline from the repo root: `python src/aiops_pipeline.py`
4. Run the tests: `python -m pytest -v`
5. Expected result: 10 records processed, 2 anomalies detected, 2 events consumed, 8 tests passed.