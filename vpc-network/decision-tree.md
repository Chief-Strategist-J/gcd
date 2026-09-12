# VPC Networks & Subnets: Decision Trees

This guide provides visual decision trees (Mermaid flowcharts & ASCII text decision paths) for selecting VPC network modes, sizing subnets, planning non-downtime CIDR expansions, and choosing inter-VPC and hybrid networking strategies.

---

## 1. VPC Network Mode Selection Decision Tree

```mermaid
flowchart TD
    START["Evaluate Network Project Requirements"] --> USE_CASE{"What is the intended deployment environment?"}

    USE_CASE -- "Sandbox / Quick Prototyping / Testing" --> AUTO_CHECK{"Are you connecting to on-premises or using VPC Peering?"}
    AUTO_CHECK -- "No (Pure standalone playground)" --> AUTO_MODE["Use Default / Auto Mode Network<br/>- Automatically creates subnets in all regions<br/>- Quick setup, default firewall rules"]
    AUTO_CHECK -- "Yes (May connect to enterprise network)" --> CUSTOM_MODE

    USE_CASE -- "Production / Enterprise / Hybrid Cloud" --> CUSTOM_MODE["Use Custom Mode Network<br/>- Explicit regional subnets<br/>- Controlled, non-overlapping CIDR blocks<br/>- Mandatory for Hybrid VPN, Inter-VPC Peering & Shared VPC"]

    AUTO_MODE --> CONVERT_CHECK{"Do you need custom CIDRs later?"}
    CONVERT_CHECK -- "Yes" --> SWITCH_MODE["Convert Auto Mode to Custom Mode<br/>'gcloud compute networks switch-mode --mode=custom'<br/>Note: Irreversible one-way action!"]

    style AUTO_MODE fill:#FBBC05,color:#333
    style CUSTOM_MODE fill:#34A853,color:#fff
    style SWITCH_MODE fill:#EA4335,color:#fff
```

### ASCII Text Breakdown:
* **Playground / Sandbox**: Default / Auto Mode (`10.128.0.0/9` pre-allocated).
* **Enterprise / Production / Hybrid VPN**: Custom Mode (mandatory to prevent CIDR overlapping).
* **Existing Auto Mode Needing Custom CIDRs**: Convert to Custom Mode (`switch-mode --mode=custom`). **Warning**: One-way conversion!

---

## 2. Subnet Mask Sizing & Non-Downtime CIDR Expansion Decision Tree

```mermaid
flowchart TD
    SIZE_START["Determine Subnet Capacity & CIDR Mask"] --> HOST_COUNT{"How many VM instances / workloads will run in the subnet?"}

    HOST_COUNT -- "1 to 200 instances" --> MASK_24["Assign /24 CIDR (256 Total IPs - 4 Reserved = 252 Usable)"]
    HOST_COUNT -- "200 to 1,000 instances" --> MASK_22["Assign /22 CIDR (1,024 Total IPs - 4 Reserved = 1,020 Usable)"]
    HOST_COUNT -- "1,000+ instances (GKE Node Pools)" --> MASK_20["Assign /20 CIDR (4,096 Total IPs - 4 Reserved = 4,092 Usable)"]

    MASK_24 --> EXPAND_CHECK{"Subnet running out of IP addresses?"}
    MASK_22 --> EXPAND_CHECK
    MASK_20 --> EXPAND_CHECK

    EXPAND_CHECK -- "Yes (Zero-Downtime Expansion Needed)" --> CHECK_COLLISION{"Will expanded CIDR overlap with adjacent subnets, VPN, or Peering?"}
    CHECK_COLLISION -- "No Collision" --> EXPAND_CMD["Expand Subnet Range<br/>'gcloud compute networks subnets expand-ip-range'<br/>Pass smaller prefix mask (e.g. /24 -> /20)"]
    CHECK_COLLISION -- "Collision Detected" --> ALTERNATIVE["Cannot Expand!<br/>Create secondary IP range or provision new regional subnet"]

    style MASK_24 fill:#4285F4,color:#fff
    style MASK_20 fill:#34A853,color:#fff
    style EXPAND_CMD fill:#34A853,color:#fff
    style ALTERNATIVE fill:#EA4335,color:#fff
```

---

## 3. Inter-Instance Communication & Routing Decision Tree

```mermaid
flowchart TD
    COMM_START["Determine Communication Path Between Two VMs"] --> NETWORK_MATCH{"Are VM-A and VM-B in the same VPC Network?"}

    NETWORK_MATCH -- "Yes (Same VPC)" --> ZONE_MATCH{"Are they in the same region / zone?"}
    ZONE_MATCH -- "Different Zones or Regions" --> GLOBAL_FIBER["Communicate via Internal IP Address<br/>- Uses Google's Global Private Fiber Backbone<br/>- Zero public internet traversal<br/>- High throughput & low latency"]
    ZONE_MATCH -- "Same Zone" --> LOCAL_FIBER["Communicate via Internal IP Address<br/>- Direct intra-zone rack routing"]

    NETWORK_MATCH -- "No (Different VPC Networks)" --> PEERING_CHECK{"Are the two VPCs peered or connected via VPN?"}
    PEERING_CHECK -- "No Connectivity Configured" --> EDGE_ROUTER["Must communicate via External IP Address<br/>- Traffic routed through Google Edge Routers<br/>- Incurs egress bandwidth charges<br/>- Requires external firewall rules"]
    PEERING_CHECK -- "VPC Peering Configured" --> PEERING_ROUTE["Communicate via Internal IP Address<br/>- Full private routing via VPC Peering"]
    PEERING_CHECK -- "Cloud VPN Configured" --> VPN_ROUTE["Communicate via Internal IP Address over IPsec Tunnel"]

    style GLOBAL_FIBER fill:#34A853,color:#fff
    style EDGE_ROUTER fill:#EA4335,color:#fff
    style PEERING_ROUTE fill:#4285F4,color:#fff
```
