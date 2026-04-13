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

Deploy a zone-redundant gateway with the correct SKU:
```azurecli
az network vnet-gateway create \
  --resource-group <rg> \
  --name <gateway-name> \
  --vnet <vnet-name> \
  --gateway-type ExpressRoute \
  --sku ErGw2AZ \
  --public-ip-address <pip-name> \
  --location <region>
```

Verify the current gateway SKU on an existing deployment:
```azurecli
az network vnet-gateway show \
  --resource-group <rg> \
  --name <gateway-name> \
  --query "{SKU:sku.name, Tier:sku.tier, ActiveActive:activeActive}" \
  --output table
```

### FastPath and the Gateway Throughput Ceiling

Gateway SKU selection determines more than zone redundancy — it determines whether the gateway sits in the data path at all.

By default, all inbound traffic from on-premises traverses the ExpressRoute gateway, making the gateway the throughput ceiling regardless of circuit bandwidth. A 10Gbps circuit with an ErGw2AZ gateway caps at the gateway's rated throughput, not the circuit's. FastPath changes this by bypassing the gateway data plane for traffic sourced from on-premises, forwarding it directly to the spoke VM. The gateway still handles control plane functions — route advertisement, BGP — but exits the data path for qualifying flows.

FastPath is supported on ErGw2AZ and higher. It is not a gateway-level setting; it is enabled per connection:

```azurecli
# Check whether FastPath is enabled on a connection
az network express-route port show \
  --resource-group <rg> \
  --name <connection-name> \
  --query "expressRouteGatewayBypass"
```

There are three separate throughput ceilings that are almost never discussed together during design:

- **Circuit bandwidth** — the contracted rate with the provider (1Gbps, 10Gbps, etc.)
- **Gateway throughput** — the rated aggregate throughput for the gateway SKU; ErGw2AZ supports up to ~10Gbps but this is aggregate across all connections
- **FastPath bypass** — removes the gateway from the return data path for on-premises-sourced traffic; the VM NIC becomes the ceiling

A workload hitting the gateway throughput ceiling looks like circuit saturation in monitoring because `BitsInPerSecond` climbs while the circuit still has headroom. Compare gateway connection throughput metrics against circuit utilization to distinguish these before assuming the circuit needs an upgrade.

### Connection Weights for Path Preference

When two circuits connect to one gateway — a primary plus a backup, or two diverse circuits — Azure uses connection weights to determine path preference. The default weight is 0 on all connections. Without explicit weights, Azure may not prefer the circuit you intend, and the preference may change after a routing event without any visible alert.

Set connection weights explicitly to match your intended primary/secondary designation:

```azurecli
# Set primary circuit connection as preferred
az network vpn-connection update \
  --resource-group <rg> \
  --name <primary-connection-name> \
  --routing-weight 10

# Secondary connection remains at default 0
```

Higher weight wins. Verify the weights match your documented routing intent during the initial deployment review and after any connection change.

### FastPath and the Gateway Throughput Ceiling

Gateway SKU selection determines more than zone redundancy — it determines whether the gateway sits in the data path at all.

By default, all inbound traffic from on-premises traverses the ExpressRoute gateway, making the gateway the throughput ceiling regardless of circuit bandwidth. A 10Gbps circuit with an ErGw2AZ gateway caps at the gateway's rated throughput, not the circuit's. FastPath changes this by bypassing the gateway data plane for traffic sourced from on-premises, forwarding it directly to the spoke VM. The gateway still handles control plane functions — route advertisement, BGP — but exits the data path for qualifying flows.

FastPath is supported on ErGw2AZ and higher. It is not a gateway-level setting; it is enabled per connection:

```azurecli
# Check whether FastPath is enabled on a connection
az network express-route port show \
  --resource-group <rg> \
  --name <connection-name> \
  --query "expressRouteGatewayBypass"
```

There are three separate throughput ceilings that are almost never discussed together during design:

- **Circuit bandwidth** — the contracted rate with the provider (1Gbps, 10Gbps, etc.)
- **Gateway throughput** — the rated aggregate throughput for the gateway SKU; ErGw2AZ supports up to ~10Gbps but this is aggregate across all connections
- **FastPath bypass** — removes the gateway from the return data path for on-premises-sourced traffic; the VM NIC becomes the ceiling

A workload hitting the gateway throughput ceiling looks like circuit saturation in monitoring because `BitsInPerSecond` climbs while the circuit still has headroom. Compare gateway connection throughput metrics against circuit utilization to distinguish these before assuming the circuit needs an upgrade.

