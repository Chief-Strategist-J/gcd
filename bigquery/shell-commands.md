# BigQuery: In-Depth Operations & Verification Manual

This document is an operational reference manual for the BigQuery CLI tool (`bq`).

Every section provides:
1. **Command to Execute**3. **How to Verify Configuration Correctness & Expected Verification Output**

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

### 1. Create Dataset with Regional Location & Expiration

```bash
bq --location=US mk \
    --dataset \
    --default_table_expiration 3600 \
    --description "Production Analytics Dataset" \
    YOUR_PROJECT_ID:prod_analytics
```

#### How to Verify Configuration Correctness:
```bash
bq show --format=prettyjson YOUR_PROJECT_ID:prod_analytics
```

#### Expected Verification Output:
```json
{
  "datasetReference": {
    "datasetId": "prod_analytics",
    "projectId": "YOUR_PROJECT_ID"
  },
  "defaultTableExpirationMs": "3600000",
  "location": "US"
}
```

---

### 2. Create Partitioned & Clustered Table with Schema

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

#### How to Verify Configuration Correctness:
```bash
bq show --format=prettyjson YOUR_PROJECT_ID:prod_analytics.user_transactions
```

#### Expected Verification Output:
```json
{
  "clustering": {
    "fields": ["region", "user_id"]
  },
  "requirePartitionFilter": true,
  "timePartitioning": {
    "field": "transaction_date",
    "type": "DAY"
  }
}
```

---

## Category 2: High-Performance Data Loading (`bq load` Parquet/JSON/CSV)

### 1. Load Compressed Parquet Files from Cloud Storage

```bash
bq load \
    --source_format=PARQUET \
    --autodetect \
    YOUR_PROJECT_ID:prod_analytics.user_transactions \
    "gs://my-prod-data-bucket/parquet/year=2026/*.parquet"
```

#### How to Verify Configuration Correctness:
```bash
bq query --use_legacy_sql=false 'SELECT COUNT(*) AS total_rows FROM `prod_analytics.user_transactions` WHERE transaction_date = "2026-09-12"'
```

#### Expected Verification Output:
```text
+------------+
| total_rows |
+------------+
|    1450201 |
+------------+
```

---

## Category 3: SQL Execution, Cost Estimation & Dry-Runs (`bq query`)

### 1. Dry-Run Query to Estimate Cost (Bytes Scanned)

```bash
bq query \
    --use_legacy_sql=false \
    --dry_run \
    'SELECT region, SUM(amount) AS total FROM `prod_analytics.user_transactions` WHERE transaction_date = "2026-09-12" GROUP BY region'
```

#### How to Verify Configuration Correctness:
```bash
# Execute query and write results to destination summary table
bq query \
    --use_legacy_sql=false \
    --destination_table=prod_analytics.daily_regional_summary \
    --write_disposition=WRITE_TRUNCATE \
    'SELECT region, SUM(amount) AS total FROM `prod_analytics.user_transactions` WHERE transaction_date = "2026-09-12" GROUP BY region'
```

#### Expected Verification Output:
```text
Waiting on bqjob_r21b8c_0000018f929... (2s) Current status: DONE
```

---

## Category 4: Data Export & Cloud Storage Extraction (`bq extract`)

### 1. Export Table to Compressed GZIP CSV Shards in GCS

```bash
bq extract \
    --destination_format=CSV \
    --compression=GZIP \
    YOUR_PROJECT_ID:prod_analytics.daily_regional_summary \
    "gs://my-prod-data-bucket/exports/summary-*.csv.gz"
```

#### How to Verify Configuration Correctness:
```bash
gcloud storage ls "gs://my-prod-data-bucket/exports/summary-*.csv.gz"
```

#### Expected Verification Output:
```text
gs://my-prod-data-bucket/exports/summary-000000000000.csv.gz
```

---

## Category 5: IAM Security, Dataset Access Controls & Encryption

### 1. Inspect & Grant Dataset Access Policy

```bash
bq show --format=prettyjson YOUR_PROJECT_ID:prod_analytics | grep -A 10 "access"
```

#### Expected Verification Output:
```json
  "access": [
    {
      "role": "WRITER",
      "userByEmail": "data-pipeline-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com"
    },
    {
      "role": "OWNER",
      "specialGroup": "projectOwners"
    }
  ]
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
