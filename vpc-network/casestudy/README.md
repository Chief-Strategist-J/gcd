# GCP VPC Network Case Study: Deep-Dive Packet Lifecycle, Distributed Firewall Engine & Compute Integration

Welcome to the **Google Cloud Virtual Private Cloud (VPC) Deep-Dive Architecture & Packet Lifecycle Case Study**. This comprehensive engineering guide explains how Google Cloud's software-defined networking (**Andromeda SDN**) operates under the hood, how packets traverse the virtual and physical network infrastructure, how the distributed stateful firewall engine filters and drops/allows traffic, and how Compute Engine VMs physically and logically connect to VPC networks and subnetworks.

---

## Case Study Directory Sitemap

| Document | Focus Area | Key Architectural Concepts Covered |
| :--- | :--- | :--- |
| [**1. High-Level Architecture (HLD)**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/hld-architecture.md) | Network Topology & SDN Control Plane | Andromeda SDN Control/Data Plane, Hoverboard Flow Programmers, Global VPC Infrastructure, Jupiter/B4 Data Center Fabric, Subnet-Route-Firewall Binding. |
| [**2. Low-Level Packet Lifecycle (LLD)**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/lld-packet-lifecycle.md) | Packet Traversal & Network Mechanics | Guest OS Socket $\rightarrow$ Virtio-Net Driver $\rightarrow$ Hypervisor Tap Interface $\rightarrow$ Andromeda Packet Processor $\rightarrow$ Geneve/STT Encapsulation $\rightarrow$ Physical Wire $\rightarrow$ Decapsulation $\rightarrow$ Destination Delivery. |
| [**3. Stateful Firewall Engine Deep Dive**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/firewall-deep-dive.md) | Firewall Filtering & Security Enforcement | Distributed Connection Tracking Engine (`conntrack`), Priority Evaluation (0-65535), Deny-Override, Target Tags, Service Accounts, Hierarchical Policies, Packet Drop Flowchart. |
| [**4. Compute & Network Integration**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/compute-network-integration.md) | Hypervisor & VM Connectivity | Virtual Network Interfaces (vNICs), Subnet CIDR Binding, 4 Reserved Subnet IPs, 1:1 NAT Transparency, Metadata Server (`169.254.169.254`), Multi-NIC, **Connectivity Scope Isolation Matrix**, and **Common GCP VPC Network Design Patterns (Multi-Zone HA, Multi-Region Globalization, Cloud NAT Outbound, Private Google Access)**. |

| [**5. Operations & Troubleshooting Manual**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/commands-and-troubleshooting.md) | CLI Commands & Diagnostic Manual | `gcloud compute` CLI, Network Intelligence Center Connectivity Tests, VPC Flow Logs, Packet Mirroring, Diagnostic Error Resolution Matrix. |
| [**6. Hands-On Configuration Guide**](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/hands-on-network-configuration-guide.md) | Step-by-Step Practical Blueprints | 6 real-world deployment scenarios: Custom VPC, Auto-to-Custom conversion, Cloud NAT, VPC Peering, Multi-NIC, PSC. |

---

## Core System Architecture Overview

```mermaid
graph TD
    classDef tenant fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef subnet fill:#0F172A,stroke:#818CF8,stroke-width:2px,color:#F8FAFC;
    classDef vm fill:#1E1B4B,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;
    classDef sdn fill:#064E3B,stroke:#34D399,stroke-width:2px,color:#F8FAFC;
    classDef phy fill:#451A03,stroke:#F97316,stroke-width:2px,color:#F8FAFC;

    subgraph GlobalVPC ["Global VPC Network Domain (Tenant Isolated)"]
        subgraph SubnetA ["Subnet A: us-central1 (10.128.0.0/20)"]
            VM1["VM 1: mynet-us-vm<br/>Internal IP: 10.128.0.2<br/>External IP: 34.67.18.181 (1:1 NAT)"]:::vm
        end
        subgraph SubnetB ["Subnet B: europe-west1 (10.132.0.0/20)"]
            VM2["VM 2: mynet-eu-vm<br/>Internal IP: 10.132.0.2<br/>External IP: 35.198.10.12 (1:1 NAT)"]:::vm
        end
    end

    subgraph AndromedaLayer ["Andromeda Software-Defined Network (SDN) Layer"]
        HB["Hoverboard Flow Programmers<br/>(Control Plane Centralized Controller)"]:::sdn
        HPP1["Host Packet Processor A<br/>(Fast-Path Kernel / DPDK)"]:::sdn
        HPP2["Host Packet Processor B<br/>(Fast-Path Kernel / DPDK)"]:::sdn
    end

    subgraph Underlay ["Physical Data Center Fabric Layer"]
        JUPITER["Jupiter Switching Fabric<br/>(100G/400G Optical Clos Network)"]:::phy
        B4WAN["B4 Software-Defined WAN<br/>(Google Private Global Fiber Backbone)"]:::phy
    end

    VM1 <-->|Virtio-Net / Tap| HPP1
    VM2 <-->|Virtio-Net / Tap| HPP2
    HB -->|Pushes OpenFlow / P4 Rules| HPP1
    HB -->|Pushes OpenFlow / P4 Rules| HPP2
    HPP1 <-->|Geneve Tunnel Encapsulation| JUPITER
    HPP2 <-->|Geneve Tunnel Encapsulation| JUPITER
    JUPITER <-->|Cross-Region Trunking| B4WAN

    class GlobalVPC,SubnetA,SubnetB tenant;
    class SubnetA,SubnetB subnet;
```

---

## Summary of Key Learnings & Engineering Concepts

1. **Software-Defined Networking (Andromeda)**: GCP does not use physical switches or hardware middleboxes for VPC routing and firewalling. All network logic is programmed into Andromeda software packet processors co-located on the Linux hypervisors hosting Google Compute Engine VMs.
2. **Stateful Connection Tracking**: GCP Distributed Firewalls operate statefully at the hypervisor level. When an egress packet or ingress packet opens a session, the hypervisor's conntrack engine stores the session tuples `(src_ip, src_port, dst_ip, dst_port, protocol)`. Return response packets bypass firewall rule evaluation automatically.
3. **Hypervisor 1:1 NAT Transparency**: Public External IPs are NOT configured on the Guest OS interface (`eth0` / `nic0`). The Guest OS only sees its private RFC 1918 internal IP address. Andromeda packet processors perform 1:1 Network Address Translation (NAT) at the hypervisor egress/ingress boundary invisibly.
4. **VPC Network Isolation Boundary**: Separate VPC networks represent completely isolated routing domains. Internal IP traffic directed across unpeered VPC networks is dropped at the source hypervisor's Andromeda routing lookup stage before ever entering the physical wire.

---

## Related Workspace References

- [Main VPC Module Readme](file:///home/btpl-lap-22/live/gcd/vpc-network/README.md)
- [VPC High-Level & Low-Level Design](file:///home/btpl-lap-22/live/gcd/vpc-network/hld-lld-design.md)
- [VPC Decision Tree Guide](file:///home/btpl-lap-22/live/gcd/vpc-network/decision-tree.md)
- [VPC CLI Operations Manual](file:///home/btpl-lap-22/live/gcd/vpc-network/shell-commands.md)
