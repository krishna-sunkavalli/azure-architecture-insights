---
title: "ExpressRoute in Production: Design Decisions, Common Failures, and Troubleshooting"
date: 2026-04-13
author: "Krishna Sunkavalli"
tags:
  - azure
  - network
  - production
  - governance
description: "Most ExpressRoute failures are not circuit failures. They are routing decisions made at design time that surface as connectivity symptoms months later."
categories:
  - Network
image: assets/images/expressroute-social.png
---

# ExpressRoute in Production
## Design Decisions, Common Failures, and Troubleshooting

---

Most ExpressRoute failures are not circuit failures. The circuit is usually fine. The BGP session is up. The MSEE is reachable. Traffic is flowing. And the application is still broken.

The failure is almost always in the routing decisions made at design time — which path traffic takes out, which path traffic takes back, and whether anyone verified those are the same path. Most teams treat ExpressRoute as a connectivity checkbox. Provision the circuit, create the gateway, peer it, done. What they skip is the operational model: what happens when one connection goes down, who owns the BGP advertisements, and how asymmetric routing gets detected before it becomes a two-day incident.

---

## The Design Decisions That Determine Whether This Works

### Gateway SKU and Subnet

Use ErGw2AZ or higher. ErGw1AZ supports zone redundancy but capacity is limited — under heavy traffic it becomes the bottleneck before the circuit does. Basic and Standard SKUs have no zone support and no SLA suitable for production hybrid workloads.

The GatewaySubnet must be /27 or larger. A /28 works for non-AZ gateways but fails silently for zone-redundant deployments. Resize the subnet before deploying an AZ gateway, not during.

### Active-Active Over Active-Passive

Every ExpressRoute circuit has two connections to two different MSEE devices. Active-passive wastes one of them. The passive connection adds to MTTR on failover — the standby instance must initialize and BGP must reconverge before traffic resumes. Active-active uses both connections simultaneously, increases aggregate throughput, and fails over in sub-second when BFD is enabled.

Default configuration is often active-passive. Verify this explicitly.

### Two Circuits, Two Peering Locations

A single circuit with two connections protects against MSEE failure. It does not protect against a peering location failure, a provider edge failure, or a facility event. Maximum resiliency requires two circuits from two different peering locations — different physical paths, different provider edge devices, different facilities.

For most enterprise workloads, two circuits from the same provider across two peering locations is the right balance of cost and protection. Equinix and Telekom, for example, offer diverse entry points in the same metro.

### Never Terminate Both Connections on the Same CPE

Two connections from the circuit into one on-premises CPE is not HA. It is a single device failure away from losing both paths simultaneously. Each connection must terminate on a separate edge router.

This is the most common misconfiguration in ExpressRoute deployments and the one most likely to be discovered for the first time during an incident.

### Enable BFD

BFD (Bidirectional Forwarding Detection) reduces link failure detection from approximately three minutes under standard BGP timers to sub-second. Without BFD, a silent link failure leaves BGP up but traffic blackholing for the full keepalive interval before reconvergence.

BFD is supported on all current MSEE devices and most enterprise CPE. Enable it. There is no meaningful downside.

### Route Discipline

Advertise only the prefixes you intend to advertise. Leaking a default route (0.0.0.0/0) into the Microsoft peering is a common misconfiguration that redirects internet-bound traffic from Azure through your on-premises network. Set explicit prefix filters on both sides.

Set a maximum prefix limit on the gateway to protect against a misconfigured on-premises router advertising a full internet routing table into your Azure environment.

---

## Common Pitfalls

**Failing to test the secondary path.** Most teams test the primary connection and assume the secondary works. The secondary path is untested until failover, which means its operational status is unknown. Simulate a connection failure in a maintenance window and verify traffic reconverges on the expected path.

**VPN as a DR path for latency-sensitive workloads.** VPN over internet provides different latency, different throughput, and different MTU characteristics than ExpressRoute. Applications tuned for ExpressRoute latency will behave differently over VPN. VPN is a last-resort backup for disaster scenarios, not a transparent failover path.

**Non-zone-redundant ExpressRoute gateway in an otherwise zone-redundant design.** A workload with zone-redundant VMs, zone-redundant databases, and a non-AZ ExpressRoute gateway has a clear single point of failure. The gateway lives in a single zone and a zone failure severs on-premises connectivity regardless of how redundant everything else is.

**No BGP monitoring.** BGP session flaps and route changes are invisible without explicit monitoring. Azure Network Watcher and Connection Monitor expose this, but require configuration. Treat BGP session health as a first-class alert, not an afterthought.

**Assuming the Microsoft peering and private peering operate the same way.** Private peering connects to your VNet address space. Microsoft peering connects to Microsoft public service endpoints (Office 365, Azure PaaS public IPs). Route advertisements, prefix requirements, and NAT requirements differ between the two. Conflating them leads to misconfigured filters and unexpected traffic paths.

---

## Troubleshooting: Asymmetric Routing

Asymmetric routing is the most consistently misdiagnosed issue in hybrid Azure environments. It is not a connectivity failure. It is a path mismatch — outbound traffic leaves through one interface, return traffic arrives on another. The application sees what looks like packet loss, intermittent failures, or a firewall problem. The network is doing exactly what the routes tell it to do.

**How it happens in practice:**

A workload in Azure sends a request to an on-premises endpoint. The return path selection on the on-premises router prefers the internet route over the ExpressRoute route, or vice versa. A stateful firewall on one side allows the initial packet, then drops the return because it never saw the session established. The application logs a timeout.

**The diagnostic sequence:**

