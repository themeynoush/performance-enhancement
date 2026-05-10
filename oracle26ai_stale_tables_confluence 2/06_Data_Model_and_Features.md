# 06 Data Model and Features

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | Data Engineer / ML Engineer |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics |
| Review cadence | Weekly during build; monthly after go-live |


## Purpose

Define the data model for stale-stat monitoring, prediction, anomaly detection, and recommendation tracking.

## Snapshot table: stale object state

```sql
CREATE TABLE aiops_stale_obj_snapshot (
  snapshot_id              NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  snapshot_ts              TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
  owner                    VARCHAR2(128) NOT NULL,
  object_type              VARCHAR2(30)  NOT NULL,
  object_name              VARCHAR2(128) NOT NULL,
  partition_name           VARCHAR2(128),
  subpartition_name        VARCHAR2(128),
  num_rows                 NUMBER,
  stale_stats              VARCHAR2(3),
  last_analyzed            DATE,
  stattype_locked          VARCHAR2(5),
  stale_percent_pref       NUMBER,
  inserts_since_stats      NUMBER,
  updates_since_stats      NUMBER,
  deletes_since_stats      NUMBER,
  observed_dml_changes     NUMBER,
  row_change_percent       NUMBER,
  truncated                VARCHAR2(3)
);
```

## Feature table

```sql
CREATE TABLE aiops_stale_obj_features (
  feature_id               NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  snapshot_ts              TIMESTAMP NOT NULL,
  owner                    VARCHAR2(128) NOT NULL,
  object_name              VARCHAR2(128) NOT NULL,
  partition_name           VARCHAR2(128),
  subpartition_name        VARCHAR2(128),
  num_rows                 NUMBER,
  row_change_percent       NUMBER,
  stale_percent_pref       NUMBER,
  last_analyzed_age_hours  NUMBER,
  dml_rate_1h              NUMBER,
  dml_rate_6h              NUMBER,
  dml_rate_24h             NUMBER,
  dml_spike_ratio          NUMBER,
  is_partitioned           NUMBER(1),
  is_stats_locked          NUMBER(1),
  is_missing_stats         NUMBER(1),
  is_truncated             NUMBER(1),
  sql_execs_24h            NUMBER,
  sql_elapsed_ms_p95       NUMBER,
  sql_buffer_gets_p95      NUMBER,
  sql_disk_reads_p95       NUMBER,
  plan_hash_change_count   NUMBER,
  business_criticality     NUMBER
);
```

## Prediction table

```sql
CREATE TABLE aiops_stale_prediction (
  prediction_id            NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  scored_ts                TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
  owner                    VARCHAR2(128) NOT NULL,
  object_name              VARCHAR2(128) NOT NULL,
  partition_name           VARCHAR2(128),
  subpartition_name        VARCHAR2(128),
  p_stale_4h               NUMBER,
  p_stale_8h               NUMBER,
  p_stale_24h              NUMBER,
  p_stale_48h              NUMBER,
  time_to_stale_hours      NUMBER,
  anomaly_score            NUMBER,
  performance_risk_score   NUMBER,
  recommended_action       VARCHAR2(100),
  reason_code              VARCHAR2(4000),
  model_name               VARCHAR2(128),
  model_version            VARCHAR2(128)
);
```

## Action log

```sql
CREATE TABLE aiops_stats_action_log (
  action_id                NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  prediction_id            NUMBER,
  action_ts                TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
  owner                    VARCHAR2(128) NOT NULL,
  object_name              VARCHAR2(128) NOT NULL,
  action_type              VARCHAR2(100) NOT NULL,
  action_status            VARCHAR2(30) NOT NULL,
  approved_by              VARCHAR2(128),
  started_ts               TIMESTAMP,
  ended_ts                 TIMESTAMP,
  duration_seconds         NUMBER,
  before_stale_stats       VARCHAR2(3),
  after_stale_stats        VARCHAR2(3),
  before_sql_p95_ms        NUMBER,
  after_sql_p95_ms         NUMBER,
  outcome                  VARCHAR2(100),
  notes                    VARCHAR2(4000)
);
```

## Feature catalog

| Feature | Description | Why it matters |
|---|---|---|
| `row_change_percent` | DML changes divided by stats row count | Main stale threshold signal |
| `dml_rate_1h/6h/24h` | Recent DML velocity | Forecast time-to-stale |
| `dml_spike_ratio` | Current DML compared with historical normal | Detect abnormal loads |
| `last_analyzed_age_hours` | Time since stats collection | Indicates stats age risk |
| `is_stats_locked` | Stats lock flag | Prevent unsafe automation |
| `is_missing_stats` | `STALE_STATS IS NULL` | Missing stats are high-priority for optimizer quality |
| `is_truncated` | Table truncated since stats gathered | Strong signal for stats refresh/review |
| `sql_execs_24h` | Number of executions using object | Measures workload importance |
| `sql_elapsed_ms_p95` | P95 elapsed time | Performance risk |
| `sql_buffer_gets_p95` | Logical I/O pressure | Capacity/performance risk |
| `plan_hash_change_count` | Number of observed plan changes | Plan instability signal |
| `business_criticality` | Manual or service metadata score | Business prioritization |

