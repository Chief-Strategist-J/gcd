# Cloud SQL: Decision Trees (ASCII & Visual)

This document provides architectural decision trees for selecting GCP relational database services (Cloud SQL vs. Cloud Spanner vs. BigQuery vs. Memorystore), connection models, and High Availability configurations.

---

## 1. GCP Database & Storage Selection (ASCII Decision Tree)

```
================================================================================
                GCP DATABASE & DATA STORE SELECTION DECISION TREE
================================================================================

                  What type of data and workload is required?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
 [ In-Memory Caching ]          [ Relational Analytics ]        [ Relational OLTP / HTAP ]
  Microsecond response,          Data Warehousing, SQL          Transactional & Hybrid SQL,
  real-time analytics, gaming    reporting over petabytes       ACID compliance
        │                              │                              │
        ▼                              ▼                              │
 [ MEMORYSTORE ]                [ BIGQUERY ]                          │
  Managed Redis/Memcached        Serverless Data Warehouse            │
                                                                      ▼
                                                   Do you need global scale / horizontal scaling?
                                                                      │
                        ┌─────────────────────────────────────────────┴─────────────────────────────────────────────┐
                        ▼                                                                                           ▼
                     [ YES ]                                                                                     [ NO ]
              [ CLOUD SPANNER ]                                                             Is this a high-demanding HTAP / PostgreSQL workload?
              Unlimited scale, global                                                                               │
              synchronous transactions                                                          ┌───────────────────┴───────────────────┐
                                                                                                ▼                                       ▼
                                                                                             [ YES ]                                  [ NO ]
                                                                                       [ ALLOYDB POSTGRES ]                       [ CLOUD SQL ]
                                                                                       >4x OLTP, 100x OLAP,                       Cost-effective managed
                                                                                       99.99% SLA, Columnar                       MySQL, Postgres, SQL Server
```

---

## 2. Connection Strategy Flowchart

```mermaid
flowchart TD
    START_CONN["Select Cloud SQL Connection Model"] --> LOCATION{"Where is the connecting application hosted?"}

    LOCATION -- "Same Project & Region inside GCP" --> PERF{"Prioritize maximum performance & private networking?"}
    PERF -- "Yes (Zero Internet Exposure)" --> PRIVATE_IP["Use Private IP Connection<br/>(VPC Private Services Access / Peering)"]
    PERF -- "Standard Connection" --> AUTH_PROXY

    LOCATION -- "Cross-Region, Cross-Project, or External / On-Prem" --> SEC_MODE{"Do you want automated IAM auth & SSL rotation?"}

    SEC_MODE -- "Yes (Recommended Default)" --> AUTH_PROXY["Use Cloud SQL Auth Proxy<br/>(Automated IAM auth, SSL & key rotation)"]
    SEC_MODE -- "Manual Certificate Management" --> SSL["Use Manual SSL/TLS Certificates<br/>(Client cert generation & manual rotation)"]
    SEC_MODE -- "Unencrypted / Simple External IP" --> AUTH_NET["Use Authorized Networks<br/>(Public IP restricted to whitelisted CIDRs)"]

    style PRIVATE_IP fill:#34A853,color:#fff
    style AUTH_PROXY fill:#4285F4,color:#fff
    style SSL fill:#FBBC05,color:#333
    style AUTH_NET fill:#EA4335,color:#fff
```

---

## 3. High Availability & Scaling Decision Tree

```mermaid
flowchart TD
    START_HA["Evaluate Availability & Scaling Requirements"] --> HA_CHECK{"Can your business tolerate downtime during zone failure?"}

    HA_CHECK -- "No (Production Critical / SLA Required)" --> REGIONAL_HA["Deploy Regional High Availability (HA)<br/>(Primary in Zone A, Standby in Zone B, Synchronous Disk Rep)"]
    HA_CHECK -- "Yes (Dev / Staging / Non-Critical)" --> ZONAL["Deploy Zonal Instance<br/>(Single Zone, lower cost)"]

    REGIONAL_HA --> SCALE_TYPE{"What type of load increase is expected?"}
    ZONAL --> SCALE_TYPE

    SCALE_TYPE -- "Read-Heavy Query Volume" --> READ_REPLICAS["Add Read Replicas<br/>(Scale out read capacity up to multiple replicas)"]
    SCALE_TYPE -- "Write Volume / CPU / RAM Saturation" --> VERTICAL_SCALE["Scale Up Machine Tier<br/>(Increase vCPUs up to 96 cores & RAM up to 624 GB)"]
    SCALE_TYPE -- "Storage Capacity Full" --> AUTO_STORAGE["Enable Storage Auto-Increase<br/>(Automatically expands up to 64 TB)"]

    style REGIONAL_HA fill:#34A853,color:#fff
    style ZONAL fill:#FBBC05,color:#333
    style READ_REPLICAS fill:#4285F4,color:#fff
    style VERTICAL_SCALE fill:#EA4335,color:#fff
    style AUTO_STORAGE fill:#34A853,color:#fff
```
