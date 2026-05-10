# 08 Risks, Assumptions, and Governance

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | Project Manager / DBA Governance |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, optimizer-statistics, agentic-ai |
| Review cadence | Weekly during build; monthly after go-live |


## Assumptions

| Assumption | Owner | Validation |
|---|---|---|
| Project has access to Oracle dictionary views required for stats monitoring | DBA Lead | Confirm grants |
| SQL performance data source is available | DBA Lead | Confirm `V$SQL`, AWR, or equivalent |
| Business criticality can be mapped by schema/module/service | Product Owner | Define score table |
| DBA team owns DBMS_STATS policy decisions | DBA Manager | Governance approval |
| Automation starts in recommend-only mode | Delivery Lead | Confirm launch plan |
| AWR/ASH/ADDM use follows Oracle licensing and environment rules | DBA Manager | License review |
| Agentic recommendation engine starts with read-only evidence tools | AI Architect | Tooling review |
| Action APIs require approval for risky DBMS_STATS operations | DBA Governance | OpenAPI/approval review |

## Risks and controls

| Risk | Impact | Control |
|---|---|---|
| False positive alerts | Alert fatigue | Tune thresholds; track accepted/rejected recommendations |
| False negative predictions | Missed stale events | Monitor recall; retrain model; keep deterministic Oracle stale checks |
| Over-gathering stats | Extra CPU/I/O and possible plan churn | Use priority score, guardrails, and maintenance windows |
| Large table stats gather affects production | Performance impact | Prefer partition/incremental strategies and runtime prediction |
| Stats are intentionally locked | Automation could violate policy | Block automation when `STATTYPE_LOCKED` is set |
| Model drift | Predictions become unreliable | Monitor prediction quality monthly |
| Missing performance data | Weak risk ranking | Fall back to stale % + business criticality |
| Incomplete object-to-SQL mapping | Impact scoring misses affected SQL | Add SQL plan/object access mapping where available |
| Dashboard lacks ownership | Stale backlog grows | Assign owner and SLA per schema/application |
| Agent calls an action tool without enough review | Unsafe remediation | Keep MVP recommend-only; use approval-required API paths; log every request |
| Supervisor routes the request to the wrong collaborator agent | Bad recommendation or missing evidence | Define routing tests and required evidence checklist |
| RAG retrieves outdated runbook content | Wrong operational guidance | Use approved knowledge sources and review cadence |
| Multi-agent design increases cost or latency | Slower recommendations | Start with single-agent MVP; add collaborators only for production use cases that need specialization |

## Governance rules

1. The first release is **recommend-only**.
2. Production automation requires DBA approval and documented guardrails.
3. Locked statistics are never changed by automation.
4. Large-object actions must respect maintenance windows.
5. Every recommendation and action must be logged.
6. Model outputs must be explainable with reason codes.
7. KPI/ROI must be reported monthly after production launch.
8. Any action that worsens performance must be reviewed and tagged in the feedback log.
9. Agentic tools that can trigger action must use approval controls and authentication.
10. Supervisor/collaborator agent outputs must include evidence and reason codes.

## Decision log

| Date | Decision | Owner | Status | Notes |
|---|---|---|---|---|
| 2026-05-10 | Create separate Problem Statement page | Product Owner | Proposed | Needed for scope alignment |
| 2026-05-10 | Create separate KPI/ROI page | Product Owner | Proposed | Needed for business-value tracking |
| 2026-05-10 | Start with recommend-only automation | DBA Lead | Proposed | Reduces operational risk |
| 2026-05-10 | Use deterministic Oracle stale signal as source of truth | DBA Lead | Proposed | AI predicts and prioritizes; Oracle metadata confirms state |
| 2026-05-10 | Use single-agent MVP, then supervisor/collaborator production design | AI Architect | Proposed | Balances low-complexity pilot with specialized production recommendations |

## Model governance

| Area | Control |
|---|---|
| Training data | Store training snapshot ranges and data filters |
| Model version | Record model name, version, training date, features, metrics |
| Explainability | Store reason codes and top contributing features |
| Monitoring | Track precision, recall, false positives, and missed stale events |
| Retraining | Retrain after schema/workload changes or quality degradation |
| Human override | Allow DBA to accept, reject, defer, or mark as not applicable |
| Agent/tool auditability | Store supervisor output, collaborator evidence, tool calls, approval state, and final action result |

## Official source basis

- Oracle stale statistics and DBMS_STATS behavior: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- Oracle AWR/ADDM and performance tools: https://docs.oracle.com/en/cloud/paas/autonomous-database/dedicated/adbaa/high-performance-features-in-autonomous-ai-database-on.html
- Oracle Diagnostics/Tuning Pack tooling context: https://docs.oracle.com/en/database/oracle/oracle-database/26/tdppt/tuning-sql-statements-using-sql-tuning-advisor.html
- OCI Generative AI Agents supervisor/collaborator agents and API tools: https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/agent-as-tool-guidelines.htm, https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/api-calling-tool-guidelines.htm
- Confluence status and page properties for governance tracking: https://support.atlassian.com/confluence-cloud/docs/insert-the-status-macro/, https://support.atlassian.com/confluence-cloud/docs/insert-the-page-properties-macro/
