---
title: "Defender for AI Services and AI SPM: KQL Reference for SOC Teams"
date: 2026-04-27
author: "Krishna Sunkavalli"
tags:
  - security
  - governance
  - production
  - azure
description: "A practical KQL reference for SOC analysts investigating AI workload threats, validating Defender for AI Services coverage, and troubleshooting AI SPM posture gaps."
categories:
  - AI Architecture
image: assets/images/defender-for-ai-services-social.png
---

# Defender for AI Services and AI SPM: KQL Reference for SOC Teams

---

Prompt injection, jailbreak attempts, and anomalous token consumption are real attack patterns in production AI environments. Defender for AI Services and AI SPM generate the signals to detect them. What most SOC teams are missing is the query layer: the KQL to turn those signals into actionable triage. This reference covers coverage validation, posture gaps, threat detection, and cross-signal correlation, organized by the question an analyst is actually trying to answer.

## Quick Navigation

| Category | What it covers |
|---|---|
| [Coverage and Posture](#coverage-and-posture) | Inventory, missing Defender coverage, public exposure, overprivileged identities, open recommendations |
| [Threat Detection](#threat-detection) | All AI alerts, prompt injection trends, jailbreak confidence, anomalous token consumption, sensitive data in responses |
| [Alert Operations](#alert-operations) | Incident conversion rate, connector health, AI resource configuration changes |
| [Cross-Signal Correlation](#cross-signal-correlation) | AI alerts with identity risk, AI alerts with impossible travel |

---

## Coverage and Posture

### Inventory All AI Resources in Scope

Before any triage, confirm what Defender for Cloud actually sees. Resources in unmanaged subscriptions produce no alerts and appear in no attack paths.

Run this in Azure Resource Graph Explorer, not in Sentinel. `SecurityResources` is an ARG table.

```kql
// Azure Resource Graph query (run in ARG Explorer, not Sentinel)
securityresources
| where type =~ "microsoft.cognitiveservices/accounts"
| extend Kind = tostring(properties.kind)
| extend PublicAccess = tostring(properties.publicNetworkAccess)
| extend ProvisioningState = tostring(properties.provisioningState)
| project subscriptionId, resourceGroup, name, Kind, PublicAccess, ProvisioningState, location
| order by subscriptionId, resourceGroup
```

### Find AI Resources Without Defender Coverage

AI SPM surfaces this as a recommendation. This KQL surfaces the same gap without navigating the Defender portal UI.

```kql
SecurityRecommendation
| where TimeGenerated > ago(1d)
| where RecommendationName has_any ("Defender for Azure AI", "Microsoft Defender for Azure AI Services", "Enable Microsoft Defender")
| where RecommendationState == "Unhealthy"
| project TimeGenerated, AffectedResourceType, AffectedResourceName, RecommendationName, RecommendationSeverity
| order by RecommendationSeverity asc
```

### Surface Publicly Exposed AI Endpoints

Default Azure OpenAI deployments have public network access enabled. This query surfaces AI resources that AI SPM has flagged for public exposure.

```kql
SecurityRecommendation
| where TimeGenerated > ago(1d)
| where RecommendationName has_any ("public network access", "private endpoint", "network access")
| where AffectedResourceType has_any ("microsoft.cognitiveservices", "microsoft.machinelearningservices")
| where RecommendationState == "Unhealthy"
| project TimeGenerated, AffectedResourceName, AffectedResourceType, RecommendationName, RecommendationSeverity, RemediationDescription
| order by RecommendationSeverity asc
```

---

## Threat Detection

### All Defender for AI Alerts in the Last 30 Days

Start here. Understand the volume and distribution of alert types before tuning anything.

```kql
SecurityAlert
| where TimeGenerated > ago(30d)
| where ProductName == "Microsoft Defender for Cloud"
| where AlertType has_any ("AI", "OpenAI", "CognitiveServices", "PromptInjection", "Jailbreak")
| summarize
    AlertCount = count(),
    FirstSeen = min(TimeGenerated),
    LastSeen = max(TimeGenerated)
    by AlertName, AlertType, AlertSeverity
| order by AlertCount desc
```

### Prompt Injection Alert Trend

Rising prompt injection volume without a corresponding application change is an active attack signal, not ambient noise.

```kql
SecurityAlert
| where TimeGenerated > ago(30d)
| where ProductName == "Microsoft Defender for Cloud"
| where AlertName has_any ("Prompt Injection", "Jailbreak", "Indirect Prompt")
| summarize DailyCount = count() by bin(TimeGenerated, 1d), AlertName
| render timechart
```

### Jailbreak Attempts With High Confidence

High-confidence jailbreak alerts are the ones to triage first. Low-confidence alerts carry higher false positive rates and should inform tuning before driving incident creation.

```kql
SecurityAlert
| where TimeGenerated > ago(7d)
| where ProductName == "Microsoft Defender for Cloud"
| where AlertName has "Jailbreak"
| extend Confidence = tostring(parse_json(ExtendedProperties).Confidence)
| where Confidence in ("High", "Medium")
| project TimeGenerated, AlertName, AlertSeverity, Confidence,
    AffectedResource = tostring(Entities),
    Description
| order by TimeGenerated desc
```

### Anomalous Token Consumption by AI Resource

Token consumption spikes on a narrow set of user identities correlate with exfiltration attempts. This query uses Azure OpenAI diagnostic logs.

```kql
AzureDiagnostics
| where TimeGenerated > ago(7d)
| where ResourceType == "MICROSOFT.COGNITIVESERVICES/ACCOUNTS"
| where Category == "RequestResponse"
| extend CompletionTokens = toint(column_ifexists("completion_tokens_s", "0"))
| extend PromptTokens = toint(column_ifexists("prompt_tokens_s", "0"))
| summarize
    TotalCompletionTokens = sum(CompletionTokens),
    TotalPromptTokens = sum(PromptTokens),
    RequestCount = count()
    by bin(TimeGenerated, 1h), ResourceId, callerIpAddress
| where TotalCompletionTokens > 100000
| order by TotalCompletionTokens desc
```

### AI Resources With Open Sensitive Data Recommendations

AI SPM flags data stores connected to AI pipelines that contain sensitive data without adequate controls. This query surfaces those recommendations.

```kql
SecurityRecommendation
| where TimeGenerated > ago(1d)
| where RecommendationName has_any ("sensitive data", "data classification", "Purview")
| where AffectedResourceType has_any ("microsoft.cognitiveservices", "microsoft.machinelearningservices", "microsoft.storage", "microsoft.sql")
| where RecommendationState == "Unhealthy"
| project TimeGenerated, AffectedResourceName, AffectedResourceType, RecommendationName, RecommendationSeverity
| order by RecommendationSeverity asc
```

### Overprivileged Identities on AI Resources

AI SPM surfaces identities with Contributor or Owner access on AI resources that only require inference-level permissions. This query surfaces the recommendation class.

```kql
SecurityRecommendation
| where TimeGenerated > ago(1d)
| where RecommendationName has_any ("overprivileged", "excessive permissions", "least privilege", "RBAC")
| where AffectedResourceType has_any ("microsoft.cognitiveservices", "microsoft.machinelearningservices")
| where RecommendationState == "Unhealthy"
| project TimeGenerated, AffectedResourceName, AffectedResourceType, RecommendationName, RecommendationSeverity, RemediationDescription
| order by RecommendationSeverity asc
```

### Sensitive Data in AI Responses

Defender for AI flags responses matching sensitive data patterns: credentials, PII, confidential document content. This query surfaces those alerts and links to the diagnostic log window for the flagged request.

```kql
SecurityAlert
| where TimeGenerated > ago(7d)
| where ProductName == "Microsoft Defender for Cloud"
| where AlertName has_any ("Sensitive Data", "Data Exfiltration", "Confidential")
| extend ResourceId = tostring(parse_json(Entities)[0].ResourceId)
| project TimeGenerated, AlertName, AlertSeverity, ResourceId, Description, RemediationSteps
| order by TimeGenerated desc
```

---

## Alert Operations

### AI Alert to Incident Conversion Rate

Alerts that are not converted to incidents are not being triaged. A low conversion rate against AI alerts means they are falling through the automation rule or analyst triage gap.

```kql
let AIAlertIds = SecurityAlert
    | where TimeGenerated > ago(30d)
    | where ProductName == "Microsoft Defender for Cloud"
    | where AlertType has_any ("AI", "OpenAI", "CognitiveServices", "PromptInjection", "Jailbreak")
    | project SystemAlertId, AlertSeverity;
let IncidentAlertIds = SecurityIncident
    | where TimeGenerated > ago(30d)
    | mv-expand AlertIds = parse_json(AlertIds)
    | project LinkedAlertId = tostring(AlertIds);
AIAlertIds
| extend HasIncident = SystemAlertId in (IncidentAlertIds)
| summarize
    TotalAlerts = count(),
    AlertsWithIncident = countif(HasIncident)
    by AlertSeverity
| extend ConversionRate = round(100.0 * AlertsWithIncident / TotalAlerts, 1)
| order by AlertSeverity asc
```

### Defender for Cloud Connector Health

A connector that stopped streaming will silently lose AI alerts. This catches it before an incident is missed.

```kql
_SentinelHealth
| where TimeGenerated > ago(1d)
| where SentinelResourceKind == "DataConnector"
| where SentinelResourceName has_any ("Defender for Cloud", "Microsoft Defender for Cloud", "Azure Security Center")
| where Status != "Success"
| project TimeGenerated, ConnectorName = SentinelResourceName, Status, Description
| order by TimeGenerated desc
```

### AI Resource Deployment Changes

Model deployments, capacity changes, and network configuration modifications on AI resources are administrative operations with security implications. This surfaces them from the activity log.

```kql
AzureActivity
| where TimeGenerated > ago(30d)
| where ResourceProviderValue == "MICROSOFT.COGNITIVESERVICES"
| where OperationNameValue has_any ("write", "delete", "action")
| where ActivityStatusValue == "Success"
| project TimeGenerated, Caller, OperationNameValue, ResourceGroup, Resource, SubscriptionId
| order by TimeGenerated desc
```

### Unresolved Critical and High AI SPM Recommendations

AI SPM recommendations that have been open longer than 30 days represent accepted risk without a documented decision. Surface them before they surface in an audit.

```kql
SecurityRecommendation
| where TimeGenerated > ago(1d)
| where RecommendationSeverity in ("Critical", "High")
| where AffectedResourceType has_any ("microsoft.cognitiveservices", "microsoft.machinelearningservices")
| where RecommendationState == "Unhealthy"
| extend DaysOpen = datetime_diff('day', now(), TimeGenerated)
| where DaysOpen > 30
| project AffectedResourceName, AffectedResourceType, RecommendationName, RecommendationSeverity, DaysOpen, RemediationDescription
| order by DaysOpen desc
```

---

## Cross-Signal Correlation

### AI Alerts With Concurrent Identity Risk

An AI alert and a high-risk Entra ID sign-in for the same user within the same hour is a compound signal worth escalating. This query surfaces those correlations.

```kql
let AIAlerts = SecurityAlert
    | where TimeGenerated > ago(7d)
    | where ProductName == "Microsoft Defender for Cloud"
    | where AlertType has_any ("AI", "OpenAI", "PromptInjection", "Jailbreak")
    | extend AlertUser = tostring(parse_json(Entities)[0].Name)
    | project AlertTime = TimeGenerated, AlertUser, AlertName, AlertSeverity;
let RiskySignins = SigninLogs
    | where TimeGenerated > ago(7d)
    | where RiskLevelDuringSignIn in ("high", "medium")
    | project SigninTime = TimeGenerated, UserPrincipalName, RiskLevelDuringSignIn, Location;
AIAlerts
| join kind=inner (RiskySignins) on $left.AlertUser == $right.UserPrincipalName
| where abs(datetime_diff('minute', AlertTime, SigninTime)) < 60
| project AlertTime, AlertUser, AlertName, AlertSeverity, RiskLevelDuringSignIn, Location, SigninTime
| order by AlertTime desc
```

### AI Alerts With Impossible Travel

A prompt injection alert from a resource combined with an impossible travel signal on the same user confirms account compromise, not just a misconfigured application. This correlation runs against Entra ID sign-in risk events.

```kql
let AIAlerts = SecurityAlert
    | where TimeGenerated > ago(7d)
    | where ProductName == "Microsoft Defender for Cloud"
    | where AlertType has_any ("AI", "OpenAI", "PromptInjection", "Jailbreak")
    | extend AlertUser = tostring(parse_json(Entities)[0].Name)
    | project AlertTime = TimeGenerated, AlertUser, AlertName, AlertSeverity;
let ImpossibleTravel = SigninLogs
    | where TimeGenerated > ago(7d)
    | where RiskEventTypes_V2 has "impossibleTravel"
    | project SigninTime = TimeGenerated, UserPrincipalName, Location, IPAddress;
AIAlerts
| join kind=inner (ImpossibleTravel) on $left.AlertUser == $right.UserPrincipalName
| where abs(datetime_diff('hour', AlertTime, SigninTime)) < 24
| project AlertTime, AlertUser, AlertName, AlertSeverity, Location, IPAddress, SigninTime
| order by AlertTime desc
```

---

## What the Queries Cannot Tell You

Every query in this reference depends on two prerequisites that KQL cannot validate: Defender for AI Services must be enabled on the resources generating the signals, and Azure OpenAI diagnostic logging must be routed to the same Log Analytics workspace. The inventory query surfaces the first gap. A missing `AzureDiagnostics` result for a known OpenAI resource surfaces the second.

Run the coverage and posture queries before investing time in the detection queries. Alert triage against an environment with incomplete coverage produces false confidence. The absence of prompt injection alerts does not mean injection is not happening. It may mean Defender for AI Services was never enabled on the resource being targeted.
