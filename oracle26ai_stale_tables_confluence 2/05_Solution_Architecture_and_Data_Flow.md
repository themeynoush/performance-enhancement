# 05 Solution Architecture and Data Flow

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | Solution Architect |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics, agentic-ai |
| Review cadence | Weekly during build; monthly after go-live |


## Architecture goal

Create a closed-loop system that detects stale optimizer statistics, predicts future stale risk, recommends the best DBA action, executes only controlled remediation, and measures the outcome.

The architecture separates three concerns:

| Layer | Purpose |
|---|---|
| Oracle truth layer | Read official database state such as `STALE_STATS`, `DBA_TAB_STATISTICS`, `DBA_TAB_MODIFICATIONS`, `STALE_PERCENT`, stats locks, and stats operation history |
| ML prediction layer | Predict stale risk, time-to-stale, DML anomalies, performance-regression risk, and stats-gather cost |
| Agentic recommendation layer | Combine rules, ML outputs, workload context, runbooks, and approval policy into a safe action recommendation |

Oracle remains the source of truth for current stale status. AI is used for prediction, prioritization, explanation, and recommendation.

## End-to-end logical architecture

```text
+--------------------------------------------------------------------------------+
|                                  Oracle DB / 26ai                               |
|--------------------------------------------------------------------------------|
| Data dictionary and workload sources                                            |
| - DBA_TAB_STATISTICS / DBA_IND_STATISTICS                                       |
| - DBA_TAB_MODIFICATIONS                                                         |
| - DBA_TAB_STAT_PREFS / DBMS_STATS preferences                                   |
| - DBA_OPTSTAT_OPERATIONS / DBA_OPTSTAT_OPERATION_TASKS                          |
| - V$SQL / V$SQLAREA / AWR / ASH / ADDM / SQL tuning data                         |
+--------------------------------------+-----------------------------------------+
                                       |
                                       v
+--------------------------------------------------------------------------------+
| Data ingestion pipeline from Oracle DB                                           |
|--------------------------------------------------------------------------------|
| Option A: in-database scheduler snapshots                                        |
| - SQL/PLSQL collector jobs write point-in-time rows into AIOPS snapshot tables    |
|                                                                                |
| Option B: OCI Data Integration                                                   |
| - Oracle Database data asset + connection                                        |
| - data flow from database source to target snapshot/history tables               |
| - pipeline to orchestrate loaders, transforms, notifications, and monitoring     |
+--------------------------------------+-----------------------------------------+
                                       |
                                       v
+--------------------------------------------------------------------------------+
| Historical store and feature layer                                               |
|--------------------------------------------------------------------------------|
| Raw snapshots -> curated history -> ML case table/view                           |
| Features: row-change %, DML velocity, last-analyzed age, workload impact,         |
|          plan changes, stats preferences, object size, partition flags,           |
|          action history, business criticality                                    |
+----------------------+-------------------------------+-------------------------+
                       |                               |
                       v                               v
+--------------------------------------+      +------------------------------------+
| ML training pipeline                  |      | ML inference pipeline              |
|--------------------------------------|      |------------------------------------|
| 1. Build labels                       |      | 1. Read latest snapshots           |
|    - stale_next_24h                   |      | 2. Rebuild same features           |
|    - time_to_stale_hours              |      | 3. Score with trained models       |
|    - perf_regression_next_24h         |      | 4. Produce risk outputs            |
|    - stats_action_helped              |      | 5. Store scored recommendations    |
| 2. Train OML models                   |      |                                    |
| 3. Evaluate and register model        |      | Outputs:                           |
| 4. Publish approved model version     |      | - p_stale_4h/8h/24h/48h            |
|                                      |      | - time_to_stale_hours              |
|                                      |      | - anomaly_score                    |
|                                      |      | - performance_risk_score           |
+----------------------+---------------+      +----------------+-------------------+
                       |                                       |
                       +-------------------+-------------------+
                                           v
+--------------------------------------------------------------------------------+
| Agentic AI recommendation engine                                                 |
|--------------------------------------------------------------------------------|
| Recommended production design: supervisor agent + specialized collaborator agents |
|                                                                                |
| Supervisor agent                                                                |
| - routes the DBA question or alert context                                       |
| - calls specialist agents/tools                                                  |
| - aggregates evidence into one recommendation                                    |
| - enforces output format and approval policy                                     |
|                                                                                |
| Collaborator agents / subagents                                                  |
| - Stale Stats Agent: current stale/missing/locked/truncated state                |
| - Performance Impact Agent: SQL workload, plan, AWR/ASH/ADDM signals             |
| - ML Risk Agent: stale forecast, anomaly score, performance risk                 |
| - Remediation Policy Agent: DBMS_STATS strategy, guardrails, maintenance window  |
| - Runbook/RAG Agent: similar incidents, runbooks, tuning notes                   |
|                                                                                |
| Agent tools                                                                     |
| - SQL/read-only API for evidence                                                 |
| - RAG knowledge base backed by Oracle AI Vector Search or supported KB store      |
| - action API for approved remediation, with required approval where needed        |
+--------------------------------------+-----------------------------------------+
                                       |
                                       v
+-----------------------------+        +------------------------------------------+
| Dashboard and alerts         |        | Controlled automation                     |
|-----------------------------|        |------------------------------------------|
| KPI, SLA, ROI, risk queue    |        | DBMS_STATS action request                 |
| DBA recommendations          |        | Approval gate                             |
| Explainability and evidence  |        | Maintenance-window execution              |
+-----------------------------+        +-------------------+----------------------+
                                                           |
                                                           v
+--------------------------------------------------------------------------------+
| Feedback loop                                                                    |
|--------------------------------------------------------------------------------|
| Save action decision, action result, before/after SQL metrics, stats job cost,    |
| model outcome, accepted/rejected recommendation, DBA notes, and drift signals.    |
+--------------------------------------------------------------------------------+
```

