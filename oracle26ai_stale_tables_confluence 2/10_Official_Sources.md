# 10 Official Sources

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | Project Manager |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics, agentic-ai |
| Review cadence | Weekly during build; monthly after go-live |


## Source policy

This project documentation uses official Oracle and Atlassian sources only.

## Oracle Database and optimizer statistics sources

| Topic | Official source |
|---|---|
| Gathering optimizer statistics, stale stats, `STALE_PERCENT`, `DBMS_STATS`, high-frequency stats, incremental stats | https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html |
| Optimizer Statistics Advisor | https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/optimizer-statistics-advisor.html |
| Table modification counters | https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/ALL_TAB_MODIFICATIONS.html |
| Table statistics reference | https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/DBA_TAB_STATISTICS.html |
| Statistics preferences reference | https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/DBA_TAB_STAT_PREFS.html |
| `DBMS_STATS` package | https://docs.oracle.com/en/database/oracle/oracle-database/26/arpls/DBMS_STATS.html |
| `DBMS_SCHEDULER` / job scheduling | https://docs.oracle.com/en/database/oracle/oracle-database/21/arpls/DBMS_SCHEDULER.html |
| AWR/ADDM and Autonomous AI Database performance tools | https://docs.oracle.com/en/cloud/paas/autonomous-database/dedicated/adbaa/high-performance-features-in-autonomous-ai-database-on.html |
| AWR, ASH, MMON, MMNL historical performance data | https://docs.oracle.com/en/database/oracle/oracle-database/26/dbiad/db_MMON_MMNL.html |
| SQL Tuning Advisor | https://docs.oracle.com/en/database/oracle/oracle-database/26/tdppt/tuning-sql-statements-using-sql-tuning-advisor.html |
| Enhanced automatic SQL Plan Management | https://docs.oracle.com/en/database/oracle/oracle-database/26/nfcoa/data_analytics_sql.html |
| Execution plan changes and optimizer inputs | https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/generating-and-displaying-execution-plans.html |

## Oracle AI, ML, and vector sources

| Topic | Official source |
|---|---|
| Oracle AI Database 26ai AI, ML, and Analytics landing page | https://docs.oracle.com/en/database/oracle/oracle-database/26/ai.html |
| Oracle Machine Learning for SQL overview | https://docs.oracle.com/en/database/oracle/machine-learning/omlad/oracle-machine-learning-sql.html |
| Oracle Machine Learning techniques, data preparation, and scoring requirements | https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/machine-learning-functions1.html |
| Oracle Machine Learning scoring and deployment | https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/scoring-and-deployment.html |
| Oracle `PREDICTION` SQL function | https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/21/dmapi/PREDICTION.html |
| `DBMS_DATA_MINING` | https://docs.oracle.com/en/database/oracle/oracle-database/26/arpls/DBMS_DATA_MINING.html |
| Oracle AI Vector Search overview | https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/overview-ai-vector-search.html |
| `DBMS_VECTOR` package | https://docs.oracle.com/en/database/oracle/oracle-database/26/arpls/dbms_vector1.html |
| ONNX text embedding pipeline | https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/onnx-pipeline-models-text-embedding.html |

## OCI Generative AI and agentic AI sources

| Topic | Official source |
|---|---|
| OCI Generative AI overview | https://docs.oracle.com/en-us/iaas/Content/generative-ai/overview.htm |
| OCI Responses API for agentic applications | https://docs.oracle.com/en-us/iaas/Content/generative-ai/get-started-agents.htm |
| OCI Generative AI Agents overview | https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/overview.htm |
| Supervisor and collaborator agents / Agent tool guidelines | https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/agent-as-tool-guidelines.htm |
| API endpoint calling tools | https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/api-calling-tool-guidelines.htm |
| Knowledge bases for Generative AI Agents | https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/create-knowledge-base.htm |
| IAM policies for Generative AI Agents | https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/iam-policies.htm |

## OCI Data Integration sources

| Topic | Official source |
|---|---|
| Creating a Data Integration data asset | https://docs.oracle.com/en-us/iaas/Content/data-integration/using/creating-data-assets.htm |
| Oracle Database data asset properties | https://docs.oracle.com/en-us/iaas/Content/data-integration/using/creating-oracle-db.htm |
| Creating a Data Integration data flow | https://docs.oracle.com/en-us/iaas/Content/data-integration/using/creating-a-data-flow.htm |
| Data Integration pipeline tutorial | https://docs.oracle.com/en-us/iaas/Content/data-integration/tutorials/06-use-a-pipeline.htm |
| Creating a pipeline task | https://docs.oracle.com/en-us/iaas/Content/data-integration/using/creating-a-pipeline-task.htm |
| Data Integration metrics | https://docs.oracle.com/en-us/iaas/Content/data-integration/using/metrics.htm |

## Atlassian Confluence sources

| Topic | Official source |
|---|---|
| Create Confluence templates | https://support.atlassian.com/confluence-cloud/docs/create-a-template/ |
| Labels for organization | https://support.atlassian.com/confluence-cloud/docs/use-labels-to-organize-your-content/ |
| Content Properties macro | https://support.atlassian.com/confluence-cloud/docs/insert-the-page-properties-macro/ |
| Status macro | https://support.atlassian.com/confluence-cloud/docs/insert-the-status-macro/ |
| Custom report / content properties report | https://support.atlassian.com/confluence-cloud/docs/create-a-custom-report/ |

## Notes for reviewers

- Any new factual claim added to these pages should be linked to an official Oracle or Atlassian source.
- Internal project facts, such as owners, dates, system names, baseline KPIs, and decisions, should be sourced from internal project records once available.
- Avoid community blogs, third-party explainers, and unofficial examples unless the team explicitly changes the source policy.