### Connection Weights for Path Preference

When two circuits connect to one gateway — a primary plus a backup, or two diverse circuits — Azure uses connection weights to determine path preference. The default weight is 0 on all connections. Without explicit weights, Azure may not prefer the circuit you intend, and the preference may change after a routing event without any visible alert.

Set connection weights explicitly to match your intended primary/secondary designation:

```azurecli
# Set primary circuit connection as preferred
az network vpn-connection update \
  --resource-group <rg> \
  --name <primary-connection-name> \
  --routing-weight 10

# Secondary connection remains at default 0
```

Higher weight wins. Verify the weights match your documented routing intent during the initial deployment review and after any connection change.

### Active-Active Over Active-Passive

Every ExpressRoute circuit has two connections to two different MSEE devices. Active-passive wastes one of them. The passive connection adds to MTTR on failover — the standby instance must initialize and BGP must reconverge before traffic resumes. Active-active uses both connections simultaneously, increases aggregate throughput, and fails over in sub-second when BFD is enabled.

Default configuration is often active-passive. Verify this explicitly:
```azurecli
az network vnet-gateway show \
  --resource-group <rg> \
  --name <gateway-name> \
  --query "activeActive"
```

If this returns `false`, the gateway is active-passive. Update it:
```azurecli
az network vnet-gateway update \
  --resource-group <rg> \
  --name <gateway-name> \
  --set activeActive=true
```

### Two Circuits, Two Peering Locations

A single circuit with two connections protects against MSEE failure. It does not protect against a peering location failure, a provider edge failure, or a facility event. Maximum resiliency requires two circuits from two different peering locations — different physical paths, different provider edge devices, different facilities.

For most enterprise workloads, two circuits from the same provider across two peering locations is the right balance of cost and protection. Equinix and Telekom, for example, offer diverse entry points in the same metro.

List available peering locations to identify diverse options:
```azurecli
az network express-route list-service-providers \
  --query "[].{Provider:name, Locations:peeringLocations}" \
  --output table
```

### Never Terminate Both Connections on the Same CPE

Two connections from the circuit into one on-premises CPE is not HA. It is a single device failure away from losing both paths simultaneously. Each connection must terminate on a separate edge router.

This is the most common misconfiguration in ExpressRoute deployments and the one most likely to be discovered for the first time during an incident.

Verify the circuit connection state — both Primary and Secondary should show `Connected`:
```azurecli
az network express-route show \
  --resource-group <rg> \
  --name <circuit-name> \
  --query "{CircuitState:circuitProvisioningState, PrimaryPort:peerings[0].primaryAzurePort, SecondaryPort:peerings[0].secondaryAzurePort, ServiceKey:serviceKey}" \
  --output table
```

### Enable BFD

BFD (Bidirectional Forwarding Detection) reduces link failure detection from approximately three minutes under standard BGP timers to sub-second. Without BFD, a silent link failure leaves BGP up but traffic blackholing for the full keepalive interval before reconvergence.

BFD is supported on all current MSEE devices and most enterprise CPE. Enable it on the peering:
```azurecli
az network express-route peering update \
  --resource-group <rg> \
  --circuit-name <circuit-name> \
  --name AzurePrivatePeering \
  --set microsoftPeeringConfig.advertisedPublicPrefixes=[] \
  --peering-type AzurePrivatePeering
```

On the CPE side, the equivalent for a Cisco IOS-XE edge router:
```
router bgp 65001
 neighbor <MSEE-IP> fall-over bfd
!
bfd-template single-hop MSEE-BFD
 interval min-tx 300 min-rx 300 multiplier 3
```

Verify BFD sessions are up on Cisco:
```
show bfd neighbors
show bgp neighbors <MSEE-IP> | include BFD
```

### Route Discipline

Advertise only the prefixes you intend to advertise. Leaking a default route (0.0.0.0/0) into the Microsoft peering is a common misconfiguration that redirects internet-bound traffic from Azure through your on-premises network. Set explicit prefix filters on both sides.

Set a maximum prefix limit on the gateway to protect against a misconfigured on-premises router advertising a full internet routing table into your Azure environment.

On a Cisco CPE, restrict outbound BGP advertisements with a prefix list:
```
ip prefix-list AZURE-OUT seq 10 permit 10.1.0.0/16
ip prefix-list AZURE-OUT seq 20 permit 10.2.0.0/16
!
route-map TO-AZURE permit 10
 match ip address prefix-list AZURE-OUT
!
router bgp 65001
 neighbor <MSEE-IP> route-map TO-AZURE out
 neighbor <MSEE-IP> maximum-prefix 200 80
```