## Recommendation engine design choice

Oracle documentation supports both simple agent workflows and supervisor/collaborator agents. The best design depends on complexity, required specialization, and governance.

| Design | When to use | Advantages | Trade-offs | Fit for this project |
|---|---|---|---|---|
| Rule engine only | MVP detection, no natural-language DBA interaction, no semantic runbook retrieval | Fast, deterministic, easy to audit | No reasoning across runbooks, incidents, ML outputs, and policy | Required baseline; not enough for the full recommendation vision |
| Single agent / single workflow | One request path, limited tools, one output type, early pilot | Lower operational complexity; simpler prompt/tool control | Can become overloaded as data, tools, and policies grow | Good MVP for “explain this object and suggest action” |
| Supervisor agent with collaborator agents / subagents | The request spans stale stats, performance impact, ML risk, remediation policy, and runbook search | Clear specialization; routing; separate local context; reusable agents; easier tool isolation by function | More endpoints, IAM policy, testing, latency, and cost management | Recommended production design |

## Recommended approach

Use a staged design:

| Phase | Recommendation engine pattern | Reason |
|---|---|---|
| MVP | Rule engine + single agent workflow | Validate data quality, stale logic, risk scores, and DBA trust with minimum complexity |
| Production | Supervisor agent + collaborator agents / subagents | The production use case requires multiple specializations: stale-stat evidence, performance impact, ML risk, remediation policy, and runbook retrieval |
| Automation stage | Supervisor agent remains advisory; action API requires approval for risky actions | Keeps DBMS_STATS execution controlled and auditable |

The production recommendation is **multi-agent in structure**, using Oracle's supervisor/collaborator pattern. In Confluence we can call the collaborator agents **subagents**, but in Oracle terminology they are **collaborator agents**.

## Data ingestion pipeline from Oracle DB

### Ingestion sources

| Source | Data collected | Main use |
|---|---|---|
| `DBA_TAB_STATISTICS` and `DBA_IND_STATISTICS` | `STALE_STATS`, `NUM_ROWS`, `LAST_ANALYZED`, object/partition stats | Current stale/missing stats state |
| `DBA_TAB_MODIFICATIONS` | inserts, updates, deletes, truncated flag, timestamp | Stale-percent calculation and DML velocity |
| `DBA_TAB_STAT_PREFS` / `DBMS_STATS` preferences | `STALE_PERCENT`, `INCREMENTAL`, `INCREMENTAL_STALENESS`, publish settings | Correct threshold and collection strategy |
| `DBA_OPTSTAT_OPERATIONS` and `DBA_OPTSTAT_OPERATION_TASKS` | stats gather history, duration, status, target | Action cost, success/failure, feedback labels |
| SQL performance sources | elapsed time, CPU, buffer gets, disk reads, executions, plan hash | Performance-risk model and prioritization |
| Runbooks/incidents/tuning notes | text records | Semantic retrieval and explanation |

### Ingestion implementation options

