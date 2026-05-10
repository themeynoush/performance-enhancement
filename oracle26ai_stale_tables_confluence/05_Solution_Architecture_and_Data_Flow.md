# 05 Solution Architecture and Data Flow

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | Solution Architect |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics |
| Review cadence | Weekly during build; monthly after go-live |


## Architecture goal

Create a closed-loop system that detects stale stats, predicts future risk, recommends action, executes only controlled remediation, and measures the result.

## Logical architecture

```text
+------------------------+
| Oracle metadata views  |
| DBA_TAB_STATISTICS     |
| DBA_IND_STATISTICS     |
| DBA_TAB_MODIFICATIONS  |
+-----------+------------+
            |
            v
+------------------------+       +------------------------+
| Performance sources    |       | Optional text sources  |
| V$SQL / V$SQLAREA      |       | Incidents, runbooks,   |
| AWR / ASH / ADDM       |       | SQL tuning notes       |
+-----------+------------+       +-----------+------------+
            |                                |
            v                                v
+---------------------------------------------------------+
| Collection jobs and snapshot tables                     |
+---------------------------------------------------------+
            |
            v
+---------------------------------------------------------+
| Feature engineering                                     |
| stale %, DML velocity, SQL impact, plan changes, etc.   |
+---------------------------------------------------------+
            |
            v
+---------------------------------------------------------+
| AI/ML scoring                                           |
| stale prediction, time-to-stale, anomaly score, ranking |
+---------------------------------------------------------+
            |
            v
+---------------------------------------------------------+
| Recommendation engine                                   |
| no action / watch / gather stats / tune SQL / review    |
+---------------------------------------------------------+
            |
            v
+------------------------+       +------------------------+
| Dashboard and alerts   |       | Controlled automation  |
| KPI, risk, SLA, ROI    |       | DBMS_STATS actions     |
+------------------------+       +------------------------+
            |
            v
+---------------------------------------------------------+
| Feedback: action outcome, before/after metrics, drift   |
+---------------------------------------------------------+
```

## Component responsibilities

| Component | Responsibility |
|---|---|
| Metadata collector | Snapshot stale stats, row counts, last analyzed dates, stats locks, preferences |
| DML collector | Snapshot inserts, updates, deletes, truncation flags, timestamps |
| Performance collector | Snapshot SQL elapsed time, CPU, buffer gets, disk reads, rows processed, plan hash values |
| Feature builder | Convert raw snapshots into model-ready features |
| Prediction service | Score future stale state, time-to-stale, anomaly, and performance risk |
| Recommendation engine | Convert score + rules into a clear action |
| Dashboard | Show health, risk, actions, and business KPIs |
| Automation controller | Execute approved DBMS_STATS actions only inside guardrails |
| Feedback store | Save before/after results and DBA decisions |

## Deterministic rule layer

Rules that do not need AI:

| Rule | Action |
|---|---|
| `STALE_STATS = 'YES'` and critical SQL uses the table | Prioritize investigation |
| `STALE_STATS IS NULL` | Treat as missing stats |
| `TRUNCATED = 'YES'` | High-priority review |
| `STATTYPE_LOCKED IS NOT NULL` | Do not auto-gather; escalate to DBA |
| Large partitioned table with one stale partition | Prefer partition/incremental strategy |
| Object stale but no workload usage | Low priority/watch |

## AI layer

AI adds probability and prioritization:

| AI output | Used by |
|---|---|
| `p_stale_24h` | Proactive alerting |
| `time_to_stale_hours` | Maintenance planning |
| `anomaly_score` | DML/performance outlier detection |
| `performance_risk_score` | DBA priority queue |
| `recommended_action` | Runbook automation |
| `similar_incident_id` | Explainability and faster diagnosis |

## Oracle AI Vector Search role

Vector search is not required to calculate stale stats. It is useful for operational intelligence:

| Text data | Vector use |
|---|---|
| Incident tickets | Find similar past stale-stat incidents |
| Runbooks | Retrieve the most relevant remediation guide |
| SQL tuning notes | Connect symptoms to previous fixes |
| ADDM/SQL tuning summaries | Retrieve related recommendations |
| DBA comments | Improve explanation and future recommendations |

## Data flow frequency

| Data | Suggested frequency | Reason |
|---|---:|---|
| `DBA_TAB_STATISTICS` | 15–60 minutes | Detect stale/missing status |
| `DBA_TAB_MODIFICATIONS` | 15–60 minutes | Track DML velocity |
| SQL performance | 15–60 minutes or AWR interval | Track performance risk |
| Stats job history | After each job | Learn cost and outcome |
| Incidents/runbooks | On change | Improve semantic retrieval |
| Model scoring | After each feature refresh | Keep risk current |

## Automation guardrails

| Guardrail | Reason |
|---|---|
| Never auto-gather locked stats | Locks may be intentional |
| Avoid full large-table gather during peak hours | Prevent resource impact |
| Require approval for first 30 days | Build trust and validate recommendations |
| Use reporting mode where possible before action | Show what would be gathered |
| Log every recommendation and action | Auditability |
| Stop automation after repeated failures | Prevent loops |

## Official source basis

- Oracle stale statistics and DBMS_STATS: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- Oracle AWR/ADDM performance diagnosis: https://docs.oracle.com/en/cloud/paas/autonomous-database/dedicated/adbaa/high-performance-features-in-autonomous-ai-database-on.html
- Oracle AWR historical performance data: https://docs.oracle.com/en/database/oracle/oracle-database/26/dbiad/db_MMON_MMNL.html
- Oracle SQL Tuning Advisor: https://docs.oracle.com/en/database/oracle/oracle-database/26/tdppt/tuning-sql-statements-using-sql-tuning-advisor.html
- Oracle AI Vector Search: https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/overview-ai-vector-search.html
