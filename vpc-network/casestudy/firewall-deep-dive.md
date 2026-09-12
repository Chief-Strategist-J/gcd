# Distributed Stateful Firewall Engine & Packet Filtering Deep Dive

This document details the architecture, evaluation engine, connection tracking mechanics (`conntrack`), rule priority hierarchy, and packet blocking behavior of Google Cloud's **Distributed Stateful Firewall Engine**.

---

## 1. Architectural Foundation: Distributed Enforcement

Unlike traditional cloud or enterprise network architectures that route traffic through centralized hardware firewall appliances, Virtual Router VMs, or perimeter middleboxes, GCP Distributed Firewalls are **enforced in parallel directly on the Linux hypervisor host of every Compute Engine instance**.

```mermaid
graph TD
    classDef trad fill:#451A03,stroke:#F97316,stroke-width:2px,color:#F8FAFC;
    classDef dist fill:#064E3B,stroke:#34D399,stroke-width:2px,color:#F8FAFC;
    classDef vm fill:#1E1B4B,stroke:#818CF8,stroke-width:2px,color:#F8FAFC;

    subgraph Traditional Architecture ["TRADITIONAL CENTRALIZED FIREWALL ARCHITECTURE"]
        VMA_T["VM A"]:::vm -->|Hair-pin Routing Bottleneck| Appliance["Hardware / Virtual Firewall Appliance<br/>(Single point of failure & throughput choke)"]:::trad
        Appliance --> VMB_T["VM B"]:::vm
    end

    subgraph GCP Architecture ["GCP DISTRIBUTED FIREWALL ARCHITECTURE"]
        VMA_G["VM A"]:::vm <-->|vNIC Firewall Kernel Tap A| HPP_A["Host Hypervisor Firewall A<br/>(Parallel Fast-Path)"]:::dist
        HPP_A <-->|Direct Jupiter Fabric Wire| HPP_B["Host Hypervisor Firewall B<br/>(Parallel Fast-Path)"]:::dist
        HPP_B <-->|vNIC Firewall Kernel Tap B| VMB_G["VM B"]:::vm
    end
```

---

## 2. Stateful Connection Tracking Mechanics (`conntrack`)

GCP Firewalls are strictly **stateful**. This means once a session is established and permitted by an egress or ingress rule, all bi-directional return packets belonging to that connection are automatically allowed.

### The Conntrack Session Tuple

The hypervisor host's Andromeda packet engine maintains a high-speed lock-free connection tracking table (`conntrack`). Each active session is keyed by a 5-tuple hash:

$$\text{Session Tuple} = \big( \text{Source IP}, \text{Source Port}, \text{Destination IP}, \text{Destination Port}, \text{Protocol} \big)$$

```mermaid
graph TD
    classDef pkt fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef check fill:#4C1D95,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;
    classDef allow fill:#14532D,stroke:#4ADE80,stroke-width:2px,color:#F8FAFC;
    classDef eval fill:#065F46,stroke:#34D399,stroke-width:2px,color:#F8FAFC;

    Incoming["INCOMING / OUTGOING PACKET"]:::pkt --> Lookup{"CONNTRACK TABLE LOOKUP:<br/>Is 5-Tuple Hash in Memory?"}:::check

    Lookup -->|MATCH FOUND| StatefulBypass["STATEFUL FAST-PATH PASS:<br/>Bypass Firewall Priority Engine<br/>Deliver Packet Immediately"]:::allow
    Lookup -->|NO MATCH| RuleEngine["EVALUATE FIREWALL RULE ENGINE:<br/>Check Priorities (0 -> 65535)"]:::eval
```

### Example: Connection State Lifecycle

```mermaid
sequenceDiagram
    autonumber
    participant VM as Guest OS (mynet-us-vm: 10.128.0.2)
    participant HPP as Host Hypervisor (Andromeda HPP)
    participant Conntrack as Stateful Conntrack Table
    participant External as External DNS (8.8.8.8)

    VM->>HPP: Outbound TCP Request (10.128.0.2:49152 -> 8.8.8.8:53)
    HPP->>HPP: Evaluate Egress Rules (Match: Priority 1000 ALLOW)
    HPP->>Conntrack: Insert Tuple (10.128.0.2, 49152, 8.8.8.8, 53, TCP, ESTABLISHED)
    HPP->>External: Transmit Frame to Internet
    External-->>HPP: Inbound Reply (8.8.8.8:53 -> 10.128.0.2:49152)
    HPP->>Conntrack: Lookup 5-Tuple Hash
    Conntrack-->>HPP: MATCH CONFIRMED (State: ESTABLISHED)
    Note over HPP: Bypasses all Ingress Firewall Rules!
    HPP-->>VM: Deliver Reply Packet directly to Guest OS
```

---

## 3. Firewall Evaluation Priority & Match Logic

Firewall rules are processed sequentially based on **Priority Integer Values** ranging from `0` (highest priority) to `65535` (lowest priority).

