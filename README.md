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
│   ├── casestudy/                    # Deep-Dive Case Study Module Index & Reports
│   │   ├── README.md                 # Case Study Sitemap
│   │   ├── mig-autoscaling-loadbalancing.md # Case Study 1: MIGs, Autoscaling & Load Balancing
│   │   ├── alb-global-routing-in-action.md  # Case Study 2: Global ALB Routing, Capacity & Failover
│   │   ├── cloud-cdn-edge-caching.md        # Case Study 3: Cloud CDN Edge Caching & Cache Modes
│   │   ├── layer4-network-load-balancers.md # Case Study 4: Layer 4 NLBs (Proxy vs Passthrough)
│   │   ├── internal-load-balancers-3tier-architecture.md # Case Study 5: Internal Load Balancers & 3-Tier Architecture
│   │   ├── internal-passthrough-nlb-hands-on-lab.md      # Case Study 6: Multi-Zone Internal Passthrough NLB Lab
│   │   └── load-balancer-selection-decision-matrix.md   # Case Study 7: Load Balancer Selection Decision Tree & Schemes
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
│   ├── networking-shell-command.md   # Unified Networking Manual: Docker, CNI & GCP VPC Underlay
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
├── cloud-sql/                        # Feature 7: Cloud SQL & Relational Databases
│   ├── README.md                     # Cloud SQL Feature Index
│   ├── hld-lld-design.md             # Regional HA Failover & Auth Proxy Architecture
│   ├── decision-tree.md              # Database Selection & Connection Strategy Decision Trees
│   ├── shell-commands.md             # In-Depth CLI Manual & Error Matrix (`gcloud sql`)
│   └── references.md                 # Official Cloud SQL Documentation Links & Best Practices
│
├── firestore/                        # Feature 8: Google Cloud Firestore (NoSQL Document DB)
│   ├── README.md                     # Firestore Feature Index
│   ├── hld-lld-design.md             # Multi-Region Replication & Live Sync Architecture
│   ├── decision-tree.md              # Database & Operating Mode Selection Trees
│   ├── shell-commands.md             # In-Depth CLI Manual & Error Matrix (`gcloud firestore`)
│   └── references.md                 # Official Firestore Documentation Links & Best Practices
│
├── cloud-run-functions/              # Feature 9: Cloud Run Functions (Serverless Compute)
│   ├── README.md                     # Cloud Run Functions Feature Index
│   ├── casestudy/                    # Deep-Dive Case Studies & Hands-On Engineering Labs
│   │   ├── README.md                 # Case Studies & Hands-On Labs Sitemap
│   │   ├── http-and-cloud-storage-event-functions-lab.md # Lab 1: HTTP & GCS Event Functions with Revisions
│   │   └── vpc-connector-redis-and-internal-vm-lab.md   # Lab 2: VPC Connector, Redis & Internal VM
│   ├── hld-lld-design.md             # Serverless Architecture & Buildpack Containerization
│   ├── triggers-vpc-workflows.md     # In-Depth Triggers, VPC Networking & Workflows Architecture
│   ├── decision-tree.md              # Source Location & Deployment Tooling Decision Trees
│   ├── shell-commands.md             # In-Depth CLI Manual & Error Matrix (`gcloud functions`)
│   └── references.md                 # Official Cloud Run Functions Documentation Links
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
* **[Case Study Index (`compute-engine/casestudy/README.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/README.md)**
* **[Case Study 1: MIGs, Autoscaling & Load Balancing (`compute-engine/casestudy/mig-autoscaling-loadbalancing.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/mig-autoscaling-loadbalancing.md)**
* **[Case Study 2: Global ALB Routing in Action (`compute-engine/casestudy/alb-global-routing-in-action.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/alb-global-routing-in-action.md)**
* **[Case Study 3: Cloud CDN Edge Caching & Cache Modes (`compute-engine/casestudy/cloud-cdn-edge-caching.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/cloud-cdn-edge-caching.md)**
* **[Case Study 4: Layer 4 NLBs (Proxy vs Passthrough) (`compute-engine/casestudy/layer4-network-load-balancers.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/layer4-network-load-balancers.md)**
* **[Case Study 5: Internal Load Balancers & 3-Tier (`compute-engine/casestudy/internal-load-balancers-3tier-architecture.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/internal-load-balancers-3tier-architecture.md)**
* **[Case Study 6: Multi-Zone Internal Passthrough NLB Lab (`compute-engine/casestudy/internal-passthrough-nlb-hands-on-lab.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/internal-passthrough-nlb-hands-on-lab.md)**
* **[Case Study 7: Load Balancer Selection Decision Matrix (`compute-engine/casestudy/load-balancer-selection-decision-matrix.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/casestudy/load-balancer-selection-decision-matrix.md)**
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
* **[Unified Networking Manual: Docker, CNI & GCP VPC (`kubernetes/networking-shell-command.md`)](file:///home/btpl-lap-22/live/gcd/kubernetes/networking-shell-command.md)**
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

---

### Feature 7: Cloud SQL & Relational Databases (`/cloud-sql/`)
* **[HLD & LLD Design (`cloud-sql/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/cloud-sql/hld-lld-design.md)**
* **[ASCII & Visual Decision Trees (`cloud-sql/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/cloud-sql/decision-tree.md)**
* **[Shell Command Reference & Verification Manual (`cloud-sql/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/cloud-sql/shell-commands.md)**
* **[Official References & Links (`cloud-sql/references.md`)](file:///home/btpl-lap-22/live/gcd/cloud-sql/references.md)**

---

### Feature 8: Google Cloud Firestore (`/firestore/`)
* **[HLD & LLD Design (`firestore/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/firestore/hld-lld-design.md)**
* **[ASCII & Visual Decision Trees (`firestore/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/firestore/decision-tree.md)**
* **[Shell Command Reference & Verification Manual (`firestore/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/firestore/shell-commands.md)**
* **[Official References & Links (`firestore/references.md`)](file:///home/btpl-lap-22/live/gcd/firestore/references.md)**

---

### Feature 9: Cloud Run Functions (`/cloud-run-functions/`)
* **[Case Studies & Hands-On Labs Index (`cloud-run-functions/casestudy/README.md`)](file:///home/btpl-lap-22/live/gcd/cloud-run-functions/casestudy/README.md)**
* **[Lab 1: HTTP & GCS Event Functions with Revisions (`cloud-run-functions/casestudy/http-and-cloud-storage-event-functions-lab.md`)](file:///home/btpl-lap-22/live/gcd/cloud-run-functions/casestudy/http-and-cloud-storage-event-functions-lab.md)**
* **[Lab 2: VPC Connector, Memorystore Redis & Internal VM (`cloud-run-functions/casestudy/vpc-connector-redis-and-internal-vm-lab.md`)](file:///home/btpl-lap-22/live/gcd/cloud-run-functions/casestudy/vpc-connector-redis-and-internal-vm-lab.md)**
* **[HLD & LLD Design (`cloud-run-functions/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/cloud-run-functions/hld-lld-design.md)**
* **[Triggers, VPC Networking & Workflows Manual (`cloud-run-functions/triggers-vpc-workflows.md`)](file:///home/btpl-lap-22/live/gcd/cloud-run-functions/triggers-vpc-workflows.md)**
* **[ASCII & Visual Decision Trees (`cloud-run-functions/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/cloud-run-functions/decision-tree.md)**
* **[Shell Command Reference & Failure Resolutions (`cloud-run-functions/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/cloud-run-functions/shell-commands.md)**
* **[Official References & Links (`cloud-run-functions/references.md`)](file:///home/btpl-lap-22/live/gcd/cloud-run-functions/references.md)**



