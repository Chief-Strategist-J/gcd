# High-Level Architecture (HLD): GCP VPC Networks & Andromeda SDN

This document details the **High-Level Architecture (HLD)** of Google Cloud Virtual Private Cloud (VPC) networks, Google's **Andromeda Software-Defined Network (SDN)** substrate, and the structural relationships between VPC networks, regional subnetworks, distributed firewalls, and routing tables.

---

## 1. Physical vs. Logical Infrastructure Stack

In traditional cloud or data center environments, networking relies on physical switches, VLAN tags, virtual wire routers, and hardware firewall appliances. Google Cloud decouples logical network constructs completely from physical infrastructure using **Andromeda SDN**.

```mermaid
graph TD
    classDef logical fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef sdn fill:#064E3B,stroke:#34D399,stroke-width:2px,color:#F8FAFC;
    classDef physical fill:#451A03,stroke:#F97316,stroke-width:2px,color:#F8FAFC;

    subgraph LogicalLayer ["LOGICAL TENANT LAYER (Virtual Private Cloud)"]
        VPC["Global VPC Network (RFC 1918 Address Space)"]:::logical
        Subnets["Regional Subnetworks (Spans all AZs in a Region)"]:::logical
        Routes["Global Virtual Routing Tables (Read-Only)"]:::logical
        Firewall["Distributed Stateful Firewall Engine"]:::logical
    end

    subgraph SDNLayer ["SDN VIRTUALIZATION LAYER (Andromeda Engine)"]
        HPP["Host Fast-Path Packet Processors (DPDK / SmartNIC)"]:::sdn
        Hoverboard["Hoverboard Flow Programmers & Gateway Clusters"]:::sdn
        Geneve["Geneve / STT Overlay Tunnel Encapsulator"]:::sdn
    end

    subgraph PhysicalLayer ["PHYSICAL UNDERLAY FABRIC (Data Center Network)"]
        Jupiter["Jupiter Data Center Switching Fabric (100G/400G Clos)"]:::physical
        B4WAN["B4 Software-Defined Global WAN Backbone"]:::physical
        EdgeRouters["Andromeda Border Routers & Edge Gateways"]:::physical
    end

    LogicalLayer -->|Programmed via Control Plane| SDNLayer
    SDNLayer -->|Encapsulated Ethernet Frames| PhysicalLayer
```

---

## 2. Andromeda SDN Control & Data Plane Architecture

Andromeda is Google's NFV (Network Function Virtualization) software stack that delivers high-throughput, low-latency VPC networking.

### Andromeda Component Architecture Flowchart

```mermaid
graph TD
    classDef cp fill:#1E1B4B,stroke:#818CF8,stroke-width:2px,color:#F8FAFC;
    classDef hb fill:#065F46,stroke:#34D399,stroke-width:2px,color:#F8FAFC;
    classDef hpp fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef vm fill:#312E81,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;

    CP["Andromeda Global Control Plane<br/>(Centralized Network State)"]:::cp
    HB["Hoverboard Regional Flow Programmer<br/>(OpenFlow / P4 Translator)"]:::hb
    
    subgraph HostHypervisorA ["Host Hypervisor A (US Central)"]
        VM1["VM 1: mynet-us-vm"]:::vm
        TAP1["Tap Interface (tap1234)"]:::hpp
        HPP1["Host Packet Processor A (Fast-Path)"]:::hpp
    end

    subgraph HostHypervisorB ["Host Hypervisor B (Europe West)"]
        VM2["VM 2: mynet-eu-vm"]:::vm
        TAP2["Tap Interface (tap5678)"]:::hpp
        HPP2["Host Packet Processor B (Fast-Path)"]:::hpp
    end

    CP -->|Pushes Network Config| HB
    HB -->|Installs Flow Tables| HPP1
    HB -->|Installs Flow Tables| HPP2
    
    VM1 <-->|Virtio Ring Buffer| TAP1
    TAP1 <--> HPP1
    
    VM2 <-->|Virtio Ring Buffer| TAP2
    TAP2 <--> HPP2
    
    HPP1 <-->|Geneve Encapsulated Tunnel over B4 Fabric| HPP2
```

