# Compute Engine: End-to-End Build, Configure & Deploy Manual

This document provides a production-grade, end-to-end reference for the complete VM lifecycle: **Phase 1: Building Custom Disk Images & Packer** $\rightarrow$ **Phase 2: Configuring Instance Templates & Subnets** $\rightarrow$ **Phase 3: Deploying Managed Instance Groups (MIGs) & Rolling Updates** $\rightarrow$ **Phase 4: Auto-Scaling & Load Balancing** $\rightarrow$ **Phase 5: Error Diagnosis & Failure Resolution Matrix**.

---

## Table of Contents
1. [Phase 1: Building Custom OS Disk Images & Packer](#phase-1-building-custom-os-disk-images--packer)
2. [Phase 2: Configuring Instance Templates, VPC Subnets & Metadata](#phase-2-configuring-instance-templates-vpc-subnets--metadata)
3. [Phase 3: Deploying Managed Instance Groups (MIGs) & Rolling Updates](#phase-3-deploying-managed-instance-groups-migs--rolling-updates)
4. [Phase 4: Auto-Scaling, Load Balancers & Network Security](#phase-4-auto-scaling-load-balancers--network-security)
5. [Phase 5: Remote SSH Access, IAP Tunnels & Troubleshooting](#phase-5-remote-ssh-access-iap-tunnels--troubleshooting)
6. [Phase 6: Professional Failure Diagnosis & Resolution Matrix](#phase-6-professional-failure-diagnosis--resolution-matrix)

---

## Phase 1: Building Custom OS Disk Images & Packer

### 1. Create Custom Golden Disk Image from Source VM
```bash
# 1. Stop source VM to ensure disk consistency
gcloud compute instances stop Golden-VM-Template --zone=us-central1-a

# 2. Create versioned Machine Image
gcloud compute machine-images create gold-image-v2-4 \
    --source-instance=Golden-VM-Template \
    --source-instance-zone=us-central1-a
```

### 2. Export & Import Machine Images
```bash
# Export custom disk image to Cloud Storage bucket
gcloud compute images export \
    --image=gold-image-v2-4 \
    --destination-uri=gs://my-image-bucket/gold-image-v2-4.tar.gz
```

---

## Phase 2: Configuring Instance Templates, VPC Subnets & Metadata

### 1. Create Reusable Instance Template
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

---

## Phase 3: Deploying Managed Instance Groups (MIGs) & Rolling Updates

### 1. Create Regional Managed Instance Group (MIG)
```bash
gcloud compute instance-groups managed create prod-web-mig \
    --template=prod-web-template-v2 \
    --size=3 \
    --region=us-central1 \
    --instance-redistribution-type=PROACTIVE
```

### 2. Zero-Downtime Rolling Update of Instance Group
```bash
gcloud compute instance-groups managed rolling-action start-update prod-web-mig \
    --version=template=prod-web-template-v3 \
    --max-surge=25% \
    --max-unavailable=0 \
    --region=us-central1
```

---

## Phase 4: Auto-Scaling, Load Balancers & Network Security

### 1. Configure Autoscaling Policy on Instance Group
```bash
gcloud compute instance-groups managed set-autoscaling prod-web-mig \
    --region=us-central1 \
    --min-num-replicas=3 \
    --max-num-replicas=20 \
    --target-cpu-utilization=0.75 \
    --cool-down-period=90
```

### 2. Attach Instance Group to HTTP Load Balancer Backend
```bash
gcloud compute backend-services add-backend prod-web-backend \
    --instance-group=prod-web-mig \
    --instance-group-region=us-central1 \
    --global
```

---

## Phase 5: Remote SSH Access, IAP Tunnels & Troubleshooting

```bash
# SSH into Private VM via IAP Tunnel (No Public IP!)
gcloud compute ssh private-vm --zone=us-central1-a --tunnel-through-iap

# Fetch Kernel Boot Logs (Troubleshoot boot failures)
gcloud compute instances get-serial-port-output prod-web-vm --zone=us-central1-a
```

---

## Phase 6: Professional Failure Diagnosis & Resolution Matrix

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
