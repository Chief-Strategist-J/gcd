# Google Cloud & Kubernetes Infrastructure Master Reference Index

Welcome to the **Google Cloud & Kubernetes Infrastructure Master Reference Index**. Every feature is organized into a dedicated directory containing separate files for **Architecture Design (HLD/LLD)**, **ASCII Decision Trees**, **Shell Command References & Error Matrix**, and **Official Reference Links**.

---

## Directory & File Structure

```
gcd/
├── compute-engine/                   # Feature 1: Compute Engine (Virtual Machines)
│   ├── README.md                     # Compute Engine Feature Index
│   ├── hld-lld-design.md             # High-Level Architecture & Low-Level State Machine
│   ├── decision-tree.md              # ASCII & Visual Decision Trees (Machine Specs & SSH)
│   ├── shell-commands.md             # In-Depth CLI Manual & Error Matrix (`gcloud compute`)
│   └── references.md                 # Official GCP Documentation Links & Best Practices
│
├── cloud-storage/                    # Feature 2: Cloud Storage (Buckets & Objects)
│   ├── README.md                     # Cloud Storage Feature Index
│   ├── hld-lld-design.md             # High-Level Architecture & Resumable Upload Sequence Flow
│   ├── decision-tree.md              # ASCII & Visual Decision Trees (Storage Class & Location)
│   ├── shell-commands.md             # In-Depth CLI Manual & Error Matrix (`gcloud storage`)
│   └── references.md                 # Official GCP Documentation Links & Best Practices
│
├── kubernetes/                       # Feature 3: Kubernetes & kubectl (Container Orchestration)
│   ├── README.md                     # Kubernetes Feature Index
│   ├── hld-lld-design.md             # Control Plane Architecture & Pod Lifecycle State Machine
│   ├── decision-tree.md              # ASCII & Visual Decision Trees (Workload Controllers & Services)
│   ├── shell-commands.md             # In-Depth CLI Manual & Error Matrix (`kubectl`)
│   └── references.md                 # Official Kubernetes Documentation Links & Best Practices
│
└── README.md                         # Master Workspace Index (This File)
```

---

## Feature Index Links

### 🖥️ Feature 1: Compute Engine (`/compute-engine/`)
* 📐 **[HLD & LLD Design (`compute-engine/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/hld-lld-design.md)**
* 🌳 **[ASCII & Visual Decision Trees (`compute-engine/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/decision-tree.md)**
* 💻 **[Shell Command Reference & Failure Resolutions (`compute-engine/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/shell-commands.md)**
* 🔗 **[Official References & Links (`compute-engine/references.md`)](file:///home/btpl-lap-22/live/gcd/compute-engine/references.md)**

---

### 🪣 Feature 2: Cloud Storage (`/cloud-storage/`)
* 📐 **[HLD & LLD Design (`cloud-storage/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/cloud-storage/hld-lld-design.md)**
* 🌳 **[ASCII & Visual Decision Trees (`cloud-storage/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/cloud-storage/decision-tree.md)**
* 💻 **[Shell Command Reference & Failure Resolutions (`cloud-storage/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/cloud-storage/shell-commands.md)**
* 🔗 **[Official References & Links (`cloud-storage/references.md`)](file:///home/btpl-lap-22/live/gcd/cloud-storage/references.md)**

---

### ☸️ Feature 3: Kubernetes & kubectl (`/kubernetes/`)
* 📐 **[HLD & LLD Design (`kubernetes/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/kubernetes/hld-lld-design.md)**
* 🌳 **[ASCII & Visual Decision Trees (`kubernetes/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/kubernetes/decision-tree.md)**
* 💻 **[Shell Command Reference & Failure Resolutions (`kubernetes/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/kubernetes/shell-commands.md)**
* 🔗 **[Official References & Links (`kubernetes/references.md`)](file:///home/btpl-lap-22/live/gcd/kubernetes/references.md)**
