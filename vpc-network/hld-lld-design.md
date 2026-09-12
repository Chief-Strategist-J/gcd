# VPC Networks & Subnets: High-Level & Low-Level Design Architecture

This document presents the **High-Level Design (HLD)** and **Low-Level Design (LLD)** for Google Cloud Platform (GCP) Virtual Private Cloud (VPC) Networks, Regional Subnetworks, IP Address Allocation, and Connectivity Mechanics.

---

## 1. High-Level Design (HLD) Architecture

GCP VPC networks are **global resources**. Unlike traditional cloud providers where a VPC is tied to a single region, a GCP VPC spans all available regions worldwide simultaneously. Inside a global VPC, **subnets are regional resources** that extend across all zones within a given region.

```mermaid
graph TD
    subgraph GCP Project Scope
        subgraph Global VPC Network: custom-vpc-prod
            
            subgraph Region 1: us-central1
                subgraph Subnet 1: 10.1.0.0/24 - Spans Zones A and B
                    VM_A["VM-A (Zone us-central1-a)<br/>Internal IP: 10.1.0.2"]
                    VM_B["VM-B (Zone us-central1-b)<br/>Internal IP: 10.1.0.3"]
                end
            end

            subgraph Region 2: europe-west1
                subgraph Subnet 2: 10.2.0.0/24
                    VM_C["VM-C (Zone europe-west1-b)<br/>Internal IP: 10.2.0.2"]
                end
            end

            VM_A <-->|Private Global Fiber Network<br/>Internal IP Communication| VM_B
            VM_A <-->|Private Global Fiber Network<br/>Cross-Region Internal IP| VM_C
        end

        subgraph Global VPC Network: isolated-vpc-qa
            subgraph Region 1: us-central1
                subgraph Subnet QA: 10.99.0.0/24
                    VM_D["VM-D (Zone us-central1-a)<br/>Internal IP: 10.99.0.2<br/>External IP: 35.192.10.4"]
                end
            end
        end

        VM_A <-->|Communicates via External IP<br/>Traverses Google Edge Routers| VM_D
    end

    subgraph On-Premises Network
        ON_PREM_GATEWAY["On-Premises VPN Router<br/>IP: 192.168.1.1"]
    end

    subgraph Hybrid Connectivity
        CLOUD_VPN["Cloud VPN Gateway<br/>(Attached to custom-vpc-prod)"]
    end

    CLOUD_VPN <-->|IPsec Tunnel| ON_PREM_GATEWAY
    VM_A <-->|Secure Internal Access via VPN| ON_PREM_GATEWAY
    VM_C <-->|Secure Internal Access via VPN| ON_PREM_GATEWAY

    style Global VPC Network: custom-vpc-prod fill:#4285F4,stroke:#333,stroke-width:2px,color:#fff
    style Global VPC Network: isolated-vpc-qa fill:#EA4335,stroke:#333,stroke-width:2px,color:#fff
    style Subnet 1: 10.1.0.0/24 - Spans Zones A and B fill:#34A853,stroke:#333,stroke-width:1px,color:#fff
    style Subnet 2: 10.2.0.0/24 fill:#34A853,stroke:#333,stroke-width:1px,color:#fff
```

---

## 2. Low-Level Design (LLD) Architecture

### A. Subnet IP Address Allocation & 4 Reserved IP Addresses

In every primary subnet CIDR block, Google Cloud automatically reserves **4 IP addresses**. You cannot assign these 4 reserved addresses to VM instances or load balancers.

