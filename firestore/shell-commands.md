# Firestore Shell Commands & Troubleshooting Manual

This reference provides generic, reusable command templates for managing Google Cloud Firestore databases in both **Native Mode** and **Datastore Mode**, along with composite index management, backup/restore, security rules, and troubleshooting error codes.

> **Note on Variable Conventions**: Replace placeholder variables formatted as `${VARIABLE}` (e.g., `${PROJECT_ID}`, `${DATABASE_NAME}`, `${LOCATION_ID}`) with target infrastructure values.

---

## 1. Database Lifecycle & Mode Provisioning

### Set Environment & Active Project
```bash
gcloud config set project ${PROJECT_ID}
```

### Create Database in Native Mode (Recommended for Web/Mobile Apps)
```bash
gcloud firestore databases create \
  --database="${DATABASE_NAME}" \
  --location="${LOCATION_ID}" \
  --type="firestore-native" \
  --delete-protection
```
*Example locations*: `nam5` (US multi-region), `eur3` (Europe multi-region), `us-central1` (regional). Use `(default)` as database name for the default database.

### Create Database in Datastore Mode (Recommended for Backend / Datastore Migration)
```bash
gcloud firestore databases create \
  --database="${DATABASE_NAME}" \
  --location="${LOCATION_ID}" \
  --type="datastore-mode" \
  --delete-protection
```

### List All Firestore Databases in Project
```bash
gcloud firestore databases list --project=${PROJECT_ID}
```

### Describe Database Configuration & Protection Status
```bash
gcloud firestore databases describe --database="${DATABASE_NAME}"
```

### Update Database (Enable/Disable Delete Protection or Encryption)
```bash
gcloud firestore databases update --database="${DATABASE_NAME}" \
  --no-delete-protection
```

### Delete a Firestore Database (Requires Delete Protection Disabled)
```bash
gcloud firestore databases delete --database="${DATABASE_NAME}" --quiet
```

---

## 2. Composite Index Management

Firestore automatically builds single-field indexes. Queries combining multiple equality or inequality operators require composite indexes.

### Create Composite Index from CLI
```bash
gcloud firestore indexes composite create \
  --database="${DATABASE_NAME}" \
  --collection-group="${COLLECTION_NAME}" \
  --query-scope="COLLECTION" \
  --field-config field-path="${FIELD_1_NAME}",order="ASCENDING" \
  --field-config field-path="${FIELD_2_NAME}",order="DESCENDING"
```

### Create Index with Array-Contains (Multiquery)
```bash
gcloud firestore indexes composite create \
  --database="${DATABASE_NAME}" \
  --collection-group="${COLLECTION_NAME}" \
  --query-scope="COLLECTION" \
  --field-config field-path="${TAGS_FIELD}",array-config="CONTAINS" \
  --field-config field-path="${TIMESTAMP_FIELD}",order="DESCENDING"
```

### Deploy Composite Indexes from JSON Configuration File (`firestore.indexes.json`)
```bash
gcloud firestore indexes composite import --database="${DATABASE_NAME}" \
  --source="${INDEX_FILE_PATH}"
```

#### Example `firestore.indexes.json` File Format:
```json
{
  "indexes": [
    {
      "collectionGroup": "orders",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    }
  ],
  "fieldOverrides": []
}
```

### List Composite Indexes
```bash
gcloud firestore indexes composite list --database="${DATABASE_NAME}"
```

### Describe Index Status (Check Building Progress)
```bash
gcloud firestore indexes composite describe ${INDEX_ID} --database="${DATABASE_NAME}"
```

### Delete Composite Index
```bash
gcloud firestore indexes composite delete ${INDEX_ID} --database="${DATABASE_NAME}" --quiet
```

---

## 3. Managed Export & Import Operations

Managed exports create a snapshot of database collections in a Cloud Storage bucket.

### Export Specific Collections to Cloud Storage
```bash
gcloud firestore export gs://${BUCKET_NAME}/${EXPORT_PREFIX} \
  --database="${DATABASE_NAME}" \
  --collection-ids="${COLLECTION_1}","${COLLECTION_2}"
```

