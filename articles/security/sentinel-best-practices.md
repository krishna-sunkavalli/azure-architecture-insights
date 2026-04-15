---
title: "Microsoft Sentinel Best Practices: Decisions, Trade-offs, and Validation"
date: 2026-04-14
author: "Krishna Sunkavalli"
tags:
  - security
  - governance
  - production
  - azure
description: "Practical Sentinel decisions every SOC architect faces, the right call for each, the trade-offs to own, and KQL to validate you got it right."
categories:
  - AI Architecture
image: assets/images/sentinel-best-practices-social.png
---

# Microsoft Sentinel Best Practices: Decisions, Trade-offs, and Validation

---

Sentinel deployments fail in predictable ways. Ingestion sprawl drives the cost wall. Stale analytics rules generate noise without catching threats. UEBA sits unconfigured because no one owns the enablement decision. The Defender portal migration gets deferred until it becomes forced. None of these are capability gaps. They are deferred architectural decisions.

This article covers the decisions that matter, the right practice for each, the trade-offs that come with it, and KQL queries to validate the state of your environment.

---

## Best Practices Reference

| Decision | Best Practice | Trade-offs |
|---|---|---|
| **Connector prioritization** | Enable Microsoft first-party connectors first (Defender XDR, Entra ID, M365). They are free-tier or already licensed, provide the highest signal density, and underpin UEBA and fusion detection. Add third-party connectors only after first-party coverage is complete. | Third-party connectors deferred too long can leave blind spots in non-Microsoft environments. Sequence must match actual threat surface, not licensing convenience. |
| **Pre-ingestion log filtering** | Use Azure Monitor Agent Data Collection Rules (DCRs) to filter Windows and Linux events before ingestion. DCRs are the preferred mechanism: they keep logs in their native table type, preserve free-tier eligibility, and maintain UEBA and ML compatibility. | DCR filtering requires Azure Monitor Agent deployment. Legacy MMA-based deployments cannot use DCRs and require Logstash or custom code for filtering. |
| **Logstash filtering trade-off** | Avoid using Logstash to filter content from logs that are on the free-tier list (Windows Security Events, Syslog via connector). Logstash re-ingests filtered output as custom logs, converting free-tier data to paid and removing UEBA, ML, and fusion support. | When Logstash is the only viable option (air-gapped environments, unsupported Linux distros), the cost and capability penalties are unavoidable. Document explicitly which tables were affected and why. |
| **Data tier placement** | Analytics tier for tables with active detection rules, UEBA signals, or real-time hunting. Data lake tier for forensics, compliance, and historical data. | Lake-tier data requires promotion jobs before rules can fire on it. Promotion adds latency and operational complexity. |
| **Retention strategy** | Set analytics-tier retention to match your detection lookback window (90-180 days for most rules). Use lake tier for long-term retention up to 12 years. | Longer analytics retention increases cost linearly. Shorter retention limits historical correlation without lake queries. |
| **External data sources** | Federate data from Fabric, ADLS Gen2, or Databricks before deciding to ingest it. Use investigation outcomes to determine which federated sources earn ingestion. | Federated sources do not support detection rules or UEBA. Queries against federated data depend on source availability and access policies. |
| **Defender portal migration** | Migrate now. The Azure portal retirement is March 31, 2027. Unified XDR plus Sentinel incident correlation is operationally superior to running both portals in parallel. | Post-migration, Sentinel workbooks and some automation playbooks require review. Legacy Azure-portal-only workflows need updating. |
| **UEBA enablement** | Enable UEBA with at minimum Azure Active Directory and sign-in log sources. UEBA enrichments on incidents reduce investigation time significantly. | UEBA increases ingestion volume from identity sources. Baseline establishment takes 7-14 days after enablement before anomaly scoring stabilizes. |
| **Analytics rule hygiene** | Audit rules monthly. Disable rules that have not fired in 60 days or consistently generate false positives above 80%. Keep the active rule set lean. | Fewer rules mean potential coverage gaps. Every disabled rule needs a documented reason and review date. |
| **Watchlist design** | Use watchlists for enrichment at query time, not as staging tables for large datasets. Keep individual watchlists under 10,000 rows. | Watchlists above the row threshold degrade query performance. Large enrichment sets belong in the data lake, queryable via KQL. |
| **KQL query optimization** | Filter on indexed columns first (`TimeGenerated`, `Computer`, `Account`). Apply `where` before `project`. Avoid `contains` in favor of `has` for string matching. | Optimized queries are less readable for junior analysts. Add inline comments to preserve intent when tuning for performance. |
| **Custom graph traversal** | Build identity relationship graphs (group membership, SP permissions) via Sentinel VS Code extension notebooks. Schedule as recurring jobs. | Graph creation requires Security Operator or higher. Graphs have 30-day default retention on on-demand schedules. Scheduled jobs require explicit retention configuration. |
| **Incident management workflow** | Close incidents within 48 hours of triage. Use the incident graph before pivoting to raw logs. Tag false positives with a consistent classification for rule tuning feedback. | 48-hour SLA requires staffed tier-1 coverage. Without it, incident backlog degrades MTTR metrics and masks real signal. |
| **Summarization rules** | Use summarization rules to pre-aggregate high-volume tables (CommonSecurityLog, Syslog, ASimNetworkSessionLogs) into compact summary tables. Run detection rules against the summary table, not the raw source. | Summarized tables lose row-level fidelity. Any rule that requires raw event detail cannot run against a summary. Raw tables must be retained in parallel for investigation. |

