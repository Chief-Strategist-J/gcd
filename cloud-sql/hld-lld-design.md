# Cloud SQL: High-Level Design (HLD) & Low-Level Design (LLD)

This document details the architectural design for **Google Cloud SQL**, covering High Availability (HA) failover mechanics, synchronous disk replication, Private IP VPC peering, and Cloud SQL Auth Proxy security architectures.

---

## 1. High-Level Design (HLD) Architecture

The High-Level Architecture illustrates regional High Availability deployment, synchronous persistent disk replication, client connectivity paths, and automated backup/recovery pipelines:

```mermaid
graph TD
    ClientApp["Client Application<br/>(App Engine / Compute VM / GKE / On-Prem)"]
    
    subgraph ConnectionMethods ["Connectivity & Authorization Layer"]
        PrivateIP["Private IP Connection<br/>(VPC Peering / Private Services Access)"]
        AuthProxy["Cloud SQL Auth Proxy<br/>(IAM Auth & Auto SSL Encryption)"]
        PublicIP["Authorized Networks Public IP<br/>(whitelisted CIDRs + TLS Certs)"]
    end
    
    ClientApp --> PrivateIP
    ClientApp --> AuthProxy
    ClientApp --> PublicIP

    subgraph GCPRegion ["GCP Region (e.g. us-central1)"]
        subgraph ZoneA ["Zone A (Primary Zone)"]
            PrimarySQL["Cloud SQL Primary Instance<br/>(MySQL / Postgres / SQL Server)"]
            DiskA["Persistent Disk A (Primary)"]
            PrimarySQL --- DiskA
        end
        
        subgraph ZoneB ["Zone B (Standby Zone)"]
            StandbySQL["Cloud SQL Standby Instance<br/>(Failover Node)"]
            DiskB["Persistent Disk B (Secondary)"]
            StandbySQL -.->|Inactive Standby| DiskB
        end
        
        DiskA ==>|Synchronous Regional Disk Replication| DiskB
    end

    PrivateIP --> PrimarySQL
    AuthProxy --> PrimarySQL
    PublicIP --> PrimarySQL

    subgraph BackupPipeline ["Backup & Disaster Recovery Pipeline"]
        GCSBackup["Cloud Storage Bucket<br/>(Automated Backups & PITR Binlogs)"]
        ReadReplica["Cross-Region / In-Region<br/>Read Replica Node"]
    end

    PrimarySQL -->|Async Replication| ReadReplica
    PrimarySQL -.->|Automated Snapshots & PITR| GCSBackup

    style PrimarySQL fill:#34A853,stroke:#333,color:#fff
    style StandbySQL fill:#FBBC05,stroke:#333,color:#333
    style ConnectionMethods fill:#4285F4,stroke:#333,color:#fff
    style DiskA fill:#34A853,stroke:#333,color:#fff
    style DiskB fill:#FBBC05,stroke:#333,color:#333
```

### Key HLD Architectural Components:
1. **Regional High Availability (HA)**:
   - **Primary Instance**: Active instance in Zone A handling all read and write queries.
   - **Standby Instance**: Standby node in Zone B.
   - **Synchronous Disk Replication**: Every write transaction is synchronously replicated to persistent disks across both Zone A and Zone B before the transaction is reported as committed.
2. **Failover Execution**: In the event of a zone or primary instance failure, the persistent disk is attached to the standby node, which becomes the new primary instance. Traffic is automatically rerouted.
3. **Connectivity Models**:
   - **Private IP**: Maximum performance and zero internet exposure using VPC Private Services Access.
   - **Cloud SQL Auth Proxy**: Recommended for external or cross-region traffic; handles IAM authentication and TLS certificate rotation automatically.

---

## 2. Low-Level Design (LLD)

### 2.1 Cloud SQL Auth Proxy Connection Sequence

The Auth Proxy uses IAM credentials to authorize access and generates ephemeral SSL/TLS certificates automatically:

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Application / Developer Tool
    participant Proxy as Local Cloud SQL Auth Proxy Client
    participant IAM as GCP IAM & STS Token Service
    participant AdminAPI as Cloud SQL Admin API
    participant DB as Cloud SQL Database Instance

    rect rgb(240, 248, 255)
    Note over Dev, DB: Phase 1: Proxy Startup & Ephemeral TLS Certificate Generation
    Dev->>Proxy: Start proxy (connectionName = PROJECT:REGION:INSTANCE)
    Proxy->>IAM: Authenticate using ADC / Service Account Key
    IAM-->>Proxy: Return IAM Access Token
    Proxy->>AdminAPI: Call getInstance(connectionName) & generateEphemeralCert()
    AdminAPI-->>Proxy: Return Server Instance Metadata & Ephemeral TLS Certificate (1 Hour)
    Proxy-->>Proxy: Listen on local port (e.g., 127.0.0.1:3306)
    end

    rect rgb(245, 255, 245)
    Note over Dev, DB: Phase 2: Client Connection & Mutual TLS Encrypted Tunnel
    Dev->>Proxy: Connect to 127.0.0.1:3306 (Standard MySQL / Postgres Driver)
    Proxy->>DB: Establish mTLS Encrypted Connection to Cloud SQL Engine (Port 3307)
    DB-->>Proxy: mTLS Connection Established & Validated
    Proxy-->>Dev: Connection Handshake Complete
    Dev->>DB: Execute SQL Queries (INSERT / SELECT / UPDATE)
    DB-->>Dev: SQL Query Results Returned via Encrypted Tunnel
    end
