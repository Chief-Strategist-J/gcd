# BigQuery: Official Reference Links & Resources

This document provides official Google Cloud BigQuery reference links, SQL optimization guides, and API documentation for `bq`.

---

## 1. Official Documentation Links

| Resource Title | URL Link | Description |
| :--- | :--- | :--- |
| **GCP BigQuery Docs** | [cloud.google.com/bigquery/docs](https://cloud.google.com/bigquery/docs) | Official landing page for BigQuery documentation. |
| **bq CLI Command Reference** | [cloud.google.com/bigquery/docs/reference/bq-cli-reference](https://cloud.google.com/bigquery/docs/reference/bq-cli-reference) | Complete command, flag, and parameter reference for `bq`. |
| **Partitioning & Clustering** | [cloud.google.com/bigquery/docs/partitioned-tables](https://cloud.google.com/bigquery/docs/partitioned-tables) | Best practices for table partitioning and clustering strategies. |
| **SQL Query Optimization** | [cloud.google.com/bigquery/docs/best-practices-performance-overview](https://cloud.google.com/bigquery/docs/best-practices-performance-overview) | Performance tuning rules for reducing query cost and latency. |
| **BigQuery Storage Write API** | [cloud.google.com/bigquery/docs/write-api](https://cloud.google.com/bigquery/docs/write-api) | High-throughput streaming data ingestion guide. |

---

## 2. Recommended Best Practices

1. **Cost Control**: Always run **`bq query --dry_run`** before executing massive SQL queries to check estimated bytes scanned.
2. **Table Optimization**: Combine **Partitioning by Date** with **Clustering by Category** (`--clustering_fields`) to minimize data scanned.
3. **Partition Filters**: Enforce `--require_partition_filter=true` on production tables to block accidental full-table scans.
