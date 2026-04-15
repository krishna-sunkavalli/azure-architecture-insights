---
title: "Sentinel Is an Architecture Decision, Not a SIEM Configuration"
date: 2026-04-14
author: "Krishna Sunkavalli"
tags:
  - security
  - governance
  - production
  - azure
description: "Most teams solve Sentinel cost and coverage problems by tuning ingestion. The real fix is a data architecture that separates visibility from ownership."
categories:
  - AI Architecture
image: assets/images/sentinel-best-practices-social.png
---

# Sentinel Is an Architecture Decision, Not a SIEM Configuration

## Why Your Coverage Gaps and Cost Problems Have the Same Root Cause

---

Most teams hit the Sentinel cost wall around six months in. Data sources proliferate. Ingestion volume climbs. The bill arrives. The response is almost always the same: start cutting connectors, apply ingestion filters, disable tables. Coverage shrinks. Detection quality degrades. The team calls this optimization. It is actually triage after a design decision they made too early, without enough information, and without a framework that fit the problem.

The ingestion-first instinct is understandable. SIEM has meant "ingest everything into one place" for twenty years. But Sentinel in 2026 is not that model. It is a security analytics platform with a layered data architecture, and the teams still operating it as a flat ingestion pipe are paying for an approach the product has already moved past.

---

## The Assumption That Creates Cost Spirals

The default Sentinel deployment puts everything into the analytics tier: high-performance, indexed, real-time queryable, priced accordingly. That tier exists for a specific purpose. It is where detection rules run, where incidents are created, where UEBA operates, where AI models consume signals. It is not where twelve months of raw firewall logs should live while you figure out whether you ever need them.

The data lake tier changes this. It stores up to 12 years of security data at a fraction of the cost, in open-format Parquet, queryable with KQL and Python. When analytics-tier data ages, it mirrors automatically to the lake tier. You retain a single copy of every event without paying analytics-tier pricing for data you query twice a year. The operational model shifts from "what can we afford to keep?" to "what do we actually need in fast storage?"

The practical architectural decision becomes: what belongs in each tier? The answer is not a ratio or a default retention policy. It is a function of your detection strategy. Data that feeds active detection rules, UEBA baselines, and near-real-time hunting belongs in analytics. Data retained for forensics, compliance, and historical correlation belongs in the lake. Data that lives in other systems and carries its own governance requirements does not belong in Sentinel at all. It belongs federated.

---

## Federation Is Not a Storage Feature

Data federation in Sentinel, now in public preview as of April 2026, allows security teams to query Microsoft Fabric, Azure Data Lake Storage Gen 2, and Azure Databricks in place, without copying data into Sentinel. This is documented as a cost and coverage improvement. That framing undersells what it actually changes.

In most enterprises, the most sensitive and analytically rich security data is in systems the SOC does not own. Finance transaction logs live in a regulated data warehouse. HR activity data is controlled by a compliance team. Application telemetry sits in a Databricks environment owned by engineering. SOC teams spend months negotiating to get copies of this data, or they run blind to signals that only appear when you correlate across domains.

Federation resolves the ownership conflict without removing it. The data does not move. Data owners retain their governance policies, their access controls, their compliance posture. The SOC team issues KQL queries that span Sentinel-native tables and federated sources in a single hunt. When a threat investigation needs twelve months of identity correlation against HR offboarding records, the analyst does not need to ask someone to export a CSV. The data is queryable where it already lives.

This matters architecturally because it separates visibility from ownership for the first time. Governance and access control are organizational contracts, not just technical configurations. The right data architecture respects those contracts rather than working around them.

The progression that makes operational sense is: federate first to establish coverage and validate signal value, then ingest the data that consistently delivers detection outcomes into the analytics tier to unlock rule-based alerting, UEBA, and AI-driven correlation. Federation is not a replacement for ingestion. It is a mechanism for making ingestion decisions with evidence rather than assumption.

---

## The Architecture Decision That Needs a Deadline

By March 31, 2027, Microsoft Sentinel will no longer be supported in the Azure portal. All customers will be redirected to the Microsoft Defender portal.

