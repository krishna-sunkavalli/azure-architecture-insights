# How to Design for Regional Failures in Azure
## Azure Resiliency Patterns That Actually Hold

---

Regional outages are rare. But rare is not never.

In the past six months: Azure Front Door went down globally, taking with it every workload that had no secondary ingress path. Geopolitical events in the Middle East forced AWS to declare hard-down status across multiple availability zones in Bahrain and Dubai. These are not edge cases in a threat model. They are recent history.

The question is not whether your region will fail. It is whether you designed for it before it did.

---

## Three core principles of Resiliency

1. **Resiliency is stack-wide, not service-wide.** You can make Cosmos DB multi-region and still lose the workload because your Key Vault is single-region, your APIM is Standard tier, or your ExpressRoute circuit terminates on one CPE. The weakest link defines your actual RTO.

2. **SLA math is not resiliency.** A 99.9% SLA allows ~8.7 hours of downtime per year. Compounding three services at 99.9% gives a composite of ~99.7% — over 26 hours. Architects who design to individual SLAs without modeling composite availability are building false confidence.

3. **Most failures happen at the seams.** Not within a service, but between them. DNS misconfiguration at failover. A managed identity with no fallback. A cold standby that was never tested. The table below addresses these seams explicitly.

---

## Service-by-Service Resiliency Guide

Resiliency is only as strong as the weakest service in the stack. The following patterns cover the most common Azure services in a production workload — for each service, we outline the resiliency strategy and the pitfalls that most teams only discover during an actual outage.

---

### Azure Front Door

| | |
|---|---|
| **Resiliency Strategy** | • Use Traffic Manager (Always Serve mode) as DNS-level failover above AFD<br>• Deploy regional Application Gateway + WAF as the failover ingress path<br>• For CDN-heavy workloads: run a secondary CDN with Traffic Manager splitting 90/10 to keep caches warm<br>• Keep DNS TTL low on custom domains for fast re-pointing<br>• Test failover paths regularly |
| **Common Pitfalls** | • Treating AFD as immune to outages because it is a "global service" — the October 2025 incidents proved otherwise<br>• Cold standby Application Gateway paths that have never been tested — they fail silently until needed<br>• WAF in Prevention mode on failover paths blocking legitimate traffic during a real incident |

---

### Azure API Management

| | |
|---|---|
| **Resiliency Strategy** | • Use Premium tier — the only tier supporting availability zones and multi-region deployment<br>• Deploy a minimum of 2 scale units distributed across availability zones<br>• For multi-region HA: deploy APIM instances per region fronted by AFD or Traffic Manager<br>• Implement circuit breaker and backend load balancing policies at the gateway layer<br>• Use APIOps (IaC + GitOps) for rapid configuration recovery — backup/restore alone is insufficient for DR |
| **Common Pitfalls** | • Running Standard or Developer tier in production — neither supports availability zones or multi-region<br>• Relying on APIM backup/restore as the DR strategy — restore is slow and does not recover in-flight traffic<br>• Single-unit deployments without zone distribution — one unit in one zone is not HA |

---

### Azure Kubernetes Service

| | |
|---|---|
| **Resiliency Strategy** | • Enable availability zones on all node pools — the control plane is zone-resilient by default, node pools are not<br>• Run a minimum of 2 system node pool nodes to survive freeze/upgrade cycles<br>• For regional HA: deploy identical clusters in two paired regions with AFD or Traffic Manager (active-active)<br>• Use deployment stamps — each region is a fully isolated unit to limit blast radius<br>• Configure maxSurge on node pool upgrades to prevent downtime during updates |
| **Common Pitfalls** | • Assuming control plane zone-resiliency protects workloads — node pools must be configured independently<br>• Single-node system node pools — a node freeze during upgrade leaves the cluster unable to schedule system pods<br>• Active-active multi-region AKS with stateful workloads and no conflict resolution strategy for shared data |

---

### Azure Service Bus

| | |
|---|---|
| **Resiliency Strategy** | • Use Premium tier — required for availability zones, Geo-Replication, and Geo-Disaster Recovery<br>• Prefer Geo-Replication over Geo-DR — Geo-DR replicates metadata only, not messages<br>• Connect applications via the alias DNS name, not the namespace endpoint directly<br>• Implement dead-letter queue monitoring and a remediation process<br>• Failover is always manually triggered — build and test runbooks |
| **Common Pitfalls** | • Using Geo-Disaster Recovery and assuming messages are protected — they are not; only metadata is replicated<br>• Connecting to the namespace endpoint directly instead of the alias — failover requires application changes<br>• No dead-letter queue remediation process — messages pile up silently after transient failures |

