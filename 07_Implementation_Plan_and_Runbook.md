# 07 Implementation Plan and Runbook

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | Delivery Lead / DBA Lead |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics |
| Review cadence | Weekly during build; monthly after go-live |


## Implementation phases

| Phase | Goal | Exit criteria |
|---|---|---|
| 0. Prerequisites | Confirm access, schemas, environments, licensing, and target objects | Access approved; source views validated |
| 1. Discovery | Baseline stale stats, DML patterns, SQL performance, and manual effort | Baseline report complete |
| 2. Deterministic MVP | Build stale-stat dashboard and row-change formula | Dashboard matches Oracle `STALE_STATS` |
| 3. Data pipeline | Store snapshots and features | Daily snapshots complete with no gaps |
| 4. AI models | Train stale prediction, anomaly, and risk models | Evaluation metrics published |
| 5. Recommendation engine | Rank objects and suggest actions | DBA validates top recommendations |
| 6. Controlled automation | Execute approved DBMS_STATS actions | Guardrails and action log working |
| 7. Production operation | Monitor, improve, and report ROI | KPI/ROI page updated monthly |

## Phase 1 discovery checklist

| Task | Owner | Status |
|---|---|---|
| Identify schemas and critical applications | Product Owner | TBD |
| List large and partitioned tables | DBA | TBD |
| Capture current `STALE_PERCENT` preferences | DBA | TBD |
| Identify locked statistics | DBA | TBD |
| Confirm AWR/ASH/ADDM availability and licensing | DBA Manager | TBD |
| Select SQL performance views available in environment | DBA | TBD |
| Define business criticality scoring | Product Owner | TBD |
| Agree alert thresholds | DBA + SRE | TBD |

## Runbook: stale object investigation

### Step 1 — Check stale state

```sql
SELECT owner,
       table_name,
       partition_name,
       subpartition_name,
       num_rows,
       stale_stats,
       last_analyzed,
       stattype_locked
FROM   dba_tab_statistics
WHERE  owner = :owner
AND    table_name = :table_name
ORDER  BY partition_name, subpartition_name;
```

### Step 2 — Check DML changes

```sql
SELECT table_owner,
       table_name,
       partition_name,
       subpartition_name,
       inserts,
       updates,
       deletes,
       truncated,
       timestamp
FROM   dba_tab_modifications
WHERE  table_owner = :owner
AND    table_name = :table_name;
```

### Step 3 — Calculate row-change percentage

```text
row_change_percent = ((INSERTS + UPDATES + DELETES) / NUM_ROWS) * 100
```

### Step 4 — Check stats preference

```sql
SELECT DBMS_STATS.GET_PREFS(
         pname   => 'STALE_PERCENT',
         ownname => :owner,
         tabname => :table_name
       ) AS stale_percent
FROM dual;
```

### Step 5 — Decide action

| Condition | Action |
|---|---|
| Missing stats | Gather table stats |
| Stale small table | Gather table stats |
| Stale large partition only | Gather partition stats or use incremental strategy |
| Many stale tables | Gather schema stats with `GATHER STALE` |
| Stats locked | Do not auto-gather; review with DBA owner |
| High SQL impact remains after stats | Run SQL tuning investigation |

## Runbook: gather table statistics

```sql
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    ownname => :owner,
    tabname => :table_name,
    degree  => DBMS_STATS.AUTO_DEGREE
  );
END;
/
```

## Runbook: gather stale schema statistics

```sql
BEGIN
  DBMS_STATS.GATHER_SCHEMA_STATS(
    ownname => :owner,
    options => 'GATHER STALE'
  );
END;
/
```

## Runbook: review high-frequency automatic optimizer statistics collection

Use when critical tables can become stale between regular maintenance windows.

```sql
-- Example only: enable high-frequency automatic optimizer statistics collection.
-- Review with DBA governance before production use.
EXEC DBMS_STATS.SET_GLOBAL_PREFS('AUTO_TASK_STATUS','ON');

-- Example: set interval to 15 minutes.
EXEC DBMS_STATS.SET_GLOBAL_PREFS('AUTO_TASK_INTERVAL','900');
```

## Runbook: enable incremental statistics for partitioned table

```sql
BEGIN
  DBMS_STATS.SET_TABLE_PREFS(
    ownname => :owner,
    tabname => :table_name,
    pname   => 'INCREMENTAL',
    pvalue  => 'TRUE'
  );
END;
/
```

## Change control checklist

Before automation executes any stats action:

| Check | Required? |
|---|---|
| Object is not stats-locked | Yes |
| Action is inside approved window | Yes for large objects |
| Object has risk score above threshold | Yes |
| Last action is not too recent | Yes |
| Expected runtime under limit | Yes |
| Rollback path documented | Yes |
| Action is logged | Yes |

## Production operating model

| Cadence | Activity |
|---|---|
| Daily | Review top high-risk objects and anomaly events |
| Weekly | Review false positives, false negatives, and accepted recommendations |
| Monthly | Update KPI/ROI page and business-value summary |
| Quarterly | Revisit thresholds, stale percent preferences, and automation scope |

## Official source basis

- Oracle DBMS_STATS gather table/schema stats and `GATHER STALE`: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- Oracle high-frequency automatic optimizer statistics collection: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- Oracle Optimizer Statistics Advisor workflow: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/optimizer-statistics-advisor.html
- Oracle SQL Tuning Advisor: https://docs.oracle.com/en/database/oracle/oracle-database/26/tdppt/tuning-sql-statements-using-sql-tuning-advisor.html