---

## When to Route Logs to the Data Lake Tier

The analytics tier is not the right home for every log source. Sending everything there is how teams hit the cost wall without improving detection coverage. The question to answer for each data source is: does this table need to support real-time detection rules, UEBA signals, or sub-minute hunting? If the answer is no, it belongs in the lake.

The following table provides guidance on log routing decisions by source type.

| Log Source | Recommended Tier | Reason |
|---|---|---|
| Azure Activity Logs | Analytics (90 days) + Lake | Activity logs feed UEBA and privileged action detection. Full history in the lake for compliance. |
| Azure Firewall / NSG Flow Logs | Lake tier (summarize for analytics) | Raw flow logs are high volume with low per-row signal. Summarize by IP, port, and direction before promoting to analytics. |
| Sign-in logs (Entra ID) | Analytics tier | UEBA baseline and identity detection rules depend on sign-in telemetry in analytics. |
| Syslog / CEF from Linux endpoints | Lake tier + summarization rule | Most Syslog volume is infrastructure noise. Summarize to detect anomalies; keep raw in the lake for forensics. |
| Microsoft Defender XDR alerts | Analytics tier | Defender alerts are already correlated and low volume. They belong in analytics for incident creation. |
| DNS query logs | Lake tier | DNS logs are extremely high volume. Promote only flagged domains or query anomalies to analytics via scheduled jobs. |
| Audit logs (M365, Entra) | Analytics (180 days) + Lake | Compliance mandates retention. Detection rules for data exfiltration and privilege escalation need analytics-tier access. |
| Custom application logs | Federate first | Evaluate signal value via federation before committing to ingestion. Most app logs deliver value only in correlation with other sources. |
| Cloud infrastructure telemetry (resource logs) | Lake tier | Resource logs support forensic investigation and cost attribution. Promotion jobs can surface anomalies to analytics on a schedule. |
| Threat intelligence feeds | Analytics tier | TI feeds power detection rules and enrichment lookups. They must remain in analytics regardless of volume. |

The pattern that works in production: ingest high-fidelity signal sources (identity, alerts, TI) into analytics. Route everything else to the lake. Use summarization rules to distill high-volume sources into compact tables that analytics rules can query without the raw ingestion cost. Promote from the lake to analytics only when a source consistently delivers confirmed detections.

---

## KQL Queries to Validate Your Environment

### Find Your Highest-Cost Tables

Run this before any retention or connector decision. The tables at the top of this list are your first optimization candidates.

```kql
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize TotalGB = round(sum(Quantity) / 1024, 2) by DataType
| order by TotalGB desc
| take 20
```

### Identify Stale Analytics Rules

Rules that have not fired in 60 days are either missing coverage or generating no value. Either outcome requires action.

```kql
SecurityAlert
| where TimeGenerated > ago(60d)
| summarize LastFired = max(TimeGenerated) by AlertName, ProductName
| where LastFired < ago(60d)
| order by LastFired asc
```