```text
Subnet Range Example: 10.1.0.0/24 (Total 256 Addresses: 10.1.0.0 to 10.1.0.255)
┌─────────────────┬───────────────────────────────┬─────────────────────────────────────────────┐
│ Address         │ Status                        │ Function / Description                      │
├─────────────────┼───────────────────────────────┼─────────────────────────────────────────────┤
│ 10.1.0.0        │ Reserved by GCP               │ Network Address (.0)                        │
│ 10.1.0.1        │ Reserved by GCP               │ Subnet Default Gateway (.1)                 │
│ 10.1.0.2        │ Available for Assignment      │ First usable host IP address                │
│ 10.1.0.3        │ Available for Assignment      │ Second usable host IP address               │
│ ...             │ Available for Assignment      │ Usable host range (10.1.0.4 to 10.1.0.253)  │
│ 10.1.0.254      │ Reserved by GCP               │ Second-to-last address (Reserved by GCP)   │
│ 10.1.0.255      │ Reserved by GCP               │ Last address / Subnet Broadcast Address     │
└─────────────────┴───────────────────────────────┴─────────────────────────────────────────────┘
```

---

### B. Network Mode Architecture & Conversion Mechanics

GCP supports two primary modes of VPC creation:

```mermaid
graph TD
    subgraph VPC Network Modes
        AUTO["Auto Mode VPC Network<br/>- Automatically creates 1 subnet per region<br/>- Uses predefined /20 subnets inside 10.128.0.0/9<br/>- Default firewall rules created (SSH, RDP, ICMP, Internal)"]
        CUSTOM["Custom Mode VPC Network<br/>- Zero subnets created automatically<br/>- Complete user control over regional CIDRs<br/>- Non-overlapping IP ranges across regions"]
    end

    AUTO -->|One-Way Irreversible Conversion| CONVERT["gcloud compute networks switch-mode --mode=custom"]
    CONVERT --> CUSTOM

    style AUTO fill:#FBBC05,stroke:#333,stroke-width:2px,color:#333
    style CUSTOM fill:#34A853,stroke:#333,stroke-width:2px,color:#fff
    style CONVERT fill:#EA4335,stroke:#333,stroke-width:2px,color:#fff
```

* **Auto Mode**:
  - Automatically provisions one subnet per region within the `10.128.0.0/9` CIDR block using a `/20` prefix mask (e.g., `10.128.0.0/20` in `us-central1`, `10.132.0.0/20` in `europe-west1`).
  - Automatically adds new `/20` subnets when new GCP regions are launched.
  - Can be converted to Custom Mode, but **Custom Mode networks can NEVER be converted back to Auto Mode**.

* **Custom Mode**:
  - Starts with zero subnets.
  - Allows full manual configuration of regional subnets and custom CIDR ranges (RFC 1918 compliant: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`).
  - Essential for enterprise production, hybrid VPN, and VPC Network Peering setups to avoid IP collisions.

---

### C. Zero-Downtime Subnet IP Range Expansion

Google Cloud allows increasing the IP address space of any existing subnetwork **without shutting down workloads or restarting VM instances**.

```text
Initial Subnet Configuration:
Subnet Name: prod-subnet-us
CIDR: 10.1.0.0/24 (256 IP Addresses)
Prefix Mask: /24

Expansion Action (Zero-Downtime):
gcloud compute networks subnets expand-ip-range prod-subnet-us --prefix-length=20

Expanded Subnet Configuration:
Subnet Name: prod-subnet-us
CIDR: 10.1.0.0/20 (4,096 IP Addresses)
Prefix Mask: /20

Expansion Rules & Constraints:
1. The new prefix mask MUST be smaller than the existing mask (e.g., /24 -> /20).
2. Expansion is ONE-WAY. You CANNOT shrink a subnet IP range.
3. Auto mode subnets start at /20 and can only be expanded up to /16 max.
4. Custom mode subnets can be expanded to any valid non-overlapping RFC range.
5. The expanded range MUST NOT overlap with any existing subnet in the same VPC.
```

---

### D. Dual-Stack IPv4 / IPv6 Architecture

Custom VPC networks support **Dual-Stack** configuration:
- **IPv4 Range**: Private internal RFC 1918 CIDR assigned to subnets and VMs.
- **IPv6 Range**: Public or internal `/64` IPv6 range assigned to subnets (`/96` mask per VM instance).
- VMs can communicate over IPv4 and IPv6 simultaneously without NAT gateway translation.
