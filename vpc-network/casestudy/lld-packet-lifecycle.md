# Low-Level Design (LLD): End-to-End Packet Lifecycle & Traversal

This document details the **Low-Level Design (LLD)** of how a network packet travels from an application running inside a Compute Engine Virtual Machine, through the Linux hypervisor kernel, Andromeda Fast-Path packet processing pipeline, stateful firewall inspection, overlay encapsulation, physical data center fabric, and into the target destination instance.

---

## 1. Complete End-to-End Packet Lifecycle Architecture

When `mynet-us-vm` (`10.128.0.2`) sends an ICMP Echo Request (`ping`) or TCP payload to `mynet-eu-vm` (`10.132.0.2`), the packet traverses **12 discrete processing stages**.

```mermaid
graph TD
    classDef guest fill:#1E1B4B,stroke:#818CF8,stroke-width:2px,color:#F8FAFC;
    classDef hyp fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef fw fill:#064E3B,stroke:#34D399,stroke-width:2px,color:#F8FAFC;
    classDef encap fill:#4C1D95,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;
    classDef phy fill:#451A03,stroke:#F97316,stroke-width:2px,color:#F8FAFC;

    S1["STEP 1: Guest Application & Socket Layer<br/>App issues socket(), connect(), sendto() sys calls"]:::guest
    S2["STEP 2: Guest OS Virtio-Net Driver<br/>Frame formatted (Src/Dst MAC), written to virtqueue ring buffer"]:::guest
    S3["STEP 3: Hypervisor Tap Interface<br/>KVM thread intercepts frame from virtqueue onto tap interface"]:::hyp
    S4["STEP 4: Andromeda Host Packet Processor (HPP)<br/>Intercepted by user-space DPDK / SmartNIC fast-path pipeline"]:::hyp
    S5["STEP 5: Stateful Firewall Egress Evaluation<br/>Conntrack lookup + Priority rules check (ALLOW)"]:::fw
    S6["STEP 6: Virtual Routing & NAT Lookup<br/>Route match: 10.132.0.0/20 -> Resolve Host B Physical IP"]:::hyp
    S7["STEP 7: Overlay Tunnel Encapsulation<br/>Attach Geneve Header (VPC Tenant ID: 0x8F4A, VNI, UDP 6081)"]:::encap
    S8["STEP 8: Physical Data Center Fabric<br/>Transmit over 100G NIC -> Jupiter Fabric -> B4 Global WAN Fiber"]:::phy
    S9["STEP 9: Destination Host Decapsulation<br/>Host B Andromeda HPP strips Geneve outer header & extracts Tenant ID"]:::encap
    S10["STEP 10: Stateful Firewall Ingress Evaluation<br/>Conntrack lookup + Ingress rules check (ALLOW)"]:::fw
    S11["STEP 11: Hypervisor Tap Interface Injection<br/>Ethernet frame injected into target VM's tap interface"]:::hyp
    S12["STEP 12: Target Guest OS Delivery & Response<br/>Virtqueue interrupt triggers Target Kernel. ICMP Reply generated"]:::guest

    S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7 --> S8 --> S9 --> S10 --> S11 --> S12
```

---

## 2. Sequence Diagram: Intra-VPC Cross-Region Packet Traversal

```mermaid
sequenceDiagram
    autonumber
    actor App as App / Ping Payload
    participant VM1 as Guest OS (mynet-us-vm)
    participant TapA as Host Hypervisor A (Tap)
    participant AndA as Andromeda HPP (Host A)
    participant B4 as Physical Fabric (B4 WAN)
    participant AndB as Andromeda HPP (Host B)
    participant TapB as Host Hypervisor B (Tap)
    participant VM2 as Guest OS (mynet-eu-vm)

    App->>VM1: Generate Packet (10.128.0.2 -> 10.132.0.2)
    VM1->>TapA: Write packet to Virtio Ring Buffer
    TapA->>AndA: Intercept frame via kernel tap
    Note over AndA: Stateful Egress Firewall Check (ALLOW)<br/>Insert session tuple into Conntrack Table
    Note over AndA: Virtual Route Lookup<br/>Destination: Host B Physical IP (192.168.4.12)
    Note over AndA: Geneve Encapsulation<br/>Attach VPC Tenant ID (0x8F4A)
    AndA->>B4: Transmit Encapsulated Frame over Jupiter/B4 Fabric
    B4->>AndB: Deliver Frame to Europe West Data Center Host B
    Note over AndB: Decapsulate Geneve Outer Header<br/>Extract VPC Tenant ID & Inner IP Frame
    Note over AndB: Stateful Ingress Firewall Check (ALLOW)<br/>Insert session tuple into Host B Conntrack Table
    AndB->>TapB: Inject Ethernet Frame into Tap Interface
    TapB->>VM2: Deliver Frame to Virtio Ring Buffer of Target VM
    VM2-->>AndB: Send ICMP Reply (Automatic Conntrack Match -> Fast Pass)
```

