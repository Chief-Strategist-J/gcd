# Compute Engine: End-to-End Build, Configure & Deploy Operations Manual

This document is an exhaustive operational reference for the complete Google Compute Engine (GCE) Virtual Machine lifecycle. 

Every section provides:
1. **Command to Execute**
2. **Expected Terminal Output (What to read & look for in terminal)**
3. **How to Verify Configuration Correctness & Expected Verification Output**

---

## Table of Contents
1. [Phase 1: Building Custom OS Disk Images & Packer](#phase-1-building-custom-os-disk-images--packer)
2. [Phase 2: Configuring Instance Templates, VPC Subnets & Metadata](#phase-2-configuring-instance-templates-vpc-subnets--metadata)
3. [Phase 3: Deploying Managed Instance Groups (MIGs) & Rolling Updates](#phase-3-deploying-managed-instance-groups-migs--rolling-updates)
4. [Phase 4: Auto-Scaling, Load Balancers & Network Security](#phase-4-auto-scaling-load-balancers--network-security)
5. [Phase 5: Instance Lifecycle Management (Start, Stop, Reset, Destroy)](#phase-5-instance-lifecycle-management-start-stop-reset-destroy)
6. [Phase 6: Remote SSH Access, IAP Tunnels & Troubleshooting](#phase-6-remote-ssh-access-iap-tunnels--troubleshooting)
7. [Phase 7: Professional Failure Diagnosis & Resolution Matrix](#phase-7-professional-failure-diagnosis--resolution-matrix)

---

## Phase 1: Building Custom OS Disk Images & Packer

### 1. Create Custom Golden Disk Image from Source VM

```bash
# 1. Stop source VM to ensure disk consistency
gcloud compute instances stop Golden-VM-Template --zone=us-central1-a

# 2. Create Machine Image
gcloud compute machine-images create gold-image-v2-4 \
    --source-instance=Golden-VM-Template \
    --source-instance-zone=us-central1-a
```

#### Expected Terminal Output:
```text
Stopping instance [Golden-VM-Template]...done.
Updated [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/zones/us-central1-a/instances/Golden-VM-Template].
Created machine image [gold-image-v2-4].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute machine-images describe gold-image-v2-4 --format="yaml(name, status, totalStorageBytes)"
```

#### Expected Verification Output:
```yaml
name: gold-image-v2-4
status: READY
totalStorageBytes: '4891283456'
```

---

## Phase 2: Configuring Instance Templates, VPC Subnets & Metadata

### 1. Create Reusable Instance Template with Startup Script

```bash
gcloud compute instance-templates create prod-web-template-v2 \
    --machine-type=n2-standard-4 \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=50GB \
    --boot-disk-type=pd-ssd \
    --tags=http-server,https-server \
    --labels=env=production,team=ops \
    --metadata-from-file=startup-script=./scripts/install_nginx.sh \
    --scopes=cloud-platform \
    --shielded-secure-boot
```

#### Expected Terminal Output:
```text
Created instance template [prod-web-template-v2].
NAME                 MACHINE_TYPE   PREEMPTIBLE  CREATION_TIMESTAMP
prod-web-template-v2  n2-standard-4               2026-09-12T13:40:12.123-07:00
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute instance-templates describe prod-web-template-v2 \
    --format="yaml(name, properties.machineType, properties.tags)"
```

#### Expected Verification Output:
```yaml
name: prod-web-template-v2
properties:
  machineType: n2-standard-4
  tags:
    items:
    - http-server
    - https-server
```

---

## Phase 3: Deploying Managed Instance Groups (MIGs) & Rolling Updates

### 1. Provision Regional Managed Instance Group (MIG)

```bash
gcloud compute instance-groups managed create prod-web-mig \
    --template=prod-web-template-v2 \
    --size=3 \
    --region=us-central1 \
    --instance-redistribution-type=PROACTIVE
```

#### Expected Terminal Output:
```text
Created instance group [prod-web-mig].
NAME          LOCATION     SCOPE   REGION       BASE_INSTANCE_NAME  SIZE  TARGET_SIZE  INSTANCE_TEMPLATE     AUTOSCALED
prod-web-mig  us-central1  region  us-central1  prod-web-mig        0     3            prod-web-template-v2  no
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute instance-groups managed list-instances prod-web-mig --region=us-central1
```

#### Expected Verification Output:
```text
NAME               ZONE           STATUS   HEALTH_STATE  ACTION  INSTANCE_TEMPLATE
prod-web-mig-a1b2  us-central1-a  RUNNING  HEALTHY       NONE    prod-web-template-v2
prod-web-mig-c3d4  us-central1-b  RUNNING  HEALTHY       NONE    prod-web-template-v2
prod-web-mig-e5f6  us-central1-c  RUNNING  HEALTHY       NONE    prod-web-template-v2
```

---

### 2. Perform Zero-Downtime Rolling Update of MIG

```bash
gcloud compute instance-groups managed rolling-action start-update prod-web-mig \
    --version=template=prod-web-template-v3 \
    --max-surge=1 \
    --max-unavailable=0 \
    --region=us-central1
```

#### Expected Terminal Output:
```text
Started rolling update of instance group [prod-web-mig].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute instance-groups managed describe prod-web-mig --region=us-central1 --format="yaml(status)"
```

#### Expected Verification Output:
```yaml
status:
  isStable: true
  versionTarget:
    isReached: true
```

---

## Phase 4: Auto-Scaling, Load Balancers & Network Security

### 1. Configure Target CPU Autoscaling Policy

```bash
gcloud compute instance-groups managed set-autoscaling prod-web-mig \
    --region=us-central1 \
    --min-num-replicas=3 \
    --max-num-replicas=20 \
    --target-cpu-utilization=0.75 \
    --cool-down-period=90
```

#### Expected Terminal Output:
```text
Updated autoscaler [prod-web-mig-autoscaler].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute instance-groups managed describe-autoscaler prod-web-mig --region=us-central1
```

#### Expected Verification Output:
```yaml
autoscalingPolicy:
  coolDownPeriodSec: 90
  cpuUtilization:
    utilizationTarget: 0.75
  maxNumReplicas: 20
  minNumReplicas: 3
status: ACTIVE
```

---

## Phase 5: Instance Lifecycle Management (Start, Stop, Reset, Destroy)

### 1. Stop, Start, Reset and Delete VM Instance

```bash
# Stop VM instance
gcloud compute instances stop prod-web-vm1 --zone=us-central1-a

# Start VM instance
gcloud compute instances start prod-web-vm1 --zone=us-central1-a

# Soft-reset VM instance (reboot)
gcloud compute instances reset prod-web-vm1 --zone=us-central1-a

# Terminate and destroy VM instance
gcloud compute instances delete prod-web-vm1 --zone=us-central1-a --quiet
```

#### Expected Terminal Output (Stop Command):
```text
Stopping instance [prod-web-vm1]...done.
Updated [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/zones/us-central1-a/instances/prod-web-vm1].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute instances describe prod-web-vm1 --zone=us-central1-a --format="value(status)"
```

#### Expected Verification Output:
```text
TERMINATED
```

---

## Phase 6: Remote SSH Access, IAP Tunnels & Troubleshooting

### 1. SSH via Identity-Aware Proxy (IAP) Tunnel & Serial Port Inspection

```bash
# Connect to private instance without external IP
gcloud compute ssh private-web-vm --zone=us-central1-a --tunnel-through-iap

# Retrieve serial console boot logs
gcloud compute instances get-serial-port-output private-web-vm --zone=us-central1-a | tail -n 25
```

#### Expected Terminal Output (Serial Console Output):
```text
[   14.891230] cloud-init[982]: Cloud-init v. 23.4 finished at Sat, 12 Sep 2026 13:50:11 +0000. Datasource DataSourceGCE.
[   15.012391] systemd[1]: Started Nginx HTTP Server.
```

---

## Phase 7: Professional Failure Diagnosis & Resolution Matrix

| Phase | Error Code / Status | Root Cause | Diagnosis Command | Immediate Resolution Command |
| :--- | :--- | :--- | :--- | :--- |
| **Build** | **`IMAGE_CREATION_FAILED`** | Source instance running during image creation | `gcloud compute instances describe VM` | Stop instance before creating image: `gcloud compute instances stop VM`. |
| **Config**| **`QUOTA_EXCEEDED (CPUS)`** | Regional vCPU quota limit reached | `gcloud compute regions describe REGION --format="value(quotas)"` | Reduce machine size or request quota increase in GCP Console. |
| **Config**| **`ZONE_RESOURCE_POOL_EXHAUSTED`** | Hardware capacity exhausted in target zone | `gcloud compute instances create VM --zone=us-central1-a` | Retry deployment in adjacent zone: `--zone=us-central1-b`. |
| **Deploy**| **`MIG_INSTANCE_CREATION_FAILED`** | Template startup script failed on boot | `gcloud compute instance-groups managed list-errors MIG` | Fix startup script syntax or update Instance Template (`gcloud compute instance-templates create`). |
| **Deploy**| **`SSH Connection Timed Out`** | Firewall blocking port 22 or no public IP | `gcloud compute instances describe VM` | Connect via IAP: `gcloud compute ssh VM --tunnel-through-iap`. |
| **Resize**| **`Disk size expanded in CLI but not OS`** | Partition table not expanded in OS | `gcloud compute ssh VM --command="df -h"` | Run filesystem expansion inside VM: `sudo resize2fs /dev/sda1`. |
| **IAM**   | **`403 Forbidden (compute.setMachineType)`** | User lacks `compute.instanceAdmin.v1` role | `gcloud config get-value account` | Grant role: `gcloud projects add-iam-policy-binding PROJECT --member="USER" --role="roles/compute.instanceAdmin.v1"`. |
| **MIG**   | **`Rolling update stalled`** | New template fails health check probes | `gcloud compute instance-groups managed describe MIG` | Stop update and rollback template version: `gcloud compute instance-groups managed rolling-action start-update MIG --version=template=PREV_TEMPLATE`. |
