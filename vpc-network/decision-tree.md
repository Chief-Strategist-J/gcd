# VPC Networks & Subnets: Decision Trees

This guide provides visual decision trees (Mermaid flowcharts & ASCII text decision paths) for selecting VPC network modes, sizing subnets, planning non-downtime CIDR expansions, external IP strategies, BYOIP, custom routing, and stateful firewall rule targeting.

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

---

## 4. External IP Address & BYOIP Strategy Decision Tree

```mermaid
flowchart TD
    EXT_START["Evaluate External Facing Requirements"] --> FACING{"Does the workload require direct public internet access?"}

    FACING -- "No (Backend DB / Internal API)" --> NO_EXT["No External IP<br/>- Access via Cloud NAT for outbound<br/>- Connect via IAP SSH for management<br/>- Zero public attack surface"]

    FACING -- "Yes (Public Web / Load Balancer)" --> OWN_IP{"Do you own custom public IPv4 prefixes?"}
    OWN_IP -- "No (Use GCP Managed IPs)" --> STATIC_REQ{"Does the public IP need to remain constant across restarts?"}
    STATIC_REQ -- "No (Dynamic / Temporary)" --> EPHEMERAL["Ephemeral External IP<br/>- Free in-use, released on instance termination"]
    STATIC_REQ -- "Yes (DNS A-record / Firewall Whitelist)" --> RESERVED_STATIC["Reserved Static External IP<br/>- Must keep attached to running VM to avoid unassigned penalty fee"]

    OWN_IP -- "Yes (Owned Public IPv4 Block)" --> SIZE_CHECK{"Is the owned block /24 or larger?"}
    SIZE_CHECK -- "Yes (/24 block or larger)" --> BYOIP["Bring Your Own IP (BYOIP)<br/>- Provision Public Advertised Prefix (PAP)<br/>- Global BGP Anycast announcement"]
    SIZE_CHECK -- "No (Smaller than /24, e.g. /28)" --> BYOIP_FAIL["Ineligible for BYOIP!<br/>BGP requires /24 minimum prefix. Must use GCP Public IPs"]

    style NO_EXT fill:#34A853,color:#fff
    style RESERVED_STATIC fill:#FBBC05,color:#333
    style BYOIP fill:#4285F4,color:#fff
    style BYOIP_FAIL fill:#EA4335,color:#fff
```

---

## 5. Internal DNS Resolution Boundary Decision Tree

```mermaid
flowchart TD
    DNS_START["Internal Hostname Resolution"] --> TARGET_VPC{"Is the target VM in the same VPC network?"}

    TARGET_VPC -- "Yes (Same VPC)" --> GCP_DNS["Automatic GCP Internal DNS Resolution<br/>- Resolves 'vm-name.zone.c.PROJECT.internal'<br/>- Handled by Metadata server 169.254.169.254"]
    TARGET_VPC -- "No (Different VPC / On-Premises)" --> CROSS_DNS{"Is Cloud DNS Private Zone or Peering set up?"}
    CROSS_DNS -- "Yes" --> PRIVATE_ZONE["Cloud DNS Private Zone Resolution<br/>- Resolves cross-VPC internal hostnames"]
    CROSS_DNS -- "No" --> DNS_FAIL["Internal DNS Resolution Fails!<br/>GCP Internal DNS is strictly scoped to a single VPC network"]

    style GCP_DNS fill:#34A853,color:#fff
    style DNS_FAIL fill:#EA4335,color:#fff
```

---

## 6. Firewall Rule Target & Security Targeting Strategy Decision Tree

```mermaid
flowchart TD
    FW_START["Determine Firewall Rule Scope"] --> TARGET_SCOPE{"Which VM instances should the rule apply to?"}

    TARGET_SCOPE -- "All VMs in VPC Network" --> ALL_VMS["Target: All Instances in Network<br/>- Leave target-tags and target-service-accounts blank<br/>- Global baseline security policy"]
    TARGET_SCOPE -- "Specific Group of VMs by Function" --> SEC_AUTH{"Do you require RBAC IAM identity enforcement?"}
    
    SEC_AUTH -- "No (Simple operational grouping)" --> NET_TAGS["Target Network Tags<br/>- e.g., --target-tags=web-server,db-server<br/>- Flexible string tags assigned to instances"]
    SEC_AUTH -- "Yes (Strict enterprise security & RBAC)" --> SA_TARGETS["Target Service Accounts<br/>- e.g., --target-service-accounts=sa-web@project.iam.gserviceaccount.com<br/>- IAM-controlled security boundary prevents unauthorized tag spoofing"]

    style ALL_VMS fill:#FBBC05,color:#333
    style NET_TAGS fill:#4285F4,color:#fff
    style SA_TARGETS fill:#34A853,color:#fff
```
