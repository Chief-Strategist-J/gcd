# Unified Infrastructure Networking Manual: Docker, Kubernetes & Google Cloud VPC

A comprehensive, annotated manual connecting **Container Networking (Docker)**, **Pod/Service Orchestration (Kubernetes)**, and **Cloud Underlay Infrastructure (Google Cloud VPC, Subnets, Firewalls, Peering, NAT, Routes, VPN & DNS)**.

Every command is written in modular sections (`# ── SECTION ──`) with explicit breakdowns of valid parameters, allowed enums, IP address calculations, and production trade-offs.

---

## High-Level Networking Architecture: How the 3 Layers Connect

```mermaid
graph TB
    subgraph VPC["Layer 1: Google Cloud VPC Infrastructure (10.0.0.0/16)"]
        direction TB
        SubnetNode["Primary Subnet: Worker Nodes & VMs<br/>10.0.0.0/20 (4,096 IPs)"]
        SubnetPod["Secondary Subnet 1: GKE Pod CIDR<br/>10.4.0.0/14 (262,144 IPs)"]
        SubnetSvc["Secondary Subnet 2: GKE Services ClusterIP<br/>10.8.0.0/20 (4,096 VIPs)"]
        CloudNAT["Cloud NAT Gateway<br/>Outbound Internet Egress (No Public IPs)"]
        TransitRouter["Cloud Router / HA VPN / Interconnect<br/>BGP Transit to On-Prem / Other VPCs"]
        
        SubnetNode -. "Egress Web" .-> CloudNAT
        SubnetPod -. "Egress Web" .-> CloudNAT
        SubnetNode --- TransitRouter
    end

    subgraph K8S["Layer 2: Kubernetes CNI & Workload Orchestration"]
        direction TB
        NodeNIC["Node Interface: eth0 (e.g. 10.0.0.15)"]
        CNI["CNI Datapath v2 / Cilium (eBPF Engine)"]
        CoreDNS["CoreDNS / NodeLocal DNSCache (169.254.20.10)"]
        NetPol["NetworkPolicy Engine<br/>Layer 3/4 Micro-segmentation (Default Deny)"]
        SvcVIP["Service Virtual IP: ClusterIP (e.g. 10.8.0.100)"]
        PodNet["Pod Network Namespace (veth pair)<br/>IP: 10.4.1.24"]

        NodeNIC --- CNI
        CNI --- PodNet
        CNI --- SvcVIP
        PodNet --> NetPol
        PodNet -. "DNS UDP/53" .-> CoreDNS
    end

    subgraph DOCKER["Layer 3: Docker Container Engine (Host / Local)"]
        direction TB
        DockerBridge["docker0 / Custom Bridge (e.g. 172.28.0.0/16)"]
        DockerHost["Host Networking Mode (Direct eth0 Access)"]
        ContainerNS["Container Network Namespace (eth0 -> vethX)"]

        DockerBridge --- ContainerNS
        DockerHost --- ContainerNS
    end

    SubnetNode === NodeNIC
    SubnetPod === PodNet
    SubnetSvc === SvcVIP
    NodeNIC === DockerHost
```

---

## Detailed Traffic Flow Architecture

### 1. Ingress Flow: Public Internet → Cloud Armor WAF → Global Load Balancer → GKE Pod

```mermaid
sequenceDiagram
    autonumber
    actor Client as External User
    participant Armor as Cloud Armor WAF
    participant GLB as Global HTTPS Load Balancer
    participant Gateway as GKE Gateway API
    participant Firewall as VPC Firewall Rules
    participant NEG as Network Endpoint Group (NEG)
    participant NetPol as K8s NetworkPolicy
    participant Pod as Backend Pod (10.4.1.24)

    Client->>Armor: HTTPS Request (api.company.com:443)
    Note over Armor: Evaluates Rate Limits (100 req/min)<br/>Evaluates OWASP CRS (SQLi, XSS, RCE)
    alt Malicious Request / Threshold Exceeded
        Armor-->>Client: 403 Forbidden / 429 Too Many Requests
    else Legitimate Traffic
        Armor->>GLB: Forward Request
        GLB->>Gateway: Match Hostname & Path in HTTPRoute
        Note over GLB,Gateway: SSL Termination & URL Rewrite
        GLB->>Firewall: Health Check Probe & Forward
        Note over Firewall: Checks Health-Check CIDRs<br/>(35.191.0.0/16, 130.211.0.0/22)
        Firewall->>NEG: Direct Routing to Target Pod IP
        Note over NEG: Bypasses kube-proxy DNAT hop!<br/>Direct L4 delivery to Pod IP
        NEG->>NetPol: Ingress Micro-segmentation Check
        alt Blocked by NetworkPolicy
            NetPol--xPod: Packet Dropped (Default-Deny)
        else Permitted by Ingress Rule
            NetPol->>Pod: Deliver to TCP 8080 Socket
            Pod-->>Client: HTTP 200 OK (Preserved True Client IP)
        end
    end
```

### 2. Egress Flow: Pod → VPC Firewall → Cloud NAT / Cloud Router / Interconnect

```mermaid
graph TD
    Pod["Kubernetes Pod<br/>(IP: 10.4.1.24)"] --> NetPolCheck{"Kubernetes Egress<br/>NetworkPolicy?"}

    NetPolCheck -- Denied --> Drop1["Drop Packet at Pod veth (Default Deny)"]
    NetPolCheck -- Allowed --> eBPF["Linux Kernel eBPF Engine<br/>Transfers packet to Node eth0"]

    eBPF --> VPCFirewall{"VPC Egress Firewall<br/>Priority 1000 Allow vs 65534 Deny"}
    VPCFirewall -- Denied (Unauthorized Destination) --> Drop2["Drop Packet at Cloud Hypervisor"]
    VPCFirewall -- Allowed --> RouteTable{"VPC Route Table<br/>Destination CIDR Match"}

    RouteTable -- "0.0.0.0/0 (Public Internet)" --> CloudNAT["Cloud NAT Gateway<br/>Translates Pod IP to NAT Pool External IP"]
    CloudNAT --> Internet["External Public APIs / Mirrors"]

    RouteTable -- "192.168.0.0/16 (Corporate On-Prem)" --> CloudRouter["Cloud Router BGP Decision"]
    
    CloudRouter -- "Primary Link (MED Priority = 100)" --> Interconnect["Dedicated Cloud Interconnect<br/>10G/100G Fiber Link (Sub-millisecond)"]
    CloudRouter -- "Standby Failover (MED Priority = 300)" --> HAVPN["Cloud HA VPN<br/>IPsec Encapsulation (MSS Clamped: 1360)"]

    Interconnect --> OnPrem["On-Premises Enterprise Data Center"]
    HAVPN --> OnPrem
```

---

## Table of Contents

