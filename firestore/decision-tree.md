# Firestore Decision Trees & Selection Logic

This document provides structured ASCII decision trees to guide database selection, operating mode determination, data modeling strategies, and multi-region replication choices.

---

## Decision Tree 1: Primary Database Selection (Firestore vs Bigtable vs Cloud SQL vs BigQuery)

Use this decision tree to evaluate whether **Firestore** or another Google Cloud database service is the optimal choice for your workload.

```
                          [What is your primary data storage requirement?]
                                                  |
       +------------------------------------------+------------------------------------------+
       |                                                                                     |
[Relational / SQL Schema?]                                                    [NoSQL / Unstructured / Semi-Structured?]
       |                                                                                     |
       +-----------------------------------+                                                 +-----------------------------------+
       |                                   |                                                 |                                   |
[Need strict SQL joins          [Global scale,                             [Analytical / Warehouse           [Operational Operational Database /
 & legacy RDBMS compat?]        multi-region SQL?]                          OLAP at petabyte scale?]          Real-time App Data Store?]
       |                                   |                                                 |                                   |
       v                                   v                                                 v                                   v
+--------------+                   +---------------+                                 +---------------+                           |
|  Cloud SQL   |                   | Cloud Spanner |                                 |   BigQuery    |                           |
| (MySQL /     |                   +---------------+                                 +---------------+                           |
| PostgreSQL / |                                                                                                                 |
| SQL Server)  |                                                                                                                 |
+--------------+                                                                                                                 |
                                                                                                                                 |
                                          +--------------------------------------------------------------------------------------+
                                          |
                        [What is the data scale, throughput, & access pattern?]
                                          |
       +----------------------------------+----------------------------------+
       |                                                                     |
[Sub-10ms latency, high-throughput                          [Adaptable dynamic schema, scale to zero,
 key-value / wide-column, petabyte scale,                   real-time live sync listeners, mobile/web SDKs,
 analytical throughput?]                                    terabyte scale, multi-document ACID transactions?]
       |                                                                     |
       v                                                                     v
+------------------+                                               +-------------------+
|  Cloud Bigtable  |                                               |  Cloud Firestore  |
+------------------+                                               +-------------------+
```

---

## Decision Tree 2: Firestore Operating Mode Selection (Native Mode vs Datastore Mode)

```
                            [Which Firestore Operating Mode should you select?]
                                                     |
            +----------------------------------------+----------------------------------------+
            |                                                                                 |
[Building a new Mobile, Web,                              [Building a server-side backend, app Engine,
 or IoT App requiring Client SDKs?]                        or migrating legacy Google Cloud Datastore?]
            |                                                                                 |
            v                                                                                 v
   +--------------------+                                                            +--------------------+
   | Needs real-time    |                                                            | Requires 100%      |
   | live sync data     |--YES------------------------------------------------------>| Datastore API      |--NO--+
   | listeners?         |                                                            | compatibility?     |      |
   +--------------------+                                                            +--------------------+      |
            |                                                                                 |                  |
            | YES                                                                             | YES              |
            v                                                                                 v                  |
+--------------------------+                                                       +---------------------+       |
| Firestore in Native Mode |                                                       | Firestore in        |       |
| - Document & Collection  |                                                       | Datastore Mode      |       |
|   data model             |                                                       | - Entity & Keys     |       |
| - Firebase Security Rules|                                                       |   data model        |       |
| - Real-time listeners    |                                                       | - Server client     |       |
| - Offline data caching   |                                                       |   libraries         |       |
+--------------------------+                                                       | - Strongly          |       |
                                                                                   |   consistent queries|       |
                                                                                   | - Removes 1 write/sec|       |
                                                                                   |   entity group limit|       |
                                                                                   +---------------------+       |
                                                                                                                 |
                                                                                                                 |
   +-------------------------------------------------------------------------------------------------------------+
   |
   v
[General Architectural Guideline]:
- **Firestore Native Mode**: Default recommendation for modern web, mobile, serverless, and multi-client apps.
- **Firestore Datastore Mode**: Recommendation for high-volume backend microservices that do not utilize real-time sync listeners and demand high single-entity write throughput or Datastore library compatibility.
```

---

## Decision Tree 3: Data Modeling Strategy (Native Mode)

```
                       [How should you structure data relationships in Firestore?]
                                                   |
       +-------------------------------------------+-------------------------------------------+
       |                                                                                       |
[Sub-documents / Child Data < 20KB                 [Large child collection, > 100 items,               [Independent top-level
 and queried strictly with parent?]                 or queried independently across parents?]           entities queried standalone?]
       |                                                                                       |
       v                                                                                       v
+-----------------------------+                                                         +-----------------------------+
| Embed as Map / Array Field  |                                                         | Create Nested Subcollection |
| in Parent Document          |                                                         | e.g. /users/{id}/orders     |
+-----------------------------+                                                         +-----------------------------+
       |                                                                                       |
       | If querying sub-items across all parents                                              | Query using Collection Group
       +-------------------------------------------------------------------------------------->| Query syntax
                                                                                               +-----------------------------+
```

---

## Decision Tree 4: Replication & Location Strategy

```
                          [Which Location Type should you select for Firestore?]
                                                     |
       +---------------------------------------------+---------------------------------------------+
       |                                                                                           |
[Primary consumers distributed across                     [Data consumers and app compute (e.g. GKE,
 multiple geographic regions, or require                   Cloud Run, App Engine) strictly co-located
 maximum SLA resiliency (99.999%)?]                        in a single region to optimize latency?]
       |                                                                                           |
       v                                                                                           v
+------------------------------------+                                                     +------------------------------------+
| Select Multi-Region Location       |                                                     | Select Regional Location           |
| (e.g., nam5 [US], eur3 [Europe])   |                                                     | (e.g., us-central1, europe-west1)  |
| - Automatic multi-region failover  |                                                     | - Lower cross-region egress cost   |
| - Highest disaster resiliency      |                                                     | - Optimized write latency for      |
+------------------------------------+                                                     |   single-region workloads          |
                                                                                           +------------------------------------+
```
