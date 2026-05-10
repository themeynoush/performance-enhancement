# 03 Oracle 26ai Stale Statistics Detection Logic

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | DBA Lead |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics |
| Review cadence | Weekly during build; monthly after go-live |


## Purpose

Document the deterministic Oracle logic used to detect stale optimizer statistics, and define how this project calculates row-change ratio, stale risk, and remediation action.

## Oracle stale-stat concepts

Oracle monitors approximate DML operations on tables. To check whether statistics are stale, query `STALE_STATS` in `DBA_TAB_STATISTICS` and `DBA_IND_STATISTICS`. The stale flag is based on `DBA_TAB_MODIFICATIONS` and the `STALE_PERCENT` preference.

`DBA_TAB_MODIFICATIONS` includes approximate counts of inserts, updates, and deletes since the last time statistics were gathered.

## Core formula

```text
observed_dml_changes = INSERTS + UPDATES + DELETES
stale_threshold_rows = NUM_ROWS * (STALE_PERCENT / 100)
row_change_percent = (observed_dml_changes / NULLIF(NUM_ROWS, 0)) * 100
```

Default Oracle behavior is usually:

```text
If row_change_percent > 10%, then statistics are considered stale.
```

Use the configured table preference instead of hardcoding 10% whenever possible:

```sql
SELECT DBMS_STATS.GET_PREFS(
         pname   => 'STALE_PERCENT',
         ownname => :owner,
         tabname => :table_name
       ) AS stale_percent
FROM dual;
```

## Direct stale-stat check

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
ORDER  BY table_name, partition_name, subpartition_name;
```

Interpretation:

| `STALE_STATS` value | Meaning |
|---|---|
| `YES` | Statistics are stale |
| `NO` | Statistics are not stale |
| `NULL` | Statistics are not collected |

## DML modification check

```sql
SELECT table_owner,
       table_name,
       partition_name,
       subpartition_name,
       inserts,
       updates,
       deletes,
       timestamp,
       truncated
FROM   dba_tab_modifications
WHERE  table_owner = :owner
ORDER  BY timestamp DESC;
```

## Row-change calculation query

```sql
WITH prefs AS (
  SELECT owner,
         table_name,
         TO_NUMBER(DBMS_STATS.GET_PREFS('STALE_PERCENT', owner, table_name)) AS stale_percent
  FROM   dba_tables
  WHERE  owner = :owner
), mods AS (
  SELECT table_owner AS owner,
         table_name,
         NVL(partition_name, '-') AS partition_name,
         NVL(subpartition_name, '-') AS subpartition_name,
         SUM(NVL(inserts, 0)) AS inserts,
         SUM(NVL(updates, 0)) AS updates,
         SUM(NVL(deletes, 0)) AS deletes,
         MAX(truncated) AS truncated
  FROM   dba_tab_modifications
  WHERE  table_owner = :owner
  GROUP  BY table_owner, table_name, NVL(partition_name, '-'), NVL(subpartition_name, '-')
)
SELECT s.owner,
       s.table_name,
       s.partition_name,
       s.subpartition_name,
       s.num_rows,
       p.stale_percent,
       NVL(m.inserts, 0) AS inserts,
       NVL(m.updates, 0) AS updates,
       NVL(m.deletes, 0) AS deletes,
       NVL(m.inserts, 0) + NVL(m.updates, 0) + NVL(m.deletes, 0) AS observed_dml_changes,
       ROUND(
         ((NVL(m.inserts, 0) + NVL(m.updates, 0) + NVL(m.deletes, 0)) / NULLIF(s.num_rows, 0)) * 100,
         2
       ) AS row_change_percent,
       s.stale_stats,
       s.last_analyzed,
       m.truncated
FROM   dba_tab_statistics s
JOIN   prefs p
       ON p.owner = s.owner
      AND p.table_name = s.table_name
LEFT JOIN mods m
       ON m.owner = s.owner
      AND m.table_name = s.table_name
      AND NVL(s.partition_name, '-') = m.partition_name
      AND NVL(s.subpartition_name, '-') = m.subpartition_name
WHERE  s.owner = :owner
ORDER  BY row_change_percent DESC NULLS LAST;
```

## Remediation options

| Situation | Recommended action |
|---|---|
| Small stale table | Gather table stats |
| Many stale objects in schema | `GATHER_SCHEMA_STATS` with `options => 'GATHER STALE'` or `GATHER AUTO` |
| Large partitioned table | Prefer partition-aware and incremental statistics strategy |
| DML spikes between maintenance windows | Consider high-frequency automatic optimizer statistics collection |
| Volatile table rebuilt often | Review dynamic statistics / locked stats strategy with DBA approval |
| Missing stats | Gather stats or verify intended exception |
| Locked stats | Review whether stats lock is intentional |

## Example DBMS_STATS actions

Gather one table:

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

Gather stale/missing stats in one schema:

```sql
BEGIN
  DBMS_STATS.GATHER_SCHEMA_STATS(
    ownname => :owner,
    options => 'GATHER STALE'
  );
END;
/
```

Enable incremental statistics for a partitioned table:

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

Use stale percent for incremental staleness:

```sql
BEGIN
  DBMS_STATS.SET_TABLE_PREFS(
    ownname => :owner,
    tabname => :table_name,
    pname   => 'INCREMENTAL_STALENESS',
    pvalue  => 'USE_STALE_PERCENT'
  );
END;
/
```

## Small vs large table strategy

| Object type | Strategy |
|---|---|
| Small table | Full stats collection is usually cheap; gather when stale |
| Medium table | Use `GATHER STALE` / `GATHER AUTO`; monitor runtime |
| Large non-partitioned table | Tune `STALE_PERCENT`, schedule carefully, track resource impact |
| Large partitioned table | Use partition-level stats and incremental stats where appropriate |
| Frequently changing table | Consider high-frequency stats collection and anomaly alerts |

## Official source basis

- Oracle stale stats, `DBA_TAB_STATISTICS`, `STALE_PERCENT`, and `GATHER STALE`: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- Oracle table modification counters: https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/ALL_TAB_MODIFICATIONS.html
- Oracle incremental statistics: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- Oracle Optimizer Statistics Advisor: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/optimizer-statistics-advisor.html