The `maximum-prefix 200 80` parameter alerts at 80% and tears down the BGP session at 200 prefixes — protecting Azure from a route leak from on-premises.

Check what prefixes are currently being received from Azure on the CPE:
```
show bgp neighbors <MSEE-IP> received-routes
show ip bgp summary
```

### MTU and TCP MSS

ExpressRoute operates at an MTU of 1500 on the Layer 2 path, but encapsulation overhead between segments — particularly when traffic traverses a firewall, NVA, or VPN fallback path — can cause silent fragmentation. The symptom is workloads that pass small packet tests (ping, basic TCP handshake) but fail under load or when transferring large payloads.

Validate MTU end-to-end during design. Document the expected MSS for each critical workload path. This is particularly relevant for iSCSI, SQL Server over named pipes, and any application sensitive to packet size or TCP segmentation behavior.

```bash
# Test path MTU from an Azure Linux VM — send a packet that must not be fragmented
ping -M do -s 1472 <on-premises-ip>
# 1472 bytes payload + 28 bytes ICMP/IP header = 1500 MTU
# If this fails but 1400 succeeds, there is an MTU mismatch in the path
```

---

## Common Pitfalls

**Failing to test the secondary path.** Most teams test the primary connection and assume the secondary works. The secondary path is untested until failover, which means its operational status is unknown. Simulate a connection failure in a maintenance window and verify traffic reconverges on the expected path.

To force traffic off the primary and onto the secondary, temporarily set a higher AS path prepend on the primary from the CPE (Cisco example):
```
route-map TO-AZURE-PRIMARY permit 10
 set as-path prepend 65001 65001 65001
!
router bgp 65001
 neighbor <MSEE-PRIMARY-IP> route-map TO-AZURE-PRIMARY out
```
Run traceroute from an Azure VM during this window. Confirm traffic reconverges on the secondary path within your expected BFD/BGP timer window. Remove the prepend after validation.

**vWAN Routing Intent is not optional.** When ExpressRoute connects to a vWAN hub rather than a standalone VNet gateway, the routing model changes significantly. Transit between ExpressRoute and VPN (branch-to-branch connectivity through the hub) requires Routing Intent to be explicitly enabled. Without it, spoke VNets can reach Azure resources, but on-premises-to-branch routing silently fails — there are no error messages, BGP stays up, and the connectivity gap only surfaces when a branch user tries to reach an on-premises resource through the hub.

Routing Intent also controls how security policies apply to inter-spoke and branch traffic in vWAN. If you are deploying Azure Firewall in the hub, Routing Intent must be configured to route spoke-to-spoke and branch-to-branch traffic through the firewall. Leaving it unconfigured means the firewall has no visibility into that traffic class.

Verify Routing Intent is configured on the vWAN hub if inter-branch or ExpressRoute-to-VPN transit is required:
```azurecli
az network vhub routing-intent show \
  --resource-group <rg> \
  --vhub-name <vhub-name> \
  --name <routing-intent-name>
```

**Route Server coexistence requires deliberate path tuning.** If you are injecting routes via Azure Route Server — typically for NVA integration in a hub-spoke design — ExpressRoute-learned routes and Route Server-injected routes can conflict. Route Server propagates NVA-originated routes to the ER gateway and ER-learned routes back to the NVA, which can cause on-premises to learn NVA-originated prefixes and prefer them over the direct ExpressRoute path. The result is an unintended traffic path that is invisible without explicit route table inspection.

When Route Server and ExpressRoute coexist, use AS path length and BGP weight to explicitly control which routes are preferred for each prefix class. Document the intended routing decision for every prefix family before deployment.

```azurecli
# Inspect what routes Route Server is propagating to the ER gateway
az network express-route list-route-tables \
  --resource-group <rg> \
  --name <circuit-name> \
  --peering-name AzurePrivatePeering \
  --path Primary \
  --query "value[].{Network:network, NextHop:nextHop, ASPath:asPath, Origin:origin}" \
  --output table
```

Look for NVA-originated prefixes appearing in the ER route table. If they are present and more specific than the on-premises prefix, they will be preferred — which may not be the intended behavior.

