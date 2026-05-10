# 09 SQL Appendix

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | DBA Lead |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics, agentic-ai |
| Review cadence | Weekly during build; monthly after go-live |


## Notes

- Replace `:owner`, `:table_name`, `:sql_id`, and other bind variables before running.
- Use `USER_` or `ALL_` views if `DBA_` views are not available.
- Test every script in a non-production environment before operational use.

## 1. List stale or missing table statistics

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
AND    (stale_stats = 'YES' OR stale_stats IS NULL)
ORDER  BY table_name, partition_name, subpartition_name;
```

## 2. List table DML modifications since last stats gather

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

## 3. Calculate stale percentage

```sql
SELECT s.owner,
       s.table_name,
       s.partition_name,
       s.subpartition_name,
       s.num_rows,
       NVL(m.inserts, 0) AS inserts,
       NVL(m.updates, 0) AS updates,
       NVL(m.deletes, 0) AS deletes,
       NVL(m.inserts, 0) + NVL(m.updates, 0) + NVL(m.deletes, 0) AS observed_dml_changes,
       ROUND(
         ((NVL(m.inserts, 0) + NVL(m.updates, 0) + NVL(m.deletes, 0)) / NULLIF(s.num_rows, 0)) * 100,
         2
       ) AS row_change_percent,
       s.stale_stats,
       s.last_analyzed
FROM   dba_tab_statistics s
LEFT JOIN dba_tab_modifications m
       ON m.table_owner = s.owner
      AND m.table_name = s.table_name
      AND NVL(m.partition_name, '-') = NVL(s.partition_name, '-')
      AND NVL(m.subpartition_name, '-') = NVL(s.subpartition_name, '-')
WHERE  s.owner = :owner
ORDER  BY row_change_percent DESC NULLS LAST;
```

## 4. Check stale percent preference

```sql
SELECT DBMS_STATS.GET_PREFS(
         pname   => 'STALE_PERCENT',
         ownname => :owner,
         tabname => :table_name
       ) AS stale_percent
FROM dual;
```

## 5. Find locked statistics

```sql
SELECT owner,
       table_name,
       partition_name,
       subpartition_name,
       stattype_locked,
       stale_stats,
       last_analyzed
FROM   dba_tab_statistics
WHERE  owner = :owner
AND    stattype_locked IS NOT NULL
ORDER  BY table_name, partition_name, subpartition_name;
```

## 6. Gather one table's statistics

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

## 7. Gather stale schema statistics

```sql
BEGIN
  DBMS_STATS.GATHER_SCHEMA_STATS(
    ownname => :owner,
    options => 'GATHER STALE'
  );
END;
/
```

## 8. Enable incremental statistics for a partitioned table

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

## 9. Set incremental staleness to use stale percent

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

## 10. Snapshot feature query skeleton

```sql
INSERT INTO aiops_stale_obj_snapshot (
  owner,
  object_type,
  object_name,
  partition_name,
  subpartition_name,
  num_rows,
  stale_stats,
  last_analyzed,
  stattype_locked,
  stale_percent_pref,
  inserts_since_stats,
  updates_since_stats,
  deletes_since_stats,
  observed_dml_changes,
  row_change_percent,
  truncated
)
SELECT s.owner,
       'TABLE' AS object_type,
       s.table_name,
       s.partition_name,
       s.subpartition_name,
       s.num_rows,
       s.stale_stats,
       s.last_analyzed,
       s.stattype_locked,
       TO_NUMBER(DBMS_STATS.GET_PREFS('STALE_PERCENT', s.owner, s.table_name)) AS stale_percent_pref,
       NVL(m.inserts, 0),
       NVL(m.updates, 0),
       NVL(m.deletes, 0),
       NVL(m.inserts, 0) + NVL(m.updates, 0) + NVL(m.deletes, 0),
       ROUND(((NVL(m.inserts, 0) + NVL(m.updates, 0) + NVL(m.deletes, 0)) / NULLIF(s.num_rows, 0)) * 100, 2),
       m.truncated