---

### Azure Key Vault

| | |
|---|---|
| **Resiliency Strategy** | • Enable soft delete and purge protection on every vault<br>• Key Vault auto-replicates to paired regions; however, the vault becomes read-only during failover — applications must handle this gracefully<br>• For non-paired regions: deploy a vault per region and sync secrets via automation<br>• Use managed identities over access policies<br>• Alert on vault access failures before they cascade to downstream services |
| **Common Pitfalls** | • Applications that fail hard when Key Vault returns read-only during a regional failover — read-only means get/list still work, but put/delete do not<br>• Soft delete not enabled on legacy vaults — accidental deletion becomes permanent<br>• Single vault serving multiple regions — one regional failure blocks secret access everywhere |

---

### Azure Cosmos DB

| | |
|---|---|
| **Resiliency Strategy** | • Enable at least one secondary region — zero downtime to add, moves SLA from 4 nines to 4.5 nines<br>• Enable service-managed failover — Azure never initiates failover without it<br>• Enable zone redundancy per region for the highest per-region SLA<br>• For 5 nines: combine multi-region + zone redundancy + multi-region writes<br>• Use Session consistency as default — stronger consistency can block writes during read-region outages<br>• Enable continuous backup with point-in-time restore (up to 30 days) |
| **Common Pitfalls** | • Assuming "globally distributed" means resilient — single-region accounts lose write access on a regional outage<br>• Enabling multi-region writes without a conflict resolution policy, then hitting write conflicts in production<br>• Treating periodic backup as a DR strategy — restore is operator-initiated and takes hours |

---

### Azure Storage

| | |
|---|---|
| **Resiliency Strategy** | • LRS is insufficient for production — use ZRS at minimum for zone-level resiliency<br>• Use GZRS (zone-redundant in primary + geo-replicated) for the highest durability tier<br>• Enable RA-GZRS for read-heavy workloads that need secondary read access without failover<br>• Enable soft delete for blobs and containers<br>• Storage failover is customer-initiated — test the process and validate application behavior |
| **Common Pitfalls** | • Defaulting to LRS because it is the cheapest option — it provides no zone or regional protection<br>• Enabling GRS and assuming read access to the secondary — secondary is read-only only with RA-GRS or RA-GZRS<br>• Never testing the failover path — customer-initiated failover has a replication lag that applications must tolerate |

---

### Azure OpenAI Service

| | |
|---|---|
| **Resiliency Strategy** | • Prefer Global Standard deployments — Microsoft manages routing across regions automatically<br>• For PTU: create a secondary PTU or Standard deployment in a failover region and duplicate all model deployments<br>• Front with APIM — APIM handles 429 and 5xx failover; AFD cannot distinguish rate-limit errors from real failures<br>• Stack Standard deployment alongside PTU as a spillover for traffic bursts<br>• Validate required models are available in both primary and failover regions before designing the failover path |
| **Common Pitfalls** | • Using AFD for OpenAI failover — AFD cannot distinguish a 429 (throttle) from a real service failure<br>• Assuming PTU guarantees regional availability — PTU guarantees throughput, not immunity to regional outages<br>• Designing a failover path to a region that does not support your required model |

---

### Azure Monitor / Log Analytics

| | |
|---|---|
| **Resiliency Strategy** | • Deploy regional Log Analytics workspaces alongside workloads — a single central workspace creates a regional dependency for diagnostic data<br>• Use workspace replication (preview) or export pipelines (Event Hub → secondary storage) for critical log retention<br>• Decouple alerting from workspace availability — configure Azure Monitor alerts at the resource level where possible<br>• Export critical data to Azure Storage (GZRS) for long-term durability |
| **Common Pitfalls** | • Single centralized workspace for all regions — a regional outage silences diagnostics exactly when you need them most<br>• Alert rules that query Log Analytics — if the workspace is degraded, alerts stop firing during an incident<br>• No log export strategy — workspace data is inaccessible during a regional event with no secondary copy |

---

### ExpressRoute

