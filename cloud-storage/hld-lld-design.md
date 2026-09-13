# Cloud Storage: High-Level Design (HLD) & Low-Level Design (LLD)

This document details the architectural design for **Google Cloud Storage (Buckets & Objects)**, covering storage classes, location strategies, data protection recovery primitives, security models, and data transfer engines.

---

## 1. High-Level Design (HLD)

The High-Level Architecture illustrates global bucket deployment, client access patterns, Object Lifecycle rules, IAM Security, KMS/CSEK encryption layers, and ingestion options:

```mermaid
graph TD
    Client["Client / Application"] -->|HTTPS / gcloud / API| GCSFront["GCS Global Anycast Edge Router"]
    
    subgraph Ingestion ["Data Ingestion Services"]
        STS["Storage Transfer Service<br/>(Online AWS S3 / HTTP / GCS Sync)"]
        TA["Transfer Appliance<br/>(100TB - 1PB Physical Hardware)"]
        OMI["Offline Media Import<br/>(3rd Party Physical Media)"]
    end
    
    STS --> GCSFront
    TA --> GCSFront
    OMI --> GCSFront

    subgraph GCS ["Google Cloud Storage Infrastructure"]
        GCSFront -->|Auth Check| IAM["IAM Policies & Fine-Grained ACLs"]
        IAM -->|Authorized| Bucket["Storage Bucket (gs://prod-app-data)"]
        
        subgraph Bucket ["Storage Bucket (Multi-Region / Dual-Region / Regional)"]
            Standard["Standard Class<br/>(Hot Data, 0-day Min Duration)"]
            Nearline["Nearline Class<br/>(Infrequent Data, 30-day Min)"]
            Coldline["Coldline Class<br/>(Cold Data, 90-day Min)"]
            Archive["Archive Class<br/>(Long-term Compliance, 365-day Min)"]
        end
        
        Lifecycle Engine["Lifecycle Engine / Autoclass"] -->|Rules / Access Patterns| Nearline
        Lifecycle Engine -->|Age / Access| Coldline
        Lifecycle Engine -->|Age / Access| Archive

        subgraph Protection ["Data Recovery & WORM Controls"]
            SoftDelete["Soft Delete Pool<br/>(Default 7d, Up to 90d Retention)"]
            Versioning["Object Versioning<br/>(Archived Generations #GEN)"]
            WORM["Object Retention Lock<br/>(Locked SEC/FINRA WORM Policy)"]
        end
        
        Bucket --> Protection
    end
    
    Bucket -->|Encryption at Rest| KMS["Cloud KMS (CMEK) / CSEK / Google Keys"]

    style GCSFront fill:#4285F4,stroke:#333,color:#fff
    style Bucket fill:#34A853,stroke:#333,color:#fff
    style Protection fill:#FBBC05,stroke:#333,color:#333
```

### Key HLD Architectural Components:
1. **Global Anycast Ingress**: Requests are automatically routed to the nearest Google Edge Point of Presence (PoP) for sub-second first-byte latency across all storage classes.
2. **Durability vs. Availability**:
   - **Durability**: All storage classes provide **11 nines ($99.999999999\%$) annual durability** (data loss prevention).
   - **Availability**: Availability varies by storage class and location type (e.g., Multi-region Standard $99.95\%$, Regional Nearline $99.0\%$).
3. **Storage Classes & Min Retention**:
   - **Standard**: Hot data, zero minimum retention, no retrieval fee.
   - **Nearline**: Backup data accessed $<1$ /month, 30-day min charge.
   - **Coldline**: Disaster recovery accessed $<1$ /90 days, 90-day min charge.
   - **Archive**: Compliance archives accessed $<1$ /year, 365-day min charge, sub-second availability.
4. **Data Ingestion Engines**:
   - **Storage Transfer Service**: Scale online transfers from AWS S3, Azure Blob, or HTTP sources into GCS.
   - **Transfer Appliance**: Secure hardware devices (100 TB to 1 PB) shipped for offline data center migration.

---

## 2. Low-Level Design (LLD)

### 2.1 Strong Global Consistency Read-After-Write & Read-After-Delete Timeline

GCS guarantees **Strong Global Consistency** across all upload, update, delete, object listing, and bucket listing operations:

