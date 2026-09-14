# GCP Compute Engine Case Study Index: Managed Instance Groups, Autoscaling & Load Balancing

Welcome to the **Google Cloud Compute Engine Deep-Dive Architecture & Case Study Index**. This module provides detailed architectural analysis, design patterns, lifecycle state machines, and step-by-step production command guides for Google Cloud Compute Engine virtual machine infrastructure, instance groups, dynamic autoscaling, and software-defined load balancing.

---

## Case Study Sitemap

| Document | Focus Area | Key Architectural Concepts Covered |
| :--- | :--- | :--- |
| [**1. Managed Instance Groups, Autoscaling & Load Balancing**](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/mig-autoscaling-loadbalancing.md) | Fleet Management, Resiliency & Scale | Managed Instance Groups (MIGs), Instance Templates, Auto-Healing Health Checks, Zonal vs Regional MIGs (Zonal Outage Resilience), Stateless vs Stateful MIGs, Dynamic Autoscaling (CPU/RPS/Queue Metrics), Zero-Downtime Rolling Updates, Layer 4 NLB vs Layer 7 ALB Software-Defined Load Balancing. |
| [**2. Global Application Load Balancer in Action**](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/alb-global-routing-in-action.md) | Global Request Lifecycle & Traffic Management | Single Global Anycast IP, Global Forwarding Rules, Target HTTPS Proxy & SSL Certs (up to 15), QUIC Protocol, Capacity & Proximity Routing, Cross-Region Failover, Backend Buckets (Cloud Storage Static Content), and Network Endpoint Groups (Zonal, Internet, Hybrid, Serverless NEGs). |
| [**3. Cloud CDN Edge Caching & Cache Modes**](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/cloud-cdn-edge-caching.md) | Edge Acceleration & Content Offload | Global 90+ Edge PoPs, Cache Miss vs. Cache Fill vs. Cache Hit Sequence Flow, Cache Modes (`USE_ORIGIN_HEADERS`, `CACHE_ALL_STATIC`, `FORCE_CACHE_ALL`), Cloud Logging Diagnostics, and Cache Key Customization. |
| [**4. Layer 4 Network Load Balancers (Proxy vs Passthrough)**](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/layer4-network-load-balancers.md) | Transport Layer Load Balancing | Proxy NLB (Target TCP/SSL Proxy, Session Termination) vs Passthrough NLB (Google Maglev, Andromeda SDN, Direct Server Return - DSR), Preserving Client Source IP, Regional Backend Services vs Legacy Target Pools (50 pool limit), Multi-Protocol (TCP, UDP, ESP, GRE, ICMP). |
| [**5. Internal Load Balancers & 3-Tier Architecture**](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/internal-load-balancers-3tier-architecture.md) | Private VPC Load Balancing & Microservices | Internal ALB (Envoy Proxy, Proxy-Only Subnet, Regional vs. Cross-Region), Internal Passthrough NLB (Andromeda SDN, Zero-Proxy Overhead), Internal Proxy NLB, and Enterprise 3-Tier Web Application Architecture Blueprint (Web Tier -> App Tier -> DB Tier). |

---

## Core System Architecture

```mermaid
graph TD
    classDef client fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef lb fill:#064E3B,stroke:#34D399,stroke-width:2px,color:#F8FAFC;
    classDef mig fill:#1E1B4B,stroke:#818CF8,stroke-width:2px,color:#F8FAFC;
    classDef vm fill:#0F172A,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;

    Client["Internet Traffic"]:::client -->|Global Anycast IP| LB["Software-Defined Load Balancer (ALB / NLB)"]:::lb

    subgraph RegionalMIG ["Regional Managed Instance Group (Multi-Zone Distribution)"]
        subgraph ZoneA ["Zone A: us-central1-a"]
            VM1["VM Instance 1"]:::vm
        end
        subgraph ZoneB ["Zone B: us-central1-b"]
            VM2["VM Instance 2"]:::vm
        end
        subgraph ZoneC ["Zone C: us-central1-c"]
            VM3["VM Instance 3"]:::vm
        end

        AutoHealer["Auto-Healing Engine<br/>(Health Check Probes & Auto-Recreation)"]:::mig
        Autoscaler["Autoscaling Controller<br/>(Dynamic Scale Up / Scale Down)"]:::mig
    end

    LB --> VM1
    LB --> VM2
    LB --> VM3

    AutoHealer --> RegionalMIG
    Autoscaler --> RegionalMIG
```

---

## Summary of Key Learnings & Engineering Concepts

1. **Instance Templates**: Record VM configurations (machine shape, boot disk image, network interfaces, startup scripts, service account scopes) so they can be repeated deterministically across thousands of VM nodes.
2. **Auto-Healing**: Health checks continuously test VM responsiveness. If an instance crashes or becomes unhealthy, the Instance Group Manager (IGM) recreates it using the **same instance name** and **same instance template**.
3. **Regional High Availability**: Regional MIGs spread instances across multiple availability zones in a region, preventing single zonal outages from bringing down services.
4. **Software-Defined Load Balancing**: GCP Cloud Load Balancing handles over 1,000,000 QPS with global Anycast IPs without requiring physical appliances or device management.

---

## Related Workspace References

* [Compute Engine Documentation Index](file:///home/btpl-lap-22/live/gcd/compute-engine/README.md)
* [Compute Engine Architecture Design (HLD/LLD)](file:///home/btpl-lap-22/live/gcd/compute-engine/hld-lld-design.md)
* [Compute Engine Decision Tree Manual](file:///home/btpl-lap-22/live/gcd/compute-engine/decision-tree.md)
* [Compute Engine Shell Command Manual](file:///home/btpl-lap-22/live/gcd/compute-engine/shell-commands.md)
* [VPC Network Case Study Index](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/README.md)
