---
title: "Azure Quotas Are a Capacity Planning Problem, Not a Support Ticket"
date: 2026-04-13
author: "Krishna Sunkavalli"
tags:
  - azure
  - governance
  - production
  - compute
description: "Most quota limits surface during a scale event or a failover, not during planning. Treating quota as a capacity planning input changes what you provision and when."
categories:
  - Governance
image: assets/images/azure-quotas-social.png
---

# Azure Quotas Are a Capacity Planning Problem, Not a Support Ticket

Quota errors show up at the worst possible moments. An AKS cluster trying to add nodes during a load spike. A DR failover exercise that turns into a real failover before the increase request is approved. A VM Scale Set that stops scaling at 80% of its declared ceiling because nobody validated the limit before setting the max count.

The frustration is real. The framing is usually wrong. Quota limits are treated as a support escalation path when they are actually a capacity planning input. Teams that manage them reactively are building operational plans on an assumption that has not been validated.

---

## What Quotas Actually Are

An Azure quota is a limit on how much of a specific resource type can be provisioned within a subscription and region. That combination matters: quota for Standard Dv5 vCPUs in West Europe is entirely independent of the same quota in North Europe. Increasing one does not affect the other. Running out in one region does not constrain the other.

Quotas are not a billing mechanism. You can have an enterprise agreement with committed spend and still hit a quota limit. A subscription with no spending cap will fail a deployment if the regional vCPU limit for that SKU family is at its ceiling. The two systems are separate and always have been.

The reason quotas exist is physical. Azure regions are not infinitely elastic. Each region is a collection of datacenters with a finite amount of hardware: CPU cores, GPU cards, NVMe storage, network switching, and power. That capacity is shared across millions of subscriptions. Quota is the control mechanism that prevents any single subscription from exhausting regional supply at the expense of everyone else. It also gives Microsoft the signal it needs to plan and build new capacity ahead of demand.

Understanding this changes the problem framing. A quota limit is not Azure saying no. It is the resource manager enforcing a ceiling that was set before your subscription had a precise demand profile. The correct response is to update that ceiling before you need it, not after.

---

## The Granularity That Catches Teams Off Guard

Compute quota under Azure Resource Manager is not a single regional pool. It is tracked per subscription, per region, and per SKU family. Standard_Dv5, Standard_Ev5, Standard_NCv3, and Standard_NDAMSv4 each have their own independent limit. Exhausting Ev5 quota does not affect your Dv5 allocation. The granularity matters when you are evaluating whether switching instance families unblocks a deployment or whether a quota increase is the only path forward.

Check what you actually have:

```azurecli
az vm list-usage \
  --location <region> \
  --query "[?contains(name.value, 'Family')].[name.localizedValue, currentValue, limit]" \
  --output table
```

For services outside of compute, the quota model works differently.

**SQL Managed Instance** quota is not about raw vCore counts. It is about the number of managed instance resources per region, with sub-limits by hardware generation. Premium-series hardware has a smaller regional footprint than Gen5, and availability varies by region. In some regions it is not a quota problem at all: the hardware is genuinely constrained and no increase request will resolve it. That is a capacity availability scenario, not a quota scenario, and the resolution path is choosing a different region or hardware generation.

**Microsoft Fabric** F-SKU availability follows the same pattern. An F64 or F128 in a given region may simply be unavailable at a point in time. The portal will tell you the SKU is in high demand. No quota increase changes that until Microsoft expands physical capacity in that region.

**GPU families** (NC, ND, NDAMSv4) are the highest-friction quota to manage. Default allocation for these families starts at zero on most subscriptions. If your AI infrastructure plan assumes GPU availability without a prior quota validation, that assumption has no backing. Request GPU quota increases well in advance. The automated approval path does not apply to these families: they require manual review and the lead time is measured in days, not minutes.

---

## Default Quotas Are Deliberately Conservative

Every new subscription starts with conservative defaults. A new subscription has no usage history and no established demand profile. Pre-allocating significant regional capacity to it reduces availability for subscriptions with known, active workloads. The defaults are sized for experimentation, not production.

The default regional vCPU limit across most subscription types is between 20 and 100 cores depending on offer type. For a standalone VM or a small development workload that is sufficient. For a production AKS cluster with autoscaling configured, or a VM Scale Set sized for peak traffic, it is not.

The gap between default quota and production requirements is predictable. You know before deployment whether your workload fits inside defaults. The only question is whether you validate this during architecture design or discover it during a deployment failure.

Request increases before you need them:

```azurecli
az quota create \
  --resource-name "standardDSv4Family" \
  --scope "/subscriptions/<sub-id>/providers/Microsoft.Compute/locations/<region>" \
  --limit-object value=200 limit-object-type=LimitValue \
  --resource-type dedicated
```

Increases for standard compute families are typically approved within minutes through the automated path. GPU families, SQL Managed Instance, and capacity-constrained services require manual review. Build that lead time into your planning cycle, not your incident response.

