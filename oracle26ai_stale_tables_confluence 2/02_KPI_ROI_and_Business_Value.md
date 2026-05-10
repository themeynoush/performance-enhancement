# 02 KPI, ROI, and Business Value

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | Product Owner / Finance Partner |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics, agentic-ai |
| Review cadence | Weekly during build; monthly after go-live |


## Should this project have a KPI, ROI, and business-value page?

Yes. This page should be created early because it defines what success means before the solution is built.

Without this page, the project can become a technical dashboard without proving value. With this page, the team can show fewer incidents, faster detection, less DBA manual effort, and improved SQL stability.

## Value hypothesis

If we predict stale optimizer statistics and prioritize remediation, then we can reduce avoidable SQL performance degradation, reduce DBA triage time, and make stats maintenance more targeted.

## KPI framework

| KPI | Definition | Measurement source | Target example |
|---|---|---|---|
| Mean time to detect stale stats | Time between stale threshold crossing and detection | Snapshot table + `DBA_TAB_STATISTICS` | Reduce by 70% |
| Mean time to remediate | Time between detection and stats refresh or decision | Action log | Reduce by 50% |
| Stale age | Hours since `LAST_ANALYZED` for stale/missing stats | `DBA_TAB_STATISTICS` | Reduce P95 stale age |
| Stale objects with high SQL impact | Stale objects used by high-load SQL | Object access + SQL performance snapshots | Downward trend |
| Prediction precision | Correct stale-risk predictions / total stale-risk alerts | Model evaluation table | ≥ 80% after tuning |
| Prediction recall | Correct stale-risk predictions / actual stale events | Model evaluation table | ≥ 80% after tuning |
| Anomaly false positive rate | Non-actionable alerts / total anomaly alerts | Alert feedback | < 20% |
| SQL P95 elapsed time | P95 elapsed time for watched SQL | `V$SQLAREA`, AWR, or SQL tuning data | Downward trend |
| Buffer gets per execution | Logical I/O per execution for watched SQL | `V$SQLAREA`, AWR, SQL tuning sets | Downward trend |
| DBA manual hours saved | Baseline manual triage hours minus new process hours | Timesheet or runbook log | Monthly hours saved |
| Stats job efficiency | Objects refreshed because they were stale/high-risk / total objects refreshed | DBMS_STATS action log | Increase ratio |
| Incident reduction | Query-performance incidents linked to stale/missing stats | Incident records | Downward trend |
| Recommendation acceptance rate | DBA-accepted recommendations / total recommendations | Recommendation response + action log | Upward trend |
| Agent evidence completeness | Recommendations with required evidence / total recommendations | Agent evidence log | ≥ 95% |
| Approval cycle time | Time from recommendation to approved action or deferral | Recommendation response + action log | Downward trend |

## ROI model

Use a simple ROI model first, then refine after the baseline period.

```text
ROI % = ((Total measurable benefit - Total project cost) / Total project cost) * 100
```

### Benefit components

```text
Total measurable benefit =
  DBA manual hours saved
+ avoided incident handling time
+ reduced application support time
+ reduced business delay from slow SQL
+ reduced infrastructure waste from inefficient plans
```

### Cost components

```text
Total project cost =
  build effort
+ testing effort
+ operating effort
+ Oracle feature/license cost, if applicable
+ dashboard/automation cost
+ model monitoring effort
```

## Baseline plan

Collect at least 4 weeks of baseline data before claiming ROI.

| Baseline area | Data to collect |
|---|---|
| Stale stats | Count of stale/missing objects by schema and object type |
| Performance | Elapsed time, CPU, buffer gets, disk reads, rows processed, executions |
| Manual work | DBA time spent on stale-stat checks, query regression triage, stats refresh planning |
| Incidents | Number and severity of performance incidents linked to stale or missing stats |
| Stats jobs | Runtime, objects gathered, failed jobs, resource usage proxy |

## Business-value narrative

| Business value | Explanation |
|---|---|
| Better application stability | Less risk that stale stats lead to poor optimizer choices |
| Faster diagnosis | AWR/ADDM, SQL metrics, and stale stats are connected in one workflow |
| Lower manual effort | DBAs review prioritized recommendations instead of raw stale lists |
| Better recommendation quality | Agentic workflow combines current stale state, ML risk, performance impact, runbook context, and policy guardrails |
| Smarter maintenance | Large and partitioned tables get targeted strategies |
| Capacity efficiency | Better plans reduce unnecessary CPU, I/O, and buffer gets |
| Improved auditability | Every recommendation and DBMS_STATS action is logged |

## KPI dashboard sections

1. **Executive summary**: value delivered this month.
2. **Operational health**: stale count, top risk objects, alerts, remediation backlog.
3. **Prediction quality**: precision, recall, false positives, model drift.
4. **Performance impact**: SQL elapsed time, CPU, I/O, buffer gets before/after remediation.
5. **Automation value**: manual hours saved, actions recommended, actions accepted.
6. **Agentic recommendation quality**: acceptance rate, evidence completeness, approval cycle time, rejected recommendation reasons.
7. **Risk and governance**: open risks, licensing assumptions, model-quality issues.

## Official source basis

- Oracle AWR/ADDM and performance statistics: https://docs.oracle.com/en/cloud/paas/autonomous-database/dedicated/adbaa/high-performance-features-in-autonomous-ai-database-on.html
- Oracle AWR historical data and SQL/segment/service statistics: https://docs.oracle.com/en/database/oracle/oracle-database/26/dbiad/db_MMON_MMNL.html
- Oracle SQL tuning advisor and high-load SQL recommendations: https://docs.oracle.com/en/database/oracle/oracle-database/26/tdppt/tuning-sql-statements-using-sql-tuning-advisor.html
- Oracle SQL performance statistics fields: https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/ALL_TAB_MODIFICATIONS.html, https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- OCI Generative AI agent workflows and supervisor/collaborator agents: https://docs.oracle.com/en-us/iaas/Content/generative-ai/get-started-agents.htm, https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/agent-as-tool-guidelines.htm
- Confluence status and reporting setup: https://support.atlassian.com/confluence-cloud/docs/insert-the-status-macro/, https://support.atlassian.com/confluence-cloud/docs/insert-the-page-properties-macro/, https://support.atlassian.com/confluence-cloud/docs/create-a-custom-report/