---

## 3. Header Framing & Encapsulation Mathematical Breakdown

To maintain tenant isolation without hardware VLAN constraints, Andromeda wraps every tenant packet inside a **Geneve (Generic Network Virtualization Encapsulation)** or **STT (Stateless Transport Tunneling)** frame.

### Encapsulated Packet Header Structure

```mermaid
graph TD
    classDef outer fill:#451A03,stroke:#F97316,stroke-width:2px,color:#F8FAFC;
    classDef udp fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef geneve fill:#4C1D95,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;
    classDef inner fill:#064E3B,stroke:#34D399,stroke-width:2px,color:#F8FAFC;
    classDef payload fill:#0F172A,stroke:#818CF8,stroke-width:2px,color:#F8FAFC;

    subgraph HeaderStack ["GENEVE OVERLAY PACKET HEADER FRAMING"]
        H1["1. Outer Physical Ethernet Header (14 Bytes)<br/>Src Host A MAC | Dst Gateway MAC | EtherType: 0x0800 (IPv4)"]:::outer
        H2["2. Outer Physical IPv4 Header (20 Bytes)<br/>Src Host A IP (192.168.1.10) | Dst Host B IP (192.168.4.12) | Proto: UDP"]:::outer
        H3["3. Outer UDP Header (8 Bytes)<br/>Src Port: Entropy Hash | Dst Port: 6081 (Geneve Standard Port)"]:::udp
        H4["4. Geneve Header (8 Bytes)<br/>Ver: 0 | Opt Len: 0 | Protocol: 0x6558 | VNI: 24-bit VPC Tenant ID"]:::geneve
        H5["5. Inner Tenant Ethernet Header (14 Bytes)<br/>Src VM MAC (Virtio) | Dst Gateway MAC (04:01:00:00:00:01)"]:::inner
        H6["6. Inner Tenant IPv4 Header (20 Bytes)<br/>Src VM IP: 10.128.0.2 | Dst VM IP: 10.132.0.2 | Protocol: 1 (ICMP)"]:::inner
        H7["7. Inner Tenant Payload<br/>ICMP Echo Request / TCP Payload Data"]:::payload
    end

    H1 --> H2 --> H3 --> H4 --> H5 --> H6 --> H7
```

### Encapsulation Overhead Formula:

$$\text{Total Encapsulation Overhead} = 14\,(\text{Outer Eth}) + 20\,(\text{Outer IP}) + 8\,(\text{Outer UDP}) + 8\,(\text{Geneve}) = 50\text{ Bytes}$$

To accommodate this 50-byte header overhead without fragmentation, the GCP physical network fabric supports Jumbo Frames with an **underlay MTU of 8800 bytes**, allowing tenant VMs to run at standard **1460-byte MTU** or **8850-byte Jumbo MTU** seamlessly.

---

## 4. Cross-VPC Network Isolation Drop Mechanics

When `mynet-us-vm` (`10.128.0.2` in `mynetwork`) attempts to ping `10.130.0.2` (`managementnet-us-vm` in `managementnet`), the packet is dropped at **Stage 6 (Virtual Routing Lookup)**:

```mermaid
graph TD
    classDef vm fill:#1E1B4B,stroke:#818CF8,stroke-width:2px,color:#F8FAFC;
    classDef sdn fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef check fill:#4C1D95,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;
    classDef drop fill:#881337,stroke:#FB7185,stroke-width:2px,color:#F8FAFC;

    VM["Guest OS: mynet-us-vm<br/>(10.128.0.2 in VPC 'mynetwork')"]:::vm
    Tap["Host Hypervisor Tap Interface"]:::sdn
    Andromeda["Andromeda Fast-Path Engine (Host A)"]:::sdn
    RouteLookup{"Virtual Route Lookup:<br/>Is 10.130.0.2 in 'mynetwork'?"}:::check

    DropNode["IMMEDIATE PACKET DROP<br/>- Reason: ENETUNREACH<br/>- Result: 100% Packet Loss<br/>- Zero bytes sent over physical wire"]:::drop

    VM -->|Ping 10.130.0.2| Tap
    Tap --> Andromeda
    Andromeda --> RouteLookup
    RouteLookup -->|Target IP Not Found in Tenant Routing Table| DropNode
```

---

## 5. User-Centric End-to-End Client Traversal Flow

This section details how an **external user on the Internet** connects to an application running inside a GCP Compute Engine VM across the entire Google network perimeter, edge infrastructure, load balancers, and hypervisor stack.

### Client-to-Application End-to-End Flowchart