### Check Data Connector Health

Connectors that stopped reporting surface as blind spots before they surface as missed incidents.

```kql
_SentinelHealth
| where TimeGenerated > ago(1d)
| where SentinelResourceKind == "DataConnector"
| where Status != "Success"
| project TimeGenerated, ConnectorName = SentinelResourceName, Status, Description
| order by TimeGenerated desc
```

### Measure Incident MTTR by Classification

If your median MTTR exceeds 24 hours, the bottleneck is either triage workflow, analyst capacity, or automation coverage.

```kql
SecurityIncident
| where TimeGenerated > ago(30d)
| where Status == "Closed"
| extend MTTR_hours = datetime_diff('hour', ClosedTime, CreatedTime)
| summarize
    Median_MTTR = percentile(MTTR_hours, 50),
    P95_MTTR = percentile(MTTR_hours, 95),
    TotalIncidents = count()
    by Classification
| order by Median_MTTR desc
```

### Check UEBA Enrichment Coverage on Incidents

A low "Yes" count means UEBA is either not enabled or not feeding incident enrichment. Either blocks AI-assisted triage.

```kql
SecurityIncident
| where TimeGenerated > ago(30d)
| extend HasUEBA = iff(
    isnotempty(tostring(AdditionalData.ueba_enrichments)),
    "Enriched",
    "Not Enriched"
  )
| summarize Count = count() by HasUEBA
```

### Detect Agents That Stopped Heartbeating

Machines missing heartbeat for over 4 hours represent gaps in endpoint telemetry coverage.

```kql
Heartbeat
| where TimeGenerated > ago(24h)
| summarize LastSeen = max(TimeGenerated) by Computer, OSType
| where LastSeen < ago(4h)
| order by LastSeen asc
```

### Audit Watchlist Row Counts

Watchlists over 10,000 rows degrade KQL query performance at detection rule execution time.

```kql
_GetWatchlist("")
| summarize Rows = count() by WatchlistName = WatchlistAlias
| where Rows > 10000
| order by Rows desc
```

### Identify Summarization Rule Candidates

Tables contributing more than 5 GB/day with no corresponding summarization rule are prime cost reduction candidates. This query surfaces the top-volume tables to evaluate.

```kql
Usage
| where TimeGenerated > ago(7d)
| where IsBillable == true
| summarize DailyAvgGB = round(sum(Quantity) / 1024 / 7, 2) by DataType
| where DailyAvgGB > 5
| order by DailyAvgGB desc
```

### Detect Custom Log Tables That May Have Lost Free-Tier Eligibility

If Logstash filtering was applied to originally free-tier sources, the output lands in custom tables (`_CL` suffix) and becomes billable. This query surfaces all custom tables with significant volume.

```kql
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| where DataType endswith "_CL"
| summarize TotalGB = round(sum(Quantity) / 1024, 2) by DataType
| order by TotalGB desc
```

### Measure Summarization Rule Coverage Savings

Compare raw table volume against its corresponding summary table to quantify how much the summarization rule is compressing. Replace `ASimNetworkSessionLogs` and `NetworkSummary` with your actual table names.

```kql
let RawGB = toscalar(
    Usage
    | where TimeGenerated > ago(7d)
    | where DataType == "ASimNetworkSessionLogs" and IsBillable == true
    | summarize round(sum(Quantity) / 1024 / 7, 2)
);
let SummaryGB = toscalar(
    Usage
    | where TimeGenerated > ago(7d)
    | where DataType == "NetworkSummary" and IsBillable == true
    | summarize round(sum(Quantity) / 1024 / 7, 2)
);
print
    RawTable_DailyGB = RawGB,
    SummaryTable_DailyGB = SummaryGB,
    CompressionRatio = round(RawGB / SummaryGB, 1)
```

---

## The Decision That Cannot Be Deferred

Every other best practice on this list can be implemented incrementally. The Defender portal migration cannot. After March 31, 2027, the Azure portal will redirect all Sentinel customers to the Defender portal. Teams that treat this as future-state planning will hit the deadline mid-investigation with untested workflows and unreviewed playbooks.

Start the migration in a non-production workspace. Validate your automation rules, playbooks, and workbooks in the unified portal before cutting over production. The operational improvement is real. Use the forcing function.