Teams treating this as a portal migration are underestimating the change. The Defender portal unifies Sentinel and Defender XDR into a single incident management surface, a single advanced hunting environment, and a single identity correlation model. Teams currently operating Sentinel in Azure portal and Defender XDR as separate products are making a hunting query in one system, pivoting to a separate incident queue in the other, and manually correlating what the unified platform does automatically.

The practical recommendation: start the Defender portal onboarding now, not because the deadline is close, but because the operational improvement is immediate. Unified incident management reduces analyst context switching. Defender XDR signals are correlated with Sentinel rules in a single timeline. The migration is also the moment to rationalize your data connectors, confirm your analytics-tier retention strategy, and validate that your UEBA baseline is drawing from the right sources.

The following table describes the key architectural decisions and their operational implications across the three platform layers.

| Decision | Analytics Tier | Data Lake Tier | Federation |
|---|---|---|---|
| Intended use | Active detection, UEBA, AI | Forensics, compliance, historical hunting | Governed external data |
| Query performance | Real-time | Batch and near-real-time | Depends on source |
| Cost profile | High | Low | Ingestion-free |
| Retention | Up to 2 years standard | Up to 12 years | Unlimited (source-controlled) |
| Supports detection rules | Yes | Via promotion jobs | No |
| Supports hunting | Yes | Yes (KQL + Python) | Yes (single KQL across both) |

---

## Graph Traversal Changes Identity Investigation

Custom graphs in the Sentinel data lake represent a different class of investigative capability than KQL alone can provide. When investigating identity-based threats, the limiting factor is not data availability. It is the cost of traversing relationships. A user account in a group that is nested inside three other groups, one of which has an overpermissioned service principal attached to a production resource, is invisible to a flat KQL query unless the analyst already knows to look for it.

Custom graphs, built via Jupyter notebooks in the Sentinel VS Code extension, model identity relationships as graph structures. A single Graph Query Language query can traverse nested group memberships up to eight levels deep, return all service principals reachable from a given group, or map every path from a compromised account to a production resource. The query that replaces what previously required multiple KQL joins and manual enrichment looks like this:

```gql
MATCH p=(g1:EntraGroup)-[cg]->{1,8}(g2)
WHERE g1.displayName = 'TargetGroup'
RETURN *
```

The operational shift is significant. During a breach investigation involving identity, the question is not just "what did this account do?" but "what could this account reach?" Answering the second question with flat table queries requires knowing the structure of the identity hierarchy before you start. Graph traversal finds the structure for you.

Custom graphs are scheduled via notebook jobs, persist in the tenant, and are queryable from the Defender portal graph experience. The RBAC model requires care: creating a graph requires a scoped user with data management permissions, but querying a persisted graph requires only security data read permissions. Designed correctly, the analyst who needs to investigate does not need the same permissions as the analyst who built the detection model.

---

## What This Requires You to Rethink

The shift from ingestion-first SIEM to layered security data architecture has consequences that go beyond the Sentinel configuration.

Your data governance conversations need to change. Federation makes it possible to query data you do not own, but those conversations with finance, HR, and engineering still need to happen. The difference is that you are now proposing read access to existing data rather than asking for a copy. That is a materially easier negotiation, but it still requires upfront agreement on scope and purpose.

Your detection strategy needs to be explicit. The analytics tier should contain exactly the data your detection rules reference. If a table has no active rule, no UEBA signal, and no watchlist dependency, it belongs in the lake. Auditing this mapping annually prevents the ingestion sprawl that drives the cost problems in the first place.

Your SOC team needs KQL depth in the lake tier. Lake-tier KQL queries behave differently from analytics-tier queries in ways that matter operationally. Job creation, data promotion mechanics, and cost implications of running full-table scans against twelve years of data are not topics covered in standard KQL training. Build this competency before you move data to the lake and find that no one knows how to get it back.

---

## The Architecture Posture Going Forward

Sentinel's current direction is a unified security data platform: tiered storage, federated external sources, graph-based identity modeling, and AI operating across the full data layer. Teams that adopt this architecture as a data design problem rather than a connector configuration problem will have materially better coverage at lower cost with fewer governance conflicts. The migration to the Defender portal is a forcing function. Use it as the moment to reconsider the data architecture, not just the portal preference.

The teams that will struggle in 2027 are the ones still ingesting everything into one table, complaining about the bill, and looking for another connector to disable.