### Export Entire Database
```bash
gcloud firestore export gs://${BUCKET_NAME}/${EXPORT_PREFIX} \
  --database="${DATABASE_NAME}"
```

### Import Collections from Managed Export
```bash
gcloud firestore import gs://${BUCKET_NAME}/${EXPORT_PREFIX}/${EXPORT_METADATA_FILE} \
  --database="${DATABASE_NAME}" \
  --collection-ids="${COLLECTION_1}"
```

### Track Export/Import Operation Status
```bash
gcloud firestore operations list --database="${DATABASE_NAME}"
gcloud firestore operations describe ${OPERATION_ID} --database="${DATABASE_NAME}"
```

---

## 4. Point-in-Time Recovery (PITR) & Backups

### Enable Point-in-Time Recovery (PITR)
PITR allows recovering data at any past microsecond timestamp within a 7-day window.
```bash
gcloud firestore databases update --database="${DATABASE_NAME}" \
  --enable-pitr
```

### Create On-Demand Database Backup
```bash
gcloud firestore backups create \
  --database="${DATABASE_NAME}" \
  --location="${LOCATION_ID}" \
  --retention="${RETENTION_PERIOD}"
```
*Note*: Retention period syntax example: `7d` (7 days), `14d` (14 days).

### List Available Database Backups
```bash
gcloud firestore backups list --location="${LOCATION_ID}"
```

### Restore Database from Backup
```bash
gcloud firestore databases restore \
  --source-backup=projects/${PROJECT_ID}/locations/${LOCATION_ID}/backups/${BACKUP_ID} \
  --destination-database="${NEW_DATABASE_NAME}"
```

---

## 5. Security Rules & IAM Access Control

### Deploy Firebase Security Rules (Native Mode)
```bash
gcloud firestore rules deploy ${RULES_FILE_PATH} --database="${DATABASE_NAME}"
```

#### Example `firestore.rules` File Content:
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    match /public_data/{document=**} {
      allow read: if true;
      allow write: if request.auth.token.admin == true;
    }
  }
}
```

### Release Security Ruleset
```bash
gcloud firestore rules release ${RULESET_ID} --database="${DATABASE_NAME}"
```

### IAM Role Granting for Server Access (Datastore/Firestore IAM)

#### Grant Read/Write Access to Service Account
```bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
  --role="roles/datastore.user"
```

#### Grant Read-Only Viewer Role
```bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
  --role="roles/datastore.viewer"
```

#### Grant Import/Export Admin Role
```bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
  --role="roles/datastore.importExportAdmin"
```

---

## 6. Document Operations (gcloud / REST Facade)

### Insert/Patch Document via REST API (Using gcloud Auth Header)
```bash
curl -X PATCH \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://firestore.googleapis.com/v1/projects/${PROJECT_ID}/databases/${DATABASE_NAME}/documents/${COLLECTION_NAME}/${DOCUMENT_ID}" \
  -d '{
    "fields": {
      "title": { "stringValue": "Sample Document Title" },
      "quantity": { "integerValue": "42" },
      "isActive": { "booleanValue": true }
    }
  }'
