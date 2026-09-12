# BigQuery: In-Depth CLI Command Reference & Failure Resolution Manual

This document is an exhaustive, production-validated reference manual for the BigQuery CLI tool (`bq`). Commands are categorized by operational domain and include explicit **Error Diagnosis & Failure Resolution Commands** for production data warehouse scenarios.

---

## Table of Contents
1. [Category 1: Dataset & Partitioned/Clustered Table Management](#category-1-dataset--partitionedclustered-table-management)
2. [Category 2: High-Performance Data Loading (`bq load` Parquet/JSON/CSV)](#category-2-high-performance-data-loading-bq-load-parquetjsoncsv)
3. [Category 3: SQL Execution, Cost Estimation & Dry-Runs (`bq query`)](#category-3-sql-execution-cost-estimation--dry-runs-bq-query)
4. [Category 4: Data Export & Cloud Storage Extraction (`bq extract`)](#category-4-data-export--cloud-storage-extraction-bq-extract)
5. [Category 5: IAM Security, Dataset Access Controls & Encryption](#category-5-iam-security-dataset-access-controls--encryption)
6. [Category 6: Professional Failure Diagnosis & Resolution Matrix](#category-6-professional-failure-diagnosis--resolution-matrix)

---

## Category 1: Dataset & Partitioned/Clustered Table Management

### 1. Create Dataset with Regional Location & Default Expiration
```bash
bq --location=US mk \
    --dataset \
    --default_table_expiration 3600 \
    --description "Production Analytics Dataset" \
    YOUR_PROJECT_ID:prod_analytics
```

### 2. Create Partitioned & Clustered Table with Schema Definition
```bash
bq mk \
    --table \
    --time_partitioning_field transaction_date \
    --time_partitioning_type DAY \
    --clustering_fields region,user_id \
    --require_partition_filter=true \
    YOUR_PROJECT_ID:prod_analytics.user_transactions \
    id:STRING,user_id:STRING,region:STRING,amount:NUMERIC,transaction_date:DATE
```
* **Parameters**:
  * `--time_partitioning_field transaction_date`: Divides storage into daily partitions by date.
  * `--clustering_fields region,user_id`: Sorts data blocks inside partitions by region and user_id.
  * `--require_partition_filter=true`: Prevents accidental full-table scans by enforcing a `WHERE transaction_date = ...` clause.

---

## Category 2: High-Performance Data Loading (`bq load` Parquet/JSON/CSV)

### 1. Load Compressed Parquet Files from Cloud Storage ($0 Load Cost)
```bash
bq load \
    --source_format=PARQUET \
    --autodetect \
    YOUR_PROJECT_ID:prod_analytics.user_transactions \
    "gs://my-prod-data-bucket/parquet/year=2026/*.parquet"
```

### 2. Load Newline-Delimited JSON with Schema Auto-Detection & Max Bad Records
```bash
bq load \
    --source_format=NEWLINE_DELIMITED_JSON \
    --autodetect \
    --max_bad_records=10 \
    --ignore_unknown_values \
    YOUR_PROJECT_ID:prod_analytics.raw_events \
    "gs://my-prod-data-bucket/events/*.json"
```

---

## Category 3: SQL Execution, Cost Estimation & Dry-Runs (`bq query`)

### 1. Dry-Run Query to Estimate Cost (Bytes Scanned) Without Execution
```bash
bq query \
    --use_legacy_sql=false \
    --dry_run \
    'SELECT region, SUM(amount) AS total FROM `prod_analytics.user_transactions` WHERE transaction_date = "2026-09-12" GROUP BY region'
```
* **Output**: `Query successfully validated. It will process 4521098 bytes when run.`

### 2. Execute Production Query Writing to Destination Table
```bash
bq query \
    --use_legacy_sql=false \
    --destination_table=prod_analytics.daily_regional_summary \
    --write_disposition=WRITE_TRUNCATE \
    --allow_large_results \
    'SELECT region, SUM(amount) AS total FROM `prod_analytics.user_transactions` WHERE transaction_date = "2026-09-12" GROUP BY region'
```

---

## Category 4: Data Export & Cloud Storage Extraction (`bq extract`)

### 1. Export Large Table to Compressed GZIP CSV Shards in GCS
```bash
bq extract \
    --destination_format=CSV \
    --compression=GZIP \
    YOUR_PROJECT_ID:prod_analytics.daily_regional_summary \
    "gs://my-prod-data-bucket/exports/summary-*.csv.gz"
```

---

## Category 5: IAM Security, Dataset Access Controls & Encryption

### 1. Grant Dataset Viewer Access to User/Service Account
```bash
bq show --format=prettyjson YOUR_PROJECT_ID:prod_analytics > dataset_acl.json
# Edit JSON to append member role, then apply:
bq update --source dataset_acl.json YOUR_PROJECT_ID:prod_analytics
```

---

## Category 6: Professional Failure Diagnosis & Resolution Matrix

| Error Code / Message | Root Cause | Diagnosis Command | Immediate Professional Resolution Command |
| :--- | :--- | :--- | :--- |
| **`Cannot query without a predicate over partitioned table`** | Table has `--require_partition_filter=true` enabled and query lacks partition date filter | `bq show --format=prettyjson PROJECT:DS.TABLE \| grep requirePartitionFilter` | Add partition filter to SQL `WHERE transaction_date >= "2026-09-01"` or disable requirement: `bq update --require_partition_filter=false PROJECT:DS.TABLE`. |
| **`Quota exceeded: Your table has exceeded quota for total size of streaming buffer`** | Table modifying schema while active streaming buffer is present | `bq show --format=prettyjson PROJECT:DS.TABLE \| grep streamingBuffer` | Wait 30 minutes for streaming buffer to flush to Capacitor columnar storage before modifying schema. |
| **`403 AccessDenied: BigQuery: Permission denied`** | User lacks `roles/bigquery.jobUser` or `roles/bigquery.dataViewer` | `gcloud config get-value account` | Grant role: `gcloud projects add-iam-policy-binding PROJECT --member="USER" --role="roles/bigquery.jobUser"`. |
| **`403 BillingNotEnabled`** | GCP Project has no active billing account attached | `gcloud beta billing projects describe PROJECT` | Attach valid billing account to project: `gcloud beta billing projects link PROJECT --billing-account=ACCOUNT_ID`. |
| **`Resources exceeded during query execution`** | Query exceeded memory or slot allocation during heavy join/shuffle | `bq query --dry_run ...` | Cluster table by join key, replace `COUNT(DISTINCT x)` with `APPROX_COUNT_DISTINCT(x)`, or reserve additional Dremel slots. |
| **`400 Bad Request: Too many bad records`** | Ingested file contains schema malformations exceeding limit | `bq load ...` | Increase limit `--max_bad_records=100` or inspect file errors via `bq show -j JOB_ID`. |
| **`409 Already Exists: Table PROJECT:DS.TABLE`** | Table creation command attempted on existing table name | `bq ls PROJECT:DS` | Use `--replace` flag in `bq load` or drop table before creation: `bq rm -f PROJECT:DS.TABLE`. |
| **`Row size too large`** | Single row JSON/CSV record exceeds 100MB limit | `gsutil cat gs://BUCKET/file.json \| head -n 1` | Pre-process data file to split large string/array attributes before loading into BigQuery. |
