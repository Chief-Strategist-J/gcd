# Cloud Storage: Decision Trees (ASCII & Visual)

This document provides architectural decision trees for selecting Cloud Storage classes, location redundancy strategies, data protection/recovery models, access security frameworks, and data ingestion services.

---

## 1. Storage Class & Automation Strategy (ASCII Decision Tree)

```
================================================================================
                    CLOUD STORAGE CLASS DECISION TREE
================================================================================

                    How frequently is your object data accessed?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
 [ Multiple Times / Month ]    [ Once Per Month ]            [ Infrequent Access ]
  Active Web Assets, API Logs,  Monthly Backups,              Archival & Disaster Recovery
  Data Processing Input         Reporting Data
        │                              │                              │
        ▼                              ▼                              ├──────────────────────────────┐
 [ STANDARD CLASS ]            [ NEARLINE CLASS ]                     ▼                              ▼
  No retrieval fees             Low storage cost               [ Once per 90 Days ]           [ Once per Year ]
  Highest availability          30-day min charge              COLDLINE CLASS                 ARCHIVE CLASS
                                                               90-day min charge              365-day min charge

────────────────────────────────────────────────────────────────────────────────
                    STORAGE CLASS AUTOMATION DECISION
────────────────────────────────────────────────────────────────────────────────

               Do object access patterns change unpredictably?
                                       │
                   ┌───────────────────┴───────────────────┐
                   ▼                                       ▼
                 [ YES ]                                 [ NO ]
    Enable Autoclass on bucket              Configure Object Lifecycle Rules (OLM)
    (Managed automated tiering)             (Static age / date / versioning rules)
```

---

## 2. Location Strategy (ASCII Decision Tree)

```
================================================================================
                    BUCKET LOCATION STRATEGY DECISION TREE
================================================================================

                    What are your data distribution requirements?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
 [ Lowest Latency ]            [ High Availability ]          [ Global Content ]
  Single Region compute         Active-Active failover         Global web assets,
  e.g. us-central1              e.g. nam4 (us-central+east)    streaming, mobile/gaming
        │                              │                              │
        ▼                              ▼                              ▼
 [ REGIONAL BUCKET ]           [ DUAL-REGION BUCKET ]         [ MULTI-REGION BUCKET ]
```

---

## 3. Data Protection & Recovery Strategy (Soft Delete vs Versioning vs Lock)

```mermaid
flowchart TD
    START["Evaluate Data Protection Need"] --> ACCIDENTAL{"Primary risk: Accidental / Malicious Deletions or Overwrites?"}

    ACCIDENTAL -- "Default Protection (Restoring Deleted Objects)" --> SOFT_DELETE["Use Soft Delete (Default 7d retention, 0-90 days)<br/>Recommended baseline by Google"]
    ACCIDENTAL -- "Maintain Full Overwrite History & Generations" --> VERSIONING["Enable Object Versioning<br/>(Track #GENERATION IDs, bill for older versions)"]

    SOFT_DELETE --> COMPLIANCE{"Requires Immutable Regulatory Compliance (WORM)?"}
    VERSIONING --> COMPLIANCE

    COMPLIANCE -- "Yes (SEC / FINRA / CFTC Rules)" --> RETENTION_LOCK["Configure Bucket Retention Policy & Lock Policy<br/>(Permanent enforcement, cannot unlock/delete early)"]
    COMPLIANCE -- "No" --> DONE["Standard Object Protection Active"]

    style SOFT_DELETE fill:#34A853,color:#fff
    style VERSIONING fill:#4285F4,color:#fff
    style RETENTION_LOCK fill:#EA4335,color:#fff
```

---

## 4. Data Ingestion Service Decision Tree

```mermaid
flowchart TD
    START_INGEST["Determine Data Transfer Source & Volume"] --> VOL{"What is the data size and source location?"}

    VOL -- "CLI Scripts / Local Files (< 1 TB)" --> GCLOUD_RSYNC["Use 'gcloud storage rsync' or 'cp'"]
    VOL -- "Online Cloud / HTTP Source (S3, Azure, GCS, HTTP)" --> STS["Use Storage Transfer Service (STS)<br/>(Managed online high-performance transfer)"]
    VOL -- "Offline On-Premises Terabytes / Petabytes (> 100 TB)" --> PHYSICAL{"Can network bandwidth support streaming?"}

    PHYSICAL -- "No (Saturated / Slow Network Connection)" --> TA["Use Google Transfer Appliance<br/>(100 TB to 1 PB physical hardware)"]
    PHYSICAL -- "Offline Legacy Tapes / Disks" --> OMI["Use Offline Media Import<br/>(3rd Party Data Center Partner)"]

    style GCLOUD_RSYNC fill:#4285F4,color:#fff
    style STS fill:#34A853,color:#fff
    style TA fill:#FBBC05,color:#333
    style OMI fill:#EA4335,color:#fff
```

---

## 5. Security & Access Control Model Decision Tree

```mermaid
flowchart TD
    START_SEC["Evaluate Access Security Requirement"] --> MODEL{"Is authentication tied to Google IAM identities?"}

    MODEL -- "Yes (Google User or Service Account)" --> IAM_SCOPE{"Centralized coarse policy or fine-grained per-object entries?"}
    IAM_SCOPE -- "Centralized / Org Standard" --> IAM["Use Cloud IAM Roles<br/>(Project & Bucket Level Roles)"]
    IAM_SCOPE -- "Legacy Fine-Grained Entries (Max 100)" --> ACL["Use Object ACLs<br/>(allUsers, allAuthenticatedUsers, READ/WRITE)"]

    MODEL -- "No (Unauthenticated End Users)" --> METHOD{"Direct file access or constrained POST form upload?"}
    METHOD -- "Time-Limited File Access (GET / PUT)" --> SIGNED_URL["Use Signed URLs<br/>(Signed via SA Key, 1m to 7d expiration)"]
    METHOD -- "Constrained Form Upload (Size, MIME, Prefix)" --> SIGNED_POLICY["Use Signed Policy Documents<br/>(HTML Form POST upload rules)"]

    style IAM fill:#34A853,color:#fff
    style ACL fill:#FBBC05,color:#333
    style SIGNED_URL fill:#4285F4,color:#fff
    style SIGNED_POLICY fill:#EA4335,color:#fff
```

---

## 6. Unstructured Storage Architecture: Filestore vs Cloud Storage

```mermaid
flowchart TD
    START_STORE["Evaluate Storage Architecture Needs"] --> TYPE{"What protocol & access model is required?"}

    TYPE -- "POSIX Filesystem / NFSv3 Shared Storage (GCE / GKE)" --> FILESTORE["Use Google Cloud Filestore<br/>(Fully managed POSIX file storage for latency-sensitive apps)"]
    TYPE -- "RESTful API / Web Blob / Exabyte Object Storage" --> GCS["Use Google Cloud Storage (GCS)<br/>(Global HTTP REST object storage with 4 storage classes)"]

    style FILESTORE fill:#4285F4,color:#fff
    style GCS fill:#34A853,color:#fff
```