```mermaid
sequenceDiagram
    autonumber
    actor Client as Client / App
    participant Edge as GCS Edge Router
    participant Meta as Metadata & Index Directory
    participant Storage as Distributed Object Shards

    rect rgb(240, 248, 255)
    Note over Client, Storage: Phase 1: Upload / Overwrite (Strong Read-After-Write)
    Client->>Edge: PUT /bucket/report.pdf (Upload File)
    Edge->>Storage: Commit Replicated Shards
    Storage-->>Edge: Replicated Ack + MD5/CRC32c Verification
    Edge->>Meta: Atomic Metadata Update
    Meta-->>Client: 200 OK (ETag & Generation ID returned)
    Client->>Edge: GET /bucket/report.pdf (Immediate Read)
    Edge->>Meta: Query Metadata
    Meta-->>Client: 200 OK (Guaranteed Latest Data returned - No Stale Reads)
    end

    rect rgb(255, 245, 238)
    Note over Client, Storage: Phase 2: Immediate Read-After-Delete
    Client->>Edge: DELETE /bucket/report.pdf
    Edge->>Meta: Remove Object Index Entry
    Meta-->>Client: 204 No Content (Delete Success)
    Client->>Edge: GET /bucket/report.pdf (Immediate Read)
    Edge-->>Client: 404 Not Found (Guaranteed Immediate 404)
    end
```

---

### 2.2 Soft Delete vs Object Versioning State Engine

```mermaid
stateDiagram-v2
    [*] --> LiveObject: Upload New Object
    
    state LiveObject {
        [*] --> ActiveState
    }

    LiveObject --> SoftDeletedState: Object Deleted / Overwritten (Soft Delete Enabled)
    LiveObject --> ArchivedGeneration: Object Overwritten / Deleted (Versioning Enabled)

    state SoftDeletedState {
        RetainedInPool: Retained for Retention Duration (7 to 90 Days)
        RestoreSoftDelete: Can Restore to Live State
    }

    state ArchivedGeneration {
        GenIdAssigned: Uniquely Identified by #GENERATION ID
        ListableVersions: Listable via gcloud storage ls --all-versions
        RestoreArchived: Can Restore Older Generation to Live State
    }

    SoftDeletedState --> LiveObject: Restored via gcloud storage restore
    SoftDeletedState --> [*]: Purged Permanently after Retention Period Expires

    ArchivedGeneration --> LiveObject: Restored via gcloud storage cp obj#GEN obj
    ArchivedGeneration --> [*]: Permanently Deleted via gcloud storage rm obj#GEN
```

---

### 2.3 Access Control & Signed Policy Document Flow

```mermaid
sequenceDiagram
    autonumber
    actor User as Unauthenticated End User Browser
    participant App as App Server (Service Account Credentials)
    participant GCS as GCS API Endpoint

    App->>App: Construct Signed Policy Document JSON<br/>(Expiration, Bucket, Prefix, Max Size, Content-Type)
    App->>App: Sign Policy Document using SA Private Key
    App-->>User: Return HTML Form with Pre-Signed Policy & Signature
    User->>GCS: POST /bucket (Multipart HTML Form Data Upload)
    GCS->>GCS: Verify SA Signature & Validate Policy Constraints (Size <= 10MB)
    GCS-->>User: 204 No Content (Direct Upload Success)
```

---

### 2.5 Autoclass State Machine Engine

```mermaid
stateDiagram-v2
    [*] --> StandardState: Upload Object (Always Lands in Standard Class)

    StandardState --> NearlineState: Object Inactive for 30 Days
    NearlineState --> ColdlineState: Object Inactive for 90 Days
    ColdlineState --> ArchiveState: Object Inactive for 365 Days

    NearlineState --> StandardState: GET Read Access Triggered (Promotes back to Standard)
    ColdlineState --> StandardState: GET Read Access Triggered (Promotes back to Standard)
    ArchiveState --> StandardState: GET Read Access Triggered (Promotes back to Standard)

    note right of StandardState
        No retrieval fees,
        no transition fees,
        no early deletion fees.
    end note
```

---

### 2.6 LLD Mechanics Summary:
* **In-Place Storage Class Mutation**: Mutating an object's storage class (e.g., Standard to Coldline) modifies the object metadata in place without changing its URL, generation key, or moving file bytes across buckets.
* **Autoclass Promotion**: Under Autoclass, all uploads land in Standard Storage regardless of request parameters. Reading an object automatically promotes it back to Standard Storage with zero retrieval or transition penalties.
* **WORM Retention Lock**: Enforces immutable storage rules (`retentionPeriod`) for SEC/FINRA compliance. Once locked, policies cannot be weakened or removed by any IAM identity.
* **CSEK Key Handling**: Customer-supplied keys (AES-256) are held in memory for the duration of the request and never stored on Google servers.

