# Google Cloud Firestore Reference Index

Welcome to the **Google Cloud Firestore Master Reference**. Firestore is a fast, fully managed, serverless, cloud-native, NoSQL document database designed for storing, syncing, and querying data for mobile, web, backend, and IoT applications at global scale.

---

## Directory Navigation

- **[HLD & LLD Architecture (`firestore/hld-lld-design.md`)](file:///home/btpl-lap-22/live/gcd/firestore/hld-lld-design.md)**
  High-Level Architecture (Spanner-based storage layer, multi-region replication, API access tier) and Low-Level Design (Real-Time Live Sync & Offline Caching Sequence, ACID Multi-Document Transaction Flow, Operating Mode State Machine).
- **[ASCII & Visual Decision Trees (`firestore/decision-tree.md`)](file:///home/btpl-lap-22/live/gcd/firestore/decision-tree.md)**
  Structured decision trees for database selection (Firestore vs Bigtable vs Cloud SQL vs BigQuery), operating mode selection (Native Mode vs Datastore Mode), data modeling, and multi-region replication.
- **[Shell Command Reference & Failure Resolutions (`firestore/shell-commands.md`)](file:///home/btpl-lap-22/live/gcd/firestore/shell-commands.md)**
  Comprehensive `gcloud firestore` CLI manual featuring database creation, composite index configuration, managed import/export, PITR backups, IAM access control, and an explicit Error Matrix.
- **[Official References & Links (`firestore/references.md`)](file:///home/btpl-lap-22/live/gcd/firestore/references.md)**
  Curated links to Google Cloud and Firebase official documentation, mode migration guides, security rule definitions, and indexing quotas.

---

## Firestore Operating Modes Comparison

Firestore operates in two distinct modes. Selecting the correct mode at database creation time is critical based on application architecture and client requirements:

| Feature / Capability | Firestore in Native Mode | Firestore in Datastore Mode |
| :--- | :--- | :--- |
| **Primary Target Workload** | Mobile, Web, and Client-facing Serverless Apps | Server Backends, App Engine, High-Throughput Batch |
| **Data Model** | Collections, Documents, and Subcollections | Entities, Keys, Properties, and Entity Groups |
| **Real-time Live Sync** | Supported (Listeners push live updates to clients) | Not Supported (Pull query based) |
| **Client SDKs** | Mobile (iOS/Android), Web, Unity, C++, Server SDKs | Server-side Client Libraries (Node, Python, Java, Go) |
| **Offline Data Persistence** | Supported (Automatic client-side local caching & offline sync) | Not Supported |
| **Consistency Model** | Strongly Consistent (Reads & Queries) | Strongly Consistent (Removes legacy Datastore eventual consistency) |
| **Transaction Limits** | ACID Multi-document / Multi-collection | ACID (Removes legacy Datastore 25 entity group limit) |
| **Entity Group Write Rate** | High throughput document mutation | Unlimited (Removes legacy Datastore 1 write/sec limit) |
| **Security Architecture** | Firebase Security Rules + IAM | Google Cloud IAM roles (`roles/datastore.user`) |
| **Backwards Compatibility** | New Firestore native protocol & APIs | 100% compatible with Datastore APIs and client libraries |

---

## Key Core Capabilities

1. **Serverless & Scale-to-Zero**: Automatic scaling up to terabytes of data without server provisioning or cluster management, automatically scaling down to zero when idle.
2. **ACID Transactions**: Full multi-document atomic operations. If any operation in a transaction fails and cannot be retried, the entire transaction rolls back cleanly.
3. **Multi-Region Replication & Strong Consistency**: Automatic geo-redundant replication across regions delivering strong consistency, data durability, and high availability during disaster events.
4. **Sophisticated NoSQL Queries**: Compound filtering and sorting on document fields without performance degradation due to automatic indexing.
