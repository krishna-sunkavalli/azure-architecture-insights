---
title: "The /32 You Didn't Put There: How Private Endpoints Really Route Traffic"
date: 2026-04-21
author: "Krishna Sunkavalli"
tags:
  - private-endpoints
  - private-link
  - network
  - azure-firewall
  - hub-spoke
  - routing
  - zero-trust
description: "Private endpoints do not route traffic. They program routes. That distinction changes how you design for inspection, segmentation, and firewall enforcement in every topology that uses them."
categories:
  - Network
---

# The /32 You Didn't Put There: How Private Endpoints Really Route Traffic

---

Most architects treat private endpoints as network objects. They appear in the portal as NICs attached to subnets, they have IP addresses, they have DNS records pointing to them. The mental model is: traffic flows from the client to the endpoint, which then proxies it to the backing PaaS service. That mental model is wrong, and building a security architecture on top of it produces surprises at exactly the wrong moment.

A private endpoint is a control plane construct. Its job is to program /32 host routes into the NICs of virtual machines in the local VNet and every directly peered VNet. The data plane tunnel that carries actual traffic is established directly between the connecting client and the PaaS service. The endpoint's NIC does not forward packets. Traffic does not transit the subnet it is attached to. The peering between a spoke hosting the endpoint and any hub does not need to be traversed for traffic to reach the service.

The practical implication: placing a private endpoint in a specific VNet or subnet does not confine traffic to that VNet or subnet. It confines the /32 route advertisement to that topology. Those are not the same thing.

---

## The Route That Lies

Every private endpoint injects a /32 host route into the effective route table of each NIC in its VNet and in peered VNets. The next hop type is `InterfaceEndpoint`. In a hub-and-spoke topology, this means the /32 appears in the hub, including on the Virtual Network Gateway subnet, and on any Azure Firewall instance located there.

This causes a problem that is easy to miss. You configure a UDR on the GatewaySubnet to redirect on-premises traffic toward the firewall. The UDR points traffic addressed to the endpoint's /32 to the firewall's private IP. You verify the route table. The routes look correct. Traffic still bypasses the firewall.

What happened: the system /32 routes injected by private endpoints in certain gateway configurations, including active/active VPN gateways, override the UDR. The UDR specifying the exact /32 to the firewall does not win. This was not always the behavior. It changed quietly, and it is not prominently documented for standalone virtual network gateways outside of the Virtual WAN context.

The fix is not to add more specific UDRs. The fix is to enable Private Endpoint Network Policy for routing on the subnet hosting the endpoint.

---

## Network Policies: The Setting That Actually Controls Routing

By default, subnets hosting private endpoints have network policies disabled. This means NSG rules do not apply to private endpoint NICs, and UDRs do not override the /32 host routes those endpoints inject. Both of those defaults work against the architecture most enterprises need.

Enabling `RouteTableEnabled` network policy on a private endpoint subnet changes the behavior: if a UDR exists for the endpoint's address space, the /32 system routes injected by the endpoint are suppressed. The UDR wins. Traffic goes where you direct it.

The operational consequence of this is significant. Once network policies are enabled, a summary UDR pointing the endpoint's /24 subnet block to the firewall is sufficient. You do not need to track and maintain per-endpoint /32 UDRs as the number of endpoints grows. The /32 system routes stop competing.

One precision matters here: the UDR prefix must be at least as specific as the endpoint's containing subnet. A broad summary like `10.0.0.0/8` does not suppress the /32 routes because the peering system route for the specific subnet prefix is more specific and wins. The policy suppression logic requires that the effective route covering the endpoint's address is a UDR, not a system route. If the most specific match for the endpoint's IP is a system route rather than a UDR, the /32 routes will be programmed regardless of whether policies are enabled.

---

## The Hub-and-Spoke Anomaly

In a hub-and-spoke topology with Azure Firewall in the hub, the /32 route injection behavior creates an access pattern that violates intuitive assumptions about spoke isolation.

