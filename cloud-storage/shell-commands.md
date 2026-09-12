# Cloud Storage: In-Depth CLI Command Reference & Failure Resolution Manual

This document is an exhaustive, production-validated reference manual for Google Cloud Storage (`gcloud storage` / `gsutil`). Commands are categorized by operational domain and include explicit **Error Diagnosis & Resolution Commands** for common failure modes.

---

## Table of Contents
1. [Category 1: Bucket Provisioning & Baseline Security](#category-1-bucket-provisioning--baseline-security)
2. [Category 2: Object Operations (Upload, Download, Metadata)](#category-2-object-operations-upload-download-metadata)
3. [Category 3: High-Performance Data Transfer & Directory Sync](#category-3-high-performance-data-transfer--directory-sync)
4. [Category 4: Lifecycle Management, Retention & Object Lock](#category-4-lifecycle-management-retention--object-lock)
5. [Category 5: IAM Access Control, Security & Signed URLs](#category-5-iam-access-control-security--signed-urls)
6. [Category 6: CORS Configuration & CMEK Encryption](#category-6-cors-configuration--cmek-encryption)
7. [Category 7: Error Diagnosis & Failure Resolution Matrix (Expanded)](#category-7-error-diagnosis--failure-resolution-matrix-expanded)

---

## Category 1: Bucket Provisioning & Baseline Security

### Primary Command: Create Production Storage Bucket
```bash
gcloud storage buckets create gs://my-prod-company-assets \
    --location=US \
    --default-storage-class=STANDARD \
    --public-access-prevention \
    --uniform-bucket-level-access
```

#### Detailed Flag Breakdown:
* `gs://my-prod-company-assets`: Globally unique bucket identifier. Must be lowercase, 3–63 characters, start/end with a letter or number.
* `--location=US`: Multi-region US deployment providing geo-redundant durability across data centers.
* `--default-storage-class=STANDARD`: Standard storage class (no retrieval fee, high availability).
* `--public-access-prevention`: Enforces an organization-wide restriction blocking public ACL assignment.
* `--uniform-bucket-level-access`: Disables legacy per-object ACLs, enforcing centralized Cloud IAM governance.

---

### Common Failure Modes & Resolution Commands

#### Failure Mode 1.1: `409 BucketAlreadyExists` / `BucketAlreadyOwnedByYou`
* **Symptom**: `StorageException: 409 The requested bucket name is already in use by another project.`
* **Root Cause**: GCS bucket names share a single global namespace across all GCP projects worldwide.
* **Resolution Command**: Verify bucket availability before creation using `describe`:
```bash
# Check if bucket exists globally
gcloud storage buckets describe gs://my-prod-company-assets 2>&1 || echo "Bucket is available"

# Fix: Create bucket with unique project-prefix or random suffix
gcloud storage buckets create gs://gcd-prod-assets-2026-us --location=US
```

#### Failure Mode 1.2: `403 AccessDenied` (Bucket Creation Forbidden)
* **Symptom**: `AccessDeniedException: 403 Caller does not have storage.buckets.create permission.`
* **Root Cause**: Principal lacks `roles/storage.admin` or `roles/storage.bucketAdmin` IAM permissions on target project.
* **Resolution Command**: Grant `storage.admin` role to current active gcloud identity:
```bash
# 1. Identify active authenticated account
gcloud config get-value account

# 2. Grant Bucket Admin permission (requires Project Admin privileges)
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
    --member="user:$(gcloud config get-value account)" \
    --role="roles/storage.admin"
```

---

## Category 2: Object Operations (Upload, Download, Metadata)

### Primary Commands: Standard Object Upload & Inspection

```bash
# 1. Upload file with custom Content-Type and Cache-Control headers
gcloud storage cp ./application-v2.tar.gz gs://my-prod-company-assets/releases/v2.tar.gz \
    --content-type="application/gzip" \
    --cache-control="public, max-age=3600"

# 2. Download object from bucket
gcloud storage cp gs://my-prod-company-assets/releases/v2.tar.gz ./v2.tar.gz

# 3. Inspect detailed object metadata (Size, Content-Type, MD5, ETag, Storage Class)
gcloud storage objects describe gs://my-prod-company-assets/releases/v2.tar.gz
```

---

### Common Failure Modes & Resolution Commands

#### Failure Mode 2.1: MD5 / CRC32c Checksum Mismatch (Corrupted Upload)
* **Symptom**: `ServiceException: 400 Bad Request: Checksum mismatch. Downloaded bytes failed validation.`
* **Root Cause**: Data stream corrupted during transit over unreliable network interfaces.
* **Resolution Command**: Enforce client-side integrity validation during upload/download:
```bash
# Force CRC32c checksum validation during transfer
gcloud storage cp ./large-data.bin gs://my-prod-company-assets/data.bin --checksums=crc32c
```

#### Failure Mode 2.2: `404 Not Found` Object Access Error
* **Symptom**: `CommandException: No URLs matched gs://my-prod-company-assets/non-existent.file`
* **Resolution Command**: Search bucket objects using wildcards and recursive listing:
```bash
# List all files matching pattern recursively
gcloud storage ls --recursive "gs://my-prod-company-assets/**/v2.tar.gz"
```

---

## Category 3: High-Performance Data Transfer & Directory Sync

### Primary Command: Parallel Directory Synchronization (`rsync`)
```bash
gcloud storage rsync -r ./media gs://my-prod-company-assets/media/ \
    --delete-unmatched-destination-objects \
    --parallel-composite-upload-component-threshold=50MiB
```

#### Detailed Flag Breakdown:
* `-r` / `--recursive`: Recursively syncs nested subdirectories.
* `--delete-unmatched-destination-objects`: Deletes files in bucket that no longer exist locally (mirror sync).
* `--parallel-composite-upload-component-threshold`: Automatically splits large files (>50MiB) into parallel chunks for faster multi-threaded upload.

---

### Common Failure Modes & Resolution Commands

#### Failure Mode 3.1: Resumable Upload Timeout / Stalled Transfer
* **Symptom**: Upload hangs on large multi-gigabyte files due to transient HTTP drops.
* **Resolution Command**: Enable parallel composite uploads and tune chunk size:
```bash
# Re-run rsync with parallel processing enabled
gcloud config set storage/parallel_composite_upload_enabled True
gcloud storage rsync -r ./media gs://my-prod-company-assets/media/
```

---

## Category 4: Lifecycle Management, Retention & Object Lock

### Primary Command: Applying Lifecycle Auto-Tiering Rules

#### 1. Lifecycle Policy JSON Configuration (`lifecycle.json`):
```json
{
  "rule": [
    {
      "action": { "type": "SetStorageClass", "storageClass": "NEARLINE" },
      "condition": { "age": 30 }
    },
    {
      "action": { "type": "SetStorageClass", "storageClass": "COLDLINE" },
      "condition": { "age": 90 }
    },
    {
      "action": { "type": "Delete" },
      "condition": { "age": 365 }
    }
  ]
}
```

#### 2. Apply Lifecycle Policy to Bucket:
```bash
gcloud storage buckets update gs://my-prod-company-assets --lifecycle-file=./lifecycle.json
```

---

### Common Failure Modes & Resolution Commands

#### Failure Mode 4.1: `403 RetentionPolicyLocked` / Cannot Overwrite Locked Object
* **Symptom**: `StorageException: 403 Retention policy is locked. Object cannot be deleted or overwritten until retention period expires.`
* **Root Cause**: Bucket has a WORM (Write Once Read Many) Retention Policy locked under compliance enforcement.
* **Resolution Command**: Inspect object retention expiration date and write to a new version URI:
```bash
# Check object retention lock expiration
gcloud storage objects describe gs://my-prod-company-assets/locked-file.pdf --format="value(retentionExpirationTime)"
```

---

## Category 5: IAM Access Control, Security & Signed URLs

### Primary Commands: Role Assignment & Signed URL Generation

```bash
# 1. Grant IAM Storage Object Admin role to service account
gcloud storage buckets add-iam-policy-binding gs://my-prod-company-assets \
    --member="serviceAccount:app-runner@YOUR_PROJECT.iam.gserviceaccount.com" \
    --role="roles/storage.objectAdmin"

# 2. Generate a 1-Hour Presigned V4 Signed URL for Secure Temporary Download
gcloud storage sign-url gs://my-prod-company-assets/releases/v2.tar.gz \
    --duration=1h \
    --private-key-file=./service-account-key.json
```

---

### Common Failure Modes & Resolution Commands

#### Failure Mode 5.1: `403 Forbidden: Public Access Prevention Enforced`
* **Symptom**: `AccessDeniedException: 403 Cannot make object public because Public Access Prevention is enforced on bucket.`
* **Root Cause**: Organization security policy explicitly blocks public `allUsers` read permissions.
* **Resolution Command**: Remove Public Access Prevention OR issue a secure Signed URL instead:
```bash
# Option 1 (Recommended): Use Signed URL for authorized external sharing
gcloud storage sign-url gs://my-prod-company-assets/file.pdf --duration=2h --private-key-file=key.json

# Option 2 (If public bucket is intended): Disable Public Access Prevention
gcloud storage buckets update gs://my-prod-company-assets --clear-public-access-prevention
```

---

## Category 6: CORS Configuration & CMEK Encryption

### Primary Command: Configuring Customer-Managed Encryption Keys (CMEK)

```bash
# Set default Cloud KMS encryption key on bucket
gcloud storage buckets update gs://my-prod-company-assets \
    --default-kms-key=projects/YOUR_PROJECT/locations/us/keyRings/my-ring/cryptoKeys/my-key
```

---

### Common Failure Modes & Resolution Commands

#### Failure Mode 6.1: `403 PermissionDenied: KMS Key Service Account Access Missing`
* **Symptom**: `AccessDeniedException: 403 Cloud KMS Service Account does not have permission to encrypt/decrypt.`
* **Root Cause**: GCS service agent lacks `roles/cloudkms.cryptoKeyEncrypterDecrypter` on Cloud KMS key.
* **Resolution Command**: Grant KMS Encrypter/Decrypter role to GCS service agent:
```bash
# 1. Retrieve GCS Service Agent email
GCS_SA=$(gcloud storage service-agent)

# 2. Grant KMS CryptoKey Encrypter/Decrypter role
gcloud kms keys add-iam-policy-binding my-key \
    --keyring=my-ring \
    --location=us \
    --member="serviceAccount:${GCS_SA}" \
    --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"
```

---

## Category 7: Error Diagnosis & Failure Resolution Matrix (Expanded)

| Error Code / Message | Root Cause | Diagnosis Command | Immediate Resolution Command |
| :--- | :--- | :--- | :--- |
| **`403 Forbidden: Caller does not have storage.objects.get`** | Missing IAM read permissions on target object | `gcloud storage buckets get-iam-policy gs://BUCKET` | `gcloud storage buckets add-iam-policy-binding gs://BUCKET --member="USER" --role="roles/storage.objectViewer"` |
| **`409 BucketAlreadyExists`** | Bucket name taken globally | `gcloud storage buckets describe gs://BUCKET` | Re-create bucket using unique prefix: `gs://UNIQUE-PREFIX-BUCKET` |
| **`400 Bad Request: Checksum mismatch`** | Network data corruption during upload | `gcloud storage cp FILE gs://BUCKET` | Re-upload enforcing CRC32c: `gcloud storage cp FILE gs://BUCKET --checksums=crc32c` |
| **`AccessDenied: Uniform bucket-level access enabled`** | Tried setting per-object ACL on IAM-governed bucket | `gcloud storage buckets describe gs://BUCKET --format="json(iamConfiguration)"` | Do not use legacy ACL flags (`--acl`). Assign Cloud IAM roles instead via `add-iam-policy-binding`. |
| **`SignatureDoesNotMatch (Signed URL)`** | Expired service account key or clock drift | `sudo ntpdate time.nist.gov` | Regenerate service account JSON key file and re-issue `gcloud storage sign-url`. |
| **`403 RetentionPolicyLocked`** | Object locked under WORM retention compliance policy | `gcloud storage objects describe gs://BUCKET/FILE --format="value(retentionExpirationTime)"` | Wait for retention lock expiration date or write to a new version URI. |
| **`403 Public Access Prevention Enforced`** | Org policy prevents assigning `allUsers` ACL | `gcloud storage buckets describe gs://BUCKET --format="value(publicAccessPrevention)"` | Disable prevention (`gcloud storage buckets update gs://BUCKET --clear-public-access-prevention`) OR use Signed URLs. |
| **`403 KMS Key Access Denied`** | GCS Service Agent lacks permission on KMS CryptoKey | `gcloud storage service-agent` | Grant KMS CryptoKey role: `gcloud kms keys add-iam-policy-binding KEY --member="serviceAccount:GCS_SA" --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"` |
| **`412 PreconditionFailed (ETag Mismatch)`** | Object modified by another process during update | `gcloud storage objects describe gs://BUCKET/FILE --format="value(etag)"` | Re-fetch latest object version before retrying update stream. |
| **`429 RateLimitExceeded (QPS Exceeded)`** | High write/delete traffic targeting single object prefix | `gcloud storage ls gs://BUCKET/prefix/` | Distribute uploads across randomized hash prefixes (e.g. `gs://BUCKET/a1b2/file.png`). |
| **`404 BucketNotFound`** | Bucket deleted or URI typo | `gcloud storage buckets list` | Verify exact bucket spelling or re-create bucket. |
| **`VPC Service Controls Violation`** | Security perimeter blocks API request outside boundary | `gcloud access-context-manager zone-describe PERIMETER` | Request security admin to add client IP or identity to VPC-SC Access Level whitelist. |
