# Compute Engine: Machine Selection, Security, Storage & Provisioning Decision Trees

This document provides decision logic trees, architectural family breakdowns, custom machine type constraint rules, security/isolation model guidance, and block storage/encryption selection matrices for Compute Engine.

---

## 1. The 4 Compute Engine Machine Families & Architecture Matrix

Compute Engine organizes VM hardware options into **four workload-optimized machine families**:

| Machine Family | Machine Series | Underlying CPU Platform | vCPU Range | Memory (RAM) per vCPU | Primary Target Workloads |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **General-Purpose** | `E2` (Shared Core) | Context-Switched Intel / AMD | 0.25 to 1 vCPU | 0.5 GB – 8 GB total | Small non-intensive apps, dev/test, cost-sensitive microservices. |
| | `E2` (Standard) | Dynamic Burst Intel / AMD | 2 to 32 vCPUs | 0.5 GB – 8 GB / vCPU | Web servers, dev environments, small/med databases. |
| | `N2` | Intel Ice Lake / Cascade Lake | Up to 128 vCPUs | 0.5 GB – 8 GB / vCPU | Enterprise applications, databases, general web serving. |
| | `N2D` | AMD EPYC Milan / Rome | Up to 224 vCPUs | 0.5 GB – 8 GB / vCPU | Scale-out enterprise workloads needing high core count. |
| | `Tau T2D` | AMD EPYC 3rd Gen | Up to 60 vCPUs | 4 GB / vCPU | Scale-out microservices, media transcoding, Java apps, GKE pools. |
| | `Tau T2A` | Arm Ampere Altra (3 GHz) | Up to 64 cores | 4 GB / core | Arm-native containerized workloads, scale-out web applications. |
| **Compute-Optimized** | `C2` | Intel Cascade Lake (3.8 GHz Turbo) | 4 to 60 vCPUs | Up to 240 GB total | AAA gaming servers, High Performance Computing (HPC), EDA. |
| | `C2D` | AMD EPYC Milan (High L3 Cache) | 2 to 112 vCPUs | 4 GB / vCPU (up to 3TB local NVMe) | HPC, EDA, compute-intensive simulation workloads. |
| | `H3` | Intel Sapphire Rapids + Google IPU | 88 cores | 352 GB DDR5 total | Large-scale scientific HPC & financial modeling. |
| **Memory-Optimized** | `M1` | Intel Xeon Skylake | Up to 160 vCPUs | Up to 4 TB total | In-memory databases, SAP HANA, memory-intensive analytics. |
| | `M2` | Intel Xeon Cascade Lake | Up to 416 vCPUs | Up to 12 TB total | Enterprise SAP HANA, massive in-memory data warehouses. |
| | `M3` | Intel Ice Lake | Up to 128 vCPUs | Up to 30.5 GB / vCPU | Genomic modeling, EDA, memory-heavy database nodes. |
| **Accelerator-Optimized**| `A2` | NVIDIA Ampere A100 GPUs (40GB) | 12 to 96 vCPUs | Up to 1,360 GB total | Deep learning ML training/inference, LLM fine-tuning (1-16 GPUs). |
| | `G2` | NVIDIA L4 GPUs / Cascade Lake | 4 to 96 vCPUs | Up to 432 GB total | CUDA ML inference, video transcoding, remote 3D visualization. |

---

## 2. Machine Type Selection (ASCII Decision Tree)

```
================================================================================
                    COMPUTE ENGINE MACHINE SELECTION TREE
================================================================================

                         What is your workload category?
                                       │
         ┌─────────────────────────────┼─────────────────────────────┬─────────────────────────────┐
         ▼                             ▼                             ▼                             ▼
  [ GENERAL-PURPOSE ]        [ COMPUTE-OPTIMIZED ]         [ MEMORY-OPTIMIZED ]        [ ACCELERATOR (GPU) ]
  Web Apps, Microservices,    High GHz Turbo, Gaming,       SAP HANA, In-Memory DBs,    AI/ML Training, CUDA,
  Dev/Test, Enterprise        HPC, Simulation, EDA          Genomic Modeling            LLM Inference, Transcoding
         │                             │                             │                             │
  ┌──────┴──────┐               ┌──────┴──────┐               ┌──────┴──────┐               ┌──────┴──────┐
  ▼             ▼               ▼             ▼               ▼             ▼               ▼             ▼
[ COST / E2 ] [ N2 / T2D ]   [ Intel C2 ]   [ AMD C2D ]    [ M1 / M2 ]    [ M3 IceLake ]  [ A2 (A100) ]  [ G2 (L4) ]
E2-Standard   N2D (224 core)  3.8GHz Turbo   Large L3 Cache  Up to 12TB     30.5 GB/vCPU    Large LLMs     ML Inference
Shared-Core   T2A (Arm)       Local NVMe     H3 (Sapphire)   SAP Certified  EDA & Genomics  1-16 GPUs      Transcoding
```

---

## 3. Custom Machine Types & Extended Memory Rules

If predefined machine shapes do not match your exact CPU-to-memory ratio requirements, Compute Engine permits provisioning **Custom Machine Types**.

