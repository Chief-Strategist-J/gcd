# Feature 4: Google Cloud BigQuery (`bq`) — High-Level Design (HLD) & Low-Level Design (LLD)

This document details the architectural design for **Google Cloud BigQuery Serverless Enterprise Data Warehouse** and **`bq` CLI Interaction**.

---

## 1. High-Level Design (HLD)

BigQuery separates compute (query engine) from storage (columnar storage), connected over Google's petabit Jupiter network fabric:

```mermaid
graph TD
    Client["Client / bq CLI / BI Tools"] -->|REST / gRPC API| FrontState["BigQuery API Frontend & Parser"]
    
    subgraph DremelEngine ["Dremel Distributed Query Engine (Compute)"]
        FrontState -->|Query Tree Optimization| RootNode["Root Server"]
        RootNode -->|Distribute Intermediate Aggregation| IntermediateNode1["Intermediate Mixer Node 1"]
        RootNode -->|Distribute Intermediate Aggregation| IntermediateNode2["Intermediate Mixer Node 2"]
        
        IntermediateNode1 -->|Assign Slots| LeafNode1["Leaf Node / Slot 1"]
        IntermediateNode1 -->|Assign Slots| LeafNode2["Leaf Node / Slot 2"]
        IntermediateNode2 -->|Assign Slots| LeafNode3["Leaf Node / Slot 3"]
    end

    subgraph ColossusStorage ["Colossus Distributed File System (Storage)"]
        LeafNode1 <-->|Colossus Petabit Jupiter Network| Capacitor1["Capacitor Columnar Shard (Partition 202609)"]
        LeafNode2 <-->|Colossus Petabit Jupiter Network| Capacitor2["Capacitor Columnar Shard (Cluster Key A)"]
        LeafNode3 <-->|Colossus Petabit Jupiter Network| Capacitor3["Capacitor Columnar Shard (Cluster Key B)"]
    end
    
    Capacitor1 -->|KMS Encryption| CMEK["Cloud KMS (Customer Keys)"]
```

### Key HLD Components:
1. **Dremel Execution Engine**: Multi-tenant compute engine that compiles SQL queries into execution trees of Mixer and Leaf nodes (Slots).
2. **Capacitor Columnar Storage**: Compressed, read-optimized binary storage format stored on Google's Colossus distributed file system.
3. **Jupiter Network Fabric**: Petabit-per-second network topology connecting thousands of compute slots to Colossus storage shards without bandwidth bottlenecks.
4. **Separation of Compute & Storage**: Slots are allocated dynamically per query, enabling serverless auto-scaling from zero to thousands of vCPUs.

---

## 2. Low-Level Design (LLD)

### BigQuery Job Execution State Machine:

```mermaid
stateDiagram-v2
    [*] --> PENDING: bq query / bq load (Job ID generated)
    PENDING --> RUNNING: Allocated Dremel slots & scheduled on Leaf nodes
    RUNNING --> Extracting: Reading Capacitor Columnar shards
    RUNNING --> Mixing: Intermediate aggregation & shuffle
    RUNNING --> DONE: Query results materializing
    RUNNING --> FAILED: Syntax Error / Exceeded Slot Limit / Quota
    DONE --> [*]: Output cached or written to destination table
    FAILED --> [*]: Job error logged to Cloud Audit Logs
```

### LLD Internal Mechanics:
* **Partitioning & Pruning**: Partitioning divides a table based on daily time-unit (`DAY`, `MONTH`) or integer ranges. During query execution, BigQuery prunes unreferenced partition shards before reading storage, cutting cost and latency.
* **Clustering**: Groups data within partitions based on up to 4 column keys. BigQuery sorts and groups contiguous data blocks, eliminating full table scans.
* **Cached Query Results**: Queries yielding identical SQL text return instant $0-cost cached results within 24 hours if underlying table data remains unchanged.