```

---

### 2.2 High Availability (HA) Failover Sequence

When a failure occurs in Zone A, Cloud SQL automatically executes failover to Zone B:

```mermaid
sequenceDiagram
    autonumber
    actor App as Application Workload
    participant Primary as Primary Instance (Zone A)
    participant Monitor as Cloud SQL Control Plane Monitor
    participant Disk as Synchronous Replicated Storage
    participant Standby as Standby Instance (Zone B)

    rect rgb(240, 248, 255)
    Note over App, Disk: Normal Operational State (Zone A Active)
    App->>Primary: Execute SQL Transaction (COMMIT)
    Primary->>Disk: Synchronous Write to Zone A & Zone B Storage Disks
    Disk-->>Primary: Write Ack from both zones
    Primary-->>App: Transaction Committed
    end

    rect rgb(255, 240, 240)
    Note over Primary, Standby: Zone A Outage Detected & Failover Triggered
    Monitor-xPrimary: Heartbeat Check Failed (Primary Unresponsive)
    Monitor->>Disk: Detach Storage from Primary Node
    Monitor->>Standby: Attach Replicated Persistent Disk to Standby Node (Zone B)
    Standby->>Standby: Initialize Recovery & Become Active Primary
    Monitor->>Monitor: Update Internal DNS Rerouting Entry
    App->>Standby: Reconnect to Cloud SQL (Automatic Rerouting to Zone B)
    Standby-->>App: 200 OK (Traffic Restored in Zone B)
    end
```

---

### 2.3 LLD Mechanics Summary:
* **Private Services Access (VPC Peering)**: Private IP connections utilize an internal Google-managed VPC peered to the customer VPC, guaranteeing zero public IP exposure.
* **Point-In-Time Recovery (PITR)**: Uses daily base disk snapshots combined with continuous binary log (`binlog` / `wal`) archiving in GCS, allowing recovery to any exact second within the retention window.
* **Vertical Scaling Restart**: Upgrading CPU or RAM tier patches the underlying compute VM, causing a brief restarting window during which HA standby nodes minimize downtime.

---

## 3. AlloyDB for PostgreSQL Disaggregated Multi-Node Architecture

AlloyDB decouples database compute processing from a disaggregated, multi-tenant storage layer to deliver **99.99% uptime SLA** (inclusive of maintenance) and $>4\times$ transactional / $100\times$ analytical performance over standard PostgreSQL:

```mermaid
graph TD
    ClientApp["Client Application / Analytics Engine"] -->|PostgreSQL Protocol| AlloyDBPrimary["AlloyDB Primary Instance<br/>(Write & Read Workloads)"]
    ClientApp -->|Read Load Balancing| ReadPool["AlloyDB Read Pool Nodes<br/>(Scale-Out Read Capacity)"]

    subgraph ComputeLayer ["Decoupled Compute Layer"]
        AlloyDBPrimary --> ColumnarEngine["Columnar Engine in Memory<br/>(Auto-accelerates OLAP queries up to 100x)"]
        ReadPool --> ColumnarEngine
        AlloyDBPrimary --> GeminiAI["Gemini / Vertex AI Integration<br/>(In-database ML Inference)"]
    end

    subgraph StorageLayer ["Disaggregated Cloud Storage Layer (Google Infrastructure)"]
        AlloyDBPrimary ==>|WAL Stream Log Replication| LogStorage["Distributed Write-Ahead Log (WAL) Storage"]
        LogStorage ==> StorageNodes["Replicated Multi-Tenant Storage Shards<br/>(Auto-Scaling Storage, Low Latency)"]
        StorageNodes -.->|Block Materialization| ReadPool
    end

    style AlloyDBPrimary fill:#34A853,stroke:#333,color:#fff
    style ReadPool fill:#4285F4,stroke:#333,color:#fff
    style ColumnarEngine fill:#FBBC05,stroke:#333,color:#333
    style StorageLayer fill:#EA4335,stroke:#333,color:#fff
```

### Key AlloyDB Mechanics:
* **Decoupled Log-Store Architecture**: Transactions commit as soon as WAL entries are persisted to the low-latency log store. Storage shard materialization occurs asynchronously in the background.
* **In-Memory Columnar Engine**: Automatically converts hot row-oriented relational data into in-memory columnar format for analytical queries, accelerating OLAP performance by up to $100\times$.
* **ML-Driven Adaptive Vacuuming**: Machine learning algorithms optimize PostgreSQL vacuuming, memory allocation, and data tiering without manual DBA tuning.

