# Google Cloud Pub/Sub: Decision Trees (ASCII & Visual)

This document provides structured decision logic trees for selecting Pub/Sub subscription architectures, delivery guarantees, message ordering strategies, and enterprise messaging platform choices.

---

## 1. Subscription Type Selection (ASCII Decision Tree)

```
================================================================================
               PUB/SUB SUBSCRIPTION TYPE DECISION TREE
================================================================================

              Where is your message consumer running and what is its workload?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
  [ Compute Cluster / Pods ]    [ Serverless / Webhook ]       [ Analytics / Storage Sink ]
  GKE, Compute Engine,          Cloud Run, Cloud Run           Data Lake, Warehouse,
  Dataflow streaming            Functions, Third-Party URL     Long-term Archival
        │                              │                              │
        ▼                              ▼                              ├───────────────────────┐
  USE PULL SUBSCRIPTION         USE PUSH SUBSCRIPTION                 ▼                       ▼
  - High throughput             - Event-driven autoscaling     Destination: BigQuery?  Destination: GCS?
  - Bidirectional gRPC stream   - HTTPS POST endpoint                 │                       │
  - Client controls batch rate  - Automatic OIDC token auth           ▼                       ▼
  - Custom ack management       - Zero idle consumer costs     USE BIGQUERY DIRECT     USE CLOUD STORAGE
                                                               SUBSCRIPTION            DIRECT SUBSCRIPTION
                                                               - Zero pipeline code    - Direct batching
                                                               - Auto table routing    - Text / Avro output
```

---

## 2. Delivery Guarantees & Reliability Strategy (ASCII Decision Tree)

```
================================================================================
           PUB/SUB DELIVERY GUARANTEES & RELIABILITY DECISION TREE
================================================================================

                 What are your consistency & ordering requirements?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
  [ Standard Throughput ]       [ Strict Idempotency ]         [ Sequential Processing ]
  Idempotent consumers          Duplicate messages             Financial transactions,
  tolerating redeliveries       cause corrupt state            CDC changelogs, audit logs
        │                              │                              │
        ▼                              ▼                              ▼
  STANDARD AT-LEAST-ONCE        USE EXACTLY-ONCE               ENABLE ORDERING KEYS
  - Default high performance    DELIVERY (EOD)                 - Publish with `orderingKey`
  - Sub-100ms latency           - Stateful lease tracking      - Sequential delivery per key
  - Minimal overhead            - Server-side deduplication    - Failures block key until acked
                                                                      │
                                                                      ▼
                                                               CRITICAL SAFETY:
                                                               Pair with Dead-Letter Topic
                                                               (DLQ) to avoid pipeline stalls!
```

---

## 3. GCP Messaging Service Selection (ASCII Decision Tree)

```
================================================================================
         GCP MESSAGING & STREAMING SERVICE SELECTION DECISION TREE
================================================================================

               What is your primary architectural communication pattern?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
  [ Event-Driven Pub/Sub ]      [ Point-to-Point Task Queue ]  [ Partitioned Streaming Log ]
  1-to-N fan-out, independent   1-to-1 dispatch, rate limiting, High throughput, low cost,
  subscribers, global fabric    scheduled future execution     manual partition management
        │                              │                              │
        ▼                              ▼                              ▼
  USE CLOUD PUB/SUB             USE CLOUD TASKS                USE PUB/SUB LITE / KAFKA
  - Fully managed, autoscaling  - Explicit rate limiting       - Zonal partition capacity
  - Global Anycast ingestion    - Task deduplication & delays  - Lowest cost per GB throughput
  - Rich GCP ecosystem bindings - HTTP webhook targets only    - Custom offset cursors
```

---

## 4. Visual Mermaid Decision Flowcharts

### 4.1 Subscription Architecture Flowchart:

```mermaid
graph TD
    StartSub["Select Subscription Architecture"] --> Dest{"What is the target consumer?"}

    Dest -->|Dedicated Microservice / GKE Worker| PullCheck{"Requires high-throughput pull or batch control?"}
    PullCheck -->|Yes: GKE / Compute Engine / Dataflow| UsePull["Use Pull Subscription<br/>• gRPC StreamingPull<br/>• Consumer controls leasing & concurrency"]

    Dest -->|Serverless Function / HTTP Endpoint| UsePush["Use Push Subscription<br/>• HTTPS POST delivery<br/>• Cloud Run / Cloud Functions<br/>• Automatic OIDC Authentication"]

    Dest -->|Direct Data Warehouse Storage| UseBQ["Use BigQuery Direct Subscription<br/>• Zero Dataflow / Cloud Run pipeline<br/>• Writes directly into BigQuery table<br/>• Lowest operational latency & cost"]

    Dest -->|Direct Object Storage Archival| UseGCS["Use Cloud Storage Direct Subscription<br/>• Batches streaming records into GCS<br/>• Writes raw Text or structured Avro<br/>• Eliminates intermediate batch runners"]
```

---

### 4.2 Delivery Guarantees & Error Handling Flowchart:

```mermaid
graph TD
    StartRel["Determine Delivery Guarantees"] --> NeedOrder{"Does processing order strictly matter?"}

    NeedOrder -->|Yes: Sequential per entity| OrderFlow["Enable Message Ordering Keys<br/>• Assign orderingKey per entity/customer<br/>• Guaranteed FIFO per key"]
    OrderFlow --> DLQCheck1{"Add Dead-Letter Topic?"}
    DLQCheck1 -->|Mandatory for safety| DLQ1["Configure Dead-Letter Topic (DLQ)<br/>• max-delivery-attempts: 5 to 10<br/>• Prevents 1 failed message from halting key!"]

    NeedOrder -->|No: Parallel independent processing| DupCheck{"Can consumers tolerate occasional duplicates?"}
    DupCheck -->|Yes: Naturally idempotent| StdAtLeastOnce["Use Standard At-Least-Once Delivery<br/>• Maximum global scalability<br/>• Sub-100ms latency"]
    DupCheck -->|No: Duplicates cause double charge| EODFlow["Enable Exactly-Once Delivery (EOD)<br/>• Stateful lease renewal<br/>• Eliminates redeliveries on acknowledged msgs"]
```

---

### 4.3 Messaging Platform Selector Flowchart:

```mermaid
graph TD
    StartPlatform["Choose Cloud Messaging Platform"] --> Pattern{"Messaging Pattern & Operational Model?"}

    Pattern -->|Event Fan-Out: 1 Producer to N Consumers| GlobalCheck{"Global managed fabric or lowest cost per partition?"}
    GlobalCheck -->|Global managed, zero maintenance| CloudPubSub["Google Cloud Pub/Sub<br/>• Fully managed global scale<br/>• Integrated Eventarc, BigQuery, GCS sinks"]
    GlobalCheck -->|Zonal predictable high volume at minimum cost| PubSubLite["Pub/Sub Lite<br/>• Provisioned throughput & storage capacity<br/>• Up to 80% lower cost for massive streams"]

    Pattern -->|Asynchronous Task Dispatch: 1-to-1 Target| CloudTasks["Google Cloud Tasks<br/>• Rate limiting & token bucket dispatch<br/>• Scheduled execution at specific future timestamp<br/>• Task deduplication windows"]
```
