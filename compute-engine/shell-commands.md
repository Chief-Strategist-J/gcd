# Compute Engine: End-to-End Build, Configure & Deploy Operations Manual

This document is an exhaustive operational reference for the complete Google Compute Engine (GCE) Virtual Machine lifecycle. 

Every section provides:
1. **Command to Execute**3. **How to Verify Configuration Correctness & Expected Verification Output**

---

## Table of Contents
1. [Phase 1: Building Custom OS Disk Images & Packer](#phase-1-building-custom-os-disk-images--packer)
2. [Phase 2: Configuring Instance Templates, VPC Subnets & Metadata](#phase-2-configuring-instance-templates-vpc-subnets--metadata)
   * [1. Create Reusable Instance Template with Startup Script](#1-create-reusable-instance-template-with-startup-script)
   * [2. Multi-Interface VM Provisioning: Console, CLI & RESTful API Payload Generation](#2-multi-interface-vm-provisioning-console-cli--restful-api-payload-generation)
   * [3. Provisioning Specialized & Custom VMs (Arm T2A, Extended Memory & GPUs)](#3-provisioning-specialized--custom-vms-arm-t2a-extended-memory--gpus)
   * [4. Provisioning Spot VMs, Sole-Tenant Nodes, Shielded VMs & Confidential VMs](#4-provisioning-spot-vms-sole-tenant-nodes-shielded-vms--confidential-vms)
   * [5. Block Storage Operations: Persistent Disks, Local SSDs, Online Resizing & Encryption](#5-block-storage-operations-persistent-disks-local-ssds-online-resizing--encryption)
   * [6. Operational Workflows: Metadata Server Querying, Relocation & Snapshot Schedules](#6-operational-workflows-metadata-server-querying-relocation--snapshot-schedules)
   * [7. Dedicated Stateful Application Server Blueprint (Formatting, Screen, GCS Backup & Cron)](#7-dedicated-stateful-application-server-blueprint-formatting-screen-gcs-backup--cron)
3. [Phase 3: Deploying Managed Instance Groups (MIGs) & Rolling Updates](#phase-3-deploying-managed-instance-groups-migs--rolling-updates)
4. [Phase 4: Auto-Scaling, Load Balancers & Network Security](#phase-4-auto-scaling-load-balancers--network-security)
5. [Phase 5: Instance Lifecycle Management, RAM Modification & Hardware Inspection](#phase-5-instance-lifecycle-management-start-stop-reset-destroy)
   * [1. Stop, Start, Reset and Delete VM Instance](#1-stop-start-reset-and-delete-vm-instance)
   * [2. Update Machine Type & RAM Memory (Pre-defined & Custom VM)](#2-update-machine-type--ram-memory-pre-defined--custom-vm)
   * [3. Inspect VM RAM & Hardware inside Linux OS (SSH)](#3-inspect-vm-ram--hardware-inside-linux-os-ssh)
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

### 2. Multi-Interface VM Provisioning: Console, CLI & RESTful API Payload Generation

GCP Compute Engine instances can be created using **three provisioning interfaces**:
1. **Google Cloud Console**: Interactive visual builder (provides dropdown options and prevents typos; includes "EQUIVALENT CODE" link to export gcloud / REST API commands).
2. **Cloud Shell CLI**: Declarative `gcloud compute instances create` command execution.
3. **RESTful API**: Programmatic HTTP `POST` requests to `compute.googleapis.com/compute/v1/projects/{PROJECT}/zones/{ZONE}/instances`.

#### Exporting Equivalent REST API Payload & gcloud Command from Console:

When configuring an instance in Console, click **"CommandLine and REST"** at the bottom of the page to inspect or export the equivalent REST JSON body:

```json
{
  "name": "utility-app-api",
  "machineType": "zones/us-central1-a/machineTypes/n2-standard-4",
  "disks": [
    {
      "boot": true,
      "autoDelete": true,
      "initializeParams": {
        "sourceImage": "projects/ubuntu-os-cloud/global/images/family/ubuntu-2204-lts",
        "diskSizeGb": "50",
        "diskType": "zones/us-central1-a/diskTypes/pd-ssd"
      }
    }
  ],
  "networkInterfaces": [
    {
      "network": "global/networks/default",
      "accessConfigs": [
        {
          "name": "External NAT",
          "type": "ONE_TO_ONE_NAT"
        }
      ]
    }
  ]
}
```

#### Executing REST API Request via `curl`:
```bash
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d @instance_request.json \
  "https://compute.googleapis.com/compute/v1/projects/YOUR_PROJECT_ID/zones/us-central1-a/instances"
```

---

### 3. Provisioning Specialized & Custom VMs (Arm T2A, Extended Memory & GPUs)

#### Example A: Provision Arm-based Tau T2A Instance (Ampere Altra 64-Core CPU)
```bash
gcloud compute instances create arm-web-vm \
    --zone=us-central1-a \
    --machine-type=t2a-standard-4 \
    --image-family=ubuntu-2204-lts-arm64 \
    --image-project=ubuntu-os-cloud
```

#### Example B: Provision Custom VM with Extended Memory (>8 GB RAM per vCPU)
```bash
# Provision custom 4 vCPU VM with 40 GB RAM (requires --custom-extensions=extended-memory)
gcloud compute instances create custom-ext-ram-vm \
    --zone=us-central1-a \
    --custom-cpu=4 \
    --custom-memory=40GB \
    --custom-extensions=extended-memory \
    --image-family=debian-12 \
    --image-project=debian-cloud
```

#### Example C: Provision Accelerator-Optimized VM with NVIDIA L4 GPU (G2 Series)
```bash
gcloud compute instances create ml-inference-vm \
    --zone=us-central1-a \
    --machine-type=g2-standard-4 \
    --accelerator=type=nvidia-l4,count=1 \
    --maintenance-policy=TERMINATE \
    --image-family=common-cu121-debian-11 \
    --image-project=deeplearning-platform-release
```

---

### 4. Provisioning Spot VMs, Sole-Tenant Nodes, Shielded VMs & Confidential VMs

#### Part A: Provisioning Spot VMs (No 24-Hour Maximum Runtime Limit)
```bash
# 1. Create Spot VM instance (60-91% discount, auto-preemptible on capacity demand)
gcloud compute instances create spot-batch-worker \
    --zone=us-central1-a \
    --machine-type=e2-standard-4 \
    --provisioning-model=SPOT \
    --instance-termination-action=STOP \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud

# 2. Simulate preemption / host maintenance event to test fault-tolerant application recovery
gcloud compute instances simulate-maintenance-event spot-batch-worker --zone=us-central1-a
```

##### How to Verify Spot VM Configuration:
```bash
gcloud compute instances describe spot-batch-worker \
    --zone=us-central1-a \
    --format="yaml(name, scheduling.provisioningModel, scheduling.instanceTerminationAction, scheduling.onHostMaintenance)"
```

##### Expected Verification Output:
```yaml
name: spot-batch-worker
scheduling:
  instanceTerminationAction: STOP
  onHostMaintenance: TERMINATE
  provisioningModel: SPOT
```

---

#### Part B: Provisioning Sole-Tenant Dedicated Node Groups (PCI-DSS / BYOL Isolation)
```bash
# 1. Create Sole-Tenant Node Template (e.g. N2 series 80 vCPUs, 640 GB RAM)
gcloud compute node-templates create pci-compliance-template \
    --node-type=n2-node-80-640 \
    --region=us-central1

# 2. Allocate Sole-Tenant Node Group (Target 1 physical host server)
gcloud compute node-groups create pci-node-group \
    --node-template=pci-compliance-template \
    --target-size=1 \
    --zone=us-central1-a

# 3. Provision VM assigned strictly to the dedicated Sole-Tenant Node Group
gcloud compute instances create isolated-payment-vm \
    --zone=us-central1-a \
    --machine-type=n2-standard-4 \
    --node-group=pci-node-group \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud
```

##### How to Verify Sole-Tenant Assignment:
```bash
gcloud compute node-groups list-instances pci-node-group --zone=us-central1-a
```

---

#### Part C: Provisioning Shielded VMs (Secure Boot, vTPM & Integrity Monitoring)
```bash
# Provision VM with complete Shielded Cloud Security Triad
gcloud compute instances create secure-bastion-vm \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud \
    --shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring
```

##### How to Verify Shielded Settings:
```bash
gcloud compute instances describe secure-bastion-vm \
    --zone=us-central1-a \
    --format="yaml(shieldedInstanceConfig)"
```

##### Expected Verification Output:
```yaml
shieldedInstanceConfig:
  enableIntegrityMonitoring: true
  enableSecureBoot: true
  enableVtpm: true
```

---

#### Part D: Provisioning Confidential VMs (Hardware RAM Encryption In-Use via AMD SEV)
```bash
# Provision Confidential VM on N2D AMD EPYC platform with inline memory encryption
gcloud compute instances create confidential-db-vm \
    --zone=us-central1-a \
    --machine-type=n2d-standard-4 \
    --confidential-compute \
    --maintenance-policy=TERMINATE \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud
```

##### How to Verify Confidential Compute Encryption:
```bash
gcloud compute instances describe confidential-db-vm \
    --zone=us-central1-a \
    --format="yaml(name, confidentialInstanceConfig)"
```

##### Expected Verification Output:
```yaml
confidentialInstanceConfig:
  enableConfidentialCompute: true
name: confidential-db-vm
```

---

### 5. Block Storage Operations: Persistent Disks, Local SSDs, Online Resizing & Encryption

#### Part A: Creating & Attaching Persistent Disks (Zonal, Regional & Extreme PD)
```bash
# 1. Create Zonal Extreme PD with Provisioned IOPS (15,000 IOPS)
gcloud compute disks create extreme-db-disk \
    --zone=us-central1-a \
    --type=pd-extreme \
    --size=1000GB \
    --provisioned-iops=15000

# 2. Create Regional Synchronous 2-Zone Replicated Persistent Disk (HA DR)
gcloud compute disks create regional-ha-disk \
    --region=us-central1 \
    --replica-zones=us-central1-a,us-central1-b \
    --type=pd-ssd \
    --size=200GB

# 3. Disable Auto-Delete on Boot Disk so disk survives VM deletion
gcloud compute instances set-disk-auto-delete prod-db-vm \
    --zone=us-central1-a \
    --disk=prod-db-vm \
    --no-auto-delete

# 4. Attach Persistent Disk in Read-Only Mode (RO) to share static data across multiple VMs
gcloud compute instances attach-disk web-node-2 \
    --zone=us-central1-a \
    --disk=shared-static-data \
    --mode=ro
```

---

#### Part B: Dynamic Online Disk Resizing & Guest OS Partition Expansion
```bash
# 1. Dynamically resize persistent disk CLI while attached and running (No VM downtime required)
gcloud compute disks resize prod-data-disk --zone=us-central1-a --size=500GB

# 2. Connect via SSH to expand file system inside Linux OS
gcloud compute ssh prod-db-vm --zone=us-central1-a

# 3. Inside Linux VM: Grow partition table and extend file system
sudo growpart /dev/sdb 1
sudo resize2fs /dev/sdb1        # For ext4 file system
# OR: sudo xfs_growfs /mnt/disks/data  # For XFS file system
```

---

#### Part C: Local SSD (Ephemeral NVMe) & RAM Disk (`tmpfs`) Provisioning
```bash
# 1. Create VM with physically attached Local SSD (375 GB NVMe partition)
gcloud compute instances create nvme-cache-vm \
    --zone=us-central1-a \
    --machine-type=n2-standard-4 \
    --local-ssd=interface=NVME \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud

# 2. Inside Linux VM: Format & mount NVMe Local SSD scratch disk
# sudo mkfs.ext4 -F /dev/nvme0n1
# sudo mkdir -p /mnt/disks/nvme-scratch
# sudo mount -o discard,defaults /dev/nvme0n1 /mnt/disks/nvme-scratch

# 3. Inside Linux VM: Mount RAM Disk (tmpfs in system memory) for ultra-low latency volatile storage
# sudo mkdir -p /mnt/ramdisk
# sudo mount -t tmpfs -o size=8g tmpfs /mnt/ramdisk
```

---

#### Part D: Customer-Managed (CMEK) & Customer-Supplied (CSEK) Encryption CLI
```bash
# 1. Provision CMEK Encrypted Disk using Cloud KMS Key
gcloud compute disks create cmek-protected-disk \
    --zone=us-central1-a \
    --type=pd-ssd \
    --size=100GB \
    --kms-key=projects/YOUR_PROJECT_ID/locations/us-central1/keyRings/my-key-ring/cryptoKeys/db-key

# 2. Provision CSEK Encrypted VM using raw Customer-Supplied Key JSON file
gcloud compute instances create csek-secure-vm \
    --zone=us-central1-a \
    --machine-type=n2-standard-4 \
    --csek-key-file=./csek_keys.json \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud
```

---

### 6. Operational Workflows: Metadata Server Querying, Relocation & Snapshot Schedules

#### Part A: Metadata Server Querying & Cloud Storage Script Deployment
```bash
# 1. Deploy VM referencing startup & shutdown scripts stored in Cloud Storage (GCS)
gcloud compute instances create web-app-vm \
    --zone=us-central1-a \
    --machine-type=n2-standard-4 \
    --metadata=startup-script-url=gs://my-app-scripts-bucket/startup.sh,shutdown-script-url=gs://my-app-scripts-bucket/shutdown.sh \
    --scopes=cloud-platform

# 2. SSH into VM and query Metadata Server (169.254.169.254) without credentials
gcloud compute ssh web-app-vm --zone=us-central1-a

# 3. Inside Linux VM: Query external IP from Metadata Server
curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip

# 4. Inside Linux VM: Query zone and instance name
curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/zone
```

---

#### Part B: Cross-Zone / Cross-Region VM Relocation Workflow
```bash
# 1. Create Machine Image of source VM instance (captures disks, configs, and RAM state)
gcloud compute machine-images create myinstance-relocation-img \
    --source-instance=myinstance \
    --source-instance-zone=europe-west1-c

# 2. Instantiate relocated VM in target zone (e.g. us-west1-b) from Machine Image
gcloud compute instances create myinstance \
    --zone=us-west1-b \
    --source-machine-image=myinstance-relocation-img

# 3. Delete old source VM instance after target verification
gcloud compute instances delete myinstance --zone=europe-west1-c --quiet
```

---

#### Part C: Incremental Persistent Disk Snapshots & Performance Upgrades
```bash
# 1. Create incremental compressed snapshot of persistent disk (Stored in Cloud Storage)
gcloud compute disks snapshot mydatadisk \
    --zone=us-central1-a \
    --snapshot-names=mydatadisk-backup-v1

# 2. Upgrade disk performance: Restore snapshot to a high-performance pd-ssd or pd-extreme disk
gcloud compute disks create upgraded-datadisk-ssd \
    --zone=us-central1-a \
    --type=pd-ssd \
    --source-snapshot=mydatadisk-backup-v1
```

---

#### Part D: Automated Snapshot Schedule Resource Policies
```bash
# 1. Create daily automated snapshot schedule policy (Retain backups for 14 days)
gcloud compute resource-policies create snapshot-schedule daily-backup-policy \
    --region=us-central1 \
    --daily-schedule \
    --start-time=02:00 \
    --max-retention-days=14 \
    --on-source-disk-delete=keep-auto-snapshots

# 2. Attach automated snapshot policy to target persistent disk
gcloud compute disks add-resource-policies mydatadisk \
    --zone=us-central1-a \
    --resource-policies=daily-backup-policy
```

---

### 7. Dedicated Stateful Application Server Blueprint (Formatting, Screen, GCS Backup & Cron)

#### Part A: VM Provisioning with Data SSD, Static IP & Storage Scopes
```bash
# 1. Reserve Static External IPv4 Address
gcloud compute addresses create mc-server-ip --region=us-central1

# 2. Provision VM with attached 50GB SSD PD, Storage API Read-Write scope, and target tags
gcloud compute instances create mc-server \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --create-disk=name=minecraft-disk,type=pd-ssd,size=50GB,auto-delete=yes \
    --scopes=storage-rw \
    --tags=minecraft-server \
    --address=mc-server-ip \
    --image-family=debian-12 \
    --image-project=debian-cloud
```

---

#### Part B: Ext4 Formatting & Mounting by Device ID (`/dev/disk/by-id/`)
```bash
# 1. Connect to VM via SSH
gcloud compute ssh mc-server --zone=us-central1-a

# 2. Create mount directory for persistent data
sudo mkdir -p /home/minecraft

# 3. Format persistent SSD disk by device ID with discard & fast initialization flags
sudo mkfs.ext4 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/disk/by-id/google-minecraft-disk

# 4. Mount disk with discard option to enable SSD TRIM optimization
sudo mount -o discard,defaults /dev/disk/by-id/google-minecraft-disk /home/minecraft
```

---

#### Part C: Headless JRE Installation & `screen` Virtual Terminal Session CLI
```bash
# 1. Update repositories and install headless JRE, screen, and wget
sudo apt-get update && sudo apt-get install -y default-jre-headless screen wget

# 2. Navigate to mounted persistent disk directory and download application server
cd /home/minecraft
sudo wget https://launcher.mojang.com/v1/objects/d0d0fe2b1dc6ab4c65554cb734270872b72dadd6/server.jar

# 3. Start server inside a named screen virtual terminal session ("mcs")
sudo screen -S mcs java -Xmx1024M -Xms1024M -jar server.jar nogui

# Keyboard shortcuts:
# Detach screen session: Press Ctrl+A, Ctrl+D
# Reattach screen session: sudo screen -r mcs

# 4. Send remote graceful stop command directly into the screen session buffer
sudo screen -r -X stuff '/stop\n'
```

---

#### Part D: VPC Firewall Rule Creation for Custom Application Port
```bash
# Create ingress firewall rule allowing TCP port 25565 targeting minecraft-server instance tag
gcloud compute firewall-rules create minecraft-rule \
    --allow=tcp:25565 \
    --target-tags=minecraft-server \
    --source-ranges=0.0.0.0/0 \
    --description="Allow incoming client traffic on port 25565"
```

---

#### Part E: Automated GCS Bucket Backup Script & Cron Scheduling
```bash
# 1. Create globally unique Cloud Storage bucket using gcloud storage CLI
export BUCKET_NAME="${GOOGLE_CLOUD_PROJECT}-minecraft-backup"
gcloud storage buckets create gs://${BUCKET_NAME}

# 2. Create backup script (/home/minecraft/backup.sh)
sudo bash -c 'cat << "EOF" > /home/minecraft/backup.sh
#!/bin/bash
screen -r mcs -X stuff '\''/save-all\n/save-off\n'\''
/usr/bin/gcloud storage cp -R ${BASH_SOURCE%/*}/world gs://'${BUCKET_NAME}'/$(date "+%Y%m%d-%H%M%S")-world
screen -r mcs -X stuff '\''/save-on\n'\''
EOF'

# 3. Make backup script executable
sudo chmod 755 /home/minecraft/backup.sh

# 4. Add cron job to execute backup every 4 hours automatically
(sudo crontab -l 2>/dev/null; echo "0 */4 * * * /home/minecraft/backup.sh") | sudo crontab -
```

---

#### Part F: Automated Lifecycle Script Metadata URLs
```bash
# Configure startup & shutdown metadata script URLs for hands-off VM startup and graceful stop
gcloud compute instances add-metadata mc-server \
    --zone=us-central1-a \
    --metadata=startup-script-url=https://storage.googleapis.com/cloud-training/archinfra/mcserver/startup.sh,shutdown-script-url=https://storage.googleapis.com/cloud-training/archinfra/mcserver/shutdown.sh
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

#### How to Verify Configuration Correctness:
```bash
gcloud compute instances describe prod-web-vm1 --zone=us-central1-a --format="value(status)"
```

#### Expected Verification Output:
```text
TERMINATED
```

---

### 2. Update Machine Type & RAM Memory (Pre-defined & Custom VM)

> [!NOTE]
> Compute Engine requires VM instances to be in the **`STOPPED` (`TERMINATED`)** state before updating CPU cores, memory size, or machine type series. You cannot modify machine parameters while the VM is in `RUNNING` status.

#### Option A: Modify Pre-defined Machine Type (e.g., e2-medium to e2-standard-4)
```bash
# 1. Stop target VM instance
gcloud compute instances stop utility-vm --zone=us-central1-a

# 2. Reconfigure machine type to e2-standard-4 (4 vCPUs, 16 GB memory)
gcloud compute instances set-machine-type utility-vm \
    --zone=us-central1-a \
    --machine-type=e2-standard-4

# 3. Start target VM instance
gcloud compute instances start utility-vm --zone=us-central1-a
```

#### Option B: Modify Custom Machine Type RAM & vCPU (e.g., utility-cm)
```bash
# 1. Stop target custom VM instance
gcloud compute instances stop utility-cm --zone=us-central1-a

# 2. Reconfigure custom vCPUs (e.g., 4 cores) and Memory (e.g., 16 GB)
gcloud compute instances set-machine-type utility-cm \
    --zone=us-central1-a \
    --custom-cpu=4 \
    --custom-memory=16GB

# 3. Start target custom VM instance
gcloud compute instances start utility-cm --zone=us-central1-a
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute instances describe utility-cm \
    --zone=us-central1-a \
    --format="yaml(name, status, machineType)"
```

#### Expected Verification Output:
```yaml
machineType: https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/zones/us-central1-a/machineTypes/custom-4-16384
name: utility-cm
status: RUNNING
```

---

### 3. Inspect VM RAM & Hardware inside Linux OS (SSH)

Run these commands inside the Linux VM terminal (via SSH) to verify allocated memory, processor cores, and memory module hardware details:

```bash
# 1. Display total, used, free memory (RAM) and swap space in human-readable format
free -h

# 2. Inspect physical hardware details for installed RAM DIMMs (capacity, locator, speed)
sudo dmidecode -t 17

# 3. Verify total number of active online processor cores
nproc

# 4. Display CPU architecture and socket topology details
lscpu
```

#### Expected Output (`free -h`):
```text
               total        used        free      shared  buff/cache   available
Mem:            15Gi       482Mi        14Gi       1.0Mi       312Mi        15Gi
Swap:             0B          0B          0B
```

#### Expected Output (`sudo dmidecode -t 17` excerpt):
```text
Handle 0x1100, DMI type 17, 40 bytes
Memory Device
	Array Handle: 0x1000
	Total Width: 64 bits
	Data Width: 64 bits
	Size: 16384 MB
	Form Factor: DIMM
	Set: None
	Locator: DIMM 0
	Bank Locator: Bank 0
	Type: RAM
	Type Detail: Synchronous
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

---

## Phase 7: Professional Failure Diagnosis & Resolution Matrix

| Phase | Error Code / Status | Root Cause | Diagnosis Command | Immediate Resolution Command |
| :--- | :--- | :--- | :--- | :--- |
| **Build** | **`IMAGE_CREATION_FAILED`** | Source instance running during image creation | `gcloud compute instances describe VM` | Stop instance before creating image: `gcloud compute instances stop VM`. |
| **Config**| **`QUOTA_EXCEEDED (CPUS)`** | Regional vCPU quota limit reached | `gcloud compute regions describe REGION --format="value(quotas)"` | Reduce machine size or request quota increase in GCP Console. |
| **Config**| **`ZONE_RESOURCE_POOL_EXHAUSTED`** | Hardware capacity exhausted in target zone | `gcloud compute instances create VM --zone=us-central1-a` | Retry deployment in adjacent zone: `--zone=us-central1-b`. |
| **Deploy**| **`MIG_INSTANCE_CREATION_FAILED`** | Template startup script failed on boot | `gcloud compute instance-groups managed list-errors MIG` | Fix startup script syntax or update Instance Template (`gcloud compute instance-templates create`). |
| **Deploy**| **`SSH Connection Timed Out`** | Firewall blocking port 22 or no public IP | `gcloud compute instances describe VM` | Connect via IAP: `gcloud compute ssh VM --tunnel-through-iap`. |
| **Resize**| **`INVALID_STATE (setMachineType while RUNNING)`** | Machine type/RAM update attempted on running VM | `gcloud compute instances describe VM --format="value(status)"` | Stop instance first: `gcloud compute instances stop VM --zone=ZONE`, then run `gcloud compute instances set-machine-type`. |
| **Resize**| **`Disk size expanded in CLI but not OS`** | Partition table not expanded in OS | `gcloud compute ssh VM --command="df -h"` | Run filesystem expansion inside VM: `sudo resize2fs /dev/sda1`. |
| **IAM**   | **`403 Forbidden (compute.setMachineType)`** | User lacks `compute.instanceAdmin.v1` role | `gcloud config get-value account` | Grant role: `gcloud projects add-iam-policy-binding PROJECT --member="USER" --role="roles/compute.instanceAdmin.v1"`. |
| **MIG**   | **`Rolling update stalled`** | New template fails health check probes | `gcloud compute instance-groups managed describe MIG` | Stop update and rollback template version: `gcloud compute instance-groups managed rolling-action start-update MIG --version=template=PREV_TEMPLATE`. |

