# 00 Project Home — AI + Oracle 26ai Stale Tables Monitoring

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | Project Lead / DBA Lead |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics |
| Review cadence | Weekly during build; monthly after go-live |


## Executive summary

This project will build an AI-assisted monitoring and prediction capability for stale optimizer statistics in Oracle AI Database 26ai.

Oracle determines stale optimizer statistics from monitored DML activity, `DBA_TAB_MODIFICATIONS`, and the `STALE_PERCENT` preference. The default staleness threshold is 10% row changes for monitored tables. This project uses that deterministic Oracle signal as the foundation, then adds AI/ML to forecast stale risk, detect anomalies, prioritize remediation, and estimate business impact.

## Main outcome

Move from reactive DBA work to proactive performance operations:

| Current state | Target state |
|---|---|
| DBAs check stale stats manually or after query slowdown | System predicts which tables or partitions will become stale |
| All stale tables look equally important | Objects are ranked by performance risk and business impact |
| Stats gathering is scheduled broadly | Stats gathering is targeted by priority, size, partition, and maintenance window |
| Query regressions are investigated after the incident | Stale-stat, DML, and SQL performance anomalies are detected early |

## Project goals

1. Detect stale table, partition, and index statistics using Oracle data dictionary views.
2. Calculate stale percentage from row counts and DML counts.
3. Predict **time-to-stale** for tables and partitions.
4. Predict **risk of SQL performance degradation** caused by stale or missing stats.
5. Detect anomalies in DML activity, stale percentage growth, SQL elapsed time, buffer gets, CPU, I/O, and plan changes.
6. Recommend the best action: no action, gather table stats, gather partition stats, gather schema stale stats, review stats preferences, or escalate to SQL tuning.
7. Measure ROI and business value with transparent KPIs.

## Recommended child pages

| Page | Purpose |
|---|---|
| 01 Problem Statement and Use Cases | Explains why this project exists and what problem it solves |
| 02 KPI, ROI, and Business Value | Defines success metrics and value model |
| 03 Detection Logic | Documents Oracle stale-stat formulas, SQL, and DBMS_STATS actions |
| 04 AI Pipeline | Describes anomaly detection and prediction pipeline |
| 05 Architecture | Shows data flow and components |
| 06 Data Model | Defines feature tables, labels, and scoring outputs |
| 07 Runbook | Gives operational steps and remediation paths |
| 08 Governance | Tracks assumptions, risks, controls, and decision log |
| 09 SQL Appendix | Provides SQL and PL/SQL snippets |
| 10 Official Sources | Lists official Oracle and Atlassian references only |

## Key design principle

The solution separates **Oracle truth** from **AI prediction**:

- Oracle truth: `STALE_STATS`, `NUM_ROWS`, `INSERTS`, `UPDATES`, `DELETES`, `STALE_PERCENT`, `LAST_ANALYZED`, and stats preferences.
- AI prediction: probability of staleness in a future window, time-to-stale, anomaly probability, likely impacted SQL, expected performance risk, and recommended priority.

## Official source basis

- Oracle optimizer statistics and stale statistics: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- Oracle Machine Learning overview and supported ML techniques: https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/introduction-machine-learning.html
- Oracle AI Vector Search overview: https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/overview-ai-vector-search.html
- AWR/ADDM performance monitoring: https://docs.oracle.com/en/cloud/paas/autonomous-database/dedicated/adbaa/high-performance-features-in-autonomous-ai-database-on.html
- Confluence templates, labels, and content properties: https://support.atlassian.com/confluence-cloud/docs/create-a-template/, https://support.atlassian.com/confluence-cloud/docs/use-labels-to-organize-your-content/, https://support.atlassian.com/confluence-cloud/docs/insert-the-page-properties-macro/