1. Pick one failing flow. Be specific: source IP, destination IP, port, protocol.

2. Determine the expected path. Should this flow use ExpressRoute private peering or internet? If ExpressRoute, which connection (primary or secondary)?

3. Run traceroute from both sides. The outbound and return paths should traverse the same network segments. If traceroute from Azure exits through ExpressRoute but the return path from on-premises exits through a different gateway or interface, the path is asymmetric.

   From an Azure VM (Linux):
   ```bash
   traceroute <on-premises-ip>
   ```

   From an Azure VM (Windows):
   ```powershell
   Test-NetConnection -ComputerName <on-premises-ip> -TraceRoute
   ```

   From on-premises toward Azure, run the equivalent and compare exit interfaces. The BGP next-hop should be consistent in both directions.

4. Check advertised prefixes. On the on-premises side, verify what prefixes are being advertised into BGP toward Azure. On the Azure side, use the ExpressRoute gateway route table to verify what prefixes Azure has learned. A missing or more-specific route on either side explains most asymmetric routing scenarios.

   Get the routes advertised from Azure to on-premises (what Azure is telling your CPE):
   ```azurecli
   az network express-route list-route-tables \
     --resource-group <rg> \
     --name <circuit-name> \
     --peering-name AzurePrivatePeering \
     --path Primary
   ```

   Get the routes Azure has learned from on-premises (what your CPE is telling Azure):
   ```azurecli
   az network express-route list-route-tables-summary \
     --resource-group <rg> \
     --name <circuit-name> \
     --peering-name AzurePrivatePeering \
     --path Primary
   ```

   Get the effective routes on the ExpressRoute gateway NIC to see the full routing table Azure is using:
   ```azurecli
   az network nic show-effective-route-table \
     --resource-group <rg> \
     --name <gateway-nic-name> \
     --output table
   ```

5. Check for UDRs overriding BGP. A User Defined Route in Azure can override a BGP-learned route and force traffic onto an unexpected path. Check all UDRs in subnets involved in the failing flow.

   Get effective routes on the VM NIC in question (shows BGP routes vs UDR overrides):
   ```azurecli
   az network nic show-effective-route-table \
     --resource-group <rg> \
     --name <vm-nic-name> \
     --output table
   ```

   Look for routes where `nextHopType` is `VirtualAppliance` or `VirtualNetworkGateway` — a UDR with `VirtualAppliance` pointing at a firewall private IP will override BGP-learned routes for that prefix.

6. Test private peering connectivity directly. Use Network Watcher packet capture or PsPing to verify traffic is reaching the destination and a return path exists.

   Start a packet capture on the VM NIC to see what is actually leaving:
   ```azurecli
   az network watcher packet-capture create \
     --resource-group <rg> \
     --vm <vm-name> \
     --name capture01 \
     --storage-account <storage-account-id> \
     --filters '[{"protocol":"TCP","localPort":"0","remotePort":"<port>"}]'
   ```

   Test TCP reachability with PsPing from the VM (download Sysinternals or use `Test-NetConnection`):
   ```powershell
   # Windows — test TCP reachability and round-trip time
   Test-NetConnection -ComputerName <on-premises-ip> -Port <port>
   ```

   ```bash
   # Linux — equivalent with netcat
   nc -zv <on-premises-ip> <port>
   ```

   Verify BGP peering state on the circuit from the Azure side:
   ```azurecli
   az network express-route show \
     --resource-group <rg> \
     --name <circuit-name> \
     --query "peerings[?peeringType=='AzurePrivatePeering'].{State:state, PrimaryPeer:primaryAzurePort, SecondaryPeer:secondaryAzurePort}" \
     --output table
   ```

**Stateful device behavior:**

A firewall that sees outbound traffic on one interface and return traffic arriving on another interface will drop the return packets. The session state table has no entry for traffic arriving on the unexpected interface. This is not a firewall misconfiguration. It is the firewall behaving correctly given an asymmetric routing design.

**Fixes:**

- Correct the routing design so outbound and return use the same path. This is the right fix.
- Apply SNAT at the gateway or firewall to force return traffic back through the same device. This is a workaround that adds complexity and masks the underlying routing issue. Use it when the routing design cannot be changed in the short term.

If outbound and return traffic do not follow the same design, the issue is path symmetry — not connectivity, not the application, and not the firewall.

---

## Monitoring ExpressRoute in Production

Four monitoring surfaces that should be configured before the circuit goes live, not after the first incident:

- **Azure Monitor ExpressRoute metrics** — bits in/out, BGP availability, ARP availability per peering. Alert on BGP availability dropping below 100%.
- **Network Watcher Connection Monitor** — end-to-end latency and reachability tests between Azure and on-premises endpoints. Baseline during normal operation; alert on deviation.
- **BGP session monitoring on CPE** — Azure metrics show the Azure side of the BGP session. CPE-side monitoring shows the on-premises perspective. Both are needed. A session that appears up in Azure and down on the CPE is a split view that diagnostic tools miss.
- **Route change alerting** — a sudden increase or decrease in advertised prefix count is a leading indicator of misconfiguration. Configure alerts on prefix count changes in both directions.

---

## Closing

ExpressRoute is a routing system that spans your on-premises network, a provider edge, the Microsoft Global Network, and your Azure environment. Every segment is managed by a different team under different operational models. The failures that matter are not the ones Azure controls — those are rare. The failures that matter are the ones that live in the routing decisions between those segments: which path traffic takes, which path it returns on, and what happens when the primary path is unavailable.

Design that explicitly. Monitor it explicitly. Drill the failover before the incident makes it mandatory.
