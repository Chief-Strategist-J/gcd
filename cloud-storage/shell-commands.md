# Cloud Storage: Generalized Operations & Reference Manual

This document is a generalized, production-ready operational reference manual for Google Cloud Storage using `gcloud storage` (and `gsutil` legacy equivalents). All commands use generic environment variables and placeholders for easy copying and customization.

Every section includes:
1. **Generalized CLI Command Template**
2. **Key Parameter Descriptions & Defaults**
3. **Expected Terminal Output & Verification Commands**

---

## Table of Contents
1. [Category 1: Bucket Provisioning, Storage Tiers & Autoclass](#category-1-bucket-provisioning-storage-tiers--autoclass)
2. [Category 2: Object Operations, In-Place Tiering & Directory Emulation](#category-2-object-operations-in-place-tiering--directory-emulation)
3. [Category 3: Data Protection — Soft Delete, Object Versioning & Retention Lock (WORM)](#category-3-data-protection--soft-delete-object-versioning--retention-lock-worm)
4. [Category 4: Automated Lifecycle Management (OLM) & Autoclass](#category-4-automated-lifecycle-management-olm--autoclass)
5. [Category 5: Access Control — IAM, Fine-Grained ACLs, Signed URLs & Policy Documents](#category-5-access-control--iam-fine-grained-acls-signed-urls--policy-documents)
6. [Category 6: Encryption (CMEK & CSEK), CORS & Pub/Sub Notifications](#category-6-encryption-cmek--csek-cors--pubsub-notifications)
7. [Category 7: Data Transfer (rsync, Storage Transfer Service & Transfer Appliance)](#category-7-data-transfer-rsync-storage-transfer-service--transfer-appliance)
8. [Category 8: Strong Global Consistency Verification](#category-8-strong-global-consistency-verification)
9. [Category 9: Error Diagnosis & Failure Resolution Matrix](#category-9-error-diagnosis--failure-resolution-matrix)
10. [Category 10: Complete Hands-On Reference Manual (Generalized Steps 1–8)](#category-10-complete-hands-on-reference-manual-generalized-steps-18)

---

## Category 1: Bucket Provisioning, Storage Tiers & Autoclass

### 1. Create Storage Buckets Across Storage Tiers & Location Types

Cloud Storage buckets require a **globally unique name**, cannot be nested, and support four storage classes (`STANDARD`, `NEARLINE`, `COLDLINE`, `ARCHIVE`) across three location types (`Regional`, `Dual-Region`, `Multi-Region`).

```bash
# Set Reference Variables
export PROJECT_ID="YOUR_PROJECT_ID"
export BUCKET_NAME="YOUR_GLOBALLY_UNIQUE_BUCKET_NAME"

# 1. Create Standard Class Bucket in Regional Location (e.g. us-central1 for low compute latency)
gcloud storage buckets create gs://${BUCKET_NAME} \
    --project=${PROJECT_ID} \
    --location=LOCATION \
    --default-storage-class=STANDARD \
    --public-access-prevention \
    --uniform-bucket-level-access

# Parameter Breakdown:
# LOCATION: us-central1 (Regional), nam4 (Dual-Region: us-central1 + us-east1), or US / EU (Multi-Region)
# --default-storage-class: STANDARD, NEARLINE, COLDLINE, or ARCHIVE
# --uniform-bucket-level-access: Enforces bucket-level IAM (disables legacy per-object ACLs)
# --public-access-prevention: Enforces IAM org policy preventing public access
```

#### Expected Terminal Output:
```text
Creating gs://YOUR_GLOBALLY_UNIQUE_BUCKET_NAME/...
```

#### How to Verify Configuration Correctness:
```bash
gcloud storage buckets describe gs://${BUCKET_NAME} \
    --format="yaml(name, location, locationType, storageClass, iamConfiguration)"
```

#### Expected Verification Output:
```yaml
iamConfiguration:
  publicAccessPrevention: enforced
  uniformBucketLevelAccess:
    enabled: true
location: US-CENTRAL1
locationType: region
name: YOUR_GLOBALLY_UNIQUE_BUCKET_NAME
storageClass: STANDARD
```

---

### 2. Update Bucket Default Storage Class

Newly uploaded objects without an explicit storage class inherit the bucket's default class.

```bash
gcloud storage buckets update gs://${BUCKET_NAME} --default-storage-class=TARGET_STORAGE_CLASS
# TARGET_STORAGE_CLASS: NEARLINE, COLDLINE, or ARCHIVE
```

#### Expected Terminal Output:
```text
Updating gs://YOUR_GLOBALLY_UNIQUE_BUCKET_NAME/...
```

---

### 3. Enable Autoclass & Filestore POSIX File Storage

#### Autoclass Rules & Cost Safeguards:
* **Upload Landing**: All objects uploaded to an Autoclass bucket **start in Standard storage** regardless of upload parameters.
* **Cold Migration**: Inactive data automatically moves to colder classes (`Nearline` $\rightarrow$ `Coldline` $\rightarrow$ `Archive`).
* **Read Promotion**: Reading an object automatically **promotes it back to Standard storage**.
* **Zero Cost Penalty**: **No retrieval fees, no transition fees, and no early deletion charges** apply under Autoclass.

```bash
# 1. Enable Autoclass on Storage Bucket
gcloud storage buckets update gs://${BUCKET_NAME} --enable-autoclass

# 2. Provision Google Cloud Filestore (High-Performance POSIX NFS Shared Storage)
# Contrast: Filestore provides NFSv3 POSIX file system access, whereas GCS provides RESTful Object Storage.
gcloud filestore instances create INSTANCE_NAME \
    --zone=ZONE \
    --tier=BASIC_HDD \
    --file-share=name="FILE_SHARE_NAME",capacity=1TiB \
    --network=name="VPC_NETWORK_NAME"
```

---

## Category 2: Object Operations, In-Place Tiering & Directory Emulation

### 1. Upload Object & Mutate Storage Class In-Place

You can mutate an existing object's storage class in-place (e.g. `STANDARD` $\rightarrow$ `COLDLINE` $\rightarrow$ `ARCHIVE`) **without moving the file or changing its access URL**.

```bash
# 1. Upload Object with Metadata Headers & Checksum Verification
gcloud storage cp /path/to/local_file.pdf gs://${BUCKET_NAME}/path/in/bucket/file.pdf \
    --default-storage-class=STANDARD \
    --content-type="application/pdf" \
    --checksums=crc32c

# 2. Mutate Storage Class of Existing Object In-Place
gcloud storage objects update gs://${BUCKET_NAME}/path/in/bucket/file.pdf \
    --storage-class=TARGET_STORAGE_CLASS
# TARGET_STORAGE_CLASS: NEARLINE, COLDLINE, or ARCHIVE
```

#### Expected Terminal Output:
```text
Copying file:///path/to/local_file.pdf to gs://YOUR_GLOBALLY_UNIQUE_BUCKET_NAME/path/in/bucket/file.pdf
Updating gs://YOUR_GLOBALLY_UNIQUE_BUCKET_NAME/path/in/bucket/file.pdf...
```

#### How to Verify Configuration Correctness:
```bash
gcloud storage objects describe gs://${BUCKET_NAME}/path/in/bucket/file.pdf \
    --format="yaml(name, storageClass, mediaLink, crc32c)"
```

---

### 2. Emulate Directories & List Exabyte Collections

GCS is an object store. Directories are emulated using slash (`/`) delimiters in object keys.

```bash
# 1. Create a Pseudo-Directory Placeholder
gcloud storage cp /dev/null gs://${BUCKET_NAME}/folder_name/

# 2. Perform Recursive Prefix Search
gcloud storage ls --recursive "gs://${BUCKET_NAME}/folder_name/**"
```

---

## Category 3: Data Protection — Soft Delete, Object Versioning & Retention Lock (WORM)

### 1. Configure Soft Delete Retention & Restore Deleted Objects

Soft Delete preserves deleted or overwritten objects for a configurable retention window (default 7 days, up to 90 days, or 0 to disable).

```bash
# 1. Set Soft Delete Retention Duration (e.g., 30d, 90d, or 0 to disable)
gcloud storage buckets update gs://${BUCKET_NAME} --soft-delete-retention-duration=RETENTION_DURATION

# 2. List Soft-Deleted Objects
gcloud storage ls --soft-deleted gs://${BUCKET_NAME}/

# 3. Restore Soft-Deleted Object using Generation URI
gcloud storage restore gs://${BUCKET_NAME}/path/to/object.ext#GENERATION_ID
```

#### Expected Terminal Output:
```text
Updating gs://YOUR_GLOBALLY_UNIQUE_BUCKET_NAME/...
Restoring gs://YOUR_GLOBALLY_UNIQUE_BUCKET_NAME/path/to/object.ext#GENERATION_ID...
```

---

### 2. Enable Object Versioning & Recover Historical Generations

Object Versioning creates an archived generation identified by `#GENERATION_ID` upon overwrite or delete.

```bash
# 1. Enable Object Versioning on Bucket
gcloud storage buckets update gs://${BUCKET_NAME} --versioning

# 2. List All Object Versions (Live + Archived Generations)
gcloud storage ls --all-versions gs://${BUCKET_NAME}/path/to/file.ext

# 3. Restore Historical Generation to Live State
gcloud storage cp gs://${BUCKET_NAME}/path/to/file.ext#GENERATION_ID \
    gs://${BUCKET_NAME}/path/to/file.ext

# 4. Permanently Purge a Specific Archived Generation
gcloud storage rm gs://${BUCKET_NAME}/path/to/file.ext#GENERATION_ID
```

---

### 3. Object Retention Lock (WORM Compliance for SEC/FINRA/CFTC)

Retention policies enforce Write-Once-Read-Many (WORM) compliance. Once locked, policies cannot be reduced or removed by any identity.

```bash
# 1. Set Bucket Retention Period (e.g., 1y, 365d, or 31536000s)
gcloud storage buckets update gs://${BUCKET_NAME} --retention-period=RETENTION_PERIOD

# 2. Lock Retention Policy (PERMANENT - Irreversible enforcement)
gcloud storage buckets update gs://${BUCKET_NAME} --lock-retention-policy
```

---

## Category 4: Automated Lifecycle Management (OLM) & Autoclass

```bash
# 1. Define Multi-Condition Lifecycle Rule Policy (lifecycle.json)
cat << 'EOF' > lifecycle.json
{
  "rule": [
    {
      "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
      "condition": {"age": 30, "matchesStorageClass": ["STANDARD"]}
    },
    {
      "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
      "condition": {"age": 90, "matchesStorageClass": ["NEARLINE"]}
    },
    {
      "action": {"type": "SetStorageClass", "storageClass": "ARCHIVE"},
      "condition": {"age": 365, "matchesStorageClass": ["COLDLINE"]}
    },
    {
      "action": {"type": "Delete"},
      "condition": {"numNewerVersions": 3, "isLive": false}
    }
  ]
}
EOF

# 2. Apply Lifecycle File to Bucket (Takes up to 24 hours to propagate across asynchronous scans)
gcloud storage buckets update gs://${BUCKET_NAME} --lifecycle-file=lifecycle.json
```

---

## Category 5: Access Control — IAM, Fine-Grained ACLs, Signed URLs & Policy Documents

### 1. IAM Policy Bindings

```bash
# Grant Bucket-Level IAM Role to Principal
gcloud storage buckets add-iam-policy-binding gs://${BUCKET_NAME} \
    --member="MEMBER_PRINCIPAL" \
    --role="ROLE_NAME"
# MEMBER_PRINCIPAL: user:email@domain.com, serviceAccount:sa@project.iam.gserviceaccount.com, allUsers, allAuthenticatedUsers
# ROLE_NAME: roles/storage.objectViewer, roles/storage.objectAdmin, roles/storage.legacyObjectReader
```

---

### 2. Fine-Grained Access Control Lists (ACLs - Max 100 Entries)

```bash
# 1. Update Object ACL to Predefined Template (private, publicRead)
gcloud storage objects update gs://${BUCKET_NAME}/path/to/file.ext --predefined-acl=private

# 2. Add ACL Entry to Specific Entity
gcloud storage objects add-acl gs://${BUCKET_NAME}/path/to/file.ext \
    --entity=ENTITY_IDENTIFIER \
    --role=PERMITTED_ROLE
# ENTITY_IDENTIFIER: allUsers, allAuthenticatedUsers, user-email@domain.com
# PERMITTED_ROLE: READ, WRITE, FULL_CONTROL

# 3. View Object ACL Entries
gcloud storage objects get-acl gs://${BUCKET_NAME}/path/to/file.ext
```

---

### 3. Time-Limited Signed URLs & Signed Policy Documents

```bash
# 1. Generate Signed URL for Object GET/PUT (Requires Service Account Private Key)
gcloud storage sign-url gs://${BUCKET_NAME}/path/to/file.ext \
    --duration=DURATION \
    --private-key-file=/path/to/service-account-key.json
# DURATION: 15m, 1h, 7d

# 2. Signed Policy Document for Form Uploads (policy_document.json)
cat << 'EOF' > policy_document.json
{
  "expiration": "2026-12-31T23:59:59Z",
  "conditions": [
    {"bucket": "YOUR_GLOBALLY_UNIQUE_BUCKET_NAME"},
    ["starts-with", "$key", "uploads/"],
    ["content-length-range", 0, 10485760]
  ]
}
EOF
```

---

## Category 6: Encryption (CMEK & CSEK), CORS & Pub/Sub Notifications

```bash
# 1. Set Bucket Default Customer-Managed Encryption Key (CMEK via KMS)
gcloud storage buckets update gs://${BUCKET_NAME} \
    --default-kms-key=projects/PROJECT_ID/locations/LOCATION/keyRings/KEYRING_NAME/cryptoKeys/KEY_NAME

# 2. Upload Object Encrypted with Customer-Supplied Key (CSEK - 256-bit Base64 Key)
gcloud storage cp /path/to/local_file.ext gs://${BUCKET_NAME}/path/to/file.ext \
    --encryption-key=BASE64_AES256_KEY

# 3. Download Object Encrypted with Customer-Supplied Key (CSEK)
gcloud storage cp gs://${BUCKET_NAME}/path/to/file.ext /path/to/restored_file.ext \
    --decryption-key=BASE64_AES256_KEY

# 4. Create Pub/Sub Object Change Notification Trigger
gcloud storage notification-configs create gs://${BUCKET_NAME} \
    --topic=projects/PROJECT_ID/topics/PUBSUB_TOPIC_NAME \
    --event-types=OBJECT_FINALIZE,OBJECT_DELETE \
    --payload-format=json
```

---

## Category 7: Data Transfer (rsync, Storage Transfer Service & Transfer Appliance)

```bash
# 1. Perform Parallel Directory Synchronization (rsync)
gcloud storage rsync -r /path/to/local_directory gs://${BUCKET_NAME}/remote_prefix/ \
    --delete-unmatched-destination-objects \
    --parallel-composite-upload-component-threshold=50MiB

# 2. Storage Transfer Service Job (Managed AWS S3 / HTTP / GCS Migration)
gcloud transfer jobs create s3://AWS_BUCKET_NAME gs://${BUCKET_NAME} \
    --name="s3-migration-job" \
    --aws-access-key-id=AWS_ACCESS_KEY_ID \
    --aws-secret-access-key=AWS_SECRET_ACCESS_KEY
```

---

## Category 8: Strong Global Consistency Verification

```bash
# 1. Upload Object (Strong Read-After-Write)
gcloud storage cp /path/to/test.txt gs://${BUCKET_NAME}/test.txt

# 2. Immediate Read / List Verification (Strong Object List Consistency)
gcloud storage ls gs://${BUCKET_NAME}/test.txt

# 3. Delete Object (Strong Read-After-Delete)
gcloud storage rm gs://${BUCKET_NAME}/test.txt

# 4. Immediate Read returns 404 Not Found (Guaranteed No Stale Reads)
gcloud storage cat gs://${BUCKET_NAME}/test.txt
```

---

## Category 9: Error Diagnosis & Failure Resolution Matrix

| Error Code / Symptom | Root Cause | Diagnosis Command | Immediate Resolution Command |
| :--- | :--- | :--- | :--- |
| **`409 BucketAlreadyExists`** | Bucket name is globally taken in GCP. | `gcloud storage buckets describe gs://BUCKET_NAME` | Re-run with unique prefix/suffix: `gs://${BUCKET_NAME}-$(date +%s)` |
| **`403 AccessDenied`** | Principal lacks required IAM role. | `gcloud config get-value account` | Grant role: `gcloud projects add-iam-policy-binding PROJECT_ID --member="MEMBER" --role="roles/storage.objectAdmin"` |
| **`400 Bad Request (Checksum)`** | Transfer corruption detected. | `gcloud storage cp FILE gs://BUCKET_NAME/` | Force validation: `--checksums=crc32c` |
| **`404 Not Found`** | Object key path does not exist. | `gcloud storage ls gs://BUCKET_NAME/` | Search recursively: `gcloud storage ls --recursive "gs://BUCKET_NAME/**"` |
| **`RetentionPolicyLocked`** | WORM retention lock prevents edit/purge. | `gcloud storage buckets describe gs://BUCKET_NAME` | Retention policy is permanent by design; data cannot be deleted until retention period expires. |
| **`401 Customer-supplied key mismatch`** | Key provided does not match encryption key. | `gsutil cp --decryption-key=KEY` | Provide original key or rotate using `gsutil rewrite -k` while `decryption_key1` is configured. |

---

## Category 10: Complete Hands-On Reference Manual (Generalized Steps 1–8)

This section provides the generalized, reusable operational templates for the 8 core lab tasks.

### Task 1. Prepare Environment & Sample Files
```bash
# Environment Setup
export PROJECT_ID=$(gcloud config get-value project)
export BUCKET_NAME_1="storecore-${PROJECT_ID}"

# Create Fine-Grained Bucket
gcloud storage buckets create gs://${BUCKET_NAME_1} \
    --project=${PROJECT_ID} \
    --location=us-central1 \
    --default-storage-class=STANDARD \
    --no-public-access-prevention

# Download Sample File
curl https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/ClusterSetup.html > setup.html
cp setup.html setup2.html
cp setup.html setup3.html
```

### Task 2. ACL & Public Read Configuration
```bash
# Upload Object
gcloud storage cp setup.html gs://${BUCKET_NAME_1}/

# Set Predefined Private ACL
gcloud storage objects update gs://${BUCKET_NAME_1}/setup.html --predefined-acl=private

# Grant Public Read Access via Legacy Reader Role
gcloud storage objects add-iam-policy-binding gs://${BUCKET_NAME_1}/setup.html \
    --member="allUsers" \
    --role="roles/storage.legacyObjectReader"

# Test Recovery
rm setup.html
gcloud storage cp gs://${BUCKET_NAME_1}/setup.html setup.html
```

### Task 3. CSEK Generation & `.boto` Encryption
```bash
# Generate 256-Bit Base64 Key
export CSEK_KEY_1=$(python3 -c 'import base64; import os; print(base64.encodebytes(os.urandom(32)).decode().strip())')

# Configure .boto Config File
if [ ! -f ~/.boto ]; then gsutil config -n; fi
sed -i "s/#\? \?encryption_key=.*/encryption_key=${CSEK_KEY_1}/" ~/.boto

# Upload Customer-Encrypted Objects
gsutil cp setup2.html gs://${BUCKET_NAME_1}/
gsutil cp setup3.html gs://${BUCKET_NAME_1}/
```

### Task 4. Rotate CSEK Keys (`gsutil rewrite -k`)
```bash
# 1. Shift Encryption Key to Decryption Key 1
sed -i "s/^encryption_key=\(.*\)/# encryption_key=\1\ndecryption_key1=\1/" ~/.boto

# 2. Generate New Encryption Key
export CSEK_KEY_2=$(python3 -c 'import base64; import os; print(base64.encodebytes(os.urandom(32)).decode().strip())')
sed -i "s/# encryption_key=.*/encryption_key=${CSEK_KEY_2}\n# encryption_key=/" ~/.boto

# 3. Rewrite Object Key (Decrypts with key1, Encrypts with key2)
gsutil rewrite -k gs://${BUCKET_NAME_1}/setup2.html

# 4. Comment Out Decryption Key 1
sed -i "s/^decryption_key1=/# decryption_key1=/" ~/.boto

# 5. Verify Download (setup2 succeeds, setup3 fails as expected)
gsutil cp gs://${BUCKET_NAME_1}/setup2.html recover2.html
gsutil cp gs://${BUCKET_NAME_1}/setup3.html recover3.html || echo "Key Rotation Verified!"
```

### Task 5. 31-Day Delete Lifecycle Management
```bash
cat << 'EOF' > life.json
{
  "rule": [
    {
      "action": {"type": "Delete"},
      "condition": {"age": 31}
    }
  ]
}
EOF
gcloud storage buckets update gs://${BUCKET_NAME_1} --lifecycle-file=life.json
```

### Task 6. Enable Versioning & Restore Generation
```bash
gcloud storage buckets update gs://${BUCKET_NAME_1} --versioning
gcloud storage cp -v setup.html gs://${BUCKET_NAME_1}/
gcloud storage ls -a gs://${BUCKET_NAME_1}/setup.html
export VERSION_NAME=$(gcloud storage ls -a gs://${BUCKET_NAME_1}/setup.html | head -n 1)
gcloud storage cp ${VERSION_NAME} recovered.txt
```

### Task 7. Recursive Directory Synchronization (`rsync`)
```bash
mkdir -p firstlevel/secondlevel
cp setup.html firstlevel/
cp setup.html firstlevel/secondlevel/
gcloud storage rsync ./firstlevel gs://${BUCKET_NAME_1}/firstlevel --recursive
gcloud storage ls -r gs://${BUCKET_NAME_1}/firstlevel
```

### Task 8. Best Practices Verification Summary
* Enforce **Uniform Bucket-Level Access** for centralized IAM governance.
* Protect data using **Soft Delete** and **Object Versioning**.
* Enforce regulatory compliance using **Object Retention Lock**.