```

### Fetch Document via REST API
```bash
curl -X GET \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://firestore.googleapis.com/v1/projects/${PROJECT_ID}/databases/${DATABASE_NAME}/documents/${COLLECTION_NAME}/${DOCUMENT_ID}"
```

---

## 7. Cloud Run Functions & Eventarc Trigger Integration (Native Mode)

Cloud Run functions can execute asynchronously in response to document lifecycle changes in Firestore.

> [!IMPORTANT]
> **Native Mode Requirement**: Firestore triggers are supported **strictly on Firestore in Native mode**. Triggers are **not supported in Datastore mode**. All document paths must omit trailing slashes.

### 7.1 Deploy Cloud Run Function Triggered by Firestore Document Mutations

```bash
# Deploy function reacting to any document write (create, update, delete)
gcloud functions deploy handle-firestore-order \
  # ── RUNTIME & ENTRY POINT ────────────────────────────────────────────────
  --gen2 \
  --region=us-central1 \
  --runtime=nodejs20 \
  --entry-point=processOrderMutation \
  --source=. \
  
  # ── EVENTARC TRIGGER CONFIGURATION ───────────────────────────────────────
  --trigger-location=nam5 \
  # Location string: Multi-region (nam5, eur3) or region (us-central1) of the Firestore database.
  --trigger-event-filters="type=google.cloud.firestore.document.v1.written" \
  # Eventarc filter: Specific Firestore event type:
  #   - google.cloud.firestore.document.v1.created (Insert)
  #   - google.cloud.firestore.document.v1.updated (Mutate)
  #   - google.cloud.firestore.document.v1.deleted (Delete)
  #   - google.cloud.firestore.document.v1.written (Any write)
  --trigger-event-filters-path-pattern="document=orders/{orderId}" \
  # Document path: Wildcard pattern matching target documents. DO NOT include trailing slash.
  
  # ── RUNTIME IDENTITY ─────────────────────────────────────────────────────
  --service-account=firestore-listener-sa@${PROJECT_ID}.iam.gserviceaccount.com
```

### 7.2 Configure IAM Permissions for Function Service Account

```bash
export RUNTIME_SA="firestore-listener-sa@${PROJECT_ID}.iam.gserviceaccount.com"

# 1. Allow function to read and modify documents in Firestore
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${RUNTIME_SA}" \
  --role="roles/datastore.user"

# 2. Allow function to receive events from Eventarc
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${RUNTIME_SA}" \
  --role="roles/eventarc.eventReceiver"
```

### 7.3 List Active Eventarc Triggers Bound to Firestore

```bash
# List all triggers registered in target location
gcloud eventarc triggers list --location=nam5
```

---

## 8. Troubleshooting & Failure Resolution Matrix

| Error Code / Message | Root Cause | Resolution Procedure |
| :--- | :--- | :--- |
| `FAILED_PRECONDITION: The query requires an index` | Query executes filtering/sorting on multiple fields missing a composite index. | Click the auto-generated indexing URL printed in the client error log, or execute `gcloud firestore indexes composite create` with the specified field paths. |
| `PERMISSION_DENIED: Missing or insufficient permissions` | Firebase Security Rules (Native mode) rejected client request, or IAM role missing on Service Account. | Validate `request.auth` context in `firestore.rules`, or assign `roles/datastore.user` IAM role to service account executing the request. |
| `DEADLINE_EXCEEDED / ABORTED: Lock contention` | Multiple concurrent ACID transactions mutating the same document or entity group. | Implement exponential backoff retry in client SDK, reduce write transaction duration, or split hot documents into distributed counter sharding. |
| `RESOURCE_EXHAUSTED: Exceeded maximum write rate` | Writing to a single document at > 1 write per second in Datastore legacy entity groups or hot key bottleneck. | Shard documents using hash keys, utilize bulk batch writes, or migrate legacy Datastore entity group constraints to Firestore Native mode. |
| `INVALID_ARGUMENT: Delete protection enabled` | Attempting to execute database deletion while `--delete-protection` flag is active. | Run `gcloud firestore databases update --database=${DATABASE_NAME} --no-delete-protection` prior to invoking delete. |
| `NOT_FOUND: Database (default) does not exist` | Query targeting default database before initial database creation. | Run `gcloud firestore databases create --location=${LOCATION_ID} --type=firestore-native` to initialize the database. |
| `ALREADY_EXISTS: Database mode cannot be changed` | Attempting to create or convert an existing database to a different operating mode. | Firestore mode selection (Native vs Datastore) is fixed upon database creation. Create a new database instance or project to use a different mode. |
| `INVALID_ARGUMENT: Trailing slash detected in document path` | The `--trigger-event-filters-path-pattern="document=..."` parameter was specified with a trailing slash. | Remove trailing slash: use `document=orders/{orderId}` rather than `document=orders/{orderId}/`. |
| `FAILED_PRECONDITION: Triggers not supported in Datastore mode` | Function trigger was targeted at a database operating in Datastore mode. | Firestore event triggers strictly require **Firestore in Native mode**. |