| | |
|---|---|
| **Resiliency Strategy** | • Never terminate both circuit connections on the same CPE — eliminates HA at the on-premises edge<br>• Operate both connections in active-active mode — active-passive increases MTTR and wastes capacity<br>• Enable BFD (Bidirectional Forwarding Detection) — reduces link failure detection from ~3 minutes to sub-second<br>• Deploy zone-redundant ExpressRoute Virtual Network Gateways (ErGw1AZ or higher)<br>• For maximum resiliency: use two circuits across two different peering locations<br>• Configure site-to-site VPN only as last-resort backup, not as a primary DR path for latency-sensitive workloads |
| **Common Pitfalls** | • Both circuit connections terminating on the same CPE — a single device failure takes down the circuit entirely<br>• Non-zone-redundant ExpressRoute gateways — a zone failure in Azure takes down on-premises connectivity<br>• VPN as a primary DR path for latency-sensitive workloads — VPN latency and throughput are not comparable to ExpressRoute |

---

### Microsoft Entra ID

| | |
|---|---|
| **Resiliency Strategy** | • Enable Conditional Access resilience defaults — allows the Backup Authentication Service to reissue tokens for existing sessions during an outage<br>• Use MSAL in all applications and persist the token cache for at least 3 days<br>• Use managed identities over service principals with secrets for service-to-service auth<br>• Export tenant configuration regularly (Entra Exporter / Graph API) — CA policies and app registrations have limited native recovery<br>• Enable soft delete awareness — CA policies deleted maliciously have no built-in point-in-time restore |
| **Common Pitfalls** | • Disabling resilience defaults on Conditional Access policies — blocks users entirely during a primary auth service outage<br>• Applications that do not use MSAL or do not persist the token cache — the backup auth system cannot serve them during an outage<br>• Assuming Entra ID is immune to outages — CA policy misconfiguration can replicate globally within minutes |

---

### Azure DNS / Private DNS

| | |
|---|---|
| **Resiliency Strategy** | • Public DNS zones are globally distributed and zone-redundant by default — no additional configuration needed<br>• Private DNS zones are also globally replicated — a regional outage does not affect name resolution in other linked VNets<br>• Link private DNS zones to VNets in every region that needs resolution — missing links cause silent DNS failures at failover time<br>• Deploy Azure DNS Private Resolver in each region for hybrid environments<br>• Test DNS resolution from all regions as part of every DR drill |
| **Common Pitfalls** | • Missing VNet links for private DNS zones in secondary regions — the workload fails over but cannot resolve internal names<br>• Single DNS Private Resolver instance shared across regions — a regional failure breaks on-premises-to-Azure name resolution<br>• DNS failures treated as application failures during DR drills — DNS is the most common silent failure mode in multi-region failovers |

---

### Azure SQL / SQL Managed Instance

| | |
|---|---|
| **Resiliency Strategy** | • Enable zone redundancy for Business Critical or Premium tier<br>• Use Failover Groups for automatic regional failover — provides a stable listener endpoint that survives failover without connection string changes<br>• Set the failover grace period to match your RTO — default is 1 hour, too long for most production workloads<br>• For SQL MI: Failover Groups are the primary DR mechanism; geo-restore is a last resort with higher RPO<br>• Test failover regularly and measure actual application reconnection behavior |
| **Common Pitfalls** | • Relying on geo-restore as the primary DR strategy — restore is slow and results in significant data loss relative to assumed RPO<br>• No retry logic in the application layer — even with Failover Groups, a brief connection interruption will surface as an error<br>• Default failover grace period of 1 hour — for most mission-critical workloads, this must be tuned down |

---

### VPN Gateway

| | |
|---|---|
| **Resiliency Strategy** | • Use Zone-Redundant SKUs (VpnGw1AZ or higher) — non-zonal SKUs are single-point-of-failure within a zone<br>• Deploy in active-active mode with two public IPs — active-passive doubles MTTR on gateway failover<br>• For critical hybrid connectivity: use VPN as backup path alongside ExpressRoute, not as the primary<br>• Configure BGP for dynamic route failover between VPN and ExpressRoute paths<br>• Set DPD timeout appropriately — default values can leave dead tunnels open for minutes |
| **Common Pitfalls** | • Non-zone-redundant gateway SKUs in production — a zone failure takes down all VPN tunnels<br>• Active-passive configuration — on failover the standby instance must initialize, adding minutes to MTTR<br>• VPN as primary DR path for latency or throughput-sensitive workloads — VPN cannot match ExpressRoute characteristics |

