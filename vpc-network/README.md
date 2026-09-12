# Feature 6: Google Cloud VPC Networks & Subnets

Welcome to the **Google Cloud Virtual Private Cloud (VPC) Networks & Subnets** module. This folder contains production-grade architecture, decision matrices, CLI manuals, and reference guides for managing global VPC networks, regional subnetworks, Auto vs Custom modes, CIDR range expansions, firewall rules, dual-stack IPv4/IPv6 addressing, internal DNS resolution, and Bring Your Own IP (BYOIP).

---

## Module Sitemap

| Document | Description |
| :--- | :--- |
| [**HLD & LLD Architecture Design**](file:///home/btpl-lap-22/live/gcd/vpc-network/hld-lld-design.md) | High-Level & Low-Level Design diagrams for Global VPC Topologies, Cross-Zone Regional Subnets, 4 Reserved Subnet IP Addresses, Internal DHCP/DNS Scoping, Ephemeral vs Static IP Billing, BYOIP, and Dual-Stack IPv4/IPv6. |
| [**Decision Tree Guide**](file:///home/btpl-lap-22/live/gcd/vpc-network/decision-tree.md) | Visual decision flowcharts for Auto Mode vs Custom Mode selection, Subnet Mask planning, Zero-Downtime CIDR Expansion rules, Ephemeral vs Static External IP selection, and BYOIP. |
| [**CLI Shell Commands & Operations Manual**](file:///home/btpl-lap-22/live/gcd/vpc-network/shell-commands.md) | Exhaustive command manual covering `gcloud compute networks`, `gcloud compute subnets`, `gcloud compute firewall-rules`, `gcloud compute addresses`, and BYOIP PAPs with **Expected Terminal Outputs** and **Verification Checks**. |
| [**Official References**](file:///home/btpl-lap-22/live/gcd/vpc-network/references.md) | Links to official GCP VPC documentation, RFC 1918 specifications, RFC 4291 IPv6, and BYOIP guides. |

---

## Key Operational & Technical Concepts Covered

1. **Global VPC Architecture**: Single global VPC network spanning all GCP regions worldwide.
2. **Network Modes**: Default Auto-Mode (`10.128.0.0/9` CIDR block) vs Custom Mode networks with explicit non-overlapping subnets.
3. **One-Way Mode Conversion**: Converting Auto Mode networks to Custom Mode networks (irreversible).
4. **Regional Subnets & Cross-Zone Span**: Subnets exist regionally and extend across all zones within that region.
5. **4 Reserved IP Addresses per Subnet**: `.0` (Network ID), `.1` (Default Gateway), `N-2` (Reserved by GCP), and `N-1` (Broadcast).
6. **Zero-Downtime Subnet Expansion**: Expanding subnet IP ranges without workload shutdown or VM reboot (smaller prefix mask; irreversible).
7. **Dual-Stack IPv4/IPv6**: Provisioning dual-stack VMs and subnets for IPv4 and IPv6 routing.
8. **Internal Global Communication vs Edge Routing**: Communication between VMs in the same VPC via Google's private global fiber network vs cross-VPC external IP communication through Edge Routers.
9. **Internal DHCP & Network-Scoped Internal DNS**: Automatic internal DHCP lease & `vm-name.zone.c.PROJECT.internal` DNS registration (scoped strictly to a single VPC network).
10. **Ephemeral vs Static External IPs & Billing Surcharge**: Dynamics of ephemeral vs static public IPs, including unassigned static IP penalty hourly fees.
11. **Bring Your Own IP (BYOIP)**: Importing customer-owned IPv4 blocks (minimum `/24` prefix) and advertising globally via BGP Anycast.