```mermaid
graph TD
    classDef client fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef edge fill:#451A03,stroke:#F97316,stroke-width:2px,color:#F8FAFC;
    classDef proxy fill:#4C1D95,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;
    classDef gcp fill:#064E3B,stroke:#34D399,stroke-width:2px,color:#F8FAFC;
    classDef vm fill:#1E1B4B,stroke:#818CF8,stroke-width:2px,color:#F8FAFC;

    subgraph UserPerimeter ["1. USER CLIENT PERIMETER"]
        User["User Browser / Client<br/>Types https://myapp.company.com"]:::client
        DNS["Cloud DNS / Anycast<br/>Resolves VIP: 34.120.50.10"]:::client
    end

    subgraph GoogleEdge ["2. GOOGLE EDGE INFRASTRUCTURE (Point of Presence)"]
        BGP["BGP Anycast Edge Router<br/>Ingress to Google Fiber Network"]:::edge
        Maglev["Maglev L4 Load Balancer<br/>Consistent Hash Flow Distribution"]:::edge
        Armor["Cloud Armor WAF Security<br/>GeoIP, DDoS, SQLi/XSS Inspection"]:::edge
        Envoy["Envoy L7 Global Proxy<br/>TLS Termination, Header Injection"]:::proxy
    end

    subgraph GoogleBackbone ["3. PRIVATE GLOBAL WAN (B4 Backbone)"]
        B4Fiber["B4 Fiber WAN Trunking<br/>Fast-Path Transit to Target Region"]:::gcp
    end

    subgraph HypervisorHost ["4. TARGET HOST HYPERVISOR (Andromeda SDN)"]
        Gateway["Subnet Virtual Gateway<br/>10.128.0.1"]:::gcp
        HPP["Andromeda Host Packet Processor<br/>1:1 NAT (34.120.50.10 -> 10.128.0.2)"]:::gcp
        Conntrack["Stateful Conntrack Table<br/>Log 5-Tuple Hash"]:::gcp
        Firewall{"Ingress Firewall Check:<br/>ALLOW tcp:80,443"}:::gcp
        Tap["Host Tap Interface (tap1234)"]:::gcp
    end

    subgraph TargetVM ["5. COMPUTE ENGINE INSTANCE (Guest OS)"]
        Virtio["Virtio-Net Ring Buffer"]:::vm
        Socket["Linux Kernel Socket"]:::vm
        App["Web Application Server<br/>NGINX / Node.js Process"]:::vm
    end

    User -->|DNS Query| DNS
    DNS -->|Returns VIP 34.120.50.10| User
    User -->|HTTPS Request| BGP
    BGP --> Maglev --> Armor --> Envoy
    Envoy -->|Decrypted HTTP/2 Frame| B4Fiber
    B4Fiber --> Gateway --> HPP
    HPP --> Conntrack --> Firewall
    Firewall -->|ALLOW| Tap
    Tap --> Virtio --> Socket --> App
```

### End-to-End Client Traversal Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor User as External Client / Browser
    participant DNS as Cloud DNS Anycast
    participant Edge as Google Edge / Maglev PoP
    participant Envoy as Envoy L7 Proxy & WAF
    participant B4 as B4 Global WAN Backbone
    participant HPP as Andromeda Host Packet Processor
    participant FW as Host Stateful Firewall Engine
    participant VM as Target Guest OS (mynet-us-vm)
    participant App as Web App (NGINX / Node.js)

    User->>DNS: Resolve myapp.company.com
    DNS-->>User: Return Public Anycast VIP (34.120.50.10)
    User->>Edge: TCP SYN to 34.120.50.10:443 (via BGP Anycast)
    Edge->>Envoy: Route packet to Maglev / Envoy Proxy Cluster
    Note over Envoy: TLS Handshake Completed<br/>Cloud Armor WAF Rules Evaluated (ALLOW)<br/>Inject X-Forwarded-For Header
    Envoy->>B4: Forward Decrypted Flow over B4 Backbone to us-central1
    B4->>HPP: Deliver Encapsulated Packet to Destination Host Hypervisor
    Note over HPP: Translate VIP 34.120.50.10 -> Private IP 10.128.0.2 (1:1 NAT)<br/>Lookup Conntrack Session Table
    HPP->>FW: Evaluate Ingress Firewall Rules
    FW-->>HPP: Rule Match: Priority 1000 ALLOW tcp:443
    HPP->>VM: Inject Ethernet Frame into Virtio Ring Buffer via Tap
    VM->>App: Deliver Socket Payload to Web Server Process
    Note over App: App processes request & generates HTTP 200 OK Response
    App->>VM: Write HTTP Response Payload to Socket
    VM->>HPP: Transmit Response Packet to Tap Interface
    Note over HPP: Matches Conntrack Session Tuple (Automatic Stateful Pass)<br/>Apply Egress NAT Translation
    HPP->>B4: Transmit Response Frame back over B4 Fiber
    B4->>Envoy: Deliver Response to Envoy Proxy
    Envoy-->>User: Send TLS Encrypted HTTP 200 OK to Client Browser
```

---

## Related Workspace Documents

- [Case Study Index](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/README.md)
- [High-Level Architecture (HLD)](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/hld-architecture.md)
- [Stateful Firewall Deep Dive](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/firewall-deep-dive.md)
- [Compute & Network Integration](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/compute-network-integration.md)
