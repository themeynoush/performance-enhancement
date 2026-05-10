# 04 AI Anomaly Detection and Prediction Pipeline

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | ML Lead / DBA Lead |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics |
| Review cadence | Weekly during build; monthly after go-live |


## Purpose

Use AI/ML to move beyond static stale-stat detection and produce predictions, anomaly alerts, and prioritized recommendations.

## Core idea

Oracle already answers:

```text
Is the object stale now?
```

The AI pipeline answers:

```text
Will it become stale soon?
How soon?
Will it matter?
Which SQL or business service may be impacted?
What should be done first?
```

## Pipeline overview

```text
Oracle dictionary views + SQL performance data
        ↓
Scheduled collection snapshots
        ↓
Feature engineering
        ↓
Model training and scoring
        ↓
Risk score + anomaly event + recommended action
        ↓
Dashboard, alert, or controlled DBMS_STATS action
        ↓
Feedback loop: action outcome and performance after remediation
```

## Input data

| Data group | Example fields | Purpose |
|---|---|---|
| Object statistics | owner, table_name, partition_name, num_rows, stale_stats, last_analyzed | Current stats health |
| DML modifications | inserts, updates, deletes, truncated, timestamp | Row-change velocity |
| Stats preferences | stale_percent, incremental, incremental_staleness | Correct threshold and strategy |
| SQL performance | sql_id, plan_hash_value, elapsed_time, cpu_time, buffer_gets, disk_reads, executions | Business/performance impact |
| Workload metadata | module, action, service, schema, criticality | Prioritization |
| Stats jobs | action type, start/end time, duration, success/failure | Cost and feedback |
| Incident/ticket notes | symptoms, remediation, root cause | Optional semantic search and explanation |

## Model types

| Model | Question answered | Example output |
|---|---|---|
| Time-series forecast | How will DML and stale percentage grow? | `time_to_stale_hours = 6.5` |
| Classification | Will the object become stale in the next N hours? | `p_stale_24h = 0.87` |
| Regression | What performance/resource impact is likely? | `predicted_elapsed_ms_delta = +35%` |
| Anomaly detection | Is this DML or SQL behavior unusual? | `anomaly_score = 0.94` |
| Ranking | Which objects should be remediated first? | priority 1–5 |
| Semantic similarity | Which past incident/runbook is similar? | top matching runbook/ticket |

## What we can predict in this project

### 1. Future stale state

Prediction target:

```text
Will object X become stale in the next 4, 8, 24, or 48 hours?
```

Example output:

| Object | Current stale % | Predicted stale in 24h | Time-to-stale | Reason |
|---|---:|---:|---:|---|
| SALES.ORDERS partition P202605 | 8.4% | 91% | 3.2 hours | DML rate above normal weekday pattern |

### 2. Time-to-stale

Formula-based estimate:

```text
remaining_changes = stale_threshold_rows - observed_dml_changes
change_rate_per_hour = recent_dml_changes / recent_hours
time_to_stale_hours = remaining_changes / NULLIF(change_rate_per_hour, 0)
```

ML improves this by learning daily, weekly, month-end, and deployment-related patterns.

### 3. Performance-regression risk

Prediction target:

```text
Will SQL using this object show abnormal elapsed time, CPU, logical I/O, physical I/O, or plan changes after stats become stale?
```

Useful features:

- object stale percent
- last analyzed age
- number of high-load SQL statements using the object
- SQL elapsed time per execution
- buffer gets per execution
- plan hash changes
- execution count trend
- business criticality

### 4. Anomaly detection

Detect anomalies such as:

| Anomaly | Example |
|---|---|
| DML spike | Inserts suddenly 5x normal for a partition |
| Update storm | Updates cross threshold quickly without corresponding maintenance plan |
| Stats age anomaly | Critical table has unusually old stats |
| Query regression | P95 elapsed time jumps after stale stats appear |
| Plan instability | SQL plan hash changes repeatedly after stats changes |
| Stats job anomaly | Stats gather duration or failure rate is unusual |
| Data skew change | Column distribution shifts after heavy DML |

### 5. Recommended remediation action

Output categories:

| Recommendation | Trigger example |
|---|---|
| No action | Low stale percentage and low SQL impact |
| Watch | High DML rate but not near threshold |
| Gather table stats | Small/medium object stale and active |
| Gather partition stats | One active partition stale in large partitioned table |
| Gather schema stale stats | Many stale objects in one schema |
| Review stats preferences | Frequent false alerts or over-gathering |
| Review locked stats | Stale/missing stats with `STATTYPE_LOCKED` |
| SQL tuning review | High impact persists after fresh stats |

## Oracle 26ai AI capabilities relevant to this design

| Capability | Use in this project |
|---|---|
| Oracle Machine Learning | Train and score prediction, regression, classification, anomaly, ranking, and time-series models |
| `DBMS_DATA_MINING` | PL/SQL API for creating, evaluating, and querying OML4SQL models |
| AI Vector Search and `VECTOR` | Store embeddings for incident notes, runbooks, SQL comments, and recommendations for semantic retrieval |
| ONNX model support via `DBMS_VECTOR` | Load compatible embedding models and generate embeddings in the database |

## Model evaluation

| Model | Evaluation metric |
|---|---|
| Future stale classification | Precision, recall, F1, calibration |
| Time-to-stale forecast | Mean absolute error, P90 error |
| Anomaly detection | Alert acceptance rate, false-positive rate, missed incident rate |
| Performance impact regression | Mean absolute percentage error, rank correlation |
| Recommendation ranking | Top-N hit rate, accepted recommendation rate |

## Feedback loop

Every action must write to `ACTION_LOG`:

| Field | Purpose |
|---|---|
| recommendation_id | Link action to model output |
| action_taken | DBMS_STATS action or investigation result |
| accepted_by | Human approver or automation rule |
| before_metrics | SQL/object metrics before action |
| after_metrics | SQL/object metrics after action |
| outcome | improved, unchanged, worsened, not enough data |
| notes | DBA explanation |

## Official source basis

- Oracle Machine Learning overview, prediction, anomaly detection, and time series techniques: https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/introduction-machine-learning.html, https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/machine-learning-functions1.html
- Oracle EM anomaly detection description: https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/expectation-maximization-anomaly-detection.html
- Oracle `DBMS_DATA_MINING`: https://docs.oracle.com/en/database/oracle/oracle-database/26/arpls/DBMS_DATA_MINING.html
- Oracle AI Vector Search and `VECTOR`: https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/overview-ai-vector-search.html
- Oracle `DBMS_VECTOR` and ONNX loading: https://docs.oracle.com/en/database/oracle/oracle-database/26/arpls/dbms_vector1.html, https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/onnx-pipeline-models-text-embedding.html
