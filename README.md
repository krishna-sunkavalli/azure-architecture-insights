# Azure Architecture Insights

Practitioner perspectives on designing, governing, and operating Azure at enterprise scale. The focus is on the architectural decisions that determine whether cloud adoption delivers — landing zone design, network topology, identity architecture, cost governance, and platform engineering. Not the documentation. The decisions behind the documentation.

Grounded in hands-on work across Azure cloud architecture and enterprise-scale implementations.

Krishna Sunkavalli — Sr. Solutions Engineer, Cloud & AI Platforms, Microsoft

---

## Articles

### Network

- [DNS Resolution in Azure: Getting It Right at Scale](articles/network/dns.md) — Enterprise DNS architecture: Private Resolver, Firewall DNS Proxy, Private Endpoints, and hybrid resolution at scale.
- [ExpressRoute in Production: Design Decisions, Common Failures, and Troubleshooting](articles/network/expressroute.md) — Why most ExpressRoute failures are routing decisions made at design time, and how to diagnose asymmetric routing before it becomes a two-day incident.

### Governance

- [Azure Quotas Are a Capacity Planning Problem, Not a Support Ticket](articles/governance/quotas.md) — Why quota limits surface at the worst moments and how to manage regional capacity headroom as a platform engineering discipline.
- [Azure Landing Zones Don't Fail at Build Time](articles/landing-zones/azure-landing-zones.md) — Most landing zone problems are operational gaps, not architecture problems. Subscription vending, policy exemption workflows, connectivity drift, and how to absorb quarterly ALZ releases.

### Reliability

- [How to Design for Regional Failures in Azure](articles/reliability/resiliency.md) — Resiliency strategy and common pitfalls for 25 Azure services, from Front Door and AKS to Fabric, Databricks, and Microsoft Foundry.

---

Licensed under [CC BY 4.0](LICENSE). Content may be shared or adapted with attribution.
Views expressed are my own and do not represent the views of Microsoft.