---

## The DR Region Problem

This is where quota gaps become an availability risk.

The common pattern: primary region has adequate quota, secondary region carries default quota. The assumption is that the secondary region is idle under normal conditions and does not need real capacity pre-allocated. The moment failover is required, provisioning requests hit default limits. The increase approval process begins during an active incident.

DR quota needs to be sized to run the failed-over workload without any additional requests. That means requesting it at environment design time, not after the first failover drill reveals the gap.

A less visible version of the same problem: AKS clusters with an autoscaler max node count that exceeds the vCPU quota for that SKU family. Everything functions at normal load. Under sustained traffic, the autoscaler requests additional nodes, VM provisioning fails silently, and pods accumulate in Pending state. The autoscaler does not surface this as a quota error. It manifests as application latency and stuck scheduling, and the root cause takes time to find.

Validate the scaling ceiling before you set it:

```azurecli
# Available quota for the SKU family
az vm list-usage \
  --location <region> \
  --query "[?name.value=='standardDSv4Family'].{allocated:limit, used:currentValue, available:join('/', [to_string(currentValue), to_string(limit)])}" \
  --output table

# Current VM consumption in the subscription
az vm list \
  --query "[?location=='<region>'].{name:name, size:hardwareProfile.vmSize}" \
  --output table
```

If the autoscaler max times node vCPU count exceeds available quota, resize the max before deployment or submit an increase request.

---

## Quota Groups: The Platform Engineering Answer

When you operate across multiple subscriptions, quota management becomes a cross-subscription coordination problem. A subscription for AKS workloads, a subscription for data platform resources, a subscription for shared services: each has its own quota ceiling, and managing increases independently across all of them does not scale.

Azure Quota Groups eliminate the per-subscription ceiling by pooling quota allocation across multiple subscriptions under a single ARM object anchored to a Management Group. Instead of filing increase requests against individual subscriptions and manually redistributing headroom, the platform team manages a group-level ceiling and transfers quota to subscriptions as needed, without Microsoft intervention for each reallocation.

One thing the feature does not change: quota operations are still scoped per region and per VM family. A Quota Group increase request for Standard_Dv5 in West Europe is a separate request from the same SKU family in North Europe. The group removes the subscription-level friction; it does not collapse the regional dimension. You still need to plan and request headroom per operating region. The benefit is that once that headroom exists at the group level, reallocating it between subscriptions in that region is self-service.

Two constraints worth knowing before designing around it: Quota Groups are available on Enterprise Agreement and Microsoft Customer Agreement subscriptions only. They cover IaaS compute resources only. PaaS services, managed databases, and analytics capacity are outside scope.

This is a platform engineering responsibility. Application teams should not be managing their own increase requests against individual subscriptions. The platform team manages group-level headroom per region and pre-allocates to subscriptions before those subscriptions need it.

---

## Quotas vs Capacity Reservations

Quota defines the ceiling: your subscription is allowed to provision up to N of this resource in this region. It does not guarantee that the physical capacity will be available when you request it.

Capacity reservations provide the physical guarantee. When you create a capacity reservation, Azure holds the underlying compute exclusively for your use. You pay for the reservation whether you deploy VMs into it or not.

For most workloads, quota is sufficient. The physical capacity exists in the region; the quota is simply the limit on how much of it your subscription can claim. For workloads with strict provisioning SLAs, predictable large-scale events, or DR plans that cannot tolerate delays, capacity reservations are the mechanism that actually backs the commitment.

The question is direct: if your business continuity plan depends on provisioning N VMs within X minutes of a regional event, does your DR region have a capacity reservation in place? If not, the plan has a dependency on an assumption rather than a guarantee.

```azurecli
az capacity reservation group create \
  --resource-group <rg> \
  --name <reservation-group-name> \
  --location <region> \
  --zones 1 2 3

az capacity reservation create \
  --resource-group <rg> \
  --reservation-group-name <reservation-group-name> \
  --name <reservation-name> \
  --location <region> \
  --sku Standard_D8s_v5 \
  --capacity 10
```

---

## Quota as a Standing Operational Concern

Quota is not a one-time configuration task. Default quotas erode as workloads grow. New regions start empty. GPU and AI-specific SKU families start at zero and require manual approval. DR regions carry defaults until someone explicitly requests otherwise.

The teams that avoid quota-related incidents are not doing anything complicated. They review quota headroom in every operating region during environment design. They request increases for production SKU families before the first deployment, not after the first failure. They set DR region quota to match primary region requirements. They use Quota Groups to centralise compute quota management across subscriptions, with the understanding that the group still requires per-region planning. They start GPU quota requests early because the lead time is long and there is no automated path.

Quota limits that surface during incidents are preventable. Every one of them represents a ceiling that was discoverable and addressable before the deployment that revealed it.