1. **Andromeda Control Plane**: Maintains global tenant network state, subnet mappings, route tables, and firewall policies across all GCP regions.
2. **Hoverboard Flow Programmers**: Regional controllers that translate VPC configurations into low-level forwarding table entries (OpenFlow/P4-like rules) and push them down to hypervisors.
3. **Host Fast-Path Packet Processor**: Software module running on each Linux hypervisor host. It executes firewall checks, NAT transformations, and encapsulation in user-space or hardware offload engine (DPDK / SmartNIC) to achieve near-wire speed.
4. **Software Gateways**: High-capacity cluster routers used for fallback path resolution (when host fast-path cache misses occur) and external internet gateway routing.

---

## 3. Structural Relationship: VPC, Subnets, Firewalls & Routes

A **VPC Network** is a global resource that serves as a container for regional subnetworks, virtual routers, and distributed firewall rules.

```mermaid
graph TD
    classDef vpc fill:#0F172A,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef subnet fill:#1E1B4B,stroke:#818CF8,stroke-width:2px,color:#F8FAFC;
    classDef fw fill:#064E3B,stroke:#34D399,stroke-width:2px,color:#F8FAFC;
    classDef vm fill:#4C1D95,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;

    VPC["GLOBAL VPC NETWORK<br/>(e.g., mynetwork)"]:::vpc

    SubnetUS["Regional Subnet: us-central1<br/>CIDR: 10.128.0.0/20<br/>Span: us-central1-a/b/c/f"]:::subnet
    SubnetEU["Regional Subnet: europe-west1<br/>CIDR: 10.132.0.0/20<br/>Span: europe-west1-b/c"]:::subnet
    FW["Global Distributed Firewall Engine<br/>Priority Policies (0 - 65535)<br/>Evaluated at Hypervisor vNIC"]:::fw

    VMUS["Compute Engine VM: mynet-us-vm<br/>Internal IP: 10.128.0.2<br/>Zone: us-central1-c"]:::vm
    VMEU["Compute Engine VM: mynet-eu-vm<br/>Internal IP: 10.132.0.2<br/>Zone: europe-west1-c"]:::vm

    VPC --> SubnetUS
    VPC --> SubnetEU
    VPC --> FW

    SubnetUS --> VMUS
    SubnetEU --> VMEU
    FW -.->|Enforced on vNIC| VMUS
    FW -.->|Enforced on vNIC| VMEU
```

---

## 4. Firewall and Route Interconnection Matrix

In Google Cloud VPC networking, **Virtual Routing Tables** and the **Distributed Firewall Engine** act as two distinct security and forwarding layers operating in sequence inside the hypervisor's Andromeda Host Packet Processor (HPP).

### Egress vs. Ingress Interconnection Evaluation Order

```mermaid
graph TD
    classDef start fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef fw fill:#064E3B,stroke:#34D399,stroke-width:2px,color:#F8FAFC;
    classDef route fill:#4C1D95,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;
    classDef drop fill:#881337,stroke:#FB7185,stroke-width:2px,color:#F8FAFC;
    classDef pass fill:#14532D,stroke:#4ADE80,stroke-width:2px,color:#F8FAFC;

    subgraph EgressPipeline ["EGRESS PROCESSING PIPELINE (Source Host Hypervisor)"]
        OutPkt["1. Guest VM Transmits Outbound Packet"]:::start --> EgressCT{"In Conntrack Table?"}:::fw
        EgressCT -->|Yes: Established Flow| EgressRoute
        EgressCT -->|No| EgressFW{"2. Egress Firewall Check:<br/>Priority 0 -> 65535"}:::fw
        
        EgressFW -->|DENY| DropEgressFW["PACKET DROPPED (Egress Firewall Block)<br/>Routing Table Never Queried!"]:::drop
        EgressFW -->|ALLOW| EgressRoute{"3. Virtual Routing Lookup:<br/>Longest Prefix Match (LPM)"}:::route

        EgressRoute -->|No Route Match| DropEgressRoute["PACKET DROPPED (ENETUNREACH)<br/>No route found to destination IP"]:::drop
        EgressRoute -->|Route Found| Encap["4. Geneve Header Encapsulation<br/>Transmit to Physical Wire"]:::pass
    end

    subgraph IngressPipeline ["INGRESS PROCESSING PIPELINE (Destination Host Hypervisor)"]
        InPkt["5. Physical Wire Frame Arrives at Host B"]:::start --> Decap["6. Decapsulate Geneve Outer Header"]:::pass
        Decap --> IngressCT{"In Conntrack Table?"}:::fw
        IngressCT -->|Yes: Established Flow| InjectTap["8. Inject Packet to Target VM Tap"]:::pass
        IngressCT -->|No| IngressFW{"7. Ingress Firewall Check:<br/>Priority 0 -> 65535"}:::fw

        IngressFW -->|DENY| DropIngressFW["PACKET DROPPED (Ingress Firewall Block)<br/>Logged as security drop"]:::drop
        IngressFW -->|ALLOW| InjectTap
    end

    Encap --> InPkt
```

