# 04 AI Anomaly Detection and Prediction Pipeline

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | ML Lead / DBA Lead |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics, mlops |
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
How expensive might the stats action be?
```

## Full pipeline overview

```text
Oracle DB metadata + workload + stats operation history
        ↓
Data ingestion pipeline
        ↓
Snapshot/history tables
        ↓
Feature engineering and case-table creation
        ↓
Machine learning training pipeline
        ↓
Model evaluation and model publication
        ↓
Machine learning inference pipeline
        ↓
Risk scores, anomaly events, and action candidates
        ↓
Agentic recommendation engine
        ↓
Dashboard, alert, approval request, or controlled DBMS_STATS action
        ↓
Feedback loop: action outcome and performance after remediation
```

## Data ingestion pipeline from Oracle DB

### Input data

| Data group | Example fields | Purpose |
|---|---|---|
| Object statistics | owner, table_name, partition_name, num_rows, stale_stats, last_analyzed | Current stats health |
| DML modifications | inserts, updates, deletes, truncated, timestamp | Row-change velocity and stale-percent calculation |
| Stats preferences | stale_percent, incremental, incremental_staleness, publish | Correct threshold and stats strategy |
| SQL performance | sql_id, plan_hash_value, elapsed_time, cpu_time, buffer_gets, disk_reads, executions | Business/performance impact |
| Workload metadata | module, action, service, schema, criticality | Prioritization |
| Stats jobs | target object, operation, start/end time, duration, success/failure | Cost, label creation, feedback |
| Incident/ticket notes | symptoms, remediation, root cause | Optional semantic search and explanation |

### Ingestion options

| Option | Best use | Output |
|---|---|---|
| SQL/PLSQL snapshot collector | Low-latency metadata capture inside Oracle DB | `AIOPS_*_SNAPSHOT` tables |
| OCI Data Integration data flow | Visual source-to-target movement and transformation | Curated target tables or lake/warehouse tables |
| OCI Data Integration pipeline | Orchestration of loaders, transforms, REST notifications, and monitoring | Repeatable ingestion workflow |
| Hybrid | In-DB raw capture plus external orchestration | Raw snapshots in DB; curated features in feature store |

## Feature engineering

Convert raw snapshots into one model-ready case table or view. Oracle Machine Learning for SQL expects training/scoring data as a table or view where each case is represented as a row.

| Feature family | Examples |
|---|---|
| Stale state | stale_stats, row_change_percent, stale_percent_pref, missing_stats flag |
| DML velocity | dml_rate_1h, dml_rate_6h, dml_rate_24h, spike ratio |
| Stats age | last_analyzed_age_hours, stats operation age, stats lock flag |
| Object shape | num_rows, partitioned flag, partition/subpartition level, object criticality |
| SQL impact | executions, elapsed time, CPU, buffer gets, disk reads, plan hash changes |
| Action history | last action, action duration, action success/failure, before/after metrics |
| Text similarity | similar incident/runbook IDs, retrieved remediation notes |

## Machine learning training pipeline

```text
Historical snapshots and action logs
        ↓
Label builder
        ↓
Training case table/view
        ↓
Train OML models
        ↓
Evaluate and compare models
        ↓
Register approved model version
        ↓
