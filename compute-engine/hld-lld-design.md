# Compute Engine: High-Level Design (HLD) & Low-Level Design (LLD)

This document details the architectural design for **Google Cloud Compute Engine (Virtual Machines)**.

---

## 1. High-Level Design (HLD)

The High-Level Architecture illustrates how Compute Engine VM instances interact with Google Cloud Virtual Private Cloud (VPC), IAM Access Controls, Storage Volume Layers, and Cloud Security perimeters:

```mermaid
graph TD
    Client["Client / Operator"] -->|HTTPS / SSH| CloudRouter["Cloud Router / Gateway"]
    CloudRouter -->|VPC Ingress Firewall| VPC["Virtual Private Cloud (VPC) Subnet"]
    
    subgraph VPC ["VPC Network (us-central1)"]
        subgraph VMInstance ["Compute Engine VM Instance (n2-standard-4)"]
            OS["Operating System (Ubuntu 22.04 LTS)"]
            Agent["Google Guest Agent & Metadata Server"]
            App["Workload Container / Web Server"]
        end
    end
    
    VMInstance -->|Hot-Plug Attach| BootDisk["Boot Disk (pd-ssd 50GB)"]
    VMInstance -->|Hot-Plug Attach| DataDisk["Data Disk (pd-balanced 200GB)"]
    VMInstance -->|IAM Role Scopes| IAM["Cloud IAM & Service Account"]
    VMInstance -->|Serial Console Logging| OpsLogging["Cloud Logging / Audit Trail"]
```

### Key HLD Components:
1. **Network Layer (VPC Subnet)**: Isolates VM instances inside custom subnets, controlling ingress/egress via Stateful Firewall Rules and Network Tags.
2. **Compute Layer**: KVM-based virtualized compute resources (`e2`, `n2`, `c2`, `t2d` families) running custom Linux/Windows distributions.
3. **Storage Layer**: Block storage volumes (Persistent Disks) attached over Google's high-speed internal NVMe network fabric.
4. **Identity & Access Layer**: IAM Service Accounts attached to VM instances, enforcing Least Privilege via OAuth2 scopes.

---

## 2. Low-Level Design (LLD)

### Compute Instance Lifecycle State Machine:

```mermaid
stateDiagram-v2
    [*] --> PROVISIONING: gcloud compute instances create
    PROVISIONING --> STAGING: Allocating CPU, RAM & Disk
    STAGING --> RUNNING: Booting OS Kernel
    RUNNING --> STOPPING: gcloud compute instances stop
    STOPPING --> STOPPED: CPU/RAM released (Disk preserved)
    STOPPED --> RUNNING: gcloud compute instances start
    RUNNING --> SUSPENDED: gcloud compute instances suspend (RAM saved to disk)
    SUSPENDED --> RUNNING: gcloud compute instances resume
    RUNNING --> TERMINATED: Preempted (Spot VM)
    STOPPED --> [*]: gcloud compute instances delete
```

### LLD Internal Mechanics:
* **Metadata Server Interaction**: Inside every VM, the local metadata server runs at `169.254.169.254:80`. The Google Guest Agent queries this IP to fetch SSH keys, startup scripts, and OAuth2 access tokens dynamically.
* **Live Migration**: GCP live-migrates running VM instances to alternative hardware nodes during host maintenance without interrupting workload execution or network connections.
* **Block Storage Detachment/Attachment**: Persistent Disks are decoupled block devices. When a VM stops, disk contents persist in regional storage and can be attached to new instances instantly.
