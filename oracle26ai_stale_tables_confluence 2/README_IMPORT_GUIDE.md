# Import Guide — Oracle AI Database 26ai Stale Tables Project

This folder contains Confluence-ready Markdown pages for a project that uses Oracle AI Database 26ai, Oracle Machine Learning, data ingestion pipelines, and an agentic AI recommendation engine to detect, predict, and prioritize stale optimizer statistics.

## Recommended Confluence page tree

```text
AI + Oracle 26ai Stale Tables Monitoring
├── 00 Project Home
├── 01 Problem Statement and Use Cases
├── 02 KPI, ROI, and Business Value
├── 03 Oracle 26ai Stale Statistics Detection Logic
├── 04 AI Anomaly Detection and Prediction Pipeline
├── 05 Solution Architecture and Data Flow
├── 06 Data Model and Features
├── 07 Implementation Plan and Runbook
├── 08 Risks, Assumptions, and Governance
├── 09 SQL Appendix
├── 10 Official Sources
└── 11 Agentic AI Recommendation Engine
```

## Confluence best-practice setup

1. Create one parent page named **AI + Oracle 26ai Stale Tables Monitoring**.
2. Create each Markdown file as a child page under that parent.
3. Add the labels below to every page:

```text
oracle26ai aiops stale-statistics db-performance optimizer-statistics agentic-ai
```

4. On each page, wrap the metadata table in Confluence’s **Content Properties** macro. Then add a **Content Properties Report** macro to the project home page to show owner, status, review cadence, and decisions across all child pages.
5. Use Confluence statuses such as **Draft**, **In Review**, **Approved**, and **Live** for page lifecycle tracking.
6. Convert repeated sections into a Confluence template after the first review cycle.

## Official Confluence basis

Atlassian documents custom templates, template variables, labels, content properties, status reporting, and content properties reports here:

- https://support.atlassian.com/confluence-cloud/docs/create-a-template/
- https://support.atlassian.com/confluence-cloud/docs/use-labels-to-organize-your-content/
- https://support.atlassian.com/confluence-cloud/docs/insert-the-page-properties-macro/
- https://support.atlassian.com/confluence-cloud/docs/insert-the-status-macro/
- https://support.atlassian.com/confluence-cloud/docs/create-a-custom-report/

## Output format

These are Markdown files designed to be copied into Confluence Cloud pages. SQL blocks are fenced and can be pasted into SQL Developer, SQLcl, or SQL*Plus after schema and privilege adjustments.