Publish model name/version to inference pipeline
```

### Training labels

| Label | Model type | Definition |
|---|---|---|
| `stale_next_4h`, `stale_next_8h`, `stale_next_24h`, `stale_next_48h` | Classification | Object became stale within the future window |
| `time_to_stale_hours` | Regression or time-series forecast | Hours until stale threshold was crossed |
| `dml_anomaly_case` | Anomaly detection | Whether the current DML behavior is unusual for the object/workload |
| `perf_regression_next_24h` | Classification/regression | Whether SQL using the object regressed, or expected magnitude of regression |
| `stats_action_helped` | Classification | Whether a stats action improved selected before/after metrics |
| `stats_gather_duration_seconds` | Regression | Expected runtime of the recommended stats action |

### Model families

| Model | Question answered | Example output |
|---|---|---|
| Classification | Will the object become stale in the next N hours? | `p_stale_24h = 0.87` |
| Regression | What numeric outcome is expected? | `predicted_elapsed_ms_delta = +35%` |
| Time-series forecast | How will DML and stale percentage grow over time? | `time_to_stale_hours = 6.5` |
| Anomaly detection | Is this DML or SQL behavior unusual? | `anomaly_score = 0.94` |
| Ranking/recommendation scoring | Which objects should be remediated first? | priority 1–5 |
| Semantic similarity | Which past incident/runbook is similar? | top matching runbook/ticket |

## Machine learning inference pipeline

```text
Latest snapshots
        ↓
Feature builder applies the same transformations used at training time
        ↓
OML scoring in SQL or DBMS_DATA_MINING scoring procedure
        ↓
Prediction table
        ↓
Recommendation candidate table
        ↓
Agentic recommendation engine
```

### Inference outputs

| Output | Meaning | Example |
|---|---|---|
| `p_stale_4h`, `p_stale_8h`, `p_stale_24h`, `p_stale_48h` | Probability of stale state in each window | `0.91` |
| `time_to_stale_hours` | Estimated hours until stale threshold | `3.2` |
| `anomaly_score` | Degree of unusual DML/performance behavior | `0.87` |
| `performance_risk_score` | Risk that SQL using the object may regress | `0.78` |
| `predicted_stats_gather_duration_seconds` | Expected collection duration | `420` |
| `recommended_action_candidate` | Rule/model-generated action candidate | `Gather partition stats` |
| `reason_codes` | Machine-readable explanation | `HIGH_DML_RATE;CRITICAL_SQL;PARTITION_ONLY` |

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

ML improves the estimate by learning daily, weekly, month-end, release-window, and batch-load patterns.

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

### 5. Stats-gather cost and timing

Prediction target:

```text
How long is the stats action likely to take, and should it fit inside the maintenance window?
```

Example output:

| Object | Recommended action | Predicted duration | Timing recommendation |
|---|---|---:|---|
| SALES.ORDERS P202605 | Gather partition stats | 7 min | Safe in current window |
| DW.FACT_SALES | Gather table stats | 95 min | Defer; use partition/incremental review |

### 6. Best remediation action

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
| DBA approval required | Large/high-risk action, locked stats, unusual anomaly, or business-critical workload |

## Model evaluation

| Model | Evaluation metric |
|---|---|
| Future stale classification | Precision, recall, F1, calibration |
| Time-to-stale forecast | Mean absolute error, P90 error |
| Anomaly detection | Alert acceptance rate, false-positive rate, missed incident rate |
| Performance impact regression | Mean absolute percentage error, rank correlation |
| Recommendation ranking | Top-N hit rate, accepted recommendation rate |
| Stats gather duration regression | Mean absolute error, P90 duration error |

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

- Oracle stale statistics and modification tracking: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- Oracle Machine Learning for SQL overview: https://docs.oracle.com/en/database/oracle/machine-learning/omlad/oracle-machine-learning-sql.html
- Oracle Machine Learning techniques, data requirements, automatic data preparation, and scoring requirements: https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/machine-learning-functions1.html
- Oracle scoring and deployment: https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/scoring-and-deployment.html
- Oracle `PREDICTION` SQL function: https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/21/dmapi/PREDICTION.html
- OCI Data Integration data assets, Oracle Database data asset properties, data flows, and pipelines: https://docs.oracle.com/en-us/iaas/Content/data-integration/using/creating-data-assets.htm, https://docs.oracle.com/en-us/iaas/Content/data-integration/using/creating-oracle-db.htm, https://docs.oracle.com/en-us/iaas/Content/data-integration/using/creating-a-data-flow.htm, https://docs.oracle.com/en-us/iaas/Content/data-integration/tutorials/06-use-a-pipeline.htm