**Microsoft peering and Private peering are not interchangeable.** Private peering connects to your VNet address space. Microsoft peering connects to Microsoft public service endpoints — Office 365, Azure PaaS public IPs. The route advertisement requirements, prefix validation rules, and NAT requirements differ between the two. Teams sometimes configure Microsoft peering to reach Azure PaaS services from on-premises and then find those services unreachable from VNets. The correct pattern for VNet-resident workloads reaching PaaS over ExpressRoute is Private Endpoint combined with Private Peering. Microsoft peering is for on-premises clients reaching public Microsoft endpoints — not for traffic originating inside Azure.

Conflating the two leads to misconfigured route filters, incorrect NAT configuration on the CPE, and unexpected traffic paths that are difficult to diagnose because BGP stays healthy throughout.

List configured peerings and their state on a circuit:
```azurecli
az network express-route peering list \
  --resource-group <rg> \
  --circuit-name <circuit-name> \
  --query "[].{Type:peeringType, State:state, PrimarySubnet:primaryPeerAddressPrefix, SecondarySubnet:secondaryPeerAddressPrefix, VLAN:vlanId}" \
  --output table
```

**VPN as a DR path for latency-sensitive workloads.** VPN over internet provides different latency, different throughput, and different MTU characteristics than ExpressRoute. Applications tuned for ExpressRoute latency will behave differently over VPN. VPN is a last-resort backup for disaster scenarios, not a transparent failover path.

Measure baseline latency and throughput over ExpressRoute before designing a VPN fallback. The delta will tell you which workloads can tolerate the fallback and which cannot:
```bash
# Latency baseline from Azure VM to on-premises
ping -c 100 <on-premises-ip> | tail -2

# Throughput test with iperf3 (requires iperf3 server on-premises)
iperf3 -c <on-premises-ip> -t 30 -P 4
```

**Non-zone-redundant ExpressRoute gateway in an otherwise zone-redundant design.** A workload with zone-redundant VMs, zone-redundant databases, and a non-AZ ExpressRoute gateway has a clear single point of failure. The gateway lives in a single zone and a zone failure severs on-premises connectivity regardless of how redundant everything else is.

Check the current gateway SKU and whether it is zone-redundant:
```azurecli
az network vnet-gateway show \
  --resource-group <rg> \
  --name <gateway-name> \
  --query "{SKU:sku.name, Zones:zones}" \
  --output table
```
SKUs ending in `AZ` (ErGw1AZ, ErGw2AZ, ErGwScale) are zone-redundant. All others are not.

**No BGP monitoring.** BGP session flaps and route changes are invisible without explicit monitoring. Azure Network Watcher and Connection Monitor expose this, but require configuration. Treat BGP session health as a first-class alert, not an afterthought.

Create a Connection Monitor test to baseline latency and catch path changes:
```azurecli
az network watcher connection-monitor create \
  --name ER-OnPrem-Monitor \
  --resource-group <rg> \
  --location <region> \
  --source-resource-id <azure-vm-resource-id> \
  --dest-address <on-premises-ip> \
  --dest-port 443 \
  --monitoring-interval 30
```

Set a BGP availability alert via Azure Monitor — alert when `BgpAvailability` drops below 100% on either Primary or Secondary path:
```azurecli
az monitor metrics alert create \
  --name "ER-BGP-Availability" \
  --resource-group <rg> \
  --scopes <expressroute-circuit-resource-id> \
  --condition "avg BgpAvailability < 100" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 1 \
  --action <action-group-id>
```

---

## Troubleshooting: Asymmetric Routing

Asymmetric routing is the most consistently misdiagnosed issue in hybrid Azure environments. It is not a connectivity failure. It is a path mismatch — outbound traffic leaves through one interface, return traffic arrives on another. The application sees what looks like packet loss, intermittent failures, or a firewall problem. The network is doing exactly what the routes tell it to do.

**How it happens in practice:**

A workload in Azure sends a request to an on-premises endpoint. The return path selection on the on-premises router prefers the internet route over the ExpressRoute route, or vice versa. A stateful firewall on one side allows the initial packet, then drops the return because it never saw the session established. The application logs a timeout.

**The diagnostic sequence:**

**Step 0: Verify DNS resolution first.**

Asymmetric routing is sometimes misdiagnosed as a DNS issue, and DNS misconfiguration is sometimes misdiagnosed as asymmetric routing. Before tracing any path, confirm the application is resolving to the expected IP address.

If a Private DNS zone is missing or misconfigured, an Azure VM resolves a PaaS service FQDN to its public IP. The routing policy sends traffic over ExpressRoute for RFC1918 addresses, but the public IP takes the internet path. The traceroute looks fine. The application connects to the wrong endpoint. Confirm the resolution is correct before proceeding:

```bash
# From the Azure VM — verify the service resolves to a private IP, not a public endpoint
nslookup <service-fqdn>
# Expected: 10.x.x.x (private endpoint IP)
# If this returns a public IP, Private DNS zone is missing or the VM is not linked to it
```

**Step 1:** Pick one failing flow. Be specific: source IP, destination IP, port, protocol.

**Step 2:** Determine the expected path. Should this flow use ExpressRoute private peering or internet? If ExpressRoute, which connection (primary or secondary)?

**Step 3:** Run traceroute from both sides. The outbound and return paths should traverse the same network segments. If traceroute from Azure exits through ExpressRoute but the return path from on-premises exits through a different gateway or interface, the path is asymmetric.

From an Azure VM (Linux):
```bash
traceroute <on-premises-ip>

# If ICMP is blocked in the environment, use TCP traceroute to a specific port
traceroute -T -p 443 <on-premises-ip>
```

From an Azure VM (Windows):
```powershell
Test-NetConnection -ComputerName <on-premises-ip> -TraceRoute

# With TCP reachability check and detailed path information
Test-NetConnection -ComputerName <on-premises-ip> -Port 443 -InformationLevel Detailed
```

From on-premises toward Azure, run the equivalent and compare exit interfaces. The BGP next-hop should be consistent in both directions.

**Step 4:** Check advertised prefixes. On the on-premises side, verify what prefixes are being advertised into BGP toward Azure. On the Azure side, use the ExpressRoute gateway route table to verify what prefixes Azure has learned. A missing or more-specific route on either side explains most asymmetric routing scenarios.

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

**Step 5:** Check for UDRs overriding BGP. A User Defined Route in Azure can override a BGP-learned route and force traffic onto an unexpected path.

**Hub-spoke UDR asymmetry** is the most common pattern here and deserves explicit attention. In a hub-spoke topology, UDRs on spoke subnets route egress traffic to an NVA in the hub for inspection. The NVA sees the outbound flow and allows it. But on-premises routes directly to the spoke prefix via BGP — it does not route through the hub NVA. The return packet arrives at the spoke VM without passing through the NVA. The firewall's session state table has no entry for the return. The packet is dropped.

This is not a firewall misconfiguration. The firewall is behaving correctly. The routing design is asymmetric: outbound traffic passes through the NVA, return traffic does not.

Check all UDRs in subnets involved in the failing flow:
```azurecli
# Get effective routes on the VM NIC — shows BGP routes vs UDR overrides
az network nic show-effective-route-table \
  --resource-group <rg> \
  --name <vm-nic-name> \
  --output table
```

Look for routes where `nextHopType` is `VirtualAppliance` — a UDR pointing at a firewall private IP will override BGP-learned routes for that prefix. If the on-premises CPE does not also route return traffic through an equivalent path, the session is asymmetric.

The fix is to ensure the on-premises return path also traverses the NVA, either by advertising a more-specific route from the hub NVA back toward on-premises or by applying SNAT at the NVA to force return traffic through the same device.

**Step 6:** Test private peering connectivity directly. Use Network Watcher packet capture or PsPing to verify traffic is reaching the destination and a return path exists.

Start a packet capture on the VM NIC to see what is actually leaving:
```azurecli
az network watcher packet-capture create \
  --resource-group <rg> \
  --vm <vm-name> \
  --name capture01 \
  --storage-account <storage-account-id> \
  --filters '[{"protocol":"TCP","localPort":"0","remotePort":"<port>"}]'
```

Test TCP reachability from the VM:
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

**Fixes:**

- Correct the routing design so outbound and return use the same path. This is the right fix.
- Apply SNAT at the gateway or firewall to force return traffic back through the same device. This is a workaround that adds complexity and masks the underlying routing issue. Use it when the routing design cannot be changed in the short term.

If outbound and return traffic do not follow the same design, the issue is path symmetry — not connectivity, not the application, and not the firewall.

---

## Monitoring ExpressRoute in Production

Four monitoring surfaces that should be configured before the circuit goes live, not after the first incident:

**Azure Monitor ExpressRoute metrics — BgpAvailability and ArpAvailability together.** BGP being up does not mean Layer 2 is healthy. ARP availability is a separate metric that surfaces MSEE-to-CPE Layer 2 issues that BGP masks. A BGP session can remain established while ARP entries age out, producing intermittent packet loss that looks like application-layer flapping. Alert on both metrics. Neither alone is sufficient.