---

### Managed Identities

| | |
|---|---|
| **Resiliency Strategy** | • Prefer user-assigned managed identities over system-assigned for workloads that span multiple resources or require failover — system-assigned identities are tied to the resource lifecycle<br>• Pre-assign RBAC roles in the secondary region before a failover event — identity propagation is not instantaneous<br>• Cache tokens in application code using MSAL with a minimum 3-day token lifetime buffer<br>• Validate managed identity role assignments as part of every DR drill — missing role assignments in the secondary are the most common silent failure |
| **Common Pitfalls** | • System-assigned identity on resources that need to failover — identity does not transfer, requiring re-assignment post-failover<br>• Assuming role assignments replicate automatically to new regions or resource instances — they do not<br>• Applications that re-acquire tokens on every request — token fetch failures cascade into full service unavailability |

---

### Azure Container Apps

| | |
|---|---|
| **Resiliency Strategy** | • Deploy Container Apps environments in availability-zone-enabled regions with zone redundancy enabled — zone redundancy must be set at environment creation time and cannot be changed<br>• For multi-region HA: deploy independent environments per region behind Azure Front Door<br>• Use minimum replica count of 3 spread across zones to maintain availability during node failures<br>• Externalize state — Container Apps are stateless; persistent data must live in a separate resilient service<br>• Use Dapr state management or sidecars for service-to-service resilience patterns (retries, circuit breakers) |
| **Common Pitfalls** | • Zone redundancy not enabled at environment creation — it cannot be retrofit; the environment must be recreated<br>• Single-replica deployments with scale-to-zero for production workloads — cold start latency during failover is user-visible<br>• Stateful workloads running inside Container Apps without external durable storage |

---

### Virtual Machines / VMSS

| | |
|---|---|
| **Resiliency Strategy** | • Use Virtual Machine Scale Sets with Flexible orchestration and zone spreading — pin instances across all three AZs<br>• For single critical VMs where VMSS is not appropriate: use availability zones and Azure Site Recovery for regional DR<br>• Enable accelerated networking — it bypasses the host vNIC and survives most host-level maintenance events<br>• Use Azure Spot VMs only for fault-tolerant workloads — eviction is not an outage, it is a design constraint<br>• Automate image builds and test VM bring-up time in DR drills — manual VM deployment during an incident is too slow |
| **Common Pitfalls** | • Availability sets instead of availability zones for new workloads — availability sets protect against rack failures only, not zone-level events<br>• Single VM without Azure Site Recovery and no RTO target — this is an undefined failure mode, not a resilience strategy<br>• VMSS with no overprovision or maxSurge setting — rolling upgrades and Azure healing events can briefly reduce capacity below load |

---

### Microsoft Foundry (Azure AI Foundry)

| | |
|---|---|
| **Resiliency Strategy** | • Treat Foundry projects as region-scoped — plan deployments per region for multi-region workloads<br>• For PTU deployments: provision capacity in two regions; use APIM to route and fail over between them<br>• Store evaluation datasets and agent run history in GZRS storage — Foundry project storage has no built-in geo-replication<br>• Use deployment slots or versioned model deployments to enable zero-downtime model updates<br>• Monitor token exhaustion and 429s at the APIM layer — these are the primary availability failure mode for AI workloads |
| **Common Pitfalls** | • Treating a single Foundry project as a globally available service — it is regional<br>• No APIM or gateway layer between applications and Foundry — 429 throttling surfaces as application errors with no retry logic<br>• Agent workflows with no timeout or circuit breaker — a slow or unavailable model endpoint will block threads indefinitely |

---

### Microsoft Fabric

| | |
|---|---|
| **Resiliency Strategy** | • Fabric capacity is region-scoped — identify critical workspaces and understand which region hosts them<br>• For disaster recovery: back up critical lakehouses and warehouses to GZRS storage accounts outside the Fabric capacity<br>• Use OneLake shortcuts to external ADLS Gen2 storage for data that must survive a Fabric regional outage<br>• Export critical semantic models and dataflows as source-controlled artifacts — platform-level recovery restores infrastructure, not your definitions<br>• Monitor capacity utilization — overloaded capacity throttles all items, which looks like an outage |
| **Common Pitfalls** | • Assuming shared Fabric capacity is HA — the SLA applies to the service, not to individual capacity availability<br>• No export of workspace definitions — a capacity rebuild without source-controlled artifacts means manual recreation<br>• OneLake treated as the only copy of critical data with no secondary storage backup |