### Core Constraints & Validation Rules:

| Constraint Category | Rule / Requirement | Technical Limit & Details |
| :--- | :--- | :--- |
| **vCPU Count** | Must be 1 vCPU or an even number | Supported values: `1`, `2`, `4`, `6`, `8`, `10`, `12` ... up to series maximum. |
| **Standard Memory Ratio** | 1 GB to 8 GB of memory per vCPU | Default allowable ratio without extended memory surcharges. |
| **Memory Increment** | Must be a multiple of 256 MB | Total memory string must align to `256MB` step boundary (e.g., `4096MB`, `6144MB`). |
| **Extended Memory** | > 8 GB of memory per vCPU | Allows adding extra RAM per core at an additional per-GB pricing rate. |

---

## 4. Specialized Provisioning Models & Security/Isolation Architecture

Compute Engine provides specialized **provisioning models** (Spot/Preemptible) and **security/isolation options** (Sole-Tenancy, Shielded VMs, Confidential VMs):

### A. Spot VMs vs. Preemptible VMs Comparison Matrix

| Property / Feature | Preemptible VMs (Legacy) | Spot VMs (Modern) | Standard VMs |
| :--- | :--- | :--- | :--- |
| **Cost Savings** | 60% – 91% discount | 60% – 91% discount (Same pricing as Spot) | Standard base rate / Committed use discount |
| **Maximum Runtime Limit** | **Strict 24-Hour Max** (Auto-preempted at 24h) | **No Maximum Runtime Limit** (Runs as long as capacity exists) | Unlimited runtime |
| **Preemption Notice** | 30-second ACPI shutdown signal | 30-second ACPI shutdown signal | N/A (Scheduled maintenance notice) |
| **First-Minute Guarantee** | No charge if preempted in 1st minute | No charge if preempted in 1st minute | Billed per second (1-min min) |
| **Live Migration & Auto-Restart** | **Disabled** | **Disabled** | **Enabled** (Host maintenance live-migrates VM) |
| **Fault-Tolerance Strategy** | Managed Instance Groups (MIGs) / Autoscalers auto-spin replacements | Managed Instance Groups (MIGs) / Autoscalers auto-spin replacements | Single instance HA or Regional MIGs |
| **CLI Flag** | `--preemptible` | `--provisioning-model=SPOT` | (Default provisioning model) |

### B. Sole-Tenant Nodes vs. Multi-Tenant Infrastructure

| Dimension | Multi-Tenant Host (Standard) | Sole-Tenant Host (Dedicated) |
| :--- | :--- | :--- |
| **Hardware Isolation** | Multiple VMs from different GCP projects share physical host server | Physical Compute Engine server host dedicated **exclusively to your project** |
| **Use Cases** | Standard web apps, microservices, dev/test | Strict compliance (PCI-DSS, HIPAA), BYOL OS licenses (Microsoft, Oracle) |
| **Capacity Allocation** | Automatically placed across Google data center pool | Manual / Automated Node Groups (`gcloud compute node-groups`) |
| **License Management** | Billed per-vCPU GCP license | In-Place Restart feature minimizes physical core billing for BYOL |

### C. Shielded VMs vs. Confidential VMs Security Matrix

| Security Feature | Shielded VMs | Confidential VMs |
| :--- | :--- | :--- |
| **Primary Threat Defended** | Bootkit & kernel-level malware / rootkits | Data exfiltration & memory inspection during active CPU execution |
| **Encryption Scope** | Verifiable boot chain integrity & secure storage | **Data In-Use Hardware Encryption** (RAM contents encrypted via AMD SEV) |
| **Core Technologies** | **1. Secure Boot**: Blocks unsigned bootloaders.<br/>**2. vTPM (Measured Boot)**: Stores boot hashes.<br/>**3. Integrity Monitoring**: Flags boot sequence drift. | **AMD Secure Encrypted Virtualization (SEV)** on `N2D` instances.<br/>Hardware keys generated per VM; Google has **zero key access**. |
| **Application Overhead** | Zero application code modifications required | Zero application code modifications required (Minimal performance penalty) |

---

## 5. Storage Architecture, Disk Types & Encryption Matrix

Compute Engine separates compute from storage using **network-attached block persistent disks** and **physically attached ephemeral local drives**:

### A. Block Storage & Disk Types Comparison Matrix

