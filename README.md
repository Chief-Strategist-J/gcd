# Google Cloud & Kubernetes Infrastructure Master Reference Index

Welcome to the **Google Cloud & Kubernetes Infrastructure Master Reference Index**. Every feature is organized into a dedicated directory containing separate files for **Architecture Design (HLD/LLD)**, **ASCII Decision Trees**, **Shell Command References & Error Matrix**, and **Official Reference Links**.

---

## Directory & File Structure

```
gcd/
├── gcp-config-project/               # Feature 1: GCP Core Config, Auth & Project Hierarchy
│   ├── README.md                     # Core Config Feature Index
│   ├── hld-lld-design.md             # Resource Hierarchy & Auth Model Architecture
│   ├── decision-tree.md              # Auth Method & Project Hierarchy Decision Trees
│   ├── shell-commands.md             # In-Depth CLI Manual & Error Matrix (`gcloud config/auth/projects`)
│   └── references.md                 # Official GCP Config & IAM Documentation Links
│
├── compute-engine/                   # Feature 2: Compute Engine (Virtual Machines)
│   ├── README.md                     # Compute Engine Feature Index
│   ├── hld-lld-design.md             # High-Level Architecture & Low-Level State Machine
│   ├── decision-tree.md              # ASCII & Visual Decision Trees (Machine Specs & SSH)
│   ├── shell-commands.md             # In-Depth CLI Manual & Error Matrix (`gcloud compute`)
│   └── references.md                 # Official GCP Documentation Links & Best Practices
│
├── cloud-storage/                    # Feature 3: Cloud Storage (Buckets & Objects)
│   ├── README.md                     # Cloud Storage Feature Index
│   ├── hld-lld-design.md             # High-Level Architecture & Resumable Upload Sequence Flow
│   ├── decision-tree.md              # ASCII & Visual Decision Trees (Storage Class & Location)
│   ├── shell-commands.md             # In-Depth CLI Manual & Error Matrix (`gcloud storage`)
│   └── references.md                 # Official GCP Documentation Links & Best Practices
│
├── kubernetes/                       # Feature 4: Kubernetes & kubectl (Container Orchestration)
│   ├── README.md                     # Kubernetes Feature Index
│   ├── hld-lld-design.md             # Control Plane Architecture & Pod Lifecycle State Machine
│   ├── decision-tree.md              # ASCII & Visual Decision Trees (Workload Controllers & Services)
│   ├── shell-commands.md             # In-Depth CLI Manual & Error Matrix (`kubectl`)
│   └── references.md                 # Official Kubernetes Documentation Links & Best Practices
│
├── bigquery/                         # Feature 5: BigQuery (Enterprise Data Warehouse)
│   ├── README.md                     # BigQuery Feature Index
│   ├── hld-lld-design.md             # Dremel Engine Architecture & Capacitor Storage Mechanics
│   ├── decision-tree.md              # ASCII & Visual Decision Trees (Partitioning vs Clustering)
│   ├── shell-commands.md             # In-Depth CLI Manual & Error Matrix (`bq`)
│   └── references.md                 # Official BigQuery Documentation Links & SQL Best Practices
│
├── vpc-network/                      # Feature 6: VPC Networks & Subnets (Global Networking)
│   ├── README.md                     # VPC Network Feature Index
│   ├── hld-lld-design.md             # Global VPC Topology & 4 Reserved IP Addresses Architecture
│   ├── decision-tree.md              # Auto vs Custom Mode & Non-Downtime CIDR Expansion Trees
│   ├── shell-commands.md             # In-Depth CLI Manual & Error Matrix (`gcloud compute networks/subnets`)
│   └── references.md                 # Official VPC Documentation Links & RFC Standards
│
└── README.md                         # Master Workspace Index (This File)
```

---

## Feature Index Links

### Feature 1: GCP Core Config & Project Management (`/gcp-config-project/`)
* **[HLD & LLD Design (`gcp-config-project/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/gcp-config-project/hld-lld-design.md)**
* **[ASCII & Visual Decision Trees (`gcp-config-project/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/gcp-config-project/decision-tree.md)**
* **[Shell Command Reference & Verification Manual (`gcp-config-project/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/gcp-config-project/shell-commands.md)**
* **[Official References & Links (`gcp-config-project/references.md`)](file:///home/btpl-lap-22/live/gcd/gcp-config-project/references.md)**

---

### Feature 2: Compute Engine (`/compute-engine/`)
* **[HLD & LLD Design (`compute-engine/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/hld-lld-design.md)**
* **[ASCII & Visual Decision Trees (`compute-engine/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/decision-tree.md)**
* **[Shell Command Reference & Failure Resolutions (`compute-engine/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/shell-commands.md)**
* **[Official References & Links (`compute-engine/references.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/references.md)**

---

### Feature 3: Cloud Storage (`/cloud-storage/`)
* **[HLD & LLD Design (`cloud-storage/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/cloud-storage/hld-lld-design.md)**
* **[ASCII & Visual Decision Trees (`cloud-storage/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/cloud-storage/decision-tree.md)**
* **[Shell Command Reference & Failure Resolutions (`cloud-storage/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/cloud-storage/shell-commands.md)**
* **[Official References & Links (`cloud-storage/references.md`)](file:///home/btpl-lap-22/live/gcd/cloud-storage/references.md)**

---

### Feature 4: Kubernetes & kubectl (`/kubernetes/`)
* **[HLD & LLD Design (`kubernetes/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/kubernetes/hld-lld-design.md)**
* **[ASCII & Visual Decision Trees (`kubernetes/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/kubernetes/decision-tree.md)**
* **[Shell Command Reference & Failure Resolutions (`kubernetes/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/kubernetes/shell-commands.md)**
* **[Official References & Links (`kubernetes/references.md`)](file:///home/btpl-lap-22/live/gcd/kubernetes/references.md)**

---

### Feature 5: BigQuery (`/bigquery/`)
* **[HLD & LLD Design (`bigquery/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/bigquery/hld-lld-design.md)**
* **[ASCII & Visual Decision Trees (`bigquery/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/bigquery/decision-tree.md)**
* **[Shell Command Reference & Failure Resolutions (`bigquery/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/bigquery/shell-commands.md)**
* **[Official References & Links (`bigquery/references.md`)](file:///home/btpl-lap-22/live/gcd/bigquery/references.md)**

---

### Feature 6: VPC Networks & Subnets (`/vpc-network/`)
* **[HLD & LLD Design (`vpc-network/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/vpc-network/hld-lld-design.md)**
* **[ASCII & Visual Decision Trees (`vpc-network/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/vpc-network/decision-tree.md)**
* **[Shell Command Reference & Verification Manual (`vpc-network/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/vpc-network/shell-commands.md)**
* **[Official References & Links (`vpc-network/references.md`)](file:///home/btpl-lap-22/live/gcd/vpc-network/references.md)**
