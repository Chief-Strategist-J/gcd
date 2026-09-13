# Cloud SQL: Official Reference Links & Resources

This document provides official Google Cloud reference links, developer guides, security specifications, and API documentation for Cloud SQL and related database offerings.

---

## 1. Official Documentation Links

| Resource Title | URL Link | Description |
| :--- | :--- | :--- |
| **GCP Cloud SQL Overview** | [cloud.google.com/sql/docs](https://cloud.google.com/sql/docs) | Official landing page and documentation for Cloud SQL. |
| **`gcloud sql` CLI Manual** | [cloud.google.com/sdk/gcloud/reference/sql](https://cloud.google.com/sdk/gcloud/reference/sql) | Complete command and parameter reference for managing instances, databases, users, and backups. |
| **High Availability (HA) Guide** | [cloud.google.com/sql/docs/mysql/high-availability](https://cloud.google.com/sql/docs/mysql/high-availability) | Specifications for regional instances, synchronous persistent disk replication, and automatic failover. |
| **Cloud SQL Auth Proxy** | [cloud.google.com/sql/docs/mysql/sql-proxy](https://cloud.google.com/sql/docs/mysql/sql-proxy) | Client proxy for secure mTLS connections with IAM authentication and automatic key rotation. |
| **Private IP & VPC Peering** | [cloud.google.com/sql/docs/mysql/private-ip](https://cloud.google.com/sql/docs/mysql/private-ip) | Configuring Private Services Access for zero-internet-exposure connectivity. |
| **Point-In-Time Recovery (PITR)** | [cloud.google.com/sql/docs/mysql/backup-recovery/pitr](https://cloud.google.com/sql/docs/mysql/backup-recovery/pitr) | Automated binary log archiving and instance cloning to exact timestamps. |
| **Instance Tiers & Quotas** | [cloud.google.com/sql/docs/mysql/instance-settings](https://cloud.google.com/sql/docs/mysql/instance-settings) | Machine types (up to 96 vCPUs, 624 GB RAM, 64 TB storage, 60k IOPS). |
| **Cloud Spanner Overview** | [cloud.google.com/spanner/docs](https://cloud.google.com/spanner/docs) | Enterprise horizontally scalable global relational database. |
| **AlloyDB for PostgreSQL** | [cloud.google.com/alloydb/docs](https://cloud.google.com/alloydb/docs) | Fully managed PostgreSQL-compatible HTAP engine (>4x OLTP, 100x OLAP, 99.99% SLA). |
| **`gcloud alloydb` CLI Manual** | [cloud.google.com/sdk/gcloud/reference/alloydb](https://cloud.google.com/sdk/gcloud/reference/alloydb) | Command reference for managing AlloyDB clusters, primary instances, and read pools. |
| **Memorystore (Redis/Memcached)** | [cloud.google.com/memorystore/docs](https://cloud.google.com/memorystore/docs) | In-memory data store for microsecond response times. |

---

## 2. Recommended Operational Best Practices

1. **High Availability**: Always select `--availability-type=REGIONAL` for production instances to ensure automatic failover and zero data loss via synchronous cross-zone disk replication.
2. **Connectivity**: Use **Private IP** for in-region workloads and **Cloud SQL Auth Proxy** for cross-region, cross-project, or external connections to eliminate manual SSL certificate management.
3. **Storage Management**: Enable `--storage-auto-increase` to prevent outages caused by unexpected database storage exhaustion.
4. **Disaster Recovery**: Maintain continuous binary logging (`--enable-bin-log`) to enable Point-In-Time Recovery (PITR) for accidental data corruption recovery.
5. **Database Selection**: Use **Cloud SQL** for cost-effective single-region OLTP, **AlloyDB** for high-demanding HTAP / PostgreSQL workloads requiring $>4\times$ transactional performance and $100\times$ analytical acceleration with 99.99% SLA, **Cloud Spanner** for global horizontal scaling, **BigQuery** for data warehousing, and **Memorystore** for in-memory caching.