## Label definitions for supervised models

| Label | Type | Definition |
|---|---|---|
| `is_stale_next_24h` | Classification | Object crossed stale threshold within 24 hours after snapshot |
| `time_to_stale_hours` | Regression/time-series | Hours until stale threshold was crossed |
| `query_regressed_next_24h` | Classification | Watched SQL showed abnormal performance after snapshot |
| `stats_action_helped` | Classification | Metrics improved after stats action |
| `stats_gather_duration_seconds` | Regression | Duration of DBMS_STATS action |

## Scoring output example

| Object | p stale 24h | ETA | Anomaly | Performance risk | Action |
|---|---:|---:|---:|---:|---|
| SALES.ORDERS P202605 | 0.91 | 3.2h | 0.87 | 0.78 | Gather partition stats |
| APP.CUSTOMERS | 0.12 | 46h | 0.15 | 0.20 | Watch |
| FIN.POSTINGS | 0.76 | 8h | 0.92 | 0.95 | DBA approval required |

## Agentic recommendation tables

### Recommendation request

```sql
CREATE TABLE aiops_recommendation_request (
  request_id               NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  request_ts               TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
  request_type             VARCHAR2(50) NOT NULL,
  owner                    VARCHAR2(128),
  object_name              VARCHAR2(128),
  partition_name           VARCHAR2(128),
  triggered_by             VARCHAR2(128),
  request_payload          CLOB CHECK (request_payload IS JSON)
);
```

### Recommendation response

```sql
CREATE TABLE aiops_recommendation_response (
  response_id              NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  request_id               NUMBER NOT NULL,
  response_ts              TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
  supervisor_agent_version VARCHAR2(128),
  current_state            VARCHAR2(50),
  risk_level               VARCHAR2(20),
  recommended_action       VARCHAR2(100),
  confidence               NUMBER,
  requires_approval        VARCHAR2(3),
  reason_codes             VARCHAR2(4000),
  evidence_json            CLOB CHECK (evidence_json IS JSON),
  next_step                VARCHAR2(4000)
);
```

### Agent evidence log

```sql
CREATE TABLE aiops_agent_evidence_log (
  evidence_id              NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  response_id              NUMBER NOT NULL,
  agent_role               VARCHAR2(100) NOT NULL,
  evidence_type            VARCHAR2(100) NOT NULL,
  source_name              VARCHAR2(256),
  source_reference         VARCHAR2(1000),
  evidence_summary         VARCHAR2(4000),
  evidence_json            CLOB CHECK (evidence_json IS JSON),
  created_ts               TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL
);
```

## Agentic output fields

| Field | Description | Why it matters |
|---|---|---|
| `current_state` | Current object state: stale, fresh, missing stats, locked, unknown | Keeps Oracle truth separate from AI prediction |
| `risk_level` | Low, medium, high, critical | Prioritizes DBA queue |
| `recommended_action` | Proposed action category | Drives dashboard, ticket, or approval workflow |
| `confidence` | Confidence in the recommendation | Helps review borderline cases |
| `requires_approval` | Whether automation must stop for approval | Protects high-risk DBMS_STATS actions |
| `reason_codes` | Machine-readable reason list | Enables reporting and audit |
| `evidence_json` | Structured evidence from agents/tools | Supports explainability |
| `next_step` | Plain-language action for DBA/operator | Converts model output into operational instruction |

## Official source basis

- Oracle stale statistics and modification tracking: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html, https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/ALL_TAB_MODIFICATIONS.html
- Oracle SQL performance and AWR concepts: https://docs.oracle.com/en/database/oracle/oracle-database/26/dbiad/db_MMON_MMNL.html, https://docs.oracle.com/en/cloud/paas/autonomous-database/dedicated/adbaa/high-performance-features-in-autonomous-ai-database-on.html
- Oracle Machine Learning scoring and anomaly functions: https://docs.oracle.com/en/database/oracle/machine-learning/omlad/oracle-machine-learning-sql.html, https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/machine-learning-functions1.html, https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/scoring-and-deployment.html
- OCI Generative AI agent workflows, supervisor/collaborator agents, and API tools: https://docs.oracle.com/en-us/iaas/Content/generative-ai/get-started-agents.htm, https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/agent-as-tool-guidelines.htm, https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/api-calling-tool-guidelines.htm
