# 01 Problem Statement and Use Cases

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | Product Owner / DBA Lead |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics |
| Review cadence | Weekly during build; monthly after go-live |


## Problem statement

Oracle query performance depends heavily on optimizer statistics. When table, index, partition, or subpartition statistics no longer reflect the real data, the optimizer can make poor estimates and choose inefficient execution plans.

Oracle provides a stale-stat signal, but large systems still face operational challenges:

| Challenge | Why it matters |
|---|---|
| Many objects can become stale | DBAs need priority, not only a list |
| Large tables are expensive to scan | Remediation must avoid unnecessary resource use |
| Partitioned tables need different handling | Incremental and partition-level strategies may be better than full-table refresh |
| DML spikes can happen between maintenance windows | Statistics may go stale before the next automatic job |
| Query slowdown may be noticed late | Business impact starts before the DBA sees the root cause |
| Manual triage takes time | Repeated analysis can be automated and ranked |

## Business problem to solve

Reduce performance incidents and DBA manual effort by predicting and prioritizing stale optimizer statistics before they cause query regressions.

## Technical problem to solve

Build a repeatable process that:

1. Reads Oracle dictionary and performance data.
2. Calculates staleness from row count and DML changes.
3. Detects tables, partitions, and indexes with stale or missing stats.
4. Forecasts future stale state from DML patterns.
5. Detects unusual DML and SQL performance behavior.
6. Recommends the safest remediation action.
7. Measures impact after the action.

## Core use cases

| Use case | Description | Output |
|---|---|---|
| Detect stale objects now | Query Oracle metadata for stale/missing stats | Object list with stale flag and row-change ratio |
| Predict future stale state | Forecast whether an object will cross `STALE_PERCENT` in the next N hours/days | Time-to-stale and probability |
| Prioritize remediation | Rank stale objects by business and SQL performance risk | Priority score and action |
| Detect anomalies | Identify unusual DML, stats-age, query elapsed time, buffer gets, CPU, I/O, and plan changes | Anomaly event with explanation |
| Optimize stats maintenance | Choose table, schema, partition, high-frequency, or incremental stats strategy | Recommended DBMS_STATS action |
| Explain to stakeholders | Show why an object is risky and what value remediation delivered | KPI and ROI page updates |

## What this project can predict

The project can predict:

1. **Will this table or partition become stale soon?**
   - Output: probability in the next 4, 8, 24, or 48 hours.
2. **When will the stale threshold be crossed?**
   - Output: estimated time-to-stale.
3. **Which stale objects are most likely to affect important SQL?**
   - Output: risk score based on SQL volume, elapsed time, plan changes, object access, and business criticality.
4. **Which SQL statements may regress after stats become stale?**
   - Output: watchlist of SQL IDs, modules, schemas, and objects.
5. **Is the current DML pattern abnormal?**
   - Output: DML anomaly flag by table, partition, and time window.
6. **Is query performance abnormal after stale stats?**
   - Output: anomaly flag for elapsed time, CPU, I/O, buffer gets, rows processed, or plan hash changes.
7. **How long might stats gathering take?**
   - Output: estimated runtime/resource cost from prior stats-gathering history.
8. **What is the best action?**
   - Output: no action, gather table stats, gather partition stats, gather schema stale stats, review locked stats, tune SQL, or review workload.

## What this project should not claim to predict exactly

| Item | Reason |
|---|---|
| Exact optimizer plan choice for every future query | Execution plans depend on many inputs, including bind values, environment, statistics, schema, and optimizer settings |
| Exact business loss from every slow query | Business impact needs application and process context |
| Root cause of every performance issue | Stale stats are one common cause, not the only cause |

## Acceptance criteria

| Requirement | Acceptance test |
|---|---|
| Detect stale tables | `STALE_STATS = 'YES'` or missing stats appears in dashboard |
| Calculate row-change ratio | `(INSERTS + UPDATES + DELETES) / NUM_ROWS` shown per object |
| Predict time-to-stale | Dashboard shows ETA for non-stale but high-growth objects |
| Detect anomalies | DML/performance outliers generate events with reason codes |
| Recommend action | Each risk event has a recommended DBMS_STATS or investigation path |
| Measure value | KPI/ROI page shows before/after metrics |

## Official source basis

- Oracle stale statistics and `DBA_TAB_STATISTICS`: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- Oracle DML modification tracking columns: https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/ALL_TAB_MODIFICATIONS.html
- Oracle Machine Learning overview and prediction capability: https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/introduction-machine-learning.html
- Oracle execution plans and optimizer inputs: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/generating-and-displaying-execution-plans.html
