---
title: "Azure Landing Zones Don't Fail at Build Time"
date: 2026-04-14
author: "Krishna Sunkavalli"
tags:
  - azure
  - governance
  - landing-zones
  - production
description: "Most landing zone problems are not architecture problems. They are operational gaps that open the moment the deployment pipeline stops running."
categories:
  - Governance
image: assets/images/azure-landing-zones-social.png
---

# Azure Landing Zones Don't Fail at Build Time

Most teams that have a landing zone problem do not have a design problem. The management group hierarchy is correct. The hub-and-spoke topology is there. Policy is deployed. The architecture diagram would pass any CAF review. And the environment is still drifted, inconsistent, and increasingly difficult to govern.

The failure is almost never in what gets built. It is in what does not happen after the build: no subscription vending, no exemption process, no connectivity versioning, and no mechanism to absorb the quarterly updates that the ALZ team ships. The project closes, the pipeline stops running, and the environment starts diverging from its own baseline on day one.

---

## The Policy Set Is the Actual Deliverable

The Azure Landing Zone reference architecture is a management group hierarchy, a hub network topology, and a set of Azure Policy assignments. Most teams focus on the first two. The third is the value.

The ALZ policy set, published and maintained by Microsoft on GitHub across both the [ALZ-Bicep](https://github.com/Azure/ALZ-Bicep) and [Terraform](https://github.com/Azure/terraform-azurerm-caf-enterprise-scale) implementations, is a curated governance baseline covering logging requirements, encryption standards, network security controls, identity configuration, and regulatory alignment across frameworks including CIS, NIST 800-53, and PCI DSS. This policy set took years to build, maps to real audit frameworks, and is updated on a published release cadence. An organisation that builds the management group hierarchy but writes their own policies from scratch is discarding most of the value and taking on ongoing maintenance they are not equipped to sustain.

Policy assignments in ALZ operate at the management group level, which means they inherit down to every subscription and resource group within that scope without per-subscription configuration. This is why the management group design matters: the structure determines which workloads fall under which policy scope, and moving subscriptions between management groups later is a governance event with audit implications, not a trivial reorganisation.

The audit and deny effects are intentional and not optional. Audit tells you what is out of compliance. Deny prevents non-compliant resources from being created at all. The correct default for a production landing zone is deny, applied at the Landing Zones management group scope, with a structured exemption process for the cases where deny is inappropriate. Teams that switch deny effects to audit because deny is causing friction are removing the security control and replacing it with a report that nobody reads.

---

## What "Refresh the Latest Release" Actually Means

The ALZ team operates on a roughly quarterly release cadence. Each release on GitHub includes new policy definitions, updates to existing initiative assignments, deprecations, and changes to the management group or network module defaults. The CHANGELOG for each release documents exactly what changed and what action is required.

"Refreshing" the landing zone means pulling the new release tag, reviewing the CHANGELOG for breaking changes or remediation steps, and rerunning the deployment pipeline against your environment. Because ALZ deployments are idempotent by design, rerunning the pipeline against an already-deployed environment applies only the delta. New policies get created. Deprecated ones get cleaned up. Updated parameter defaults take effect where you have not overridden them.

This only works if the landing zone was deployed from a pipeline in the first place.

Teams that provisioned their landing zone through the portal, or from a pipeline that was run once and then abandoned, have no practical path to consuming quarterly updates. Every ALZ release that ships without being applied is governance debt. After twelve months of missed updates, the policy set your environment is running is materially behind the published baseline, and the gap between what your landing zone enforces and what current ALZ guidance recommends is no longer a version difference. It is a posture difference.

The operational requirement is: the ALZ deployment pipeline runs on a schedule or on every release tag, not on initial deployment only. AzOps ([github.com/Azure/AzOps](https://github.com/Azure/AzOps)) provides the CI/CD framework to manage this for management group, policy, and role assignment state as code. The pipeline is the governance mechanism, not the initial deployment.

---

## Three Places Drift Gets Encoded

### Subscription lifecycle without vending automation

Every subscription that gets created manually creates the conditions for drift. Manual creation means inconsistent naming, missed VNet peering to the hub, skipped budget alert configuration, and management group placement that someone guessed at. The first manual subscription is a one-off. The fortieth is a structural problem.

The ALZ Bicep [subscription vending module](https://github.com/Azure/bicep-lz-vending) and the Terraform equivalent are the intended solution. Subscription vending means a workload team submits a request through a defined intake process, the pipeline creates the subscription, places it in the correct management group, peers the spoke VNet, assigns required tags, and deploys baseline monitoring. The subscription arrives policy-compliant by construction.

Without vending, every team that needs a subscription is a custom engagement with your platform team. The platform team becomes a bottleneck, manual exceptions accumulate, and the management group hierarchy that was designed to enforce governance by structure stops reflecting reality.

### Deny policy with no exemption workflow

Restrictive policy is correct. The problem is not the policy; it is the absence of a process for handling legitimate exceptions.

When developers encounter a deny effect that blocks a deployment and there is no defined path to request an exception, they route around governance instead of through it. They deploy to a subscription that is outside the policy scope. They ask someone to switch the effect to audit. They find a resource configuration that technically satisfies the policy while defeating its intent. Every one of those outcomes is worse than a properly managed exemption.

Azure Policy supports time-bound, scoped exemptions natively. An exemption has a defined expiry date, a category (Waiver or Mitigated), and a description field for documenting the justification. This is the correct mechanism for exceptions. The exemption record is auditable, the expiry triggers a review, and the scope can be narrowed to the specific resource or resource group that needs it rather than modifying the policy assignment that covers the entire management group.

The platform team owns the exemption process. The process should have an intake path, an approver, and a maximum duration. Exemptions that need to survive longer than 90 days are a signal that the policy itself needs review, not that the exemption should be extended indefinitely.

### Connectivity treated as a one-time deployment

The connectivity subscription contains the hub VNet, the Azure Firewall, DNS forwarding configuration, ExpressRoute or VPN gateways, and Private Link DNS zones. It also changes every time a new workload team onboards, every time a new private endpoint DNS zone is required, and every time a firewall rule collection needs updating.

If the connectivity subscription is not version-controlled and deployed through a pipeline with the same rigour as application code, it will drift. Firewall rules get added through the portal. DNS zones get created outside the standard pattern. Private endpoint DNS zones get linked to the wrong VNet scope. None of these changes appear in any deployment record. None of them will get rolled back if they break something.

When connectivity drifts, the security boundary drifts with it. A hub firewall that does not reflect its own Terraform state is not a firewall you can trust to enforce the network policy the landing zone was designed around.

---

## Starting from a Drifted State

If the landing zone already exists and is already drifted, the practical path is not to rebuild from scratch. It is to bring the existing state under management and then begin aligning it to a supported baseline.

AzOps provides a pull-based pipeline that reads the current state of Azure Policy assignments, management group hierarchy, and role assignments and generates the corresponding IaC in your repository. This gives you a working baseline from the current real state rather than from a design document that no longer reflects reality. Once the current state is in code, you can begin comparing it against the ALZ reference and applying corrections incrementally.

The corrections that matter most are:

1. Policy assignments at the Landing Zones management group scope that are missing from the current deployment
2. Subscriptions that are in the wrong management group scope and therefore under the wrong policy set
3. Connectivity subscription resources that have no IaC representation

Drift is not corrected in a single remediation. It is corrected by establishing the pipeline, pulling current state, and running quarterly updates until the gap closes. The goal is a landing zone that has a deployment pipeline, absorbs ALZ releases as they ship, and has a subscription vending process that prevents new subscriptions from arriving outside the baseline.

---

## The Landing Zone as a Platform Product

The pattern across every engagement that works: the landing zone has a named owner, a backlog, a deployment pipeline that runs on a schedule, and a process for subscription requests and policy exemptions. It is operated as a platform product with a roadmap, not as a project with a delivery date.

The pattern across every engagement that drifts: the landing zone was delivered by a consulting team or a central project, handed over with documentation, and left to the operations team to maintain without any of the tooling, processes, or ownership that maintenance requires.

Azure Landing Zones are a starting point, not a destination. The ALZ team ships updates. Policies get refined. New services require new governance controls. Workload teams have requirements that the initial design did not anticipate. The environments that stay aligned to the baseline are the ones with a platform engineering discipline that treats governance as an ongoing operational concern and a pipeline that makes applying updates cheaper than skipping them.