```mermaid
graph TD
    classDef start fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef check fill:#4C1D95,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;
    classDef allow fill:#14532D,stroke:#4ADE80,stroke-width:2px,color:#F8FAFC;
    classDef drop fill:#881337,stroke:#FB7185,stroke-width:2px,color:#F8FAFC;

    PKT["Packet Arrives at vNIC Interface"]:::start --> CT{"In Conntrack Table?"}:::check
    CT -->|Yes| PASS["ALLOW PACKET (Stateful Bypass)"]:::allow
    CT -->|No| P0{"Priority 0 Check:<br/>Matches Rule?"}:::check
    
    P0 -->|Match| ACT0{"Action: Allow or Deny?"}:::check
    P0 -->|No Match| P1000{"Priority 1000 Check:<br/>Matches Rule?"}:::check
    
    P1000 -->|Match| ACT1000{"Action: Allow or Deny?"}:::check
    P1000 -->|No Match| P65535{"Priority 65535 Check:<br/>Implied Rule Evaluation"}:::check

    ACT0 -->|ALLOW| PASS
    ACT0 -->|DENY| DROP["DROP PACKET (Log Failure)"]:::drop

    ACT1000 -->|ALLOW| PASS
    ACT1000 -->|DENY| DROP

    P65535 -->|Ingress Context| DROP
    P65535 -->|Egress Context| PASS
```

### First-Match Engine Rule Execution

1. Firewall rules are evaluated from **lowest priority number to highest priority number** (e.g. Priority `1` evaluated before Priority `1000`).
2. The **first rule that matches** the packet attributes (Source IP, Destination IP, Protocol, Port, Target Tag/Service Account) terminates the evaluation chain.
3. If a higher priority rule evaluates to `DENY`, the packet is dropped immediately—even if a lower priority rule exists that would allow it (**Deny-Override**).

---

## 4. Implied Firewall Rules

Every GCP VPC network contains **two unalterable implied firewall rules** operating at the lowest priority (`65535`):

| Rule Name | Direction | Priority | Source / Target | Action | Modification Allowed? |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`implied-deny-ingress`** | Ingress | `65535` | `0.0.0.0/0` $\rightarrow$ All Instances | **DENY** | **NO** (System Immutable) |
| **`implied-allow-egress`** | Egress | `65535` | All Instances $\rightarrow$ `0.0.0.0/0` | **ALLOW** | **NO** (System Immutable) |

> **Architectural Implication**: All inbound traffic to a VM is blocked by default unless an explicit ingress firewall rule (e.g. Priority 1000 `ALLOW tcp:22`) is created. All outbound traffic from a VM is allowed by default.

---

## 5. Targeting Selectors: Tags vs. Service Accounts

GCP Firewalls support three granular targeting mechanisms:

```mermaid
graph TD
    classDef target fill:#0F172A,stroke:#818CF8,stroke-width:2px,color:#F8FAFC;
    classDef opt fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;
    classDef rbac fill:#064E3B,stroke:#34D399,stroke-width:2px,color:#F8FAFC;

    Selector["FIREWALL TARGET SELECTORS"]:::target

    Selector --> T1["1. ALL INSTANCES IN NETWORK<br/>Applies universally to every vNIC in the VPC network"]:::opt
    Selector --> T2["2. TARGET NETWORK TAGS (e.g. 'web-server')<br/>String labels attached to VM instances.<br/>Non-IAM protected (Managed by compute.admin)"]:::opt
    Selector --> T3["3. TARGET SERVICE ACCOUNTS (e.g. 'app-sa@proj.iam.gserviceaccount.com')<br/>Identity-based enforcement.<br/>Strictly RBAC protected via IAM serviceAccountUser permission"]:::rbac
```

---

## 6. Hierarchical Policy Evaluation Pipeline

In enterprise organizations, firewall evaluation follows a 4-tier hierarchy before reaching the VPC Network level:

```mermaid
graph TD
    classDef org fill:#4C1D95,stroke:#C084FC,stroke-width:2px,color:#F8FAFC;
    classDef folder fill:#1E1B4B,stroke:#818CF8,stroke-width:2px,color:#F8FAFC;
    classDef net fill:#064E3B,stroke:#34D399,stroke-width:2px,color:#F8FAFC;
    classDef vpc fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC;

    Tier1["1. ORGANIZATION HIERARCHICAL POLICIES<br/>Enforced by SecOps team globally across all projects.<br/>Supports 'Deny', 'Allow', and 'Goto Next' delegates"]:::org
    Tier2["2. FOLDER HIERARCHICAL POLICIES<br/>Scoped to specific environments (Production vs Staging)"]:::folder
    Tier3["3. GLOBAL & REGIONAL NETWORK FIREWALL POLICIES<br/>Modern policy containers attached to VPC networks"]:::net
    Tier4["4. TRADITIONAL VPC FIREWALL RULES<br/>Legacy rules evaluated per VPC network"]:::vpc

    Tier1 -->|If Delegate or No Match| Tier2 -->|If Delegate or No Match| Tier3 -->|If Delegate or No Match| Tier4
```

---

## Related Workspace Documents

- [Case Study Index](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/README.md)
- [High-Level Architecture (HLD)](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/hld-architecture.md)
- [Low-Level Packet Lifecycle (LLD)](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/lld-packet-lifecycle.md)
- [Compute & Network Integration](file:///home/btpl-lap-22/live/gcd/vpc-network/casestudy/compute-network-integration.md)
