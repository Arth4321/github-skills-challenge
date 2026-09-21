# AIOps Service Monitoring Simulation

## Scenario

This project simulates monitoring for a `payment-service`. The operational problem is detecting
slow or unhealthy payment requests, high resource utilization, and service errors early enough for
the operations team to investigate them. AIOps connects telemetry analysis to event processing so
that an abnormal observation becomes a structured event that downstream components can consume.

The workflow is:

```text
Operational data -> anomaly detector -> anomaly event -> producer -> topic -> consumer -> AIOps output
```

## Repository Components

- `data/service_data.json` contains ten timestamped payment-service observations.
- `src/anomaly_detector.py` applies response-time, CPU, memory, and log-level thresholds.
- `src/event_producer.py` publishes detected events.
- `src/event_topic.py` provides the in-memory topic used by this simulation.
- `src/event_consumer.py` reads events from the topic.
- `src/aiops_pipeline.py` loads the data and connects detection, production, and consumption.
- `tests/` contains unit and pipeline regression tests.

## Operational Data Analysis

Each record uses an ISO-like timestamp at one-minute intervals from `10:00` through `10:09` on
2026-09-20. The metric fields are `response_time_ms`, `cpu_percent`, and `memory_percent`.
`log_level` and `message` are log fields, while `service` identifies the monitored service.

The observations from `10:00` to `10:04` and `10:07` to `10:09` are normal: response time is
120-150 ms, CPU is 42-50%, memory is 51-57%, and the log is `INFO`. The `10:05` record is
abnormal with 610 ms response time and an `ERROR` timeout. The `10:06` record is also abnormal
with 640 ms response time, 94% CPU, 91% memory, and an `ERROR` database timeout.

## Detection Findings

The detector uses these thresholds:

- response time greater than 500 ms
- CPU greater than 80%
- memory greater than 80%
- `WARNING` or `ERROR` log level

The run detects two anomalies:

| Timestamp | Metrics/log evidence | Reasons |
| --- | --- | --- |
| `10:05` | 610 ms, 75% CPU, 70% memory, `ERROR`: Payment service timeout | High response time; Error log detected |
| `10:06` | 640 ms, 94% CPU, 91% memory, `ERROR`: Database connection timeout | High response time; High CPU utilization; High memory utilization; Error log detected |

No normal record is flagged by the current data. The detector is threshold-based, so it may miss
gradual degradation that stays below a fixed threshold and does not compare observations with a
historical baseline. A useful improvement would be adaptive, service-specific thresholds and
trend-based detection.

## Corrections Made

The original workflow contained three issues:

1. `ERROR` logs were not recognized because the detector checked only `WARNING`.
2. The producer published to `service-events`, while the consumer listened to a different,
	empty `anomaly-events` topic.
3. The pull-request coverage workflow had steps outside the `steps` block, so its YAML structure
	was invalid.

The corrections keep the existing architecture and connect both producer and consumer to the
same `anomaly-events` topic. The completed run processes 10 records, detects 2 anomalies, and
consumes 2 events.

## Reproduce the Demonstration

From the repository root, activate the provided environment if available and run:

```bash
source .venv/calculations/bin/activate
pytest -q
python -m src.aiops_pipeline
```

The expected test result is 10 passing tests. The pipeline prints `Records processed: 10`,
`Anomalies detected: 2`, `Events consumed: 2`, followed by both detected events and their reasons.

The pipeline can also be run from the repository root without activating the environment if
`pytest` and the project dependencies are already installed.

