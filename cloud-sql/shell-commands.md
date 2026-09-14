# Cloud SQL: In-Depth Operations & Verification Manual

This document is an exhaustive, production-grade operational reference manual for Google Cloud SQL using the `gcloud sql` CLI tool and Cloud SQL Auth Proxy client. All commands use generalized variable placeholders for long-term reference.

Every command snippet includes:
1. **Generalized CLI Command Template**
2. **Key Parameter Descriptions & Defaults**
---

## Table of Contents
1. [Category 1: Instance Provisioning Across Database Engines](#category-1-instance-provisioning-across-database-engines)
2. [Category 2: Database & Native User Administration](#category-2-database--native-user-administration)
3. [Category 3: Connection Strategies (Private IP, Auth Proxy, SSL & Authorized Networks)](#category-3-connection-strategies-private-ip-auth-proxy-ssl--authorized-networks)
4. [Category 4: High Availability (HA) & Failover Testing](#category-4-high-availability-ha--failover-testing)
5. [Category 5: Automated Backups, Point-In-Time Recovery (PITR) & Import/Export](#category-5-automated-backups-point-in-time-recovery-pitr--importexport)
6. [Category 6: Vertical Tier Scaling & Read Replicas](#category-6-vertical-tier-scaling--read-replicas)
7. [Category 7: Error Diagnosis & Failure Resolution Matrix](#category-7-error-diagnosis--failure-resolution-matrix)
8. [Category 8: Official Qwiklabs Hands-On Lab Operations Manual](#category-8-official-qwiklabs-hands-on-lab-operations-manual)
9. [Category 9: AlloyDB for PostgreSQL Operational Reference (`gcloud alloydb`)](#category-9-alloydb-for-postgresql-operational-reference-gcloud-alloydb)

---

## Category 1: Instance Provisioning Across Database Engines

Cloud SQL supports **MySQL** (5.6, 5.7, 8.0), **PostgreSQL** (9.6 to 15), and **Microsoft SQL Server** (2017, 2019 - Web, Express, Standard, Enterprise) with up to 64 TB storage capacity, 60,000 IOPS, 624 GB RAM, and 96 vCPU processor cores.

```bash
# Reference Variables
export PROJECT_ID="YOUR_PROJECT_ID"
export INSTANCE_NAME="YOUR_INSTANCE_NAME"
export REGION="us-central1"
export ROOT_PASSWORD="SECURE_DATABASE_PASSWORD"

# 1. Provision Production MySQL 8.0 Regional HA Instance (Primary in Zone A, Standby in Zone B)
gcloud sql instances create ${INSTANCE_NAME}-mysql \
    --project=${PROJECT_ID} \
    --database-version=MYSQL_8_0 \
    --tier=db-custom-4-16384 \
    --region=${REGION} \
    --availability-type=REGIONAL \
    --storage-size=100GB \
    --storage-type=SSD \
    --storage-auto-increase \
    --root-password=${ROOT_PASSWORD}

# 2. Provision Production PostgreSQL 15 Regional HA Instance
gcloud sql instances create ${INSTANCE_NAME}-pg \
    --project=${PROJECT_ID} \
    --database-version=POSTGRES_15 \
    --tier=db-custom-8-32768 \
    --region=${REGION} \
    --availability-type=REGIONAL \
    --storage-size=200GB \
    --storage-type=SSD \
    --storage-auto-increase \
    --root-password=${ROOT_PASSWORD}

# 3. Provision Microsoft SQL Server 2019 Standard Instance
gcloud sql instances create ${INSTANCE_NAME}-mssql \
    --project=${PROJECT_ID} \
    --database-version=SQLSERVER_2019_STANDARD \
    --tier=db-custom-16-65536 \
    --region=${REGION} \
    --availability-type=REGIONAL \
    --storage-size=300GB \
    --root-password=${ROOT_PASSWORD}

# Key Parameter Breakdown:
# --availability-type=REGIONAL: Deploys primary instance in Zone A and standby in Zone B with synchronous disk replication
# --tier=db-custom-N-M: Specifies custom vCPU count N and RAM size M in MB (e.g. db-custom-4-16384 = 4 vCPUs, 16 GB RAM)
# --storage-auto-increase: Dynamically increases storage capacity up to 64 TB when space runs low
```

#### How to Verify Configuration Correctness:
```bash
gcloud sql instances describe ${INSTANCE_NAME}-mysql \
    --format="yaml(name, state, databaseVersion, settings.tier, settings.availabilityType, gceZone, secondaryGceZone)"
```

---

## Category 2: Database & Native User Administration

Administer native database schemas and user credentials using Cloud SQL tools.

```bash
# 1. Create a Target Application Database Schema
gcloud sql databases create app_production \
    --instance=${INSTANCE_NAME}-mysql \
    --charset=utf8mb4 \
    --collation=utf8mb4_unicode_ci

# 2. Create Application User Account
gcloud sql users create app_user \
    --instance=${INSTANCE_NAME}-mysql \
    --host="%" \
    --password="USER_STRONG_PASSWORD"

# 3. List Databases and Users
gcloud sql databases list --instance=${INSTANCE_NAME}-mysql
gcloud sql users list --instance=${INSTANCE_NAME}-mysql
```

---

## Category 3: Connection Strategies (Private IP, Auth Proxy, SSL & Authorized Networks)

### 1. Private IP Connection (VPC Private Services Access)

Recommended for workloads co-located in the same project/region for maximum performance and security (traffic never touches the public internet).

```bash
# 1. Enable Service Networking API & Allocate Private Range
gcloud services enable servicenetworking.googleapis.com --project=${PROJECT_ID}
gcloud compute addresses create google-managed-services-default \
    --global \
    --purpose=VPC_PEERING \
    --prefix-length=16 \
    --network=default \
    --project=${PROJECT_ID}

# 2. Create Private VPC Peering Connection
gcloud services peering connect \
    --service=servicenetworking.googleapis.com \
    --ranges=google-managed-services-default \
    --network=default \
    --project=${PROJECT_ID}

# 3. Create Private IP Cloud SQL Instance (Disabling Public IP)
gcloud sql instances create ${INSTANCE_NAME}-private \
    --project=${PROJECT_ID} \
    --database-version=MYSQL_8_0 \
    --tier=db-custom-2-7680 \
    --region=${REGION} \
    --network=default \
    --no-assign-ip \
    --root-password=${ROOT_PASSWORD}
```

#### How to Verify Private IP:
```bash
gcloud sql instances describe ${INSTANCE_NAME}-private --format="value(ipAddresses[0].ipAddress)"
```

---

### 2. Cloud SQL Auth Proxy (Recommended for Cross-Region / External Connections)

The Cloud SQL Auth Proxy automatically manages authentication via IAM credentials, enforces TLS encryption, and handles automatic key rotation.

```bash
# 1. Download & Install Cloud SQL Auth Proxy Binary
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.8.0/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy

# 2. Start Cloud SQL Auth Proxy Client Tunneling to Instance Connection Name
# Format: PROJECT_ID:REGION:INSTANCE_NAME
export INSTANCE_CONNECTION_NAME="${PROJECT_ID}:${REGION}:${INSTANCE_NAME}-mysql"

./cloud-sql-proxy ${INSTANCE_CONNECTION_NAME} --port=3306 &

# 3. Connect Local Client Tool (MySQL Workbench / psql / CLI) to Proxy Localhost
mysql -h 127.0.0.1 -u app_user -p --database=app_production
```

---

### 3. Authorized Networks (Public IP Restriction)

```bash
gcloud sql instances patch ${INSTANCE_NAME}-mysql \
    --authorized-networks="203.0.113.5/32,198.51.100.0/24" \
    --require-ssl
```

---

## Category 4: High Availability (HA) & Failover Testing

High Availability regional instances maintain a primary instance in Zone A and a standby instance in Zone B with synchronous persistent disk replication.

```bash
# 1. Trigger Manual Test Failover (Fails over from Primary in Zone A to Standby in Zone B)
gcloud sql instances failover ${INSTANCE_NAME}-mysql --project=${PROJECT_ID} --quiet

# 2. Monitor Failover State & Primary Zone Switch
gcloud sql instances describe ${INSTANCE_NAME}-mysql \
    --format="yaml(name, state, gceZone, secondaryGceZone)"
```

---

## Category 5: Automated Backups, Point-In-Time Recovery (PITR) & Import/Export

### 1. Configure Automated Backups & Point-In-Time Recovery (PITR)

```bash
# Enable Daily Automated Backups at 03:00 UTC with Point-In-Time Recovery
gcloud sql instances patch ${INSTANCE_NAME}-mysql \
    --backup-start-time=03:00 \
    --enable-bin-log \
    --retained-backups-count=7

# Trigger an On-Demand Backup
gcloud sql backups create --instance=${INSTANCE_NAME}-mysql --description="Pre-deployment snapshot"
```

---

### 2. Perform Point-In-Time Recovery (Clone to New Instance)

```bash
# Clone Instance to a Specific Timestamp for PITR Recovery
gcloud sql instances clone ${INSTANCE_NAME}-mysql ${INSTANCE_NAME}-pitr-restored \
    --point-in-time="2026-09-13T14:30:00.000Z"
```

---

### 3. Import & Export SQL Dumps / CSV Files via Cloud Storage

```bash
# 1. Export Database Dump to Cloud Storage Bucket
gcloud sql export sql ${INSTANCE_NAME}-mysql gs://YOUR_STORAGE_BUCKET/backups/db-export.sql.gz \
    --database=app_production

# 2. Import SQL Dump from Cloud Storage Bucket
gcloud sql import sql ${INSTANCE_NAME}-mysql gs://YOUR_STORAGE_BUCKET/backups/db-export.sql.gz \
    --database=app_production
```

---

## Category 6: Vertical Tier Scaling & Read Replicas

### 1. Vertical Tier Scaling (Scale Up Processor Cores & RAM)

Vertical scaling updates instance CPU and RAM specs (requires a brief instance restart).

```bash
gcloud sql instances patch ${INSTANCE_NAME}-mysql \
    --tier=db-custom-8-32768
```

---

### 2. Scale Out Read Replicas (Offload Read Queries)

Read replicas offload read-heavy traffic from the primary instance.

```bash
# 1. Create a Read Replica in Same or Different Region
gcloud sql instances create ${INSTANCE_NAME}-replica-1 \
    --master-instance-name=${INSTANCE_NAME}-mysql \
    --region=${REGION} \
    --tier=db-custom-4-16384

# 2. Promote Read Replica to Independent Master (In Emergency Recovery Scenarios)
gcloud sql instances promote-replica ${INSTANCE_NAME}-replica-1
```

---

## Category 7: Error Diagnosis & Failure Resolution Matrix

| Error Code / Symptom | Root Cause | Diagnosis Command | Immediate Resolution Command |
| :--- | :--- | :--- | :--- |
| **`403 AccessDenied`** | User/SA lacks `roles/cloudsql.client` or `roles/cloudsql.admin`. | `gcloud config get-value account` | Grant role: `gcloud projects add-iam-policy-binding PROJECT_ID --member="MEMBER" --role="roles/cloudsql.client"` |
| **`Connection Refused (127.0.0.1:3306)`** | Cloud SQL Auth Proxy is not running or listening on target port. | `./cloud-sql-proxy INSTANCE_CONNECTION_NAME` | Ensure Auth Proxy is executing and port 3306 is open locally. |
| **`Private IP VPC Peering Failed`** | Service Networking API not enabled or IP range collision. | `gcloud services peering list --network=default` | Verify Service Networking connection and address allocation. |
| **`Storage Full (Disk Space Exceeded)`** | Instance ran out of storage space and auto-increase was disabled. | `gcloud sql instances describe INSTANCE_NAME` | Enable auto-increase: `gcloud sql instances patch INSTANCE_NAME --storage-auto-increase` |
| **`Failover In Progress (503 Service Unavailable)`** | Primary instance failed over to standby node; transient reconnection state. | `gcloud sql instances describe INSTANCE_NAME` | Retry request; Cloud SQL Auth Proxy / Private IP automatically reroutes traffic to standby once primary state is RUNNABLE. |

---

## Category 8: Official Qwiklabs Hands-On Lab Operations Manual

This section provides the complete step-by-step execution guide for the official **"Implementing Cloud SQL"** Qwiklabs / Skills Boost lab sequence.

---

### Task 1. Provision Cloud SQL Instance & Private IP Allocation

Create a Cloud SQL MySQL database instance (`wordpress-db`) with Private IP allocation and database schema (`wordpress`).

```bash
# 1. Export Environment Variables
export PROJECT_ID=$(gcloud config get-value project)
export REGION="us-central1"
export INSTANCE_NAME="wordpress-db"
export ROOT_PASSWORD="password"

# 2. Allocate Private IP Range via VPC Peering (Private Services Access)
gcloud compute addresses create google-managed-services-default \
    --global \
    --purpose=VPC_PEERING \
    --prefix-length=16 \
    --network=default \
    --project=${PROJECT_ID}

gcloud services peering connect \
    --service=servicenetworking.googleapis.com \
    --ranges=google-managed-services-default \
    --network=default \
    --project=${PROJECT_ID}

# 3. Create Cloud SQL Instance with Private IP Enabled
gcloud sql instances create ${INSTANCE_NAME} \
    --project=${PROJECT_ID} \
    --database-version=MYSQL_5_7 \
    --tier=db-n1-standard-1 \
    --region=${REGION} \
    --network=default \
    --root-password=${ROOT_PASSWORD}

# 4. Create WordPress Application Schema
gcloud sql databases create wordpress --instance=${INSTANCE_NAME}
```

#### How to Verify Configuration Correctness:
```bash
gcloud sql instances describe ${INSTANCE_NAME} --format="yaml(name, state, ipAddresses)"
```

---

### Task 2. Download & Run Cloud SQL Auth Proxy in Background

SSH into the application proxy VM (`wordpress-europe-proxy`), install the Cloud SQL Auth Proxy binary, and launch the proxy background daemon listening on `127.0.0.1:3306`.

```bash
# 1. SSH into Proxy Virtual Machine
gcloud compute ssh wordpress-europe-proxy --zone=us-central1-a

# --- INSIDE PROXY LINUX VM ---

# 2. Download & Make Executable Cloud SQL Auth Proxy
wget https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.8.0/cloud-sql-proxy.linux.amd64 -O cloud-sql-proxy
chmod +x cloud-sql-proxy

# 3. Export Instance Connection Name Variable
# Format: PROJECT_ID:REGION:wordpress-db
export SQL_CONNECTION="$(gcloud config get-value project):us-central1:wordpress-db"
echo "SQL Connection Name: $SQL_CONNECTION"

# 4. Launch Auth Proxy in Background Daemon Mode
./cloud-sql-proxy $SQL_CONNECTION --port=3306 &
```

---

### Task 3. Connect Application via Auth Proxy (Localhost 127.0.0.1)

Configure the web application (WordPress) using the local Auth Proxy endpoint `127.0.0.1:3306`.

```bash
# 1. Retrieve Public IP of Proxy VM
export PROXY_PUBLIC_IP=$(gcloud compute instances describe wordpress-europe-proxy --zone=us-central1-a --format="value(networkInterfaces[0].accessConfigs[0].natIP)")
echo "Open Browser to: http://${PROXY_PUBLIC_IP}"

# 2. Application Database Configuration Values (WordPress Setup Wizard):
# Database Name: wordpress
# Username:      root
# Password:      password
# Database Host: 127.0.0.1 (Connects locally via Cloud SQL Auth Proxy on port 3306)

# 3. Test HTTP Connection to Application
curl -I "http://${PROXY_PUBLIC_IP}"
```

---

### Task 4. Connect Application Directly via Cloud SQL Private IP Address

Bypass public IP endpoints and connect co-located workloads directly to the Cloud SQL Instance Private IP address (`10.x.x.x`) for zero internet exposure and lower latency.

```bash
# 1. Retrieve Cloud SQL Instance Private IP Address
export SQL_PRIVATE_IP=$(gcloud sql instances describe wordpress-db --format="value(ipAddresses[1].ipAddress)")
echo "Cloud SQL Private IP: ${SQL_PRIVATE_IP}"

# 2. Retrieve Public IP of Second Application VM (wordpress-private-ip)
export APP2_PUBLIC_IP=$(gcloud compute instances describe wordpress-private-ip --zone=us-central1-a --format="value(networkInterfaces[0].accessConfigs[0].natIP)")
echo "Open Browser to: http://${APP2_PUBLIC_IP}"

# 3. Application Database Configuration Values (Direct Private IP):
# Database Name: wordpress
# Username:      root
# Password:      password
# Database Host: 10.x.x.x (Direct Private IP Address of Cloud SQL Instance)

# 4. Verify Application Loading via Direct Private IP
curl -I "http://${APP2_PUBLIC_IP}"
```

---

## Category 9: AlloyDB for PostgreSQL Operational Reference (`gcloud alloydb`)

AlloyDB for PostgreSQL is Google Cloud's fully managed, 100% PostgreSQL-compatible enterprise database service designed for demanding hybrid transactional and analytical processing (HTAP) workloads.

### Core Performance & Architectural Specifications:
* **Transactional (OLTP)**: $>4 \times$ faster than standard PostgreSQL for high transaction throughput.
* **Analytical (OLAP)**: Up to $100 \times$ faster than standard PostgreSQL for analytical queries via columnar in-memory engine.
* **Availability SLA**: **99.99% Uptime SLA** inclusive of maintenance windows.
* **Adaptive Intelligence**: ML-driven vacuum management, storage/memory management, data tiering, and built-in integration with Gemini / Vertex AI platform for in-database ML model invocation.

```bash
# Reference Variables
export PROJECT_ID="YOUR_PROJECT_ID"
export REGION="us-central1"
export CLUSTER_ID="prod-alloydb-cluster"
export PRIMARY_ID="prod-alloydb-primary"
export READ_POOL_ID="prod-alloydb-read-pool"
export NETWORK_NAME="default"
export PASSWORD="SECURE_ALLOYDB_PASSWORD"

# 1. Create AlloyDB Cluster in Target Region
gcloud alloydb clusters create ${CLUSTER_ID} \
    --project=${PROJECT_ID} \
    --region=${REGION} \
    --network=${NETWORK_NAME} \
    --password=${PASSWORD}

# 2. Provision AlloyDB Primary Instance (8 vCPUs)
gcloud alloydb instances create ${PRIMARY_ID} \
    --project=${PROJECT_ID} \
    --region=${REGION} \
    --cluster=${CLUSTER_ID} \
    --cpu-count=8 \
    --instance-type=PRIMARY

# 3. Create Scale-Out Read Pool (2 Nodes for High Read Throughput)
gcloud alloydb instances create ${READ_POOL_ID} \
    --project=${PROJECT_ID} \
    --region=${REGION} \
    --cluster=${CLUSTER_ID} \
    --cpu-count=4 \
    --instance-type=READ_POOL \
    --read-pool-node-count=2

# 4. Trigger Manual On-Demand Backup
gcloud alloydb backups create alloydb-backup-01 \
    --project=${PROJECT_ID} \
    --region=${REGION} \
    --cluster=${CLUSTER_ID}
```

#### How to Verify Configuration Correctness:
```bash
gcloud alloydb instances list --cluster=${CLUSTER_ID} --region=${REGION} \
    --format="table(name.basename(), instanceType, state, ipAddress)"
```


