# Feature 10: Google Cloud Pub/Sub (Enterprise Messaging & Event Streaming)

Welcome to the dedicated documentation index for **Google Cloud Pub/Sub**. This feature module provides architectural blueprints, low-level message lifecycle mechanics, decision trees, production CLI command manuals, failure recovery runbooks, and enterprise integration patterns for global asynchronous messaging and real-time stream processing.

---

## Module Documentation Index

| Section | Document File | Description |
| :--- | :--- | :--- |
| **Architecture (HLD & LLD)** | [`hld-lld-design.md`](./hld-lld-design.md) | High-Level Architecture (Global Data Plane, Distributed Routers, Storage Brokers, and Fan-Out topology) & Low-Level Design (Message Acknowledgment State Machine, Exactly-Once Delivery leases, Ordering Key partitions, Dead-Letter Queues, and Avro/Protobuf Schema validation). |
| **Decision Trees** | [`decision-tree.md`](./decision-tree.md) | ASCII & Visual Mermaid decision trees for Subscription Type Selection (Pull vs Push vs BigQuery Direct vs Cloud Storage Direct), Delivery Guarantee Tuning (At-Least-Once vs Exactly-Once vs Ordering), and Service Selection (Pub/Sub vs Pub/Sub Lite vs Cloud Tasks vs Kafka). |
| **CLI Commands & Error Matrix** | [`shell-commands.md`](./shell-commands.md) | Comprehensive `gcloud pubsub` production command reference, detailed parameter breakdowns (topics, pull/push/bigquery/storage subscriptions, ordering keys, filter expressions, snapshots, seeking), and an exhaustive Troubleshooting Error Matrix. |
| **Official References** | [`references.md`](./references.md) | Official Google Cloud Pub/Sub documentation links, Client Library SDKs, Quotas & Limits guides, and enterprise reliability best practices. |

---

## Key Technical & Operational Concepts Covered

1. **Global Ingestion Fabric & Decoupled Topology**:
   - Asynchronous 1-to-N message fan-out separating message producers (publishers) from consumers (subscribers).
   - Global endpoint availability with regional message storage policy enforcement for data sovereignty and compliance.
2. **Four Distinct Subscription Models**:
   - **Pull Subscriptions**: Clients lease messages on demand via unary or high-throughput bidirectional gRPC StreamingPull.
   - **Push Subscriptions**: Google Cloud pushes messages via HTTPS POST to webhooks, Cloud Run, or Cloud Functions with automatic OIDC authentication.
   - **BigQuery Direct Subscriptions**: Ingests streaming messages directly into BigQuery tables without intermediary Dataflow or Cloud Run pipelines, eliminating operational overhead.
   - **Cloud Storage Direct Subscriptions**: Batches and writes streaming event streams directly to Cloud Storage buckets (Text or Avro format).
3. **Advanced Reliability & Delivery Guarantees**:
   - **At-Least-Once Delivery**: Default high-throughput guarantee with redelivery on unacknowledged messages.
   - **Exactly-Once Delivery (EOD)**: Guaranteed deduplication per subscription with stateful lease tracking across distributed subscriber clients.
   - **Message Ordering with Ordering Keys**: Sequential, FIFO delivery per ordering key partition without global throughput bottlenecks.
   - **Dead-Letter Topics (DLQ)**: Automatic isolation of poisonous/malformed messages after configurable maximum delivery attempts (`max-delivery-attempts`).
4. **Message Attribute Filtering & Schema Validation**:
   - Server-side attribute filtering using boolean SQL-like predicates, discarding irrelevant messages before billing and network egress.
   - Strict contract enforcement using **Apache Avro** and **Protocol Buffers (Protobuf)** schemas with backward/forward compatibility rules.
5. **Time-Travel Replay & Snapshot Operations**:
   - Replaying message history up to 31 days using **Snapshots** and **Seek** operations to roll back state after consumer failures.

---

## Quick Navigation

1. [View High-Level & Low-Level Design (`hld-lld-design.md`)](./hld-lld-design.md)
2. [View Decision Trees (`decision-tree.md`)](./decision-tree.md)
3. [View Shell Command Reference & Error Matrix (`shell-commands.md`)](./shell-commands.md)
4. [View Official Reference Links (`references.md`)](./references.md)