| Option | Use when | Design |
|---|---|---|
| In-database snapshot jobs | The project wants low latency and minimal external movement | `DBMS_SCHEDULER` or approved scheduler runs SQL collectors and writes to AIOPS snapshot tables |
| OCI Data Integration | The project wants visual orchestration, cross-system movement, notifications, and reusable pipeline tasks | Create an Oracle Database data asset, build a data flow, publish integration/data-loader tasks, and orchestrate them in a Data Integration pipeline |
| Hybrid | The project wants in-DB raw capture plus external orchestration/reporting | Use in-DB snapshots for core metadata; use OCI Data Integration for downstream feature/history movement |

## ML training pipeline

```text
Historical snapshots
  ↓
Label builder
  - stale_next_4h / stale_next_8h / stale_next_24h / stale_next_48h
  - time_to_stale_hours
  - perf_regression_next_24h
  - stats_action_helped
  - stats_gather_duration_seconds
  ↓
Feature table / case view
  ↓
Train models with Oracle Machine Learning for SQL / DBMS_DATA_MINING
  ↓
Evaluate model quality
  ↓
Register approved model version
  ↓
Publish to inference pipeline
```

## ML inference pipeline

```text
Latest Oracle DB snapshot
  ↓
Feature builder using the same transformations as training
  ↓
OML scoring in SQL / DBMS_DATA_MINING scoring procedure
  ↓
Risk outputs
  - p_stale_4h / p_stale_8h / p_stale_24h / p_stale_48h
  - time_to_stale_hours
  - anomaly_score
  - performance_risk_score
  - predicted_stats_gather_duration_seconds
  ↓
Recommendation candidates
  ↓
Agentic recommendation engine
  ↓
Dashboard, alert, approval request, or controlled action
```

## What the system can predict

| Prediction | Meaning | Example output |
|---|---|---|
| Future stale state | Probability that an object or partition crosses the stale threshold in a time window | `p_stale_24h = 0.91` |
| Time-to-stale | Estimated hours until stale threshold is crossed | `time_to_stale_hours = 3.2` |
| DML anomaly | Whether row changes are unusual compared with learned normal behavior | `anomaly_score = 0.87` |
| SQL performance risk | Probability or score that SQL using the object may regress | `performance_risk_score = 0.78` |
| Stats gather cost | Expected duration/resource risk of a stats action | `predicted_stats_gather_duration_seconds = 420` |
| Best action | Ranked remediation candidate with reason | `Gather partition stats`, `Watch`, `DBA approval required` |
| Similar incident/runbook | Past operational knowledge relevant to this case | `similar_runbook_id = RB-STAT-004` |

## Automation guardrails

| Guardrail | Reason |
|---|---|
| Never auto-gather locked stats | Locks may be intentional |
| Treat `STALE_STATS IS NULL` as missing statistics and route for review | Missing stats require different handling than stale stats |
| Avoid full large-table gather during peak hours | Prevent resource impact |
| Prefer partition/incremental strategy when one partition changed | Reduce unnecessary full-table work |
| Require approval for risky actions | Keep DBMS_STATS execution controlled |
| Use API approval flags for action endpoints where supported | Prevent unapproved action calls |
| Log every recommendation and action | Auditability and feedback labels |
| Stop automation after repeated failures | Prevent loops |

## Official source basis

- Oracle stale statistics and DBMS_STATS: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- Oracle incremental statistics on partitioned objects: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html#gathering-incremental-statistics-on-partitioned-objects
- Oracle Machine Learning for SQL overview and scoring: https://docs.oracle.com/en/database/oracle/machine-learning/omlad/oracle-machine-learning-sql.html, https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/scoring-and-deployment.html
- Oracle `PREDICTION` SQL function: https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/21/dmapi/PREDICTION.html
- OCI Generative AI Responses API: https://docs.oracle.com/en-us/iaas/Content/generative-ai/get-started-agents.htm
- OCI Generative AI Agents overview: https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/overview.htm
- OCI supervisor/collaborator agents: https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/agent-as-tool-guidelines.htm
- OCI Generative AI Agents API endpoint calling tools: https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/api-calling-tool-guidelines.htm
- OCI Generative AI Agents knowledge bases: https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/create-knowledge-base.htm
- Oracle AI Vector Search: https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/overview-ai-vector-search.html
- OCI Data Integration data assets, data flows, and pipelines: https://docs.oracle.com/en-us/iaas/Content/data-integration/using/creating-data-assets.htm, https://docs.oracle.com/en-us/iaas/Content/data-integration/using/creating-oracle-db.htm, https://docs.oracle.com/en-us/iaas/Content/data-integration/using/creating-a-data-flow.htm, https://docs.oracle.com/en-us/iaas/Content/data-integration/tutorials/06-use-a-pipeline.htm
