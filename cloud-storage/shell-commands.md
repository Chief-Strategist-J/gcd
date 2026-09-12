# Cloud Storage: In-Depth Operations & Verification Manual

This document is an operational reference manual for Google Cloud Storage (`gcloud storage` / `gsutil`).

Every command snippet includes:
1. **Command to Execute**
2. **Expected Terminal Output (What to read & look for in terminal)**
3. **How to Verify Configuration Correctness & Expected Verification Output**

---

## Table of Contents
1. [Category 1: Bucket Provisioning & Baseline Security](#category-1-bucket-provisioning--baseline-security)
2. [Category 2: Object Operations (Upload, Download, Metadata)](#category-2-object-operations-upload-download-metadata)
3. [Category 3: High-Performance Data Transfer & Directory Sync](#category-3-high-performance-data-transfer--directory-sync)
4. [Category 4: Lifecycle Management, Retention & Object Lock](#category-4-lifecycle-management-retention--object-lock)
5. [Category 5: IAM Access Control, Security & Signed URLs](#category-5-iam-access-control-security--signed-urls)
6. [Category 6: CORS Configuration & CMEK Encryption](#category-6-cors-configuration--cmek-encryption)
7. [Category 7: Error Diagnosis & Failure Resolution Matrix](#category-7-error-diagnosis--failure-resolution-matrix)

---

## Category 1: Bucket Provisioning & Baseline Security

### 1. Create Production Storage Bucket with Security Guardrails

```bash
gcloud storage buckets create gs://gcd-prod-company-assets \
    --location=US \
    --default-storage-class=STANDARD \
    --public-access-prevention \
    --uniform-bucket-level-access
```

#### Expected Terminal Output:
```text
Creating gs://gcd-prod-company-assets/...
```

#### How to Verify Configuration Correctness:
```bash
gcloud storage buckets describe gs://gcd-prod-company-assets --format="yaml(name, location, storageClass, iamConfiguration)"
```

#### Expected Verification Output:
```yaml
iamConfiguration:
  publicAccessPrevention: enforced
  uniformBucketLevelAccess:
    enabled: true
location: US
name: gcd-prod-company-assets
storageClass: STANDARD
```

---

## Category 2: Object Operations (Upload, Download, Metadata)

### 1. Upload Object with Metadata Headers & CRC32c Checksum

```bash
gcloud storage cp ./application-v2.tar.gz gs://gcd-prod-company-assets/releases/v2.tar.gz \
    --content-type="application/gzip" \
    --cache-control="public, max-age=3600" \
    --checksums=crc32c
```

#### Expected Terminal Output:
```text
Copying file://./application-v2.tar.gz to gs://gcd-prod-company-assets/releases/v2.tar.gz
  Completed files 1/1 | 24.2MiB/24.2MiB                                         
```

#### How to Verify Configuration Correctness:
```bash
gcloud storage objects describe gs://gcd-prod-company-assets/releases/v2.tar.gz \
    --format="yaml(name, contentType, cacheControl, md5Hash, crc32c)"
```

#### Expected Verification Output:
```yaml
cacheControl: public, max-age=3600
contentType: application/gzip
crc32c: 7a8B9w==
md5Hash: e281a94f872e128a1728e219ba489a2e
name: releases/v2.tar.gz
```

---

## Category 3: High-Performance Data Transfer & Directory Sync

### 1. Parallel Directory Synchronization (`rsync`)

```bash
gcloud storage rsync -r ./media gs://gcd-prod-company-assets/media/ \
    --delete-unmatched-destination-objects \
    --parallel-composite-upload-component-threshold=50MiB
```

#### Expected Terminal Output:
```text
Building synchronization state...
At gs://gcd-prod-company-assets/media/, copying 14 files, deleting 1 file.
Completed 14/14 operations.
```

#### How to Verify Configuration Correctness:
```bash
gcloud storage ls --long "gs://gcd-prod-company-assets/media/"
```

#### Expected Verification Output:
```text
  1048576  2026-09-12T13:45:12Z  gs://gcd-prod-company-assets/media/hero.png
   524288  2026-09-12T13:45:12Z  gs://gcd-prod-company-assets/media/banner.jpg
TOTAL: 14 objects, 15728640 bytes (15.0 MiB)
```

---

## Category 4: Lifecycle Management, Retention & Object Lock

### 1. Set Bucket Lifecycle Policy (Auto-Delete after 30 days)

```bash
# Create JSON lifecycle rule configuration
cat << 'EOF' > lifecycle.json
{
  "rule": [
    {
      "action": {"type": "Delete"},
      "condition": {"age": 30}
    }
  ]
}
EOF

# Apply lifecycle policy to bucket
gcloud storage buckets update gs://gcd-prod-company-assets --lifecycle-file=lifecycle.json
```

#### Expected Terminal Output:
```text
Updating gs://gcd-prod-company-assets/...
```

#### How to Verify Configuration Correctness:
```bash
gcloud storage buckets describe gs://gcd-prod-company-assets --format="yaml(lifecycle)"
```

#### Expected Verification Output:
```yaml
lifecycle:
  rule:
  - action:
      type: Delete
    condition:
      age: 30
```

---

## Category 5: IAM Access Control, Security & Signed URLs

### 1. Grant Storage Object Viewer Role & Generate Time-Limited Signed URL

```bash
# Grant Object Viewer permission to user account
gcloud storage buckets add-iam-policy-binding gs://gcd-prod-company-assets \
    --member="user:analyst@company.com" \
    --role="roles/storage.objectViewer"

# Generate 15-minute Signed URL for private file access
gcloud storage sign-url gs://gcd-prod-company-assets/releases/v2.tar.gz \
    --duration=15m
```

#### Expected Terminal Output:
```text
Signed URL:
https://storage.googleapis.com/gcd-prod-company-assets/releases/v2.tar.gz?GoogleAccessId=service-account@project.iam.gserviceaccount.com&Expires=178921827&Signature=a87f62s87f...
```

#### How to Verify Configuration Correctness:
```bash
gcloud storage buckets get-iam-policy gs://gcd-prod-company-assets \
    --filter="bindings.role:roles/storage.objectViewer"
```

#### Expected Verification Output:
```yaml
bindings:
- members:
  - user:analyst@company.com
  role: roles/storage.objectViewer
```

---

## Category 6: CORS Configuration & CMEK Encryption

### 1. Apply Customer-Managed Encryption Key (CMEK) to Bucket

```bash
gcloud storage buckets update gs://gcd-prod-company-assets \
    --default-kms-key=projects/YOUR_PROJECT/locations/us/keyRings/my-ring/cryptoKeys/my-key
```

#### Expected Terminal Output:
```text
Updating gs://gcd-prod-company-assets/...
```

#### How to Verify Configuration Correctness:
```bash
gcloud storage buckets describe gs://gcd-prod-company-assets --format="value(encryption.defaultKmsKeyName)"
```

#### Expected Verification Output:
```text
projects/YOUR_PROJECT/locations/us/keyRings/my-ring/cryptoKeys/my-key
```

---

## Category 7: Error Diagnosis & Failure Resolution Matrix

| Error Code / Symptom | Root Cause | Diagnosis Command | Immediate Resolution Command |
| :--- | :--- | :--- | :--- |
| **`409 BucketAlreadyExists`** | Bucket name is already globally reserved by another project. | `gcloud storage buckets describe gs://BUCKET` | Re-run with unique suffix: `gs://gcd-prod-assets-$(date +%s)` |
| **`403 AccessDenied`** | User lacks `roles/storage.admin` or `roles/storage.objectAdmin`. | `gcloud config get-value account` | Grant permission: `gcloud projects add-iam-policy-binding PROJECT --member="USER" --role="roles/storage.admin"` |
| **`400 Bad Request (Checksum Mismatch)`** | Corrupted data stream during transfer. | `gcloud storage cp FILE gs://BUCKET/` | Force CRC32c checksum validation: `--checksums=crc32c` |
| **`404 Not Found`** | Object key or URI path does not exist. | `gcloud storage ls gs://BUCKET/` | Search recursively: `gcloud storage ls --recursive "gs://BUCKET/**/file"` |