---

### Azure Databricks

| | |
|---|---|
| **Resiliency Strategy** | • Deploy workspaces in zone-enabled regions; use instance pools pinned to multiple zones to reduce cold start time<br>• For regional DR: deploy a secondary workspace in a paired region with mirrored cluster policies and job definitions managed via IaC<br>• Store all notebooks, job definitions, and cluster configs in source control — the workspace is ephemeral; the code is not<br>• Use Delta Lake with GZRS-backed ADLS Gen2 as the storage layer — Delta transaction logs are the recovery mechanism<br>• Automate workspace provisioning with Terraform or ARM — manual reconstruction during an incident is not a DR plan |
| **Common Pitfalls** | • Notebooks stored only in the workspace UI with no source control — a workspace loss means code loss<br>• Jobs configured manually with no IaC equivalent — rebuilding job definitions from memory during an incident<br>• Single-AZ instance pools — spot eviction or zone failure drains the pool and triggers slow cold starts under load |

---

### Azure Firewall

| | |
|---|---|
| **Resiliency Strategy** | • Deploy with availability zone support — zone-redundant Azure Firewall spreads instances across all three AZs automatically<br>• Use Azure Firewall Policy over classic rules — Policy is globally shareable, version-controlled, and survives firewall replacement<br>• For multi-region: deploy a firewall per region; use a shared Firewall Policy hierarchy to push rules consistently<br>• Use Firewall Manager to manage policies centrally — direct rule management per firewall does not scale to multi-region<br>• Back up Firewall Policy definitions in source control; the policy resource itself can be deleted |
| **Common Pitfalls** | • Non-zone-redundant firewall — a zone failure removes the single instance and takes down all egress<br>• Classic rules instead of Firewall Policy — rules are local to the firewall; a replacement loses all configuration<br>• Single Firewall Policy with no source control backup — accidental rule deletion can propagate to all regions with no restore path |

---

### Azure Load Balancer

| | |
|---|---|
| **Resiliency Strategy** | • Use Standard SKU — Basic SKU has no SLA, no zone support, and no availability guarantees<br>• Enable zone-redundant frontend IPs — the load balancer frontend survives individual zone failures without reconfiguration<br>• Configure health probes correctly — an incorrect probe URL or port causes the LB to mark all backends unhealthy<br>• Use connection draining (TCP reset on idle timeout) to avoid dropping in-flight connections during backend scaling events<br>• For cross-region load balancing: use Azure Front Door or cross-region Load Balancer (preview) |
| **Common Pitfalls** | • Basic SKU in production — no SLA and no zone support means no availability guarantee<br>• Missing or misconfigured health probes — the most common cause of "the load balancer is broken" incidents is a probe hitting a wrong path<br>• Zone-pinned frontend IPs instead of zone-redundant — a zone failure removes the frontend IP and breaks all traffic |

---

### Azure Application Gateway

| | |
|---|---|
| **Resiliency Strategy** | • Deploy Application Gateway v2 with availability zones — v2 is required for zone support; v1 is end-of-support<br>• Set minimum instance count to 2 (preferably 3 for zone spread) — autoscale minimum of 0 or 1 creates a cold-start window during traffic spikes<br>• Use WAF Policy (per-site) over global WAF config — WAF Policy supports exclusions per listener and survives gateway replacement<br>• For multi-region: deploy an Application Gateway per region behind Azure Front Door; use AFD health probes to detect regional failure<br>• Monitor the backend health API — unhealthy backend pools are silent unless explicitly alarmed |
| **Common Pitfalls** | • Application Gateway v1 in production — v1 has no zone support and is on the end-of-support path<br>• Minimum autoscale instance count of 0 — the gateway must scale from zero under sudden traffic, introducing latency exactly when it matters<br>• WAF in Prevention mode on a failover path that was never tested — legitimate traffic gets blocked during the first real failover |

---

## Closing

Resiliency is not a feature you turn on. It is a set of deliberate decisions made per service, per failure domain, and per workload criticality. Not every workload needs active-active across regions. But every workload needs an honest answer to the question: what happens when this region is unavailable, and have we tested it?

The patterns above are a starting point. The real work is translating them into architecture decisions, validating them in DR drills, and revisiting them every time the stack changes. Untested resiliency is not resiliency — it is an assumption.
