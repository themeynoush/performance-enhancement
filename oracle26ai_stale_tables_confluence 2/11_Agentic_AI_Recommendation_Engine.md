# 11 Agentic AI Recommendation Engine

> **Confluence setup**: place the table below inside a **Content Properties** macro so the project home page can roll up status, owner, and decision fields.

| Field | Value |
|---|---|
| Page owner | AI Architect / DBA Lead |
| Status | Draft |
| Last updated | 2026-05-10 |
| Related labels | oracle26ai, aiops, stale-statistics, db-performance, agentic-ai, recommendations |
| Review cadence | Weekly during build; monthly after go-live |


## Purpose

Define the recommendation engine as an agentic AI component that converts database evidence, ML predictions, workload impact, and runbook knowledge into a safe DBA recommendation.

## Scope

The recommendation engine does **not** decide whether a table is stale by itself. Oracle determines the current stale state through optimizer statistics metadata. The agentic layer explains and prioritizes what to do next.

## Recommended production design

Use a **supervisor agent with collaborator agents**. In project language, we can call the collaborator agents **subagents**. In Oracle documentation, they are **collaborator agents** configured as Agent tools under a supervisor agent.

```text
DBA question, alert, or scheduled review
        ↓
Supervisor Recommendation Agent
        ↓
+-------------------------+-------------------------+-------------------------+
| Stale Stats Agent       | Performance Impact Agent | ML Risk Agent           |
| Current stale/missing   | SQL workload, plans,     | p_stale, time-to-stale, |
| stats, stats locks,     | AWR/ASH/ADDM signals     | anomaly and risk scores |
| stale percent           |                         |                         |
+-------------------------+-------------------------+-------------------------+
        ↓                         ↓                         ↓
+-------------------------+-------------------------+
| Remediation Policy Agent| Runbook/RAG Agent        |
| DBMS_STATS strategy,    | Similar incidents,       |
| guardrails, approvals   | runbooks, tuning notes   |
+-------------------------+-------------------------+
        ↓
Supervisor aggregates evidence, applies output contract, and returns one recommendation.
```

## Why supervisor + collaborator agents fit this project

The production recommendation requires multiple specializations:

| Specialization | Why it should be separated |
|---|---|
| Stale-stat evidence | Uses deterministic Oracle metadata and should avoid mixing current truth with prediction |
| Performance impact | Looks at SQL workload, plan changes, and historical performance evidence |
| ML risk | Interprets prediction, anomaly, regression, and duration scores |
| Remediation policy | Applies DBMS_STATS guardrails, maintenance windows, stats locks, and approval rules |
| Runbook/RAG retrieval | Retrieves similar operational cases and explanations from text knowledge |

A supervisor pattern lets the system route sub-questions to the right collaborator agent, then aggregate the responses into a single recommendation.

## Single agent vs multi-agent/subagent comparison

| Case | Single agent / single workflow | Supervisor + collaborator agents / subagents |
|---|---|---|
| MVP | Best fit | Possible, but heavier than needed |
| Number of tools | Few tools | Many tools with different permissions and purposes |
| Number of specializations | One combined prompt can handle it | Better when stale stats, performance, ML, policy, and runbook retrieval are separate concerns |
| Governance | Simpler to review | Easier to isolate high-risk tools and approval paths by agent role |
| Observability | One set of logs/metrics | Separate metrics and billing per supervisor, collaborator agents, and tools |
| Latency/cost | Usually lower | Usually higher because multiple agents/tools may be called |
| Scalability | Prompt and tool logic can become crowded | Easier to add or replace specialist agents |
| Failure modes | One agent may confuse evidence, prediction, and action policy | Supervisor must handle routing, aggregation, and dependency failures |
| Recommendation | Use for pilot | Use for production recommendation engine |

## Staged implementation

| Stage | Pattern | Outcome |
|---|---|---|
| Stage 1 | Rule engine only | Deterministic stale-stat inventory and priority queue |
| Stage 2 | Single agent workflow | Explain one object, one schema, or one alert using rules + ML scores |
| Stage 3 | Supervisor + collaborator agents | Production-grade recommendation engine with specialized evidence, ML, performance, policy, and RAG agents |
| Stage 4 | Controlled action tool | Agent recommends an action; approved API submits a DBMS_STATS job or ticket |

## Agent roles

### 1. Supervisor Recommendation Agent

Responsibilities:

- Accept DBA question, alert payload, or scheduled review context
- Decide which collaborator agents/tools are required
- Merge collaborator outputs
- Apply the final recommendation output contract
- Enforce “recommend first, act only through approved action API”

Output contract:

```json
{
  "object": "OWNER.TABLE_OR_PARTITION",
  "current_state": "STALE | FRESH | MISSING_STATS | LOCKED | UNKNOWN",
  "risk_level": "LOW | MEDIUM | HIGH | CRITICAL",
  "recommended_action": "NO_ACTION | WATCH | GATHER_TABLE_STATS | GATHER_PARTITION_STATS | GATHER_SCHEMA_STALE_STATS | REVIEW_LOCKED_STATS | SQL_TUNING_REVIEW | DBA_APPROVAL_REQUIRED",
  "confidence": 0.0,
  "evidence": [],
  "reason_codes": [],
  "requires_approval": true,
  "next_step": "string"
}
```

### 2. Stale Stats Agent

Inputs:

- `DBA_TAB_STATISTICS`
- `DBA_IND_STATISTICS`
- `DBA_TAB_MODIFICATIONS`
- `DBA_TAB_STAT_PREFS`
- `STALE_PERCENT`
- stats lock state

Outputs:

- current stale/missing/fresh state
- row-change percent
- stale threshold rows
- lock/truncate/missing-stats flags

### 3. Performance Impact Agent

Inputs:

- watched SQL metrics
- execution counts
- plan hash changes
- AWR/ASH/ADDM or equivalent performance summaries
- service/module/action mapping

Outputs:

- impacted SQL list
- workload criticality
- before/after metric trend
- plan-instability evidence

### 4. ML Risk Agent

Inputs:

- prediction table
- model metadata
- evaluation metrics
- drift indicators

Outputs:

- stale probability
- time-to-stale
- anomaly score
- performance-risk score
- expected stats-gather duration

### 5. Remediation Policy Agent

Inputs:

- action guardrails
- maintenance windows
- object size/partitioning
- stats locks
- previous action failures

Outputs:

- approved action candidate
- required approval level
- safe execution window
- “do not automate” reason when applicable

### 6. Runbook/RAG Agent

Inputs:

- runbooks
- incident notes
- SQL tuning notes
- DBA comments
- post-action summaries

Outputs:

- similar incidents
- relevant runbook sections
- explanation snippets
- known caveats

## Tooling model

| Tool | Purpose | Guardrail |
|---|---|---|
| Read-only SQL/API tool | Query current evidence and model outputs | No DML or DBMS_STATS execution |
| Knowledge base / RAG tool | Retrieve runbooks and similar incidents | Use official/internal approved content only |
| Action API endpoint tool | Submit approved remediation request | OpenAPI schema, authentication, network controls, and approval flags |
| Notification tool | Send alert or approval request | Include evidence and reason codes |
| Audit log tool | Store recommendation and decision | Required for all recommendations |

## Recommendation decision matrix

| Current state | ML/performance signal | Recommended action |
|---|---|---|
| `STALE_STATS = 'NO'` | Low risk | No action |
| `STALE_STATS = 'NO'` | High DML velocity and high `p_stale_24h` | Watch or schedule future stats review |
| `STALE_STATS = 'YES'` | Low workload impact | Watch or gather during normal maintenance |
| `STALE_STATS = 'YES'` | Critical SQL impact | DBA-priority recommendation |
| `STALE_STATS IS NULL` | Object active | Gather missing stats or DBA review |
| Stats locked | Any risk level | Do not auto-gather; DBA approval required |
| Large partitioned table, one active stale partition | High risk | Gather partition stats or review incremental strategy |
| SQL remains slow after fresh stats | High performance risk | SQL tuning review |
| Stats job duration predicted above window | Any stale state | Defer, split, partition, or approval-required plan |

## Final recommendation

Use **single-agent workflow for the MVP** and **supervisor + collaborator agents for production**.

Reason:

- The MVP needs simple validation of data collection, stale-state logic, ML score quality, and DBA trust.
- The production use case crosses multiple expert domains and requires routing, aggregation, tool separation, and approval control.
- Oracle documentation explicitly describes supervisor agents that route queries to collaborator agents and aggregate their responses, and also supports single-step and multi-step workflows through the OCI Responses API.

## Official source basis

- Oracle stale statistics and DBMS_STATS: https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/gathering-optimizer-statistics.html
- OCI Generative AI Responses API: https://docs.oracle.com/en-us/iaas/Content/generative-ai/get-started-agents.htm
- OCI Generative AI Agents overview: https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/overview.htm
- OCI supervisor/collaborator agents and Agent tool: https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/agent-as-tool-guidelines.htm
- OCI API endpoint calling tools: https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/api-calling-tool-guidelines.htm
- OCI Generative AI Agents knowledge bases: https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/create-knowledge-base.htm
- Oracle AI Vector Search: https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/overview-ai-vector-search.html
- Oracle Machine Learning for SQL and scoring: https://docs.oracle.com/en/database/oracle/machine-learning/omlad/oracle-machine-learning-sql.html, https://docs.oracle.com/en/database/oracle/machine-learning/oml4sql/23/mlsql/scoring-and-deployment.html