Pull current metrics to establish a baseline:
```azurecli
az monitor metrics list \
  --resource <expressroute-circuit-resource-id> \
  --metric "BgpAvailability" "ArpAvailability" "BitsInPerSecond" "BitsOutPerSecond" \
  --interval PT1M \
  --output table
```

Alert on ARP availability alongside BGP:
```azurecli
az monitor metrics alert create \
  --name "ER-ARP-Availability" \
  --resource-group <rg> \
  --scopes <expressroute-circuit-resource-id> \
  --condition "avg ArpAvailability < 100" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 1 \
  --action <action-group-id>
```

**Network Watcher Connection Monitor** — end-to-end latency and reachability tests between Azure and on-premises endpoints. Baseline during normal operation; alert on deviation. See the pitfalls section for the creation command.

**Traffic Engineering baseline.** Connection Monitor gives you reachability and latency. It does not tell you which physical path traffic is taking. A BGP route change that does not break connectivity — reconvergence to a longer AS path after a flap, for example — is invisible to Connection Monitor but degrades latency for every flow using that path. Without a documented baseline, there is no way to detect it.

At deployment time, record the expected BGP next-hop and full AS path for each critical flow. Store it alongside your architecture documentation. After any maintenance window or BGP event, compare the current state against that baseline:

```
# On-premises CPE (Cisco) — capture the AS path for Azure prefixes before and after any change
show bgp neighbors <MSEE-IP> received-routes | include <azure-prefix>
show ip bgp <azure-prefix>
```

```azurecli
# Azure side — capture what the gateway is learning from on-premises
az network express-route list-route-tables \
  --resource-group <rg> \
  --name <circuit-name> \
  --peering-name AzurePrivatePeering \
  --path Primary \
  --query "value[].{Network:network, NextHop:nextHop, ASPath:asPath}" \
  --output table
```

A route change that increases AS path length by two hops may add 5-15ms of latency on a path that was tuned for a latency-sensitive workload. Connection Monitor will not catch it. The baseline will.

**BGP session monitoring on CPE.** Azure metrics show the Azure side of the BGP session. CPE-side monitoring shows the on-premises perspective. Both are needed. A session that appears up in Azure and down on the CPE is a split view that diagnostic tools miss — and it produces exactly the intermittent, hard-to-reproduce symptoms that waste the most incident time.

On Cisco IOS-XE, get a full BGP session view including uptime, prefixes received, and message counters:
```
show bgp neighbors <MSEE-IP>
show bgp neighbors <MSEE-IP> | include BGP state|prefixes|MsgRcvd|MsgSent|Uptime
show ip route bgp
```

On Juniper:
```
show bgp neighbor <MSEE-IP>
show route protocol bgp
```

**Route change alerting and quarterly failover drills.** A sudden increase or decrease in advertised prefix count is a leading indicator of misconfiguration. Configure alerts on prefix count changes in both directions.

```azurecli
# Primary path — stateOrPfxRcd column shows prefix count
az network express-route list-route-tables-summary \
  --resource-group <rg> \
  --name <circuit-name> \
  --peering-name AzurePrivatePeering \
  --path Primary \
  --query "value[].{Neighbor:neighbor, V:v, MsgRcvd:msgRcvd, MsgSent:msgSent, Up:upDown, State:stateOrPfxRcd}" \
  --output table

# Secondary path
az network express-route list-route-tables-summary \
  --resource-group <rg> \
  --name <circuit-name> \
  --peering-name AzurePrivatePeering \
  --path Secondary \
  --output table
```

Monitoring tells you the circuit is healthy. It does not tell you failover works. Treat the secondary path the same way you treat a DR site: it must be exercised on a schedule, not assumed. Quarterly is the minimum cadence for production workloads. The test is straightforward — AS path prepend on the primary to shift traffic to the secondary, measure reconvergence time against your RTO, verify the expected path in traceroute, restore. Document the result. The test also validates that the secondary path's latency and throughput are within acceptable bounds for your workloads — something that degrades silently between tests if the underlying path changes.

---

## Closing

ExpressRoute is a routing system that spans your on-premises network, a provider edge, the Microsoft Global Network, and your Azure environment. Every segment is managed by a different team under different operational models. The failures that matter are not the ones Azure controls — those are rare. The failures that matter are the ones that live in the routing decisions between those segments: which path traffic takes, which path it returns on, and what happens when the primary path is unavailable.

Design that explicitly. Monitor it explicitly. Drill the failover before the incident makes it mandatory.