1. [Docker Container Networking (Local & Host Bridge)](#1-docker-container-networking-local--host-bridge)
2. [GCP VPC Network & Custom Subnet Provisioning](#2-gcp-vpc-network--custom-subnet-provisioning)
3. [Cloud NAT & Egress Internet Gateway](#3-cloud-nat--egress-internet-gateway)
4. [VPC Firewall Rules for Kubernetes Clusters & Microservices](#4-vpc-firewall-rules-for-kubernetes-clusters--microservices)
5. [Kubernetes CNI & Micro-Segmentation Network Policies](#5-kubernetes-cni--micro-segmentation-network-policies)
6. [Kubernetes Service Traffic & Cloud Load Balancer Wiring](#6-kubernetes-service-traffic--cloud-load-balancer-wiring)
7. [VPC Network Peering (Inter-VPC Interconnection)](#7-vpc-network-peering-inter-vpc-interconnection)
8. [VPC Custom Routes & Virtual Gateway Routing](#8-vpc-custom-routes--virtual-gateway-routing)
9. [Enterprise Cloud HA VPN (Hybrid & Cross-Cloud Transit)](#9-enterprise-cloud-ha-vpn-hybrid--cross-cloud-transit)
10. [Cloud DNS & Internal CoreDNS Resolution](#10-cloud-dns--internal-coredns-resolution)
11. [Multi-Zone High Availability (HA) & Workload Anti-Affinity](#11-multi-zone-high-availability-ha--workload-anti-affinity)
12. [Sub-Second BGP Failover via BFD & HA Routing Policies](#12-sub-second-bgp-failover-via-bfd--ha-routing-policies)
13. [Private Service Connect (PSC) — Zero-Egress Perimeter Security](#13-private-service-connect-psc--zero-egress-perimeter-security)
14. [Cloud Armor Enterprise WAF & Layer 7 DDoS Mitigation](#14-cloud-armor-enterprise-waf--layer-7-ddos-mitigation)
15. [Zero-Trust Service Mesh & Kernel-Level WireGuard Encryption](#15-zero-trust-service-mesh--kernel-level-wireguard-encryption)
16. [Multi-Cluster High Availability & Global Ingress](#16-multi-cluster-high-availability--global-ingress)
17. [Cloud Interconnect (Dedicated/Partner 10G/100G) & 99.99% VPN Failover](#17-cloud-interconnect-dedicatedpartner-10g100g--9999-vpn-failover)
18. [Network Connectivity Center (NCC) — Hub-and-Spoke Transit](#18-network-connectivity-center-ncc--hub-and-spoke-transit)
19. [Shared VPC Architecture (Host & Service Project Multi-Team Isolation)](#19-shared-vpc-architecture-host--service-project-multi-team-isolation)
20. [Workload Identity Federation for GKE (GCP IAM ↔ Kubernetes SA)](#20-workload-identity-federation-for-gke-gcp-iam--kubernetes-sa)
21. [VPC Service Controls (VPC-SC) — Data Exfiltration Perimeter Security](#21-vpc-service-controls-vpc-sc--data-exfiltration-perimeter-security)
22. [Organization Policy Constraints & Enterprise Network Guardrails](#22-organization-policy-constraints--enterprise-network-guardrails)
23. [Kubernetes Gateway API (Modern L7 Routing) vs Legacy Ingress](#23-kubernetes-gateway-api-modern-l7-routing-vs-legacy-ingress)
24. [NodeLocal DNSCache & Dynamic Secondary Range Capacity Planning](#24-nodelocal-dnscache--dynamic-secondary-range-capacity-planning)
25. [Pod Security Standards (PSA), Binary Authorization & Envelope Encryption](#25-pod-security-standards-psa-binary-authorization--envelope-encryption)
26. [Enterprise Scale Quotas, Limits Governance & MTU/MSS Clamping](#26-enterprise-scale-quotas-limits-governance--mtumss-clamping)
27. [Cross-Layer Diagnostic & Troubleshooting Playbook](#27-cross-layer-diagnostic--troubleshooting-playbook)

---

## 1. Docker Container Networking (Local & Host Bridge)

Controls standalone container isolation, user-defined subnets, DNS alias discovery, and port binding.

```mermaid
graph TB
    subgraph ContainerNS["Container Network Namespace (netns)"]
        AppProc["Application Process (e.g. Nginx:80)"]
        ContEth0["Container eth0<br/>(IP: 172.28.5.10)"]
        ContDNS["Embedded DNS Stub<br/>(127.0.0.11:53)"]
        AppProc --> ContEth0
        AppProc -. "Resolve api.internal.local" .-> ContDNS
    end

    subgraph HostRootNS["Host Root Network Namespace (Linux Kernel)"]
        direction TB
        VethHost["Peer Interface: veth-xyz<br/>(Attached to Host Bridge)"]
        Bridge["Linux Bridge: br-app-01<br/>(IP: 172.28.0.1 - Default Gateway)"]
        DockerDNS["Docker Embedded DNS Engine<br/>(Container Name ↔ IP Map)"]
        
        subgraph Netfilter["Linux iptables / Netfilter Engine"]
            NATPost["POSTROUTING MASQUERADE<br/>(SNAT: Replaces 172.28.5.10 with Host IP)"]
            NATPre["PREROUTING DNAT<br/>(Port Binding: Host 8080 → Container 80)"]
        end

        HostNIC["Host Physical Interface: eth0<br/>(IP: 10.0.0.15 - Connected to VPC)"]

        ContEth0 == "veth pair (Virtual Wire)" ==> VethHost
        VethHost === Bridge
        ContDNS -.-> DockerDNS
        Bridge --> NATPost --> HostNIC
        HostNIC --> NATPre --> Bridge
    end

    HostNIC -. "Egress to Cloud VPC" .-> VPC[("Google Cloud VPC Subnet")]

    style ContainerNS fill:#e1f5fe,stroke:#0288d1
    style HostRootNS fill:#fff3e0,stroke:#f57c00
    style Netfilter fill:#fbe9e7,stroke:#d84315
```

### A. Create Custom User-Defined Docker Network (`docker network create`)

```bash
docker network create my-application-net \
  # ── NETWORK DRIVER ───────────────────────────────────────────────────────
  --driver=bridge \
  # Enum: "bridge" | "host" | "overlay" | "macvlan" | "none"
  #   bridge: Standard software bridge on single host (recommended for microservices).
  #   host: Bypasses container network virtualization; uses host network directly (fastest).
  #   overlay: Multi-host networking across Swarm or daemon-to-daemon mesh.
  #   macvlan: Assigns physical MAC address directly to container (appears as physical host).
  #   none: Complete isolation; disables all networking interfaces inside container.

  # ── IP ADDRESS MANAGEMENT (IPAM) ─────────────────────────────────────────
  --subnet=172.28.0.0/16 \
  # CIDR string: Private IPv4 range allocated to this bridge.
  # Valid values: Standard RFC 1918 blocks (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16).
  --gateway=172.28.0.1 \
  # IPv4 address: Default gateway for containers in this network (must reside within --subnet).
  --ip-range=172.28.5.0/24 \
  # CIDR string: Sub-pool within --subnet from which Docker dynamically leases container IPs.

  # ── SCOPE & SECURITY OPTIONS ─────────────────────────────────────────────
  --internal=false \
  # Boolean: "true" | "false".
  # If true, restricts traffic strictly to internal containers; blocks external egress/NAT to internet.
  --attachable=true \
  # Boolean: "true" | "false". Allows standalone manual containers to attach dynamically.

  # ── MTU & NETWORK DRIVER OPTIONS ─────────────────────────────────────────
  --opt com.docker.network.bridge.name=br-app-01 \
  # String: Linux bridge interface name on host OS (visible in `ip link show`).
  --opt com.docker.network.driver.mtu=1460 \
  # Integer: Maximum Transmission Unit.
  # Critical rule: Must match cloud provider underlying network MTU (GCP uses 1460; AWS uses 9001/1500).

  # ── METADATA ─────────────────────────────────────────────────────────────
  --label="environment=production" \
  --label="owner=devops"
```

### B. Launch Container Attached to Custom Network (`docker run`)

```bash
docker run -d \
  # ── CONTAINER IDENTITY & NETWORK ATTACHMENT ──────────────────────────────
  --name=api-gateway \
  --network=my-application-net \
  # String: Name or ID of Docker network created above.
  --network-alias=api.internal.local \
  # String: Embedded Docker DNS record. Other containers on same network resolve this alias!
  --ip=172.28.5.10 \
  # Static IP string: Explicitly assigns static IP from network's --ip-range.

  # ── PORT BINDING & NAT PUBLISHING ────────────────────────────────────────
  --publish=127.0.0.1:8080:80/tcp \
  # Syntax: [HOST_IP:][HOST_PORT:]CONTAINER_PORT[/PROTOCOL]
  # Examples:
  #   "127.0.0.1:8080:80": Binds host loopback only (secure, requires reverse proxy).
  #   "0.0.0.0:8080:80": Binds on all host interfaces (accessible from VPC/internet).
  #   "8080:80/udp": Binds UDP traffic instead of default TCP.

  # ── DNS & HOST OVERRIDES ─────────────────────────────────────────────────
  --dns=10.0.0.10 \
  # IP string: Custom upstream DNS resolver (e.g., Cloud DNS or corporate internal resolver).
  --add-host=database.legacy.corp:10.0.1.50 \
  # Syntax: "HOSTNAME:IP". Appends custom entries directly into container's /etc/hosts.

  # ── IMAGE ────────────────────────────────────────────────────────────────
  nginx:1.26-alpine
```

### C. Connect Running Container to Second Network (`docker network connect`)

```bash
docker network connect \
  # ── NETWORK & TARGET CONTAINER ───────────────────────────────────────────
  database-backend-net \
  api-gateway \
  # Allows api-gateway to talk to both frontend and backend networks without bridge routing.
  --ip=172.29.0.15 \
  --alias=api-internal
```

---

## 2. GCP VPC Network & Custom Subnet Provisioning

Defines the global virtual network and regional subnets. In Kubernetes/GKE, subnets require **Primary CIDRs** (for worker nodes) and **Secondary CIDRs** (for Pods and ClusterIP Services).

### A. Create Custom Mode VPC Network (`gcloud compute networks create`)

```bash
gcloud compute networks create enterprise-production-vpc \
  # ── SUBNET MODE ──────────────────────────────────────────────────────────
  --subnet-mode=custom \
  # Enum: "custom" | "auto"
  #   custom: Production standard! Only explicitly created subnets exist.
  #   auto: Creates /20 subnets in every single GCP region (never use in enterprise production).

  # ── BGP ROUTING MODE ─────────────────────────────────────────────────────
  --bgp-routing-mode=global \
  # Enum: "regional" | "global"
  #   global: Cloud Routers dynamically share routes across ALL GCP regions worldwide.
  #   regional: Cloud Routers only propagate routes within their local GCP region.

  # ── MAXIMUM TRANSMISSION UNIT (MTU) ──────────────────────────────────────
  --mtu=1460 \
  # Integer: 1300, 1460, 1500, or 8896 (Jumbo Frames).
  #   1460: Standard GCP default.
  #   1500: Standard Ethernet (requires 1500 MTU on all connected nodes and peers).
  #   8896: Jumbo frames (only supported inside same VPC/subnet).

  # ── PROJECT IDENTIFIER ───────────────────────────────────────────────────
  --project=my-gcp-project-id
```

### B. Create Regional Subnet with GKE Secondary Ranges (`gcloud compute networks subnets create`)

```bash
gcloud compute networks subnets create gke-us-central1-subnet \
  # ── VPC & LOCATION ───────────────────────────────────────────────────────
  --network=enterprise-production-vpc \
  --region=us-central1 \
  # String: Target GCP region (e.g., us-central1, europe-west1, asia-east1).

  # ── PRIMARY CIDR (WORKER NODES & COMPUTE VMs) ────────────────────────────
  --range=10.0.0.0/20 \
  # CIDR: 4,096 total host IP addresses (10.0.0.1 to 10.0.15.254).
  # Used for GKE Worker Node OS interfaces (eth0) and standard Compute Engine VMs.

  # ── SECONDARY CIDRS (KUBERNETES PODS & SERVICES) ─────────────────────────
  --secondary-range=gke-pods-range=10.4.0.0/14 \
  # CIDR: 262,144 total Pod IPs (10.4.0.0 to 10.7.255.255).
  # GKE assigns a /24 (256 Pod IPs) to each worker node by default.
  # A /14 supports up to 1,024 GKE worker nodes!
  --secondary-range=gke-services-range=10.8.0.0/20 \
  # CIDR: 4,096 ClusterIP addresses (10.8.0.0 to 10.8.15.255).
  # Allocated strictly to Kubernetes internal Service Virtual IPs (ClusterIP).

  # ── PRIVATE GOOGLE ACCESS (PGA) ──────────────────────────────────────────
  --enable-private-ip-google-access \
  # Boolean flag: Enables VMs and Pods without public IPs to access Google APIs
  # (Cloud Storage, Artifact Registry, BigQuery) via private internal routing!

  # ── VPC FLOW LOGGING (SECURITY & TRAFFIC TELEMETRY) ──────────────────────
  --enable-flow-logs \
  --logging-aggregation-interval=interval-5-sec \
  # Enum: "interval-5-sec" | "interval-30-sec" | "interval-1-min" | "interval-5-min"
  --logging-flow-sampling=0.5 \
  # Float: 0.0 to 1.0 (0.5 = 50% packet sampling rate; 1.0 = 100% full capture).
  --logging-metadata=include-all \
  # Enum: "include-all" | "exclude-all" | "custom"
  # Captures pod name, pod namespace, source/destination instance names in Cloud Logging.
  --project=my-gcp-project-id
```

---

## 3. Cloud NAT & Egress Internet Gateway

Private Kubernetes worker nodes and containers must never have public external IP addresses. **Cloud NAT** grants outbound internet access (for downloading OS updates, pulling third-party packages, or calling external APIs) while blocking all unsolicited inbound connections.

### A. Create Cloud Router for NAT Engine (`gcloud compute routers create`)

```bash
gcloud compute routers create nat-router-us-central1 \
  # ── LOCATION & NETWORK ───────────────────────────────────────────────────
  --network=enterprise-production-vpc \
  --region=us-central1 \

  # ── BGP AUTONOMOUS SYSTEM NUMBER (ASN) ───────────────────────────────────
  --asn=65001 \
  # Integer: Private ASN for BGP routing.
  # Valid private ASN ranges: 64512 to 65534, or 4200000000 to 4294967294.
  --project=my-gcp-project-id
```

### B. Deploy Cloud NAT Gateway (`gcloud compute routers nats create`)

```bash
gcloud compute routers nats create production-cloud-nat \
  # ── ROUTER & LOCATION ────────────────────────────────────────────────────
  --router=nat-router-us-central1 \
  --region=us-central1 \

  # ── SUBNET IP MAPPING SPECIFICATION ──────────────────────────────────────
  --nat-all-subnet-ip-ranges \
  # Enum / Choice:
  #   --nat-all-subnet-ip-ranges: NATs Primary ranges AND all GKE Pod Secondary ranges!
  #   --nat-custom-subnet-ip-ranges: Selectively NATs specific subnets or secondary ranges.

  # ── EXTERNAL IP ALLOCATION (NAT EGRESS IPS) ──────────────────────────────
  --auto-allocate-nat-external-ips \
  # Alternatives:
  #   --auto-allocate-nat-external-ips: GCP automatically scales pool of static external IPs.
  #   --nat-external-ip-pool=IP_NAME1,IP_NAME2: Uses specific pre-reserved static IPs
  #   (critical when external vendors require IP-whitelisting!).

  # ── PORT ALLOCATION & CONNECTION TIMEOUTS ────────────────────────────────
  --min-ports-per-vm=64 \
  # Integer: Minimum TCP/UDP ports reserved per node/VM. Default: 64. Max: 65536.
  --max-ports-per-vm=512 \
  # Integer: Maximum ports VM can dynamically consume under high socket usage.
  --enable-dynamic-port-allocation \
  # Boolean: Automatically scales ports per node without dropping connections.
  --tcp-established-idle-timeout=1200s \
  # Duration: Timeout before killing established TCP connections (default: 1200s / 20m).
  --tcp-transitory-idle-timeout=30s \
  # Duration: Timeout for half-closed or SYN TCP connections (default: 30s).
  --udp-idle-timeout=30s
```

---

## 4. VPC Firewall Rules for Kubernetes Clusters & Microservices

Google Cloud VPC firewalls are **stateful** (return traffic is automatically permitted by the connection tracking engine). Default VPC posture blocks all inbound ingress.

```mermaid
graph TD
    InPacket["Inbound Packet Arrives at Host NIC"] --> Conntrack{"Existing Connection<br/>in State Table?"}

    Conntrack -- Yes (Return Traffic) --> AllowDirect["Allow Immediately<br/>(Bypasses Firewall Evaluation)"]
    Conntrack -- No (New Connection) --> OrgPolicy{"Hierarchical Org Firewall<br/>(Priority 0 - 999)"}

    OrgPolicy -- Denied --> DropOrg["Drop Packet (Org Level Policy)"]
    OrgPolicy -- Allowed / Delegate --> VPCFirewall{"VPC Network Firewalls<br/>(Evaluated by Priority: 0 → 65535)"}

    VPCFirewall --> RuleMatch{"Rule Attributes Match?<br/>- Source IP CIDR<br/>- Protocols & Ports<br/>- Target Tags / SAs"}

    RuleMatch -- No Match --> NextRule["Evaluate Next Priority Rule"]
    NextRule --> VPCFirewall

    RuleMatch -- Match: Action = DENY --> DropVPC["Drop Packet (Log to Cloud Logging)"]
    RuleMatch -- Match: Action = ALLOW --> AllowVPC["Create Connection in State Table"]

    AllowVPC --> K8sNode["Pass Packet to Node Kernel (iptables / eBPF)"]
    K8sNode --> NetPolCheck{"Kubernetes NetworkPolicy<br/>Ingress Evaluation"}
    NetPolCheck -- Allowed --> PodApp["Deliver to Container TCP Socket"]
    NetPolCheck -- Denied --> DropK8s["Drop Packet at Pod Interface"]

    style InPacket fill:#e3f2fd,stroke:#1565c0
    style AllowDirect fill:#e8f5e9,stroke:#2e7d32
    style DropVPC fill:#ffebee,stroke:#c62828
    style DropK8s fill:#ffebee,stroke:#c62828
    style PodApp fill:#e8f5e9,stroke:#2e7d32
```

### A. Allow Internal Cluster East-West Traffic (Node-to-Node & Pod-to-Pod)

```bash
gcloud compute firewall-rules create allow-gke-internal-all \
  # ── NETWORK SCOPE ────────────────────────────────────────────────────────
  --network=enterprise-production-vpc \
  --direction=INGRESS \
  # Enum: "INGRESS" (incoming) | "EGRESS" (outgoing). Default: INGRESS.
  --priority=1000 \
  # Integer: 0 to 65535 (lower numbers evaluated first). Default: 1000.

  # ── ACTION & PROTOCOLS ───────────────────────────────────────────────────
  --action=ALLOW \
  # Enum: "ALLOW" | "DENY"
  --rules=tcp:1-65535,udp:1-65535,icmp \
  # Syntax: PROTOCOL[:PORT[-PORT]]. E.g., "tcp:80,tcp:443", "udp", "icmp", "all".

  # ── SOURCE IP RANGES ─────────────────────────────────────────────────────
  --source-ranges=10.0.0.0/20,10.4.0.0/14,10.8.0.0/20 \
  # CIDR list: Allows traffic originating from Nodes (10.0.0.0/20),
  # Pods (10.4.0.0/14), and Services (10.8.0.0/20).

  # ── TARGET INSTANCE FILTER ───────────────────────────────────────────────
  --target-tags=gke-node,k8s-worker \
  # Tag list: Applies this firewall strictly to VMs matching these network tags.
  --description="Allow intra-cluster node, pod, and service communication"
```

### B. Allow Google Cloud Load Balancer Health Checks

```bash
gcloud compute firewall-rules create allow-gcp-health-checks \
  --network=enterprise-production-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:80,tcp:443,tcp:8080,tcp:10256 \
  # Port 10256 is default kube-proxy health check; other ports are app targets.
  --source-ranges=35.191.0.0/16,130.211.0.0/22 \
  # REQUIRED IP RANGES: These exact two CIDRs are hardcoded Google Cloud Load Balancers
  # and Google health check prober source IPs. NEVER block these ranges!
  --target-tags=gke-node,k8s-worker \
  --description="Allow Google Cloud LB health check probers to verify backend pod health"
```

### C. Allow Managed GKE Control Plane to Worker Node Webhooks

```bash
gcloud compute firewall-rules create allow-gke-master-to-nodes \
  --network=enterprise-production-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:443,tcp:10250,tcp:8443,tcp:9443 \
  # 443/8443/9443: Mutating/validating admission webhooks (Cert-Manager, Argo, Istio).
  # 10250: Kubelet API for `kubectl logs` and `kubectl exec`.
  # ── TARGET INSTANCE FILTER ───────────────────────────────────────────────
  --source-ranges=172.16.0.0/28 \
  # CIDR: Private Control Plane master CIDR block allocated during GKE cluster creation.
  --target-tags=gke-node

> [!NOTE]
> **Health Check CIDR Verification**: The source ranges `35.191.0.0/16` and `130.211.0.0/22` apply to Classic LBs and Global External Application LBs. Regional LBs and Envoy proxy-based load balancers may probe backends from dedicated regional proxy-only subnets (`--purpose=REGIONAL_MANAGED_PROXY`). Always verify active health check sources against current Google documentation.

### D. Default-Deny Outbound Egress & Granular Allowlisting

Production enterprise security requires shifting from open egress to **default-deny egress**, explicitly permitting only approved destinations.

```bash
# 1. Base Default-Deny Egress Firewall Rule
gcloud compute firewall-rules create deny-all-egress \
  --network=enterprise-production-vpc \
  --direction=EGRESS \
  --priority=65534 \
  # Lowest priority before GCP default allow (65535).
  --action=DENY \
  --rules=all \
  --destination-ranges=0.0.0.0/0 \
  --description="Block all outbound traffic by default across the entire VPC"

# 2. Granular Egress Rule: Allow Strictly Internal RFC 1918 & Cloud NAT Ports
gcloud compute firewall-rules create allow-egress-internal-and-web \
  --network=enterprise-production-vpc \
  --direction=EGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:80,tcp:443,udp:53,tcp:53 \
  # Permitted egress: Web outbound via Cloud NAT, and internal DNS queries.
  --destination-ranges=10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,0.0.0.0/0 \
  --target-tags=gke-node,k8s-worker
```

---

## 5. Kubernetes CNI & Micro-Segmentation Network Policies

Kubernetes NetworkPolicies enforce **Layer 3 and Layer 4 micro-segmentation** between Pods. By default, any Pod in a cluster can talk to any other Pod. Applying a NetworkPolicy shifts the Pod into **Default-Deny** mode.

```mermaid
graph TB
    subgraph Node1["Worker Node 1 (10.0.0.15)"]
        direction TB
        PodA["Pod A (Frontend)<br/>IP: 10.4.1.24"]
        VethA["veth pair interface"]
        PodA --> VethA

        subgraph DatapathOption["Cluster Service Routing Engine (VIP: 10.8.0.100)"]
            direction LR
            subgraph LegacyKubeProxy["Legacy kube-proxy (iptables)"]
                IPTablesChain["O(N) Sequential Chain Search<br/>PREROUTING → KUBE-SERVICES<br/>→ KUBE-SVC-XYZ → KUBE-SEP-XYZ<br/>(Heavy CPU overhead at scale)"]
            end
            subgraph ModernDatapath["GKE Datapath v2 / Cilium (eBPF)"]
                eBPFEngine["O(1) BPF Hash Map Lookup<br/>Direct Socket Layer Rewrite<br/>(Zero-copy, bypasses iptables)"]
            end
        end

        VethA --> DatapathOption
        DatapathOption --> NodeNIC1["Node 1 Physical NIC: eth0"]
    end

    NodeNIC1 == "Google Cloud VPC Fabric (10.0.0.0/16)" ==> NodeNIC2["Node 2 Physical NIC: eth0"]

    subgraph Node2["Worker Node 2 (10.0.0.16)"]
        direction TB
        NodeNIC2 --> CNI2["CNI eBPF Packet Filter"]
        CNI2 --> NetPolEngine{"Target NetworkPolicy<br/>Ingress Rule Engine"}
        NetPolEngine -- Allowed --> VethB["veth pair interface"]
        NetPolEngine -- Denied --> DropPacket["Drop Packet (Silence)"]
        VethB --> PodB["Pod B (Backend API)<br/>IP: 10.4.2.50:8080"]
    end

    style Node1 fill:#f3e5f5,stroke:#7b1fa2
    Node2 fill:#e8f5e9,stroke:#388e3c
    ModernDatapath fill:#e0f2f1,stroke:#00796b
    LegacyKubeProxy fill:#ffebee,stroke:#d32f2f
```

### A. Micro-Segmentation NetworkPolicy (Ingress & Egress)
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-backend-microservice
  namespace: production
spec:
  # ── TARGET POD SELECTION ─────────────────────────────────────────────────
  podSelector:
    matchLabels:
      app: backend-api
      tier: application
  # Policy applies strictly to pods possessing these exact labels.

  # ── POLICY ENFORCEMENT TYPES ─────────────────────────────────────────────
  policyTypes:
    - Ingress
    - Egress
  # Declares that both incoming and outgoing traffic will be filtered.

  # ── INGRESS RULES (WHO CAN TALK TO THIS POD) ─────────────────────────────
  ingress:
    - from:
        # Source Rule 1: Allow strictly Frontend pods within production namespace
        - podSelector:
            matchLabels:
              app: web-frontend
              tier: frontend
        # Source Rule 2: Allow monitoring system from dedicated telemetry namespace
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
        # Source Rule 3: Allow external corporate subnet CIDR block
        - ipBlock:
            cidr: 10.100.0.0/16
            except:
              - 10.100.5.0/24  # Blocks untrusted contractor subnet inside CIDR
      ports:
        - protocol: TCP
          port: 8080
        - protocol: TCP
          port: 9090  # Prometheus metrics port

  # ── EGRESS RULES (WHERE THIS POD CAN INITIATE TRAFFIC TO) ────────────────
  egress:
    # Egress Rule 1: Allow CoreDNS resolution (MANDATORY or DNS lookups fail!)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

    # Egress Rule 2: Allow reaching PostgreSQL database in database namespace
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: database
          podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
EOF
```

---

## 6. Kubernetes Service Traffic & Cloud Load Balancer Wiring

Binds Kubernetes Pods to cloud networking. Specifying annotations triggers Google Cloud to automatically provision **Internal Passthrough Network Load Balancers** or **External Application Load Balancers**.

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: internal-microservice-lb
  namespace: production
  annotations:
    # ── GOOGLE CLOUD LOAD BALANCER ANNOTATIONS ─────────────────────────────
    networking.gke.io/load-balancer-type: "Internal"
    # Enum: "Internal" | "External"
    #   "Internal": Provisions internal VPC Passthrough LB (no public internet access).
    #   "External": Provisions public-facing Google Cloud Load Balancer.
    networking.gke.io/internal-load-balancer-subnet: "gke-us-central1-subnet"
    # String: Subnet where the LoadBalancer private IP will be leased.
    networking.gke.io/internal-load-balancer-allow-global-access: "true"
    # Boolean: "true" | "false".
    # If true, allows clients in ALL GCP regions (and on-prem via VPN) to reach this internal LB!
spec:
  # ── TRAFFIC POLICY & SOURCE IP PRESERVATION ──────────────────────────────
  externalTrafficPolicy: Local
  # Enum: "Cluster" | "Local"
  #   Local: Bypasses kube-proxy SNAT; PRESERVES true client IP on packet!
  #          Only routes to pods residing on the receiving node (eliminates extra hop).
  #   Cluster: Balances across all nodes; client IP is masked by node IP (extra hop).

  type: LoadBalancer
  selector:
    app: backend-api
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
EOF
```

---

## 7. VPC Network Peering (Inter-VPC Interconnection)

Connects two separate VPC networks (e.g., `production-vpc` and `shared-services-vpc` or `database-vpc`) with zero gateway hops, ultra-low latency, and internal pricing.

```bash
# STEP 1: Request Peering from Production VPC to Shared Services VPC
gcloud compute networks peerings create prod-to-shared-peering \
  # ── SOURCE & TARGET VPC NETWORKS ─────────────────────────────────────────
  --network=enterprise-production-vpc \
  --peer-network=shared-services-vpc \
  --peer-project=shared-services-project-id \

  # ── CUSTOM ROUTE EXPORT & IMPORT (BGP TRANSIT) ───────────────────────────
  --export-custom-routes \
  # Boolean flag: Shares custom static and dynamic BGP routes from this VPC to peer.
  --import-custom-routes \
  # Boolean flag: Accepts custom routes exported by the peer network.

  # ── GKE POD CIDR SHARING (CRITICAL FOR K8S PEERING!) ─────────────────────
  --export-subnet-routes-with-public-ip \
  --import-subnet-routes-with-public-ip \
  # If GKE Pod secondary CIDRs use non-RFC1918 space (Class E / public IP ranges),
  # this flag MUST be enabled to exchange Pod routes across the peering link!
  --project=my-gcp-project-id

# STEP 2: Complete Peering from Shared Services VPC back to Production VPC
# (Peering is bilateral; connection remains INACTIVE until configured on both sides!)
gcloud compute networks peerings create shared-to-prod-peering \
  --network=shared-services-vpc \
  --peer-network=enterprise-production-vpc \
  --peer-project=my-gcp-project-id \
  --export-custom-routes \
  --import-custom-routes \
  --project=shared-services-project-id
```

> [!CAUTION]
> **VPC Peering Non-Transitivity Landmine**: VPC Peering is strictly **non-transitive**. If `VPC-A` peers with `VPC-B`, and `VPC-B` peers with `VPC-C`, workloads in `VPC-A` **CANNOT** reach `VPC-C` through `VPC-B`! Google Cloud drops transit packets at the hypervisor. For transit hub topologies connecting dozens of VPCs and on-premises sites without full pairwise mesh limits, deploy **Network Connectivity Center (NCC)** (see Section 19).

```mermaid
graph TD
    subgraph NonTransitive["The VPC Peering Non-Transitivity Landmine"]
        VPCA["VPC-A<br/>(10.1.0.0/16)"] <== "Direct Peering A ↔ B (Allowed)" ==> VPCB["VPC-B (Transit Attempt)<br/>(10.2.0.0/16)"]
        VPCB <== "Direct Peering B ↔ C (Allowed)" ==> VPCC["VPC-C<br/>(10.3.0.0/16)"]
        VPCA -. "x Transit Traffic Blocked by Hypervisor! x<br/>(A cannot reach C through B)" .-x VPCC
    end

    subgraph NCCTransit["The Network Connectivity Center (NCC) Solution"]
        Hub["NCC Global Transit Hub<br/>(Centralized Routing Fabric)"]
        SpokeA["Spoke: VPC-A"] === Hub
        SpokeB["Spoke: VPC-B"] === Hub
        SpokeC["Spoke: VPC-C"] === Hub
        HybridSpoke["Spoke: Hybrid WAN / Interconnect"] === Hub
        
        SpokeA <== "Full Any-to-Any Transit Enabled" ==> SpokeC
    end

    style NonTransitive fill:#ffebee,stroke:#c62828
    style NCCTransit fill:#e8f5e9,stroke:#2e7d32
```

---

## 8. VPC Custom Routes & Virtual Gateway Routing

Routes determine where packets destined for specific IP CIDRs are forwarded (e.g. Next-Hop VM, Next-Hop VPN Tunnel, or Next-Hop Internal Load Balancer).

```bash
gcloud compute routes create route-to-onprem-datacenter \
  # ── NETWORK SCOPE ────────────────────────────────────────────────────────
  --network=enterprise-production-vpc \

  # ── DESTINATION CIDR BLOCK ───────────────────────────────────────────────
  --destination-range=192.168.0.0/16 \
  # CIDR string: Destination IP block matching on-prem enterprise datacenter.

  # ── PRIORITY & PRECEDENCE ────────────────────────────────────────────────
  --priority=100 \
  # Integer: 0 to 65535 (lower values take precedence). Default: 1000.

  # ── NEXT-HOP DESTINATION SELECTION (MUTUALLY EXCLUSIVE) ───────────────────
  --next-hop-vpn-tunnel=us-central1-vpn-tunnel-1 \
  --next-hop-vpn-tunnel-region=us-central1 \
  # Options:
  #   --next-hop-gateway=default-internet-gateway (default route 0.0.0.0/0)
  #   --next-hop-instance=firewall-vm-instance --next-hop-instance-zone=us-central1-a
  #   --next-hop-ilb=internal-firewall-lb-ip
  #   --next-hop-vpn-tunnel=vpn-tunnel-name

  # ── TAG FILTERING (POLICY-BASED ROUTING) ──────────────────────────────────
  --tags=needs-onprem-access \
  # If set, route ONLY applies to VMs possessing this network tag.
  --description="Route corporate datacenter traffic over encrypted VPN tunnel" \
  --project=my-gcp-project-id
```

---

## 9. Enterprise Cloud HA VPN (Hybrid & Cross-Cloud Transit)

High Availability (HA) VPN delivers a **99.99% service availability SLA** using two active IPsec tunnels over BGP routing.

### A. Create Cloud HA VPN Gateway (`gcloud compute vpn-gateways create`)

```bash
gcloud compute vpn-gateways create corp-ha-vpn-gw \
  --network=enterprise-production-vpc \
  --region=us-central1 \
  --project=my-gcp-project-id
# Automatically allocates TWO public IPv4 interface IPs (interface 0 and interface 1).
```

### B. Define External Peer Gateway (On-Premises Cisco / AWS / Azure VPN)

```bash
gcloud compute external-vpn-gateways create onprem-cisco-gw \
  # ── PEER REDUNDANCY TYPE ─────────────────────────────────────────────────
  --interfaces=0=203.0.113.1,1=203.0.113.2 \
  # Syntax: 0=IP,1=IP.
  # Represents physical WAN public IP addresses of remote customer gateway.
  --project=my-gcp-project-id
```

### C. Create Encrypted IPsec Tunnel 0 with BGP Session

```bash
# 1. Provision VPN Tunnel 0
gcloud compute vpn-tunnels create vpn-tunnel-0 \
  --region=us-central1 \
  --vpn-gateway=corp-ha-vpn-gw \
  --interface=0 \
  # Integer: 0 or 1. Associates tunnel to local HA VPN gateway interface.
  --peer-external-gateway=onprem-cisco-gw \
  --peer-external-gateway-interface=0 \
  --shared-secret='MyComplexSharedKeySecret123!' \
  # String: Pre-shared IKE key (must match remote router secret exactly).
  --ike-version=2 \
  # Enum: 1 | 2. Production standard: IKEv2.
  --router=nat-router-us-central1 \
  # Cloud Router managing BGP peering for this tunnel.
  --project=my-gcp-project-id

# 2. Add Virtual BGP Interface to Cloud Router
gcloud compute routers add-interface nat-router-us-central1 \
  --region=us-central1 \
  --interface-name=bgp-if-0 \
  --ip-address=169.254.0.1 \
  # RFC 3927 Link-Local IP: Local BGP endpoint. Subnet mask must be /30.
  --mask-length=30 \
  --vpn-tunnel=vpn-tunnel-0 \
  --project=my-gcp-project-id

# 3. Establish BGP Peer Session
gcloud compute routers add-bgp-peer nat-router-us-central1 \
  --region=us-central1 \
  --peer-name=onprem-bgp-peer-0 \
  --interface=bgp-if-0 \
  --peer-ip-address=169.254.0.2 \
  # Link-Local IP of remote on-premises BGP neighbor.
  --peer-asn=65002 \
  # Remote Autonomous System Number (ASN) of corporate router.
  --advertised-route-priority=100 \
  --project=my-gcp-project-id
```

---

## 10. Cloud DNS & Internal CoreDNS Resolution

Enables split-horizon DNS where Kubernetes Pods, VMs, and on-premises infrastructure resolve each other via private zones.

### A. Create Private Cloud DNS Zone Bound to VPC

```bash
gcloud dns managed-zones create corp-internal-zone \
  # ── DNS DOMAIN NAME & DESCRIPTION ────────────────────────────────────────
  --dns-name=internal.corp. \
  # DNS root format: MUST end with trailing dot "." (e.g. "internal.corp.").
  --description="Private corporate internal DNS zone" \

  # ── VISIBILITY & VPC BINDING ─────────────────────────────────────────────
  --visibility=private \
  # Enum: "public" | "private"
  #   private: Records only resolvable within authorized VPC networks.
  #   public: Globally queryable on public internet.
  --networks=enterprise-production-vpc,shared-services-vpc \
  # Comma-delimited list: All VPC networks permitted to resolve this zone!
  --project=my-gcp-project-id
```

### B. Register DNS Record Sets (`gcloud dns record-sets transaction`)

```bash
# 1. Initialize Record Transaction
gcloud dns record-sets transaction start --zone=corp-internal-zone

# 2. Append A Record (Mapping hostname to internal IP)
gcloud dns record-sets transaction add 10.0.1.50 \
  --name=database.internal.corp. \
  --ttl=300 \
  # Integer: Time-To-Live in seconds (e.g., 60, 300, 3600).
  --type=A \
  # Enum: "A" | "AAAA" | "CNAME" | "MX" | "TXT" | "SRV" | "PTR"
  --zone=corp-internal-zone

# 3. Commit Transaction to DNS Servers
gcloud dns record-sets transaction execute --zone=corp-internal-zone
```

---

## 11. Multi-Zone High Availability (HA) & Workload Anti-Affinity

Guarantees 99.99% workload availability by spreading compute nodes and Pod replicas across distinct geographical failure domains (zones), protected by admission disruption budgets.

### A. Regional Private GKE Cluster Creation (`gcloud container clusters create`)

```bash
gcloud container clusters create enterprise-ha-cluster \
  # ── REGIONAL CONTROL PLANE & WORKER TOPOLOGY ─────────────────────────────
  --region=us-central1 \
  # Region string: Creates multi-zone replicated master control planes across 3 zones!
  # (Zonal clusters fail if one zone goes down; Regional clusters survive zone loss).
  --node-locations=us-central1-a,us-central1-b,us-central1-c \
  # Comma-delimited zones: Worker node pools are distributed equally across all 3 zones.
  --num-nodes=2 \
  # Integer: Nodes per zone (2 nodes * 3 zones = 6 total worker nodes).

  # ── VPC SUBNET & IP ALIAS INTEGRATION ────────────────────────────────────
  --network=enterprise-production-vpc \
  --subnetwork=gke-us-central1-subnet \
  --enable-ip-alias \
  # Boolean flag: Enables VPC-native routing (Pods receive native VPC secondary IPs).
  --cluster-secondary-range-name=gke-pods-range \
  --services-secondary-range-name=gke-services-range \

  # ── ZERO-TRUST PRIVATE CONTROL PLANE & NODES ─────────────────────────────
  --enable-private-nodes \
  # Boolean: Worker nodes receive ZERO public external IPs; accessible only internally.
  --enable-private-endpoint \
  # Boolean: Disables public internet access to the Kubernetes API server!
  --master-ipv4-cidr=172.16.0.0/28 \
  # CIDR: Dedicated /28 block (16 IPs) for Google-managed Kubernetes master VMs.
  --enable-master-global-access \
  # Boolean: Allows reaching master endpoint from any GCP region or on-prem via VPN.

  # ── HIGH-PERFORMANCE eBPF DATAPATH & SECURITY ────────────────────────────
  --enable-dataplane-v2 \
  # Cilium eBPF datapath: Replaces slow iptables with direct kernel eBPF packet routing!
  --shielded-secure-boot \
  --shielded-integrity-monitoring \
  --project=my-gcp-project-id
```

### B. Kubernetes Multi-Zone Topology Spread Constraints & Anti-Affinity

Spreads application replicas across zones so that losing an entire Google Cloud data center does not impact availability.

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resilient-backend-api
  namespace: production
spec:
  replicas: 6
  selector:
    matchLabels:
      app: resilient-backend
  template:
    metadata:
      labels:
        app: resilient-backend
    spec:
      # ── TOPOLOGY SPREAD CONSTRAINTS (ZONE & NODE BALANCING) ──────────────
      topologySpreadConstraints:
        # Constraint 1: Strict distribution across availability zones
        - maxSkew: 1
          # Max difference in replica count allowed between any two zones.
          topologyKey: topology.kubernetes.io/zone
          # Standard k8s node label representing GCP zone (e.g. us-central1-a).
          whenUnsatisfiable: DoNotSchedule
          # Enum: "DoNotSchedule" (hard constraint) | "ScheduleAnyway" (soft/best-effort).
          labelSelector:
            matchLabels:
              app: resilient-backend

        # Constraint 2: Anti-colocation on the exact same physical node
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: resilient-backend

      containers:
        - name: api
          image: us-central1-docker.pkg.dev/company-gcp-prod/apps/api:v1.2.0
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
---
# ── POD DISRUPTION BUDGET (PROTECTING RUNNING REPLICAS DURING MAINTENANCE) ─
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-ha-pdb
  namespace: production
spec:
  minAvailable: 4
  # Integer or Percentage (e.g. "75%"): Guarantees at least 4 pods are online
  # during voluntary disruptions (node drains, cluster upgrades, auto-repairs).
  selector:
    matchLabels:
      app: resilient-backend
EOF
```

---

## 12. Sub-Second BGP Failover via BFD & HA Routing Policies

Standard BGP detects peer failure through keepalive timers (typically taking 60 to 180 seconds). **Bidirectional Forwarding Detection (BFD)** provides hardware-assisted microsecond health checks, triggering failover routes in **under 900 milliseconds**.

### A. Enable BFD on Cloud Router BGP Peer (`gcloud compute routers update-bgp-peer`)

```bash
gcloud compute routers update-bgp-peer nat-router-us-central1 \
  # ── ROUTER & LOCATION ────────────────────────────────────────────────────
  --region=us-central1 \
  --peer-name=onprem-bgp-peer-0 \

  # ── BFD PROBING TIMERS & MULTIPLIER ──────────────────────────────────────
  --bfd-min-receive-interval=300 \
  # Integer: Milliseconds between receiving BFD control packets. Min: 300ms.
  --bfd-min-transmit-interval=300 \
  # Integer: Milliseconds between transmitting BFD control packets. Min: 300ms.
  --bfd-multiplier=3 \
  # Integer: Number of consecutive missed packets before declaring link dead.
  # Failover calculation: 300ms * 3 = 900ms link down detection!
  --bfd-session-initialization-mode=ACTIVE \
  # Enum: "ACTIVE" | "PASSIVE" | "DISABLED"
  #   ACTIVE: Cloud Router actively negotiates BFD session with on-prem peer.
  #   PASSIVE: Waits for peer router to initiate.
  --project=my-gcp-project-id
```

### B. Deterministic Active-Passive HA Routing (BGP AS-Path Prepending & MED)

Controls which link receives production traffic by advertising custom BGP priorities (Multi-Exit Discriminator - MED).

```bash
# Set PRIMARY VPN Tunnel (Lowest MED Priority = Active)
gcloud compute routers update-bgp-peer nat-router-us-central1 \
  --region=us-central1 \
  --peer-name=onprem-bgp-peer-primary \
  --advertised-route-priority=100 \
  # Integer: Lower number = higher BGP preference.
  --project=my-gcp-project-id

# Set SECONDARY VPN Tunnel (Higher MED Priority = Standby Backup)
gcloud compute routers update-bgp-peer nat-router-us-central1 \
  --region=us-central1 \
  --peer-name=onprem-bgp-peer-secondary \
  --advertised-route-priority=300 \
  # If primary fails, traffic fails over to this secondary peer in < 1 second.
  --project=my-gcp-project-id
```

---

## 13. Private Service Connect (PSC) — Zero-Egress Perimeter Security

Replaces VPC Peering with a **unidirectional private endpoint**. Prevents IP CIDR overlapping collisions, eliminates shared routing table risks, and isolates services completely behind a Layer 4 proxy.

```mermaid
graph LR
    subgraph ConsumerVPC["Consumer VPC (10.200.0.0/16)"]
        ConsumerApp["Client Container / Pod"]
        ConsumerEndpoint["PSC Endpoint (Forwarding Rule)<br/>Local Subnet IP: 10.200.1.50"]
        ConsumerApp -->|Connects to local IP| ConsumerEndpoint
    end

    ConsumerEndpoint == "Google Andromeda SDN (Layer 4 Proxy)<br/>No VPC Peering • Zero Shared Routes" ==> PSCAttachment

    subgraph ProducerVPC["Producer Enterprise VPC (10.0.0.0/16)"]
        direction TB
        PSCAttachment["Service Attachment<br/>psc-backend-service-attachment"]
        PSCNATSubnet["Dedicated PSC NAT Subnet<br/>(10.90.0.0/24 - Translates Consumer IP)"]
        ProducerILB["Internal Passthrough Load Balancer<br/>(VIP: 10.0.5.100)"]
        ProducerPods["Target GKE Pod Replicas<br/>(10.4.1.0/24)"]

        PSCAttachment --> PSCNATSubnet
        PSCNATSubnet --> ProducerILB
        ProducerILB --> ProducerPods
    end

    style ConsumerVPC fill:#e3f2fd,stroke:#1565c0
    style ProducerVPC fill:#e8f5e9,stroke:#2e7d32
    style PSCAttachment fill:#fff9c4,stroke:#fbc02d
```

### A. Producer Side: Publish Service via Service Attachment

```bash
# 1. Create Dedicated NAT Subnet for Private Service Connect
gcloud compute networks subnets create psc-producer-nat-subnet \
  --network=enterprise-production-vpc \
  --region=us-central1 \
  --range=10.90.0.0/24 \
  --purpose=PRIVATE_SERVICE_CONNECT \
  # Enum: Mandatory purpose flag for PSC translation.
  --project=my-gcp-project-id

# 2. Expose Internal Load Balancer as a Published Service Attachment
gcloud compute service-attachments create psc-backend-service-attachment \
  --region=us-central1 \
  --producer-forwarding-rule=internal-microservice-lb-forwarding-rule \
  # Name of internal passthrough or proxy LB forwarding rule to expose.
  --connection-preference=ACCEPT_AUTOMATIC \
  # Enum: "ACCEPT_AUTOMATIC" | "ACCEPT_MANUAL"
  #   ACCEPT_MANUAL requires approving every consumer project ID explicitly.
  --nat-subnets=psc-producer-nat-subnet \
  --description="Private Service Connect attachment for backend API" \
  --project=my-gcp-project-id
```

### B. Consumer Side: Create Private Endpoint Inside Consumer VPC

```bash
# 1. Reserve Static Internal IP in Consumer Subnet
gcloud compute addresses create psc-consumer-ip \
  --region=us-central1 \
  --subnet=consumer-subnet \
  --addresses=10.200.1.50 \
  --project=consumer-project-id

# 2. Bind Consumer Forwarding Rule to Producer Attachment URI
gcloud compute forwarding-rules create psc-consumer-endpoint \
  --region=us-central1 \
  --network=consumer-vpc \
  --address=psc-consumer-ip \
  --target-service-attachment=projects/my-gcp-project-id/regions/us-central1/serviceAttachments/psc-backend-service-attachment \
  # URI: Canonical resource path to producer's Service Attachment.
  --project=consumer-project-id
# Now any container/pod in consumer-vpc reaches backend API simply by calling 10.200.1.50!
```

---

## 14. Cloud Armor Enterprise WAF & Layer 7 DDoS Mitigation

Protects container workloads and Ingress endpoints from Layer 7 attacks, HTTP floods, and OWASP Top 10 exploits directly at Google's global edge infrastructure.

### A. Provision Cloud Armor Security Policy (`gcloud compute security-policies create`)

```bash
gcloud compute security-policies create edge-waf-security-policy \
  --description="Enterprise Layer 7 WAF policy with OWASP rules and rate limiting" \
  --type=CLOUD_ARMOR \
  --project=my-gcp-project-id
```

### B. Enforce Rate Limiting (Prevent Brute Force & HTTP Flood Attacks)

```bash
gcloud compute security-policies rules create 1000 \
  # ── RULE PRIORITY & POLICY BINDING ───────────────────────────────────────
  --security-policy=edge-waf-security-policy \
  --priority=1000 \
  # Integer: 0 to 2147483647 (evaluated lowest to highest).

  # ── RATE LIMITING ALGORITHM ──────────────────────────────────────────────
  --action=rate-based-ban \
  # Enum: "rate-based-ban" | "throttle"
  --rate-limit-threshold-count=100 \
  # Integer: Max allowed requests per client within the time interval.
  --rate-limit-threshold-interval-sec=60 \
  # Integer: 10, 20, 30, 60, 120, 180, 240, 300 seconds.
  --ban-duration-sec=600 \
  # Integer: Seconds client IP is banned once threshold is exceeded (600s = 10 minutes).
  --conform-action=allow \
  --exceed-action=deny-429 \
  # Enum: "deny-403" | "deny-404" | "deny-429" | "deny-502"
  --enforce-on-key=IP \
  # Enum: "IP" | "ALL" | "HTTP-HEADER" | "X-FORWARDED-FOR-IP"
  --project=my-gcp-project-id
```

### C. Enable Preconfigured OWASP Top 10 Core Rules (SQLi, XSS, RCE, LFI)

```bash
gcloud compute security-policies rules create 2000 \
  --security-policy=edge-waf-security-policy \
  --priority=2000 \
  --action=deny-403 \
  # Expression syntax: Evaluates Google-maintained ModSecurity Core Rule Sets (CRS).
  --expression="evaluatePreconfiguredExpr('sqli-v33-stable') || evaluatePreconfiguredExpr('xss-v33-stable') || evaluatePreconfiguredExpr('rce-v33-stable') || evaluatePreconfiguredExpr('lfi-v33-stable')" \
  --description="Block SQL Injection, Cross-Site Scripting, Remote Code Execution & Local File Inclusion" \
  --project=my-gcp-project-id
```

### D. Bind Cloud Armor Policy to Kubernetes Ingress via `BackendConfig` CRD

```bash
kubectl apply -f - <<'EOF'
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: api-backend-security-config
  namespace: production
spec:
  # ── CLOUD ARMOR INTEGRATION ──────────────────────────────────────────────
  securityPolicy:
    name: "edge-waf-security-policy"
  # ── CLOUD CDN CACHE SETTINGS ─────────────────────────────────────────────
  cdn:
    enabled: true
    cachePolicy:
      includeHost: true
      includeProtocol: true
      includeQueryString: false
  # ── CONNECTION DRAINING (ZERO-DROPPED CONNECTIONS ON POD TERMINATION) ────
  timeoutSec: 30
  connectionDraining:
    drainingTimeoutSec: 60
---
# Annotate Kubernetes Service to attach BackendConfig:
apiVersion: v1
kind: Service
metadata:
  name: backend-api-service
  namespace: production
  annotations:
    beta.cloud.google.com/backend-config: '{"default": "api-backend-security-config"}'
spec:
  type: NodePort
  selector:
    app: resilient-backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
EOF
```

---

## 15. Zero-Trust Service Mesh & Kernel-Level WireGuard Encryption

Enforces end-to-end cryptographic confidentiality and cryptographic workload identities across the container mesh.

```mermaid
graph LR
    subgraph SrcNode["Source Node (10.0.0.15)"]
        direction TB
        App1["Frontend Pod"] -->|Plaintext HTTP| Envoy1["Envoy Sidecar Proxy<br/>(SPIFFE x509 Identity)"]
        Envoy1 -->|Layer 7: TLS Encrypted| Kernel1["Linux Kernel Datapath v2<br/>(WireGuard wg0 Device)"]
        Kernel1 -->|Layer 3: ChaCha20-Poly1305 Encrypted| Eth0_1["Node Physical eth0"]
    end

    Eth0_1 == "Google Cloud VPC Network<br/>(Opaque UDP/51820 WireGuard Tunnel)" ==> Eth0_2

    subgraph DstNode["Destination Node (10.0.0.16)"]
        direction TB
        Eth0_2["Node Physical eth0"] -->|Kernel WireGuard Decrypt| Kernel2["Linux Kernel Datapath v2<br/>(Validates Cryptokey Routing)"]
        Kernel2 -->|Layer 7 TLS Payload| Envoy2["Envoy Sidecar Proxy<br/>(Validates Client Cert & RBAC)"]
        Envoy2 -->|Plaintext HTTP (Socket)| App2["Backend API Pod"]
    end

    style SrcNode fill:#e8eaf6,stroke:#3f51b5
    style DstNode fill:#e8f5e9,stroke:#2e7d32
    style Envoy1 fill:#fff3e0,stroke:#e65100
    style Envoy2 fill:#fff3e0,stroke:#e65100
    style Kernel1 fill:#ede7f6,stroke:#512da8
    style Kernel2 fill:#ede7f6,stroke:#512da8
```

### A. Kernel-Level Transparent WireGuard Encryption (eBPF / Cilium Datapath v2)

All node-to-node and pod-to-pod packet payloads across the VPC are encrypted in transit with ChaCha20-Poly1305 at the Linux kernel layer:

```bash
# Enable WireGuard encryption transparently on GKE Datapath v2
gcloud container clusters update enterprise-ha-cluster \
  --region=us-central1 \
  --enable-dataplane-v2-encryption \
  --project=my-gcp-project-id
# Zero sidecar proxies needed! Packets leaving worker node eth0 are automatically encrypted.
```

### B. Strict Mutual TLS (mTLS) Mesh Enforcement (`PeerAuthentication`)

Ensures that any plaintext connection between microservices is immediately dropped at the socket layer.

```bash
kubectl apply -f - <<'EOF'
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default-strict-mtls
  namespace: production
spec:
  # ── MUTUAL TLS MODE ──────────────────────────────────────────────────────
  mtls:
    mode: STRICT
    # Enum: "STRICT" | "PERMISSIVE" | "DISABLE"
    #   STRICT: Workload accepts ONLY TLS connections with cryptographically valid SPIFFE x509 certs.
    #   PERMISSIVE: Accepts both plaintext and mTLS (use only during migration phases).
    #   DISABLE: Disables mTLS.
---
# ── ZERO-TRUST AUTHORIZATION POLICY (LEAST PRIVILEGE) ────────────────────
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: billing-api-access-control
  namespace: production
spec:
  selector:
    matchLabels:
      app: billing-api
  action: ALLOW
  rules:
    # Explicitly allow ONLY frontend service account to call GET /invoices
    - from:
        - source:
            principals: ["cluster.local/ns/production/sa/frontend-service-account"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/v1/invoices/*"]
EOF
```

---

## 16. Multi-Cluster High Availability & Global Ingress

For multi-region disaster recovery, Multi-Cluster Ingress (MCI) and Multi-Cluster Services (MCS) route incoming user traffic across active clusters in `us-central1` and `europe-west1` via a single global Anycast VIP.

```mermaid
graph TB
    User["Global User Client"] --> AnycastVIP["Google Anycast Global VIP<br/>(34.120.50.10)"]
    
    AnycastVIP --> GLB["Google Cloud Global HTTPS Load Balancer<br/>MultiClusterIngress (MCI)"]

    subgraph REGION1["Region 1: us-central1"]
        MCI1["MultiClusterService (MCS)"]
        GKE1["GKE Cluster 1 (Active)"]
        Pods1["Pod Replicas (us-central1)"]
        MCI1 --> GKE1 --> Pods1
    end

    subgraph REGION2["Region 2: europe-west1"]
        MCI2["MultiClusterService (MCS)"]
        GKE2["GKE Cluster 2 (Active / DR)"]
        Pods2["Pod Replicas (europe-west1)"]
        MCI2 --> GKE2 --> Pods2
    end

    GLB -- "Primary Latency-Based Routing" --> MCI1
    GLB -- "Automatic Cross-Region Failover" --> MCI2
```

### A. Deploy MultiClusterService (MCS) Across All Member Clusters

```bash
kubectl apply -f - <<'EOF'
apiVersion: net.gke.io/v1
kind: MultiClusterService
metadata:
  name: global-resilient-service
  namespace: production
spec:
  template:
    spec:
      selector:
        app: resilient-backend
      ports:
        - name: http
          protocol: TCP
          port: 80
          targetPort: 8080
---
# ── MULTI-CLUSTER INGRESS (GLOBAL ANYCAST VIP WITH AUTOMATIC FAILOVER) ────
apiVersion: networking.gke.io/v1
kind: MultiClusterIngress
metadata:
  name: global-application-mci
  namespace: production
  annotations:
    networking.gke.io/static-ip: "34.120.50.10"
    # Pre-reserved Google Global Anycast static IP address.
spec:
  template:
    spec:
      backend:
        serviceName: global-resilient-service
        servicePort: 80
EOF
```

---

## 17. Cloud Interconnect (Dedicated/Partner 10G/100G) & 99.99% VPN Failover

When enterprise workloads require guaranteed private bandwidth (10 Gbps or 100 Gbps circuits), predictable single-digit millisecond latency, and physical isolation, **Cloud Interconnect** replaces IPsec over public internet. Combining Interconnect with Cloud HA VPN delivers an active-passive **99.99% availability topology**.

```mermaid
graph TB
    subgraph ONPREM["On-Premises Corporate Data Center"]
        OnPremRouter["Edge Core Router (BGP ASN 65002)"]
    end

    subgraph GCP["Google Cloud Platform (VPC 10.0.0.0/16)"]
        direction TB
        CloudRouter["Cloud Router (BGP ASN 65001)<br/>BFD Sub-Second Failure Detection (900ms)"]
        
        subgraph WORKLOADS["Production GKE Infrastructure"]
            GKENodes["GKE Worker Nodes (10.0.0.0/20)"]
            GKEPods["Kubernetes Pods (10.4.0.0/14)"]
        end
    end

    OnPremRouter == "Primary: 10G/100G Dedicated Interconnect<br/>BGP Advertised Priority = 100 (Active Traffic)" ==> CloudRouter
    OnPremRouter -. "Standby: Cloud HA VPN (IPsec IKEv2)<br/>BGP Advertised Priority = 300 (Hot Standby)" .-> CloudRouter

    CloudRouter === GKENodes
    GKENodes === GKEPods

    style OnPremRouter fill:#f9f,stroke:#333,stroke-width:2px
    style CloudRouter fill:#bbf,stroke:#333,stroke-width:2px
```

### A. Create Interconnect VLAN Attachment (`gcloud compute interconnects attachments dedicated create`)

```bash
gcloud compute interconnects attachments dedicated create prod-vlan-attachment-zone1 \
  # ── REGION & ROUTER ATTACHMENT ───────────────────────────────────────────
  --region=us-central1 \
  --router=nat-router-us-central1 \
  # BGP Cloud Router managing route advertisements for this circuit.

  # ── PHYSICAL INTERCONNECT CIRCUIT & AVAILABILITY DOMAIN ──────────────────
  --interconnect=dedicated-cross-connect-1 \
  # String: Name of the physical 10G/100G port pre-ordered in colocation facility.
  --edge-availability-domain=AVAILABILITY_DOMAIN_1 \
  # Enum: "AVAILABILITY_DOMAIN_1" | "AVAILABILITY_DOMAIN_2"
  # Production requirement: Redundant attachments MUST use separate domains for 99.99% SLA.

  # ── BANDWIDTH ALLOCATION ─────────────────────────────────────────────────
  --bandwidth=10G \
  # Enum: "50M" | "100M" | "200M" | "500M" | "1G" | "2G" | "5G" | "10G" | "20G" | "50G" | "100G"
  --vlan=400 \
  # Integer: 2 to 4094. 802.1Q VLAN ID negotiated with network engineering team.
  --mtu=1440 \
  # Integer: 1440 or 1500. Standard Interconnect MTU.
  --project=my-gcp-project-id
```

### B. Configure Active-Passive HA Interconnect + HA VPN Failover

```bash
# 1. Primary Circuit: Interconnect BGP Peer (Highest Preference: Priority = 100)
gcloud compute routers update-bgp-peer nat-router-us-central1 \
  --region=us-central1 \
  --peer-name=interconnect-bgp-peer \
  --advertised-route-priority=100 \
  --project=my-gcp-project-id

# 2. Standby Backup: Cloud HA VPN BGP Peer (Lower Preference: Priority = 300)
gcloud compute routers update-bgp-peer nat-router-us-central1 \
  --region=us-central1 \
  --peer-name=ha-vpn-standby-peer \
  --advertised-route-priority=300 \
  --project=my-gcp-project-id
# Under normal operations, 100% of traffic flows over the high-speed 10G/100G circuit.
# If a fiber cut occurs, Cloud Router automatically redirects traffic to the encrypted HA VPN tunnel in < 1s!
```

---

## 18. Network Connectivity Center (NCC) — Hub-and-Spoke Transit

VPC Peering does not scale past a handful of connections due to **strict non-transitivity** and peering quotas (default 25 peerings per VPC). **Network Connectivity Center (NCC)** provides centralized transit, allowing hundreds of VPCs, hybrid routers, and third-party SD-WAN virtual appliances to communicate through a single hub.

```bash
# STEP 1: Create Centralized Transit Hub
gcloud network-connectivity hubs create enterprise-global-hub \
  --description="Global Enterprise Hub-and-Spoke Transit Fabric" \
  --project=my-gcp-project-id

# STEP 2: Attach Production VPC as a Spoke
gcloud network-connectivity spokes linked-vpc-network create prod-vpc-spoke \
  --hub=enterprise-global-hub \
  --global \
  --vpc-network=enterprise-production-vpc \
  --exclude-export-ranges="" \
  # CIDR list: Allows selectively suppressing certain subnets from being exported to the hub.
  --project=my-gcp-project-id

# STEP 3: Attach Cloud Router Hybrid Spoke (Propagates On-Prem Routes to All Spokes)
gcloud network-connectivity spokes linked-router-appliances create onprem-router-spoke \
  --hub=enterprise-global-hub \
  --region=us-central1 \
  --router=nat-router-us-central1 \
  --project=my-gcp-project-id
```

---

## 19. Shared VPC Architecture (Host & Service Project Multi-Team Isolation)

In enterprise organizations, network infrastructure is governed centrally by Platform/NetOps in a **Host Project**, while autonomous developer teams deploy GKE clusters in **Service Projects** consuming centralized subnets without permissions to alter firewall rules or routing.

```mermaid
graph TB
    subgraph HostProject["CENTRAL HOST PROJECT (Project ID: host-network-project)"]
        direction TB
        HostAdmin["Platform / NetOps Team<br/>(Full Network Control)"]
        VPCNet["Enterprise Production VPC Network"]
        SharedSubnet["Shared Subnet: gke-us-central1-subnet<br/>(Primary: 10.0.0.0/20, Pods: 10.4.0.0/14, Svcs: 10.8.0.0/20)"]
        Firewalls["Central Security Firewall Policies"]
        CloudNATGW["Shared Cloud NAT Gateway"]

        HostAdmin --> VPCNet
        VPCNet --- SharedSubnet
        VPCNet --- Firewalls
        VPCNet --- CloudNATGW
    end

    SharedSubnet == "IAM Delegation: roles/compute.networkUser<br/>roles/container.hostServiceAgentUser" ==> ServiceGKE
    SharedSubnet == "IAM Delegation: roles/compute.networkUser" ==> ServiceVMs

    subgraph ServiceProject1["SERVICE PROJECT A: GKE Workloads (gke-workloads-project)"]
        direction TB
        GKEAdmin["App Dev Team (Zero Network Perms)"]
        ServiceGKE["GKE Production Cluster<br/>(Worker Nodes lease IPs directly from Host Subnet!)"]
        GKEPodsWorkload["Pods run on Host Secondary Range (10.4.0.0/14)"]
        GKEAdmin --> ServiceGKE
        ServiceGKE --- GKEPodsWorkload
    end

    subgraph ServiceProject2["SERVICE PROJECT B: Data Analytics (data-analytics-project)"]
        direction TB
        DataTeam["Data Engineering Team"]
        ServiceVMs["Compute Engine VMs & Dataproc<br/>(VM eth0 leases IP from Host 10.0.0.0/20)"]
        DataTeam --> ServiceVMs
    end

    style HostProject fill:#f3e5f5,stroke:#7b1fa2
    style ServiceProject1 fill:#e8f5e9,stroke:#388e3c
    style ServiceProject2 fill:#e1f5fe,stroke:#0288d1
```

### A. Shared VPC Host & Service Project Configuration Sequence
# STEP 1: Enable Host Project (Run by Network Security Team)
gcloud compute shared-vpc enable host-network-project-id

# STEP 2: Associate Developer/Workload Project as a Service Project
gcloud compute shared-vpc associate-service-project gke-workloads-project-id \
  --host-project=host-network-project-id

# STEP 3: Delegate Subnet Permissions to Service Project Developer Group
gcloud compute networks subnets add-iam-policy-binding gke-us-central1-subnet \
  --project=host-network-project-id \
  --region=us-central1 \
  --member="group:gke-platform-admins@company.com" \
  --role="roles/compute.networkUser"
  # Grants permission to provision GKE nodes, VMs, and internal LBs inside this specific subnet.

# STEP 4: Grant GKE Robot Account Permission on Host Subnet (MANDATORY for GKE)
gcloud compute networks subnets add-iam-policy-binding gke-us-central1-subnet \
  --project=host-network-project-id \
  --region=us-central1 \
  --member="serviceAccount:service-GKE_SERVICE_PROJECT_NUM@container-engine-robot.iam.gserviceaccount.com" \
  --role="roles/container.hostServiceAgentUser"
```

---

## 20. Workload Identity Federation for GKE (GCP IAM ↔ Kubernetes SA)

Eliminates dangerous, exportable Google Cloud service account JSON private keys. Workload Identity binds a **Kubernetes ServiceAccount (KSA)** directly to a **Google Cloud IAM ServiceAccount (GSA)** using short-lived OIDC tokens.

```mermaid
sequenceDiagram
    autonumber
    participant Pod as Backend Pod (database-reader-ksa)
    participant MetaServer as GKE Node Metadata Server (169.254.169.254)
    participant K8sAPI as Kubernetes API Server (OIDC Provider)
    participant GCPSTS as Google Cloud Security Token Service (STS)
    participant GCPIAM as Google Cloud IAM (GSA Credentials API)
    participant GCS as Google Cloud Storage (Bucket API)

    Pod->>MetaServer: GET /computeMetadata/v1/instance/service-accounts/default/token
    Note over MetaServer: Intercepts request via eBPF/iptables
    MetaServer->>K8sAPI: Verify Pod KSA Projected Token (JWT)
    K8sAPI-->>MetaServer: Token Valid (sub: production:database-reader-ksa)
    MetaServer->>GCPSTS: Exchange K8s JWT for Federated GCP Token
    GCPSTS-->>MetaServer: Returns Short-Lived Federated Token
    MetaServer->>GCPIAM: Assume GSA (database-reader-gsa@project.iam.gserviceaccount.com)
    Note over GCPIAM: Validates "roles/iam.workloadIdentityUser"<br/>binding for "production/database-reader-ksa"
    GCPIAM-->>MetaServer: Returns Google OAuth2 Access Token (Valid 1 hour)
    MetaServer-->>Pod: Injects Bearer Token into Pod Memory
    Pod->>GCS: GET gs://company-secure-bucket/data.parquet (Bearer Token)
    GCS-->>Pod: HTTP 200 OK (Authorized via GSA IAM Roles)
```

### A. Workload Identity Configuration & Binding Sequence
# STEP 1: Create Google IAM Service Account (GSA) with Least Privilege
gcloud iam service-accounts create database-reader-gsa \
  --display-name="GSA for Production Backend Database Reader" \
  --project=my-gcp-project-id

# STEP 2: Grant GSA Specific Resource Permissions (e.g. Cloud Storage Read)
gcloud projects add-iam-policy-binding my-gcp-project-id \
  --member="serviceAccount:database-reader-gsa@my-gcp-project-id.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# STEP 3: Bind KSA to GSA via workloadIdentityUser Role
gcloud iam service-accounts add-iam-policy-binding \
  database-reader-gsa@my-gcp-project-id.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:my-gcp-project-id.svc.id.goog[production/database-reader-ksa]" \
  # Format: serviceAccount:PROJECT_ID.svc.id.goog[K8S_NAMESPACE/KSA_NAME]
  --project=my-gcp-project-id

# STEP 4: Create & Annotate Kubernetes ServiceAccount
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: database-reader-ksa
  namespace: production
  annotations:
    # ── WORKLOAD IDENTITY BINDING ANNOTATION ───────────────────────────────
    iam.gke.io/gcp-service-account: database-reader-gsa@my-gcp-project-id.iam.gserviceaccount.com
EOF
# Any Pod running with "serviceAccountName: database-reader-ksa" automatically
# authenticates to Google Cloud APIs seamlessly with zero credentials in code!
```

---

## 21. VPC Service Controls (VPC-SC) — Data Exfiltration Perimeter Security

> [!IMPORTANT]
> **VPC-SC vs Private Service Connect (PSC)**:
> - **PSC** (Section 13) provides *private network transport* (Layer 4 IP connectivity) to services.
> - **VPC-SC** creates an *authorization firewall perimeter* around Google multi-tenant managed APIs (Cloud Storage, BigQuery, KMS) preventing malicious insiders or compromised pods from exfiltrating data to an unapproved external GCP bucket!

```bash
# 1. Create Access Policy in Google Cloud Access Context Manager
gcloud access-context-manager policies create \
  --organization=123456789012 \
  --title="Enterprise Security Policy"

# 2. Define Authorized Ingress Access Level (IP-based whitelist)
gcloud access-context-manager levels create corp-trusted-perimeter \
  --policy=ACCESS_POLICY_ID \
  --title="Corporate Trusted Range" \
  --basic-level-spec=<(cat <<'EOF'
ipSubnetworks:
  - 10.0.0.0/8
  - 172.16.0.0/12
EOF
)

# 3. Create Security Perimeter Enclosing Protected Projects
gcloud access-context-manager perimeters create production-data-perimeter \
  --policy=ACCESS_POLICY_ID \
  --title="Production Data Shield" \
  --resources=projects/112233445566,projects/998877665544 \
  --restricted-services=storage.googleapis.com,bigquery.googleapis.com,container.googleapis.com \
  --access-levels=corp-trusted-perimeter
# Any API call to BigQuery/Storage originating from outside this perimeter is instantly rejected!
```

---

## 22. Organization Policy Constraints & Enterprise Network Guardrails

Organization Policies enforce unbypassable guardrails at the organization root, preventing teams from inadvertently exposing clusters or creating vulnerable configurations.

```bash
# 1. Enforce Private-Only Compute & GKE Worker Nodes (No Public IPs)
gcloud org-policies set-policy - <<'EOF'
constraint: constraints/compute.vmExternalIpAccess
listPolicy:
  allValues: DENY
EOF

# 2. Block Default VPC Auto-Creation on New Projects (Forces Custom VPC Architecture)
gcloud org-policies set-policy - <<'EOF'
constraint: constraints/compute.skipDefaultNetworkCreation
booleanPolicy:
  enforced: true
EOF

# 3. Restrict VPC Peering (Prevents Shadow Interconnections Without Security Review)
gcloud org-policies set-policy - <<'EOF'
constraint: constraints/compute.restrictVpcPeering
listPolicy:
  allValues: DENY
EOF

# 4. Mandatory Shielded VMs (Hardware-Rooted vTPM & Kernel Integrity Verification)
gcloud org-policies set-policy - <<'EOF'
constraint: constraints/compute.requireShieldedVm
booleanPolicy:
  enforced: true
EOF
```

---

## 23. Kubernetes Gateway API (Modern L7 Routing) vs Legacy Ingress

The **Gateway API** replaces legacy `Ingress` and `BackendConfig` CRDs with an expressive, role-oriented standard for Layer 7 traffic routing, cross-namespace routing, and security policy attachment.

```mermaid
graph TB
    subgraph InfraRole["INFRASTRUCTURE PROVIDER (GCP / GKE)"]
        GClass["GatewayClass: gke-l7-global-external-managed<br/>(Provisions Global External Application LB)"]
    end

    subgraph ClusterOperatorRole["CLUSTER OPERATOR / NETOPS (namespace: production)"]
        Gateway["Gateway: enterprise-edge-gateway<br/>- Listener: HTTPS:443 (TLS Terminate)<br/>- allowedRoutes: namespaces.from: All"]
        TLSCert["Certificate: production-tls-cert"]
        Gateway --- TLSCert
    end

    GClass --> Gateway

    subgraph AppTeam1["APPLICATION TEAM A (namespace: store)"]
        direction TB
        Route1["HTTPRoute: store-api-route<br/>Matches: /v1/store/*"]
        Svc1["Service: store-service"]
        Pods1["Store Pod Replicas (NEG)"]
        Route1 --> Svc1 --> Pods1
    end

    subgraph AppTeam2["APPLICATION TEAM B (namespace: auth)"]
        direction TB
        Route2["HTTPRoute: auth-api-route<br/>Matches: /v1/auth/*"]
        Svc2["Service: auth-service"]
        Pods2["Auth Pod Replicas (NEG)"]
        Route2 --> Svc2 --> Pods2
    end

    subgraph SecurityRole["SECOPS POLICY ATTACHMENT"]
        Policy["GCPBackendPolicy: gateway-security-policy<br/>Cloud Armor WAF: edge-waf-security-policy"]
    end

    Gateway == "Cross-Namespace Attachment" ==> Route1
    Gateway == "Cross-Namespace Attachment" ==> Route2
    Policy -. "Enforces WAF Rules" .-> Svc1
    Policy -. "Enforces WAF Rules" .-> Svc2

    style InfraRole fill:#f5f5f5,stroke:#9e9e9e
    style ClusterOperatorRole fill:#e1f5fe,stroke:#0288d1
    style AppTeam1 fill:#e8f5e9,stroke:#388e3c
    style AppTeam2 fill:#fff3e0,stroke:#f57c00
    style SecurityRole fill:#fce4ec,stroke:#c2185b
```

### A. Declarative Gateway & HTTPRoute Manifests
kubectl apply -f - <<'EOF'
# ── GATEWAY (INFRASTRUCTURE DEFINITION - MANAGED BY NETOPS) ───────────────
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: enterprise-edge-gateway
  namespace: production
spec:
  gatewayClassName: gke-l7-global-external-managed
  # Standard GKE GatewayClasses:
  #   - "gke-l7-global-external-managed" (Global External Application LB with Cloud Armor)
  #   - "gke-l7-regional-internal" (Internal Application Load Balancer inside VPC)
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: production-tls-cert
      allowedRoutes:
        namespaces:
          from: All
          # Allows microservice teams in any namespace to attach their HTTPRoutes!
---
# ── HTTPROUTE (APPLICATION ROUTING - MANAGED BY DEVELOPER TEAMS) ───────────
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-routing-rules
  namespace: production
spec:
  parentRefs:
    - name: enterprise-edge-gateway
      namespace: production
  hostnames:
    - "api.company.com"
  rules:
    # Rule 1: High-priority Canary Traffic (Header-based Routing)
    - matches:
        - headers:
            - type: Exact
              name: X-Release-Channel
              value: canary
      backendRefs:
        - name: backend-api-canary
          port: 8080

    # Rule 2: Standard Production Traffic with URL Rewrite
    - matches:
        - path:
            type: PathPrefix
            value: /v2/users
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /users
      backendRefs:
        - name: backend-api-production
          port: 8080
---
# ── SECURITY ATTACHMENT (BINDING CLOUD ARMOR DIRECTLY TO GATEWAY) ──────────
apiVersion: networking.gke.io/v1
kind: GCPBackendPolicy
metadata:
  name: gateway-security-policy
  namespace: production
spec:
  targetRef:
    group: ""
    kind: Service
    name: backend-api-production
  default:
    securityPolicy: "edge-waf-security-policy"
    # Links Cloud Armor WAF defined in Section 14 directly to the Gateway backend!
EOF
```

---

## 24. NodeLocal DNSCache & Dynamic Secondary Range Capacity Planning

High-throughput container platforms frequently face **DNS throttling** and **IP exhaustion**. These two mechanisms protect large-scale production clusters.

```mermaid
graph TD
    subgraph ClassicProblem["CLASSIC PROBLEM: CoreDNS UDP Conntrack Exhaustion"]
        PodOld["Kubernetes Pod"] -->|UDP DNS Query| KubeProxy["kube-proxy iptables DNAT<br/>(ClusterIP: 10.8.0.10:53)"]
        KubeProxy -->|Creates Conntrack Entry| ConntrackTable["Linux Netfilter Conntrack Table<br/>(Exhaustion & Race Conditions!)"]
        ConntrackTable -. "Dropped Packets / Table Full" .-> Timeout["5-Second DNS Lookup Timeouts / Failures"]
        ConntrackTable -->|Forward across Node| CoreDNSPod["CoreDNS Pod Replica"]
    end

    subgraph NodeLocalSolution["ENTERPRISE SOLUTION: NodeLocal DNSCache (169.254.20.10)"]
        PodNew["Kubernetes Pod"] -->|Direct Loopback Query| LocalCache["NodeLocal DNSCache Agent<br/>(Runs on Node Host: 169.254.20.10)"]
        LocalCache -- "Cache Hit (0.1ms)" --> InstantResponse["Instant Resolution (Zero Netfilter DNAT!)"]
        LocalCache -- "Cache Miss" --> TCPCoreDNS["Persistent Single TCP Stream<br/>(Zero UDP Drops)"]
        TCPCoreDNS --> CoreDNSUpstream["CoreDNS Deployment → Cloud DNS Upstream"]
    end

    style ClassicProblem fill:#ffebee,stroke:#c62828
    style NodeLocalSolution fill:#e8f5e9,stroke:#2e7d32
```

### A. Deploy NodeLocal DNSCache DaemonSet (Eliminating Conntrack Bottlenecks)

Standard Kubernetes pods route DNS queries to CoreDNS pods over UDP. Under high connection churn, Linux `conntrack` table limits are reached, resulting in silent packet drops and **5-second DNS timeouts**. NodeLocal DNSCache runs a local cache instance on node IP `169.254.20.10`:

```bash
# Enable NodeLocal DNSCache on GKE Cluster
gcloud container clusters update enterprise-ha-cluster \
  --region=us-central1 \
  --enable-dns-cache \
  --project=my-gcp-project-id
# Pod DNS requests now hit the node local memory cache directly over loopback!
# Result: 0% conntrack table overhead, sub-millisecond response latency.
```

### B. Dynamic Secondary Range Capacity & Autoscaling Sizing Math

> [!WARNING]
> **Autoscaling IP Exhaustion Gotcha**:
> By default, GKE allocates a `/24` (256 IP addresses) to *every single worker node* for its Pods, regardless of whether that node runs 5 pods or 110 pods!
> - A `/14` secondary Pod range contains **262,144 IPs**.
> - When Cluster Autoscaler and Node Auto-Provisioning (NAP) scale up, 1,024 nodes consume the ENTIRE `/14` block!
>
> **Best Practice Solution**:
> Create node pools with reduced max pods per node to preserve secondary range space:
> ```bash
> gcloud container node-pools create high-density-pool \
>   --cluster=enterprise-ha-cluster \
>   --region=us-central1 \
>   --max-pods-per-node=32 \
>   # Reduces node Pod CIDR reservation from /24 (256 IPs) down to /26 (64 IPs)!
>   # Instantly yields 4x more node scaling capacity from the exact same subnet!
> ```

---

## 25. Pod Security Standards (PSA), Binary Authorization & Envelope Encryption

Secures the workload supply chain, host runtime privileges, and encryption at rest.

### A. Enforce Pod Security Standards (PSA) at Namespace Level

PodSecurityPolicies (PSP) are dead; modern Kubernetes mandates **Pod Security Admission (PSA)** labels:

```bash
kubectl label --overwrite namespace production \
  # ── ENFORCEMENT LEVEL ────────────────────────────────────────────────────
  pod-security.kubernetes.io/enforce=restricted \
  # Enum: "privileged" | "baseline" | "restricted"
  #   privileged: Completely unconstrained (root, host network, capabilities).
  #   baseline: Prevents known privilege escalations (default Docker profile).
  #   restricted: Hardened zero-trust! Disallows root, host namespaces, and privilege escalation.
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

### B. Supply Chain Security: Binary Authorization Attestation Policy

Ensures only cryptographically signed, vulnerability-scanned container images can be scheduled:

```bash
# 1. Enable Binary Authorization on Cluster
gcloud container clusters update enterprise-ha-cluster \
  --region=us-central1 \
  --enable-binauthz \
  --project=my-gcp-project-id

# 2. Apply Strict Policy Requiring Attestation Signatures
gcloud container binauthz policy import - <<'EOF'
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  # Requires cryptographic signature from approved security pipeline before scheduling!
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  requireAttestationsBy:
    - projects/my-gcp-project-id/attestors/vuln-scan-attestor
EOF
```

### C. Application-Layer Envelope Encryption for Secrets at Rest (Cloud KMS)

Protects `etcd` secrets using a Customer-Managed Encryption Key (CMEK), so stolen disk backups or compromised storage cannot expose plaintext credentials:

```bash
gcloud container clusters update enterprise-ha-cluster \
  --region=us-central1 \
  --database-encryption-key=projects/my-gcp-project-id/locations/us-central1/keyRings/k8s-security-ring/cryptoKeys/etcd-secrets-key \
  --project=my-gcp-project-id
```

---

## 26. Enterprise Scale Quotas, Limits Governance & MTU/MSS Clamping

### A. The MTU Chain & IPsec TCP MSS Clamping Breakdown

A classic enterprise networking failure: Packets pass fine under small pings, but **hang or drop under high payloads**. This is caused by MTU mismatch across encapsulating tunnels:

| Network Layer | Maximum Transmission Unit (MTU) | Encapsulation Overhead | Maximum Segment Size (MSS) |
| :--- | :--- | :--- | :--- |
| **Standard Ethernet / Docker Host** | 1500 bytes | None | 1460 bytes |
| **Google Cloud VPC Network** | **1460 bytes** | None | 1420 bytes |
| **Cloud Interconnect (VLAN Attachment)**| **1440 or 1500 bytes** | 802.1Q tag (4 bytes) | 1400 / 1460 bytes |
| **Cloud HA VPN (IPsec ESP Encapsulation)**| **1460 bytes link** | **ESP / IV / Auth: ~60 bytes** | **Clamp to 1360 bytes!** |

```mermaid
graph TD
    subgraph StandardPacket["Standard Ethernet Packet (1500 Bytes)"]
        IP1["IPv4 Header<br/>(20 Bytes)"] --- TCP1["TCP Header<br/>(20 Bytes)"] --- Data1["Application Payload (MSS)<br/>(1460 Bytes)"]
    end

    subgraph VPCPacket["Google Cloud VPC Packet (Max MTU: 1460 Bytes)"]
        IP2["IPv4 Header<br/>(20 Bytes)"] --- TCP2["TCP Header<br/>(20 Bytes)"] --- Data2["Safe Payload (MSS)<br/>(1420 Bytes)"]
    end

    subgraph VPNPacket["IPsec ESP Encapsulated Packet (Causes Fragmentation!)"]
        OuterIP["Outer IPv4 Header<br/>(20 Bytes)"] --- ESP["IPsec ESP Header + IV<br/>(16 Bytes)"] --- InnerPacket["Inner Original Packet<br/>(1460 Bytes)"] --- ESPOther["ESP Trailer + ICV Auth<br/>(24 Bytes)"]
        NoteOverhead["Total: 1520 Bytes! Exceeds 1460 MTU!<br/>Result: Dropped if DF (Don't Fragment) bit is set!"]
    end

    subgraph MSSClampedPacket["SOLUTION: TCP MSS Clamping to 1360 Bytes"]
        OuterIP_OK["Outer IPv4 (20B)"] --- ESP_OK["ESP + IV (16B)"] --- InnerIP_OK["Inner IPv4 (20B)"] --- InnerTCP_OK["Inner TCP (20B)"] --- ClampedPayload["Clamped Payload (MSS: 1360B)"] --- ESPOther_OK["Trailer (24B)"]
        NoteOK["Total: 1460 Bytes (Exact Match for VPC Link! Zero Drops!)"]
    end

    StandardPacket -. "Crosses VPC Boundary" .-> VPCPacket
    VPCPacket -. "Crosses IPsec VPN Tunnel without MSS Clamping" .-> VPNPacket
    VPNPacket -. "Apply TCP MSS Clamping = 1360" .-> MSSClampedPacket

    style StandardPacket fill:#e3f2fd,stroke:#1565c0
    style VPCPacket fill:#fff3e0,stroke:#f57c00
    style VPNPacket fill:#ffebee,stroke:#c62828
    style MSSClampedPacket fill:#e8f5e9,stroke:#2e7d32
```

**How to Enforce MSS Clamping on Cloud Router**:
```bash
# Clamp TCP MSS to prevent packet fragmentation over IPsec tunnels
gcloud compute routers update-bgp-peer nat-router-us-central1 \
  --region=us-central1 \
  --peer-name=onprem-bgp-peer-0 \
  --advertised-route-priority=100
# Ensure on-premise firewall/router applies: `iptables -t mangle -A POSTROUTING -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu`
```

### B. Enterprise Hard Scale Quotas Cheat Sheet

Monitor these hard thresholds in Cloud Monitoring before launching massive multi-cluster migrations:

| Component | Default GCP Quota / Limit | Mitigation When Approaching Cap |
| :--- | :--- | :--- |
| **VPC Peering Connections per VPC** | **25 peerings** | Migrate to **Network Connectivity Center (NCC)** (Section 18). |
| **Dynamic BGP Routes per Cloud Router** | **100 routes** | Summarize BGP prefixes on on-premises edge routers into larger CIDRs. |
| **Internal Load Balancer Backends** | **250 backends** | Use GKE NEG (Network Endpoint Groups) with GKE Datapath v2. |
| **VPC Firewall Rules per Network** | **1000 rules** | Consolidate into hierarchical firewall policies and service accounts. |
| **Pods per Service (kube-proxy)** | **1000 endpoints** | Automatically resolved by **EndpointSlice** in modern Kubernetes. |

---

## 27. Cross-Layer Diagnostic & Troubleshooting Playbook

When communication fails between Docker containers, Kubernetes Pods, and Cloud VPC resources, follow this systematic multi-layer diagnostic sequence:

```mermaid
graph LR
    Container["Docker Container<br/>(veth / bridge)"] -->|Hop 1: CNI| Host["Host / K8s Worker Node<br/>(eth0: 10.0.0.15)"]
    Host -->|Hop 2: NetworkPolicy| Pod["Pod CNI Namespace<br/>(10.4.1.24)"]
    Pod -->|Hop 3: VPC Firewall| Subnet["VPC Primary Subnet<br/>(10.0.0.0/20)"]
    Subnet -->|Hop 4: Route Table| NATRouter["Cloud NAT / Router<br/>(Egress Gateway)"]
    NATRouter -->|Hop 5: Interconnect / VPN / Peering| Remote["Remote VPC / On-Premises<br/>(192.168.0.0/16)"]

    style Container fill:#e1f5fe,stroke:#0288d1
    style Pod fill:#e8f5e9,stroke:#388e3c
    style Host fill:#fff3e0,stroke:#f57c00
    style Subnet fill:#ede7f6,stroke:#512da8
    style NATRouter fill:#fce4ec,stroke:#c2185b
    style Remote fill:#eceff1,stroke:#455a64
```

### Diagnostic One-Liners:

```bash
# 1. Test Layer 3 Node-to-Node Reachability from inside a Kubernetes Pod
kubectl exec -it pod/web-app-74b89-x8q2z -n production -- ping -c 4 10.0.0.15

# 2. Verify Layer 4 Port Availability & TLS Handshake
kubectl exec -it pod/web-app-74b89-x8q2z -n production -- nc -zvw3 postgres.internal.corp 5432

# 3. Trace Network Route Hops to Detect Routing Loops or VPN Dropouts
kubectl exec -it pod/web-app-74b89-x8q2z -n production -- traceroute -n 192.168.10.5

# 4. Verify Internal CoreDNS Resolution & Upstream Response
kubectl exec -it pod/web-app-74b89-x8q2z -n production -- dig +short database.internal.corp

# 5. Live Packet Capture on Worker Node Interface (Detecting Dropped Packets)
sudo tcpdump -i any -nn "host 10.4.1.24 and port 8080" -vv

# 6. GCP Cloud VPC Simulated Packet Tracer (Tests Firewalls, Routes & Peering in API)
gcloud compute network-management connectivity-tests run test-pod-to-onprem \
  --source-ip=10.4.1.24 \
  --destination-ip=192.168.10.5 \
  --destination-port=5432 \
  --protocol=TCP \
  --network=enterprise-production-vpc
```
