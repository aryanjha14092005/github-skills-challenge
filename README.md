# AIOps Monitoring and Event Processing Assessment

## Scenario

This project monitors a synthetic `payment-service`. The service records request
latency, CPU usage, memory usage, and application log information. The operational
problem is to identify payment requests affected by slow responses, resource
pressure, or service/database timeout errors, then deliver those findings as events
for downstream processing.

AIOps is used here to combine metric threshold checks and log-level checks into
readable anomaly events. The repository uses an in-memory simulation of event
streaming rather than Kafka or another external service.

## Repository Components

- `data/service_data.json`: ten timestamped operational records for `payment-service`.
- `src/anomaly_detector.py`: checks metrics and `ERROR` log records and creates
	anomaly events with reasons and the original record.
- `src/event_producer.py`: publishes an event to an `EventTopic`.
- `src/event_topic.py`: stores published messages in memory.
- `src/event_consumer.py`: reads messages from the topic.
- `src/aiops_pipeline.py`: loads data, detects anomalies, publishes events, consumes
	them, and prints the final AIOps result.
- `tests/`: validation for normal/anomalous detection and event delivery.

## Operational Data Analysis

Each record has an ISO-like timestamp at one-minute intervals from
`2026-09-20T10:00:00` through `10:09:00`. The `timestamp` supports chronological
correlation of metric and log observations.

Metric fields are `response_time_ms`, `cpu_percent`, and `memory_percent`. Log fields
are `log_level` and `message`. `service` identifies the monitored service.

Records from 10:00 through 10:04 and 10:07 through 10:09 represent normal behaviour:
response time is 120-150 ms, CPU is 42-50 percent, memory is 51-57 percent, and each
record is an `INFO` success message.

The records at 10:05 and 10:06 are unusual:

| Timestamp | Metric/log evidence |
| --- | --- |
| 10:05 | 610 ms response time and `ERROR`: `Payment service timeout` |
| 10:06 | 640 ms response time, 94 percent CPU, 91 percent memory, and `ERROR`: `Database connection timeout` |

## Detection Findings

The detector thresholds are response time greater than 500 ms, CPU greater than 80
percent, and memory greater than 80 percent. An `ERROR` log level is also flagged.
The run detected two anomalies, at 10:05 and 10:06. No expected anomaly was missed
and no normal record was flagged for this dataset.

Detected reasons were:

- 10:05: high response time and error log detected.
- 10:06: high response time, high CPU utilization, high memory utilization, and
	error log detected.

## Event-Processing Flow

The complete flow is:

`Operational data -> AnomalyDetector -> Event -> EventProducer -> anomaly-events topic -> EventConsumer -> AIOps output`

The detector creates an event only when it has at least one reason. The producer
publishes that event to the shared in-memory `anomaly-events` topic. The consumer
reads the same topic and returns the processed event data to the pipeline, which
prints the service, timestamp, type, and detection reasons.

## Issues Found and Corrected

1. The producer was connected to `service-events`, while the consumer was connected
	 to a separate `anomaly-events` instance. The consumer therefore received zero
	 events. Both now use the same `anomaly-events` topic.
2. The detector checked for `WARNING`, but the supplied concerning records use
	 `ERROR`. The detector now flags `ERROR` log records.
3. Source modules used only top-level imports, which failed when tests imported them
	 as the `src` package. Package-relative imports with a direct-script fallback were
	 added, and `src` now has an explicit package marker.

## Final Execution Result

Running the pipeline processes 10 records, detects 2 anomalies, and consumes 2
events. The final output identifies `payment-service` timeouts at 10:05 and 10:06,
including the associated metric and log reasons.

## Limitation and Improvement

The detector uses fixed, global thresholds and evaluates each record independently.
It does not learn a service baseline, correlate adjacent timeout events, or account
for seasonal traffic patterns. A possible improvement is a configurable per-service
baseline with a rolling window and a correlation rule that groups related timeout
events into one incident.

## Reproduce the Demonstration

From the repository root:

```bash
python3 -m pytest -q
python3 -m src.aiops_pipeline
```