FROM   dba_tab_statistics s
LEFT JOIN dba_tab_modifications m
       ON m.table_owner = s.owner
      AND m.table_name = s.table_name
      AND NVL(m.partition_name, '-') = NVL(s.partition_name, '-')
      AND NVL(m.subpartition_name, '-') = NVL(s.subpartition_name, '-')
WHERE  s.owner = :owner;
```

## 11. Performance snapshot query skeleton from `V$SQLAREA`

```sql
SELECT sql_id,
       parsing_schema_name,
       module,
       action,
       plan_hash_value,
       executions,
       elapsed_time,
       cpu_time,
       buffer_gets,
       disk_reads,
       rows_processed,
       last_active_time
FROM   v$sqlarea
WHERE  parsing_schema_name = :owner
ORDER  BY elapsed_time DESC
FETCH FIRST 100 ROWS ONLY;
```

## 12. SQL Tuning Advisor candidate rule

Use this when a SQL statement remains high-risk after statistics are refreshed.

```text
IF stale_stats_after_action = 'NO'
AND sql_elapsed_p95_after > sql_elapsed_p95_baseline * 1.5
THEN recommend SQL tuning review.
```

## 13. ML inference scoring skeleton with `PREDICTION`

Use this pattern after a model is trained and published. Replace `STALE_MODEL` with the approved model name and replace feature columns with the real case-table columns.

```sql
INSERT INTO aiops_stale_prediction (
  owner,
  object_name,
  partition_name,
  p_stale_24h,
  model_name,
  model_version
)
SELECT owner,
       object_name,
       partition_name,
       PREDICTION_PROBABILITY(STALE_MODEL, 1 USING *) AS p_stale_24h,
       'STALE_MODEL' AS model_name,
       :model_version AS model_version
FROM   aiops_stale_obj_features
WHERE  snapshot_ts >= SYSTIMESTAMP - INTERVAL '1' HOUR;
```

## 14. Recommendation request skeleton

```sql
INSERT INTO aiops_recommendation_request (
  request_type,
  owner,
  object_name,
  partition_name,
  triggered_by,
  request_payload
) VALUES (
  'STALE_RISK_REVIEW',
  :owner,
  :object_name,
  :partition_name,
  :triggered_by,
  JSON_OBJECT(
    'p_stale_24h' VALUE :p_stale_24h,
    'anomaly_score' VALUE :anomaly_score,
    'performance_risk_score' VALUE :performance_risk_score,
    'requires_approval' VALUE 'TBD'
  )
);
```

## 15. Agentic recommendation output insert skeleton

```sql
INSERT INTO aiops_recommendation_response (
  request_id,
  supervisor_agent_version,
  current_state,
  risk_level,
  recommended_action,
  confidence,
  requires_approval,
  reason_codes,
  evidence_json,
  next_step
) VALUES (
  :request_id,
  :supervisor_agent_version,
  :current_state,
  :risk_level,
  :recommended_action,
  :confidence,
  :requires_approval,
  :reason_codes,
  :evidence_json,
  :next_step
);
```

## Official source basis

- Oracle stale statistics and DBMS_STATS: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- Oracle table modifications: https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/ALL_TAB_MODIFICATIONS.html
- Oracle SQL performance views and elapsed/buffer/disk metrics: https://docs.oracle.com/en/database/oracle/oracle-database/26/dbiad/db_MMON_MMNL.html
- Oracle SQL tuning: https://docs.oracle.com/en/database/oracle/oracle-database/26/tdppt/tuning-sql-statements-using-sql-tuning-advisor.html
- Oracle Machine Learning scoring and `PREDICTION`: https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/scoring-and-deployment.html, https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/21/dmapi/PREDICTION.html
- OCI Generative AI agent API tools: https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/api-calling-tool-guidelines.htm
