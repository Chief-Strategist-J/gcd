# Google Cloud SQL & AlloyDB: Architecture, Operations & Decision Manual

Welcome to the **Google Cloud SQL & AlloyDB for PostgreSQL** documentation module. This folder contains production-grade architecture designs, High Availability (HA) failover mechanics, connection security strategies, generalized `gcloud sql` and `gcloud alloydb` CLI operational manuals, and relational database decision trees.

---

## Module Sitemap

| Document | Description |
| :--- | :--- |
| [**HLD & LLD Architecture Design**](file:///home/btpl-lap-22/live/gcd/cloud-sql/hld-lld-design.md) | High-Level Architecture for Regional HA failover, synchronous disk replication, Private IP via VPC peering, Cloud SQL Auth Proxy client flows, and AlloyDB disaggregated multi-node HTAP architecture. |
| [**Decision Tree Guide**](file:///home/btpl-lap-22/live/gcd/cloud-sql/decision-tree.md) | Visual decision flowcharts for selecting Relational & In-Memory storage (Cloud SQL vs. AlloyDB vs. Cloud Spanner vs. BigQuery vs. Memorystore), HA vs. Standalone, and Connection models (Private IP vs. Auth Proxy vs. SSL vs. Authorized Networks). |
| [**CLI Shell Commands & Operations Manual**](file:///home/btpl-lap-22/live/gcd/cloud-sql/shell-commands.md) | Exhaustive, generalized operational command manual for `gcloud sql` and `gcloud alloydb` covering instance creation (MySQL, PostgreSQL, SQL Server, AlloyDB clusters/instances), storage scaling, HA failover testing, backups, PITR, Private IP, Auth Proxy, and read replicas/pools. |
| [**Official References & Best Practices**](file:///home/btpl-lap-22/live/gcd/cloud-sql/references.md) | Official Google Cloud links, engine version support matrices, limits, AlloyDB columnar engine specs, Gemini AI platform integration, and security best practices. |

---

## Key Operational Capabilities Covered

1. **Cloud SQL Engine Support & Specifications**:
   - **Supported Engines**: MySQL (5.6, 5.7, 8.0), PostgreSQL (9.6 to 15), Microsoft SQL Server (2017, 2019 - Web, Express, Standard, Enterprise).
   - **Performance Limits**: Up to 64 TB storage capacity, 60,000 IOPS, 624 GB RAM, and scale up to 96 vCPU processor cores per instance.
2. **AlloyDB for PostgreSQL Enterprise Capabilities**:
   - **Full Managed PostgreSQL Compatibility**: 100% PostgreSQL-compatible hybrid transactional and analytical processing (HTAP) database engine.
   - **HTAP Performance**: $>4 \times$ faster transactional (OLTP) performance and up to $100 \times$ faster analytical (OLAP) queries via columnar in-memory acceleration.
   - **Availability & SLA**: **99.99% Uptime SLA** inclusive of maintenance windows.
   - **Adaptive ML & AI Platform**: ML-driven adaptive vacuuming, memory management, and built-in integration with Gemini / Vertex AI platform for in-database ML inference.
3. **High Availability (HA) & Recovery**:
   - Cloud SQL Regional HA configuration: Primary instance in Zone A, Standby instance in Zone B with synchronous disk replication.
   - AlloyDB multi-node cluster configuration with decoupled, disaggregated storage.
   - Automated and on-demand backups with Point-In-Time Recovery (PITR).
4. **Secure Connection Strategies**:
   - **Private IP Connection**: High-performance, zero-internet-exposure connectivity via VPC Private Services Access.
   - **Cloud SQL Auth Proxy**: Secure proxy managing IAM authentication, mTLS encryption, and automatic key rotation.