### Routing Table Precedence & Selection Rules

When a packet passes the Egress Firewall check, Andromeda evaluates the VPC's global virtual routing table using the following strict hierarchy:

1. **Longest Prefix Match (LPM)**: The route with the most specific subnet mask wins. For example, a route to `10.128.0.0/24` is chosen over `10.0.0.0/8`.
2. **Route Priority (Metric)**: If two routes have identical destination prefixes, the route with the **lowest priority integer** wins (e.g. Priority `100` beats Priority `1000`).
3. **Equal-Cost Multi-Path (ECMP)**: If multiple routes have identical destination prefixes and identical priorities, Andromeda spreads traffic across all matching next-hops using a 5-tuple hash.

---

## 5. Auto Mode vs. Custom Mode Topology

```mermaid
graph LR
    classDef auto fill:#1E293B,stroke:#F59E0B,stroke-width:2px,color:#F8FAFC;
    classDef custom fill:#0F172A,stroke:#10B981,stroke-width:2px,color:#F8FAFC;
    classDef action fill:#4C1D95,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;

    subgraph AutoMode ["AUTO MODE VPC NETWORK"]
        A1["Auto-provisions one /20 subnet in every GCP region"]:::auto
        A2["Predefined CIDRs (10.128.0.0/20, 10.132.0.0/20...)"]:::auto
        A3["Preset Firewall Rules (Allow ICMP, SSH, RDP, Internal)"]:::auto
    end

    Convert["gcloud compute networks update --switch-to-custom-mode<br/>(One-Way Irreversible Migration)"]:::action

    subgraph CustomMode ["CUSTOM MODE VPC NETWORK"]
        C1["Zero subnets created by default"]:::custom
        C2["Administrator explicitly defines CIDR blocks (/24, /16...)"]:::custom
        C3["Recommended for production to avoid CIDR overlap"]:::custom
    end

    AutoMode --> Convert --> CustomMode
```

---

## 6. Cross-VPC Network Isolation Architecture

GCP VPC networks enforce **strict multi-tenant RFC 1918 isolation**.

```mermaid
graph TD
    classDef vpc1 fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef vpc2 fill:#0F172A,stroke:#818CF8,stroke-width:2px,color:#F8FAFC;
    classDef drop fill:#881337,stroke:#FB7185,stroke-width:2px,color:#F8FAFC;
    classDef allow fill:#14532D,stroke:#4ADE80,stroke-width:2px,color:#F8FAFC;

    subgraph VPC1 ["VPC NETWORK: mynetwork"]
        VM1["VM: mynet-us-vm<br/>Internal IP: 10.128.0.2"]:::vpc1
    end

    subgraph VPC2 ["VPC NETWORK: managementnet"]
        VM2["VM: mgmt-us-vm<br/>Internal IP: 10.130.0.2<br/>External IP: 35.188.20.220"]:::vpc2
    end

    LookupInternal["Andromeda Routing Lookup:<br/>Destination: 10.130.0.2 in mynetwork"]:::drop
    LookupExternal["Andromeda Gateway Lookup:<br/>Destination: 35.188.20.220 (External IP)"]:::allow

    VM1 -->|Ping Internal IP 10.130.0.2| LookupInternal
    LookupInternal -->|Target Not in VPC Domain| DropAction["PACKET DROPPED AT SOURCE HYPERVISOR<br/>Reason: ENETUNREACH<br/>Packet Loss: 100%"]:::drop

    VM1 -->|Ping External IP 35.188.20.220| LookupExternal
    LookupExternal -->|Match Default Gateway 0.0.0.0/0| PassAction["PACKET ROUTED OVER EDGE GATEWAY<br/>Ingress Rule: ALLOW ICMP<br/>Status: SUCCESS"]:::allow
    PassAction --> VM2
```

---

## Related Workspace Documents

- [Case Study Index](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/README.md)
- [Low-Level Packet Lifecycle (LLD)](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/lld-packet-lifecycle.md)
- [Stateful Firewall Deep Dive](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/firewall-deep-dive.md)
- [Compute & Network Integration](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/compute-network-integration.md)