Expected validation is nine passing tests. Expected pipeline output includes:
`Records processed: 10`, `Anomalies detected: 2`, and `Events consumed: 2`.

The module command uses the repository root as the working directory and requires
Python 3. The pipeline has no external infrastructure or service dependencies.

## Evidence Collection Commands

Run these commands from the repository root in the Codespace terminal. Capture a
screenshot after each command finishes. Keep the terminal prompt visible so the
repository context and command can be identified.

### 1. Verify Repository and Submission State

This proves that the work is in the fork, the latest commit is present, and there
are no uncommitted changes:

```bash
git remote -v
git status --short
git log -1 --oneline
```

The `origin` URL should be the completed fork:
`https://github.com/aryanjha14092005/github-skills-challenge`.

Repository URL:
<https://github.com/aryanjha14092005/github-skills-challenge>

Pull request:
<https://github.com/DebbieAUG/github-skills-challenge/pull/7>

### 2. Show the Operational Data, Metrics, and Logs

```bash
python3 -m json.tool data/service_data.json
```

Capture the fields `timestamp`, `response_time_ms`, `cpu_percent`,
`memory_percent`, `log_level`, and `message`. These show the source data being
analysed. The unusual records are at `10:05` and `10:06`.

### 3. Show Anomaly Detection Results

```bash
python3 -m src.aiops_pipeline
```

Capture the two detected events and their reasons:

- `10:05`: high response time and error log detected.
- `10:06`: high response time, high CPU, high memory, and error log detected.

This same output also proves event generation, event consumption, and the final
AIOps output. The expected summary is `10` records processed, `2` anomalies
detected, and `2` events consumed.

### 4. Show the Producer, Topic, and Consumer Flow

```bash
sed -n '1,140p' src/aiops_pipeline.py
sed -n '1,100p' src/event_producer.py
sed -n '1,100p' src/event_topic.py
sed -n '1,100p' src/event_consumer.py
```

Capture the code showing that `EventProducer` publishes to the shared
`anomaly-events` topic and `EventConsumer` reads from that same topic. Use the
pipeline output from step 3 as the runtime proof that the event travelled through
the flow.

### 5. Show Successful Validation

```bash
python3 -m pytest -q
```

The expected result is:

```text
9 passed
```

Capture the complete test result, including the command and the passing summary.

### 6. Optional Single Evidence Summary

This command creates one compact terminal view containing the source anomalies,
detection reasons, event-flow label, and final counts:

```bash
python3 - <<'PY'
import json
from src.aiops_pipeline import run_pipeline

data = json.load(open("data/service_data.json"))
result = run_pipeline("data/service_data.json")
print("=== OPERATIONAL DATA / METRICS / LOGS ===")
print("records:", len(data))
print("metric fields: response_time_ms, cpu_percent, memory_percent")
print("log fields: log_level, message")
for record in data:
		if record["log_level"] == "ERROR":
				print(record["timestamp"], record["response_time_ms"],
							record["cpu_percent"], record["memory_percent"],
							record["log_level"], record["message"])
print("=== ANOMALY DETECTION ===")
for event in result["anomalies_detected"]:
		print(event["timestamp"], event["service"], "; ".join(event["reasons"]))
print("=== EVENT FLOW / FINAL AIOPS OUTPUT ===")
print("Operational Data -> Detector -> Event -> Producer -> anomaly-events Topic -> Consumer -> AIOps Output")
print("records_processed=", result["records_processed"])
print("anomalies_detected=", len(result["anomalies_detected"]))
print("events_consumed=", len(result["events_consumed"]))
PY
```

### Evidence Checklist

- Screenshot of `data/service_data.json` showing metrics and log fields.
- Screenshot of the pipeline showing anomaly reasons and event counts.
- Screenshot of the producer/topic/consumer source code and the matching pipeline output.
- Screenshot of `python3 -m pytest -q` showing `9 passed`.
- Screenshot of `git remote -v`, clean `git status`, and the final commit.
- Verify that the screenshots were captured after the final pushed commit and that
	they show the same results documented above.























