# Feature 6: Google Cloud VPC Networks & Subnets

Welcome to the **Google Cloud Virtual Private Cloud (VPC) Networks & Subnets** module. This folder contains production-grade architecture, decision matrices, CLI manuals, and reference guides for managing global VPC networks, regional subnetworks, Auto vs Custom modes, CIDR range expansions, firewall rules, dual-stack IPv4/IPv6 addressing, internal DNS resolution, Bring Your Own IP (BYOIP), Virtual Routers, and Cloud DNS.

---

## Module Sitemap

| Document | Description |
| :--- | :--- |
| [**Case Study & Deep-Dive Module Index**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/README.md) | Deep-dive case study covering Andromeda SDN, packet traversal lifecycle, stateful firewall conntrack engine, and Compute Engine integration. |
| [**1. Case Study: High-Level Architecture (HLD)**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/hld-architecture.md) | High-level topology, Andromeda SDN control/data plane, and network component relationships. |
| [**2. Case Study: Low-Level Packet Lifecycle (LLD)**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/lld-packet-lifecycle.md) | 12-stage packet lifecycle from Guest OS virtio-net through tap interface, Andromeda HPP, Geneve header framing, to wire. |
| [**3. Case Study: Stateful Firewall Engine Deep Dive**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/firewall-deep-dive.md) | Stateful conntrack table mechanics, 5-tuple lookup, rule priority evaluation (0-65535), and implicit rules. |
| [**4. Case Study: Compute & Network Integration**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/compute-network-integration.md) | VM vNIC tap interface binding, 4 reserved subnet IPs, 1:1 NAT transparency, GCP Metadata Server (169.254.169.254), DHCP, and Multi-NIC. |
| [**5. Case Study: Commands & Diagnostics Manual**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/commands-and-troubleshooting.md) | CLI command manual (`gcloud compute`), Network Intelligence Center Connectivity Tests, VPC Flow Logs, Packet Mirroring, and Error Matrix. |
| [**6. Case Study: Hands-On Configuration Guide**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/hands-on-network-configuration-guide.md) | 6 step-by-step practical deployment blueprints: Custom VPC, Auto-to-Custom conversion, Cloud NAT, VPC Peering, Multi-NIC, PSC. |
| [**HLD & LLD Architecture Design**](file:///home/btpl-lap-22/live/gcd/vpc-network/hld-lld-design.md) | High-Level & Low-Level Design diagrams for Global VPC Topologies, Cross-Zone Regional Subnets, 4 Reserved Subnet IP Addresses, Internal DHCP/DNS Scoping, Ephemeral vs Static IP Billing, BYOIP, Dual-Stack IPv4/IPv6, Virtual Routers, and Distributed Stateful Firewalls. |
| [**Decision Tree Guide**](file:///home/btpl-lap-22/live/gcd/vpc-network/decision-tree.md) | Visual decision flowcharts for Auto Mode vs Custom Mode selection, Subnet Mask planning, Zero-Downtime CIDR Expansion rules, Ephemeral vs Static External IP selection, BYOIP, and Firewall Rule targeting. |
| [**CLI Shell Commands & Operations Manual**](file:///home/btpl-lap-22/live/gcd/vpc-network/shell-commands.md) | Exhaustive command manual covering `gcloud compute networks`, `gcloud compute subnets`, `gcloud compute firewall-rules`, `gcloud compute routes`, `gcloud dns`, and BYOIP with **Verification Checks**. |
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
9. **Internal DHCP & Network-Scoped Internal DNS**: Automatic internal DHCP lease & `vm-name.zone.c.PROJECT.internal` Zonal DNS registration (scoped strictly to a single VPC network).
10. **Ephemeral vs Static External IPs & Billing Surcharge**: Dynamics of ephemeral vs static public IPs, including unassigned static IP penalty hourly fees.
11. **Bring Your Own IP (BYOIP)**: Importing customer-owned IPv4 blocks (minimum `/24` prefix) and advertising globally via BGP Anycast.
12. **Hypervisor 1:1 NAT Transparency**: OS non-awareness of external IPs (`ifconfig` / `ip addr` inside guest OS displays private internal IP only).
13. **Cloud DNS 100% SLA**: Global Anycast authoritative DNS infrastructure backed by a 100% uptime SLA.
14. **Alias IP Ranges**: Sub-allocating secondary internal CIDR ranges to a single VM interface (`nic0`) for native container/pod networking.
15. **Virtual Router & Distributed Stateful Firewall**: Massively scalable software-defined virtual router, per-instance read-only routing tables, stateful session tracking, and implied default rules (`Deny Ingress 65535`, `Allow Egress 65535`).
16. **No VM Creation Without VPC Network**: Attempting to launch Compute Engine instances, containers, or App Engine apps without a VPC network fails with `No more networks available`.
17. **Identity-Aware Proxy (IAP) Firewall Rule**: Secure SSH tunneling via `35.235.240.0/20` TCP port 22 (`--tunnel-through-iap`) without public IP exposure.
18. **Multi-Protocol Ingress Firewall Policies**: Defining ICMP, TCP 22 (SSH), and TCP 3389 (RDP) ingress allowances within unified firewall rules (`managementnet-allow-icmp-ssh-rdp`, `privatenet-allow-icmp-ssh-rdp`).
19. **Logical VPC Network Isolation Boundary**: Physical zone co-location (e.g. both VMs in `us-central1-a`) does NOT allow internal IP communication across different VPC networks without VPC Peering or Cloud VPN.
20. **Network Pricing & Egress Cost Flow**: Ingress ($0/GB), Intra-zone internal IP ($0/GB), Intra-zone external IP leak (billed at $0.01/GB inter-zone rate), Inter-zone ($0.01/GB), Inter-region ($0.02-$0.12/GB), and Private Google Access ($0/GB).