Consider: a private endpoint is deployed in spoke-2. No route table is attached to the endpoint's subnet. A VM in spoke-1 has a UDR sending traffic toward the firewall in the hub for eastbound traffic. The expectation is that traffic from spoke-1 to the endpoint would fail or would require routing through the hub and then through spoke-2's peering.

What actually happens: the /32 route for the endpoint is programmed into all NICs in spoke-1 via the peering. The outbound packet from spoke-1's VM hits the firewall UDR as expected and reaches the firewall. The firewall's NIC also has the /32 system route. The firewall terminates the connection and establishes a Private Link tunnel directly to the PaaS service. The spoke-2 peering is never traversed. Return traffic comes back directly. Access succeeds, firewall logs show the connection, but the endpoint subnet in spoke-2 never saw the traffic.

Two things to note here. First, if the intent was to restrict access to spoke-2 peers only, attaching no route table to the endpoint subnet is insufficient. The /32 routes propagate across peerings regardless. Second, when access does go through the firewall, the cost model is also affected: you pay for the spoke-1 peering egress, but not the spoke-2 peering, because the data plane path never crosses it.

To enforce that all traffic destined for the endpoint passes through the firewall and is inspectable, the correct architecture is: route table on the endpoint subnet with `RouteTableEnabled` policy enabled, plus a UDR matching the endpoint subnet prefix directing traffic to the firewall. This closes both the routing anomaly and the inspection gap.

---

## Firewall Inspection: The SNAT Question

When Azure Firewall handles traffic addressed to a private endpoint, the /32 host route on the firewall NIC normally means the firewall sends the packet directly to the PaaS service via the Private Link data plane. NSG and UDR network policies at the endpoint subnet do not apply in the same way they apply to VMs.

For Azure Firewall network rules, this works without SNAT. The connection is logged, policy is applied, traffic is allowed or denied. For some PaaS services, particularly Azure SQL, the Private Link data plane implementation requires that the inner and outer source address match. If traffic arrives at the PaaS service with the firewall's IP as source but the original client IP as inner tunnel source, the return path breaks and connections fail silently.

The historical mitigation was to enable SNAT at the firewall so all traffic to private endpoints carries the firewall's IP as source end-to-end. As of 2024 this requirement has been relaxed for several services, but the safest posture for production designs is still to verify per-service behavior and enable SNAT where asymmetric routing symptoms appear.

Application rules in Azure Firewall do not work with private endpoints via FQDN matching in the same way they work with public service endpoints. Because the traffic resolves to a private IP rather than traversing the public DNS path, application rule FQDN matching requires additional configuration to function correctly.

---

## What This Changes About Your Design

The architecture implications group into three decisions:

**Route control:** Enable `RouteTableEnabled` network policy on every subnet hosting private endpoints in any topology that includes a firewall or gateway. Without it, /32 system routes will compete with or override your UDRs in ways that depend on gateway type and configuration, and that behavior is not guaranteed to stay consistent across platform updates.

**Endpoint placement vs. access scope:** Deploying a private endpoint in a specific spoke does not restrict which VNets can reach it. Any VNet with peering path to the endpoint's VNet receives the /32 routes. Access scope is controlled through the firewall and NSG policies, not through endpoint placement. Architects who assume endpoint placement implies access restriction will find that assumption fails silently.

**Inspection coverage:** Private endpoint traffic is inspectable at the firewall when routing is correctly configured. NSGs on the endpoint subnet do function when `NetworkSecurityGroupEnabled` policy is enabled, but they apply at the NIC level with the same asymmetric return path caveats. The firewall is the right enforcement point for inspection of on-premises and inter-spoke traffic to private endpoints.

---

The portal's representation of a private endpoint as a NIC attached to a subnet is a useful abstraction for resource management. It is a misleading abstraction for network design. The endpoint is not in the data path. It is a route programming mechanism with a subnet address, and the routes it programs ignore peering topology in ways that conflict with standard hub-and-spoke assumptions. Design your firewall policy, your route tables, and your network policies as if the endpoint is invisible to traffic, because for the data plane, it essentially is.