| Storage Category | Disk Type Flag | Underlying Medium | Max Size / Limit | Persistence & Durability | Primary Use Case |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Standard PD** | `pd-standard` | Standard HDD | 64 TB / disk | High Durability (Reboot/Stop/Delete resilient) | Large sequential batch I/O, cold data storage. |
| **Balanced PD** | `pd-balanced` | Solid-State Drive (SSD) | 64 TB / disk | High Durability (Reboot/Stop/Delete resilient) | General-purpose enterprise workloads, web apps. |
| **Performance SSD** | `pd-ssd` | Solid-State Drive (SSD) | 64 TB / disk | High Durability (Reboot/Stop/Delete resilient) | Low-latency databases, enterprise DB engines. |
| **Extreme PD** | `pd-extreme` | High-Perf SSD | Custom IOPS | High Durability (Provisioned IOPS up to 120k) | Mission-critical SAP HANA, Oracle, MS SQL Server. |
| **Regional PD** | `pd-balanced` / `pd-ssd` | Synchronous 2-Zone Replication | 64 TB / disk | High Availability (Active-Active 2-Zone Sync) | Zero-RPO database disaster recovery. |
| **Local SSD** | `--local-ssd` | Physically Attached NVMe | 375 GB / partition (Max 24 = 9 TB) | **Ephemeral** (Survives Reboot; **LOST on VM Stop**) | Ultra-high IOPS scratch space, cache, temp DBs. |
| **RAM Disk** | Linux `tmpfs` | Host Volatile Memory (RAM) | Capped by VM RAM | **Volatile** (LOST on Reboot / VM Stop) | Ultra-low latency transient data structures. |

### B. Disk Encryption at Rest Options

| Encryption Option | Managed By | Key Location | Setup Requirement |
| :--- | :--- | :--- | :--- |
| **GMEK** (Default) | Google Cloud | Google Internal KMS | Enabled automatically on all disks without extra cost or configuration. |
| **CMEK** (Customer-Managed) | Customer in Cloud KMS | Cloud Key Management Service | Pass `--kms-key=PROJECT_KMS_KEY_PATH` during disk/VM creation. |
| **CSEK** (Customer-Supplied) | Customer On-Premises | Customer raw 256-bit AES file | Pass `--csek-key-file=PATH_TO_KEY` per API request; Google never persists key. |

### C. Attachment Limits & Bandwidth Contention Rules

1. **Max Disk Attachment Limits**:
   - **Shared-Core VMs** (`e2-micro`, `e2-small`, `e2-medium`): Up to **16 persistent disks**.
   - **Standard, High-Memory, Compute, Memory, Accelerator VMs**: Up to **128 persistent disks**.
2. **Network Egress vs. Disk I/O Bandwidth Contention**:
   - Compute Engine allocates network egress bandwidth proportionally to vCPU count.
   - **Critical Nuance**: Disk write/read I/O throughput traffic shares the **same underlying physical vCPU egress bandwidth budget** as network egress traffic. Heavy disk I/O directly competes with network egress bandwidth!

---

## 6. Remote Access & SSH Strategy (ASCII Decision Tree)

```
================================================================================
                    REMOTE SSH & ACCESS STRATEGY DECISION TREE
================================================================================

                  How do you need to access the VM instance?
                                       │
                    Does the VM have a Public IPv4 Address?
                                       │
                        ┌──────────────┴──────────────┐
                        ▼                             ▼
                    [ YES ]                        [ NO ]
                        │                     Is IAP Enabled?
                        ▼                             │
            Direct SSH Connection              ┌──────┴──────┐
            gcloud compute ssh VM              ▼             ▼
                                            [ YES ]        [ NO ]
                                               │             │
                                               ▼             ▼
                                        Use IAP Tunnel   Use VPN / Bastion
                                        --tunnel-through-iap
```

---

## 7. Visual Mermaid Decision Flowcharts

### Storage Type & Encryption Selection Flowchart:
```mermaid
graph TD
    StorageStart["Select VM Storage Requirements"] --> DataDurability{"Durability & Persistence Required?"}
    
    DataDurability -->|Persistent Data (Survives Stop)| PDType{"Performance & IOPS Level?"}
    DataDurability -->|Ephemeral Scratch Space| SpeedCheck{"Speed vs Risk Tolerance?"}
    
    PDType -->|Lowest Cost / Batch HDD| StandardPD["pd-standard (Standard HDD)"]
    PDType -->|Balanced Price/Performance| BalancedPD["pd-balanced (SSD)"]
    PDType -->|High-Perf DB / Low Latency| SSDPD["pd-ssd (Performance SSD)"]
    PDType -->|Provisioned IOPS / Mission Critical| ExtremePD["pd-extreme (Provisioned IOPS up to 120k)"]
    PDType -->|Multi-Zone HA Disaster Recovery| RegionalPD["Regional Persistent Disk (2-Zone Synchronous Sync)"]
    
    SpeedCheck -->|Ultra High NVMe IOPS| LocalSSD["Local SSD (NVMe 375GB partitions, up to 9TB)"]
    SpeedCheck -->|Maximum Speed / In-Memory| RAMDisk["RAM Disk (tmpfs in system memory)"]
    
    StandardPD --> EncryptChoice{"Encryption Control Level?"}
    BalancedPD --> EncryptChoice
    SSDPD --> EncryptChoice
    ExtremePD --> EncryptChoice
    RegionalPD --> EncryptChoice
    
    EncryptChoice -->|Google Automated| GMEK["GMEK (Default Google-Managed)"]
    EncryptChoice -->|Cloud KMS Key Control| CMEK["CMEK (--kms-key=KEY_PATH)"]
    EncryptChoice -->|Raw Customer-Supplied Key| CSEK["CSEK (--csek-key-file=KEY_FILE)"]
```
