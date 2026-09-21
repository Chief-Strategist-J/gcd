# Cloud Run Functions (`gcloud functions`) Production Command Reference

A comprehensive, production-grade CLI manual for building, deploying, and managing **Google Cloud Run Functions (2nd Generation)**. Every command is structured into logical architectural sections (`# ── SECTION ──`), with explicit breakdowns of valid values, data formats, enums, units, and security trade-offs.

---

## 1. Project Prerequisites & API Activation

Cloud Run functions (2nd gen) executes builds via Cloud Build and provisions serverless containers on Cloud Run, requiring core service APIs to be active in your project before initial deployment.

```bash
# Enable required Google Cloud Service APIs
gcloud services enable \
  cloudfunctions.googleapis.com \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  eventarc.googleapis.com \
  logging.googleapis.com \
  storage.googleapis.com
```

---

## 2. Comprehensive `gcloud functions deploy` Master Command

The master command below demonstrates an enterprise-grade, fully annotated deployment of a 2nd-generation Cloud Run function.

```bash
gcloud functions deploy order-processing-service \
  # ── GENERATION & REGIONAL SCOPE ──────────────────────────────────────────
  --gen2 \
  # Flag: Enables 2nd generation (Cloud Run functions).
  # Required for: 60-minute execution timeouts, concurrency, and Eventarc.
  --region=us-central1 \
  # String: Target Google Cloud deployment region.
  # Values: us-central1, us-east1, europe-west1, asia-east1, etc.
  # Best practice: Co-locate with primary databases (Firestore, Cloud SQL).

  # ── RUNTIME & ENTRY POINT SPECIFICATION ──────────────────────────────────
  --runtime=nodejs20 \
  # String: Target language runtime environment.
  # Values:
  #   Node.js:  nodejs18, nodejs20, nodejs22
  #   Python:   python310, python311, python312
  #   Go:       go121, go122
  #   Java:     java11, java17, java21
  #   .NET:     dotnet6, dotnet8
  #   Ruby:     ruby32
  #   PHP:      php82, php83
  --entry-point=processOrder \
  # String: Name of function or class in source code executed on invocation.
  # Node.js: Exported function name in index.js (e.g., exports.processOrder).
  # Python: Function name defined in main.py (e.g., def process_order(request):).

  # ── SOURCE CODE INGESTION ────────────────────────────────────────────────
  --source=. \
  # Path/URI: Location of function source code.
  # Valid values:
  #   1. Local filesystem directory: "." or "./src/service"
  #   2. Cloud Storage ZIP: "gs://my-bucket/artifacts/source.zip"
  #   3. Source Repo: "https://source.developers.google.com/p/PROJ/r/REPO/revisions/main/paths/src"
  --stage-bucket=my-prod-gcf-staging-bucket \
  # Bucket name (optional): Cloud Storage bucket for uploading source archives.
  # Omit to let Google Cloud provision and manage a default staging bucket.

  # ── INGRESS & TRIGGER DEFINITION ─────────────────────────────────────────
  --trigger-http \
  # Flag: Configures HTTP/HTTPS endpoint trigger.
  # Alternatives:
  #   --trigger-topic=TOPIC_NAME (Pub/Sub trigger)
  #   --trigger-event-filters="type=google.cloud.storage.object.v1.finalized"
  --allow-unauthenticated \
  # Flag: Allows public, unauthenticated invocations over the public internet.
  # Enterprise alternative:
  #   --no-allow-unauthenticated (Requires IAM Cloud Run Invoker or OIDC token)

  # ── RUNTIME SERVICE ACCOUNT & SECURITY ───────────────────────────────────
  --service-account=order-fn-sa@my-gcp-project.iam.gserviceaccount.com \
  # Email: Custom IAM Service Account used by the running container.
  # Default (if omitted): Default Compute Engine Service Account (over-privileged!).
  # Security rule: Always use a dedicated least-privilege service account.

  # ── HARDWARE RESOURCES & CONCURRENCY SCALING ─────────────────────────────
  --memory=1024Mi \
  # Memory string: Memory allocated per container instance.
  # Values: 128Mi, 256Mi, 512Mi, 1024Mi (1Gi), 2048Mi (2Gi), 4096Mi (4Gi), up to 32Gi.
  --cpu=1 \
  # String/Number: Number of vCPUs allocated.
  # Values: 0.08 to 8 (e.g., 0.58, 1, 2, 4, 8).
  --timeout=60s \
  # Time string: Maximum execution duration before timing out.
  # Gen 2 HTTP Max: 3600s (60 minutes). Gen 2 Event Max: 540s (9 minutes).
  --concurrency=80 \
  # Integer: Max simultaneous requests processed by a single container instance.
  # Gen 2 Range: 1 to 1000. Default: 1 (safe for CPU-bound code).
  # High-throughput async Node.js/Go: 80-100 recommended.

  # ── INSTANCE AUTOSCALING BOUNDARIES ──────────────────────────────────────
  --min-instances=1 \
  # Integer: Minimum number of warm instances kept alive at all times.
  # Default: 0 (Scale-to-zero). Set >= 1 to completely eliminate cold starts.
  --max-instances=100 \
  # Integer: Upper scaling ceiling to protect downstream databases from flooding.
  # Default: 100.

  # ── ENVIRONMENT VARIABLES & SECRET MANAGER INTEGRATION ───────────────────
  --set-env-vars=ENVIRONMENT=production,LOG_LEVEL=info \
  # Key-value pairs: Runtime environment variables exposed to container.
  --set-secrets=DATABASE_PASSWORD=projects/123456789/secrets/db-pass:latest
  # Secret Manager mapping: Injects secret as environment variable or mounted file.
```

---

## 3. Source Code Ingestion Patterns

### Pattern 1: Deploying from Local Filesystem with `.gcloudignore`

When deploying from a local machine (`--source=.`), `gcloud` creates a ZIP archive and uploads it to Cloud Storage. To avoid multi-gigabyte uploads and accidental secret leakage, maintain a `.gcloudignore` file in the root directory.

#### Create `.gcloudignore`:
```bash
cat << 'EOF' > .gcloudignore
.gcloudignore
.git/
.gitignore
node_modules/
npm-debug.log
yarn-error.log
.env
.env.*
.venv/
__pycache__/
*.pyc
tests/
*.md
Dockerfile
.dockerignore
EOF
```

#### Deploy from Local Directory:
```bash
gcloud functions deploy user-service \
  --gen2 \
  --region=us-central1 \
  --runtime=nodejs20 \
  --entry-point=handler \
  --source=. \
  --trigger-http \
  --allow-unauthenticated
```

---

### Pattern 2: Deploying from Cloud Storage ZIP Archive

When automated CI systems produce pre-built source bundles, package the files into a ZIP archive and upload to Cloud Storage.

> [!IMPORTANT]
> **Archive Root Constraint**: All source files (`package.json`, `index.js`, or `main.py`) **MUST reside at the root of the ZIP file**, NOT inside an enclosing parent folder.

```bash
# Correct Packaging: Compress directory contents directly (without parent folder)
cd /path/to/my-function-code
zip -r /tmp/source.zip .

# Upload archive to Cloud Storage
gcloud storage cp /tmp/source.zip gs://my-function-artifacts-bucket/releases/v1.0.0.zip

# Deploy directly referencing Cloud Storage URI
gcloud functions deploy payment-service \
  --gen2 \
  --region=us-central1 \
  --runtime=python311 \
  --entry-point=process_payment \
  --source=gs://my-function-artifacts-bucket/releases/v1.0.0.zip \
  --trigger-http \
  --no-allow-unauthenticated
```

---

### Pattern 3: Deploying from Cloud Source Repositories (or GitHub / Bitbucket)

Deploy directly from revision-controlled Git repositories without uploading local archives.

```bash
# Format:
# https://source.developers.google.com/p/[PROJECT_ID]/r/[REPO_NAME]/revisions/[REVISION]/paths/[SUBDIR]

# Deploy from a specific Git tag in a monorepo sub-directory:
gcloud functions deploy inventory-service \
  --gen2 \
  --region=us-central1 \
  --runtime=go122 \
  --entry-point=HandleInventory \
  --source="https://source.developers.google.com/p/prod-corp-cloud/r/backend-services/revisions/v2.1.0/paths/services/inventory" \
  --trigger-http

# Deploy from the 'main' branch root directory:
gcloud functions deploy webhook-listener \
  --gen2 \
  --region=us-central1 \
  --runtime=python311 \
  --entry-point=webhook_entry \
  --source="https://source.developers.google.com/p/prod-corp-cloud/r/webhook-repo/revisions/main" \
  --trigger-topic=webhook-incoming
```

---

## 4. IAM Permissions & Service Account Delegation

Deploying Cloud Run functions requires two distinct IAM configurations: permissions for the deployer, and permissions for Google service agents.

### Step 4.1: Grant Deployer Permissions (User or CI/CD Service Account)

The developer or CI pipeline deploying the function must have permissions to manage functions and impersonate the runtime service account.

```bash
export PROJECT_ID="my-gcp-project"
export DEPLOYER_USER="developer@company.com"
export RUNTIME_SA="fn-runner-sa@${PROJECT_ID}.iam.gserviceaccount.com"

# 1. Grant Cloud Functions Developer role at the project level
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="user:${DEPLOYER_USER}" \
  --role="roles/cloudfunctions.developer"

# 2. Grant Service Account User role ON the specific runtime Service Account
gcloud iam service-accounts add-iam-policy-binding ${RUNTIME_SA} \
  --member="user:${DEPLOYER_USER}" \
  --role="roles/iam.serviceAccountUser"
```

---

### Step 4.2: Grant Permissions to Cloud Run Functions Service Agent

When deploying from Cloud Storage or Cloud Source Repositories, Google Cloud's managed service agent requires read permissions.

```bash
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)")
export GCF_SERVICE_AGENT="service-${PROJECT_NUMBER}@gcf-admin-robot.iam.gserviceaccount.com"

# Grant Storage Object Viewer to read deployment ZIP archives from GCS
gcloud storage buckets add-iam-policy-binding gs://my-function-artifacts-bucket \
  --member="serviceAccount:${GCF_SERVICE_AGENT}" \
  --role="roles/storage.objectViewer"

# Grant Source Repository Reader to fetch code from Cloud Source Repositories
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${GCF_SERVICE_AGENT}" \
  --role="roles/source.reader"
```

---

## 5. Build Inspection & Observability Commands

Because Cloud Run functions (2nd gen) builds container images with Cloud Build and stores them in Artifact Registry, all build output is visible in your project's Cloud Logging.

```bash
# ── BUILD PIPELINE OBSERVABILITY ───────────────────────────────────────────

# List recent builds executed by Cloud Run Functions
gcloud builds list --limit=5

# Stream real-time build logs for a specific Cloud Build ID
gcloud builds log <BUILD_ID> --stream

# List container images generated for Cloud Run functions in Artifact Registry
gcloud artifacts docker images list us-central1-docker.pkg.dev/${PROJECT_ID}/gcf-artifacts

# ── RUNTIME LOGS & TELEMETRY ───────────────────────────────────────────────

# Read live runtime execution logs (stdout/stderr) from Cloud Run
gcloud functions logs read order-processing-service \
  --gen2 \
  --region=us-central1 \
  --limit=50

# Describe deployed Cloud Run function details, revision status, and endpoints
gcloud functions describe order-processing-service \
  --gen2 \
  --region=us-central1 \
  --format="yaml(state,serviceConfig.uri,buildConfig.build)"

# ── REVISION MANAGEMENT & TRAFFIC SPLITTING ─────────────────────────────────

# List all revisions of the underlying Cloud Run service
gcloud run revisions list \
  --service=order-processing-service \
  --region=us-central1 \
  --format="table(name,active,traffic_percent)"

# Split traffic across two revisions (e.g. 50% Canary / A-B testing)
gcloud run services update-traffic order-processing-service \
  --region=us-central1 \
  --to-revisions=order-processing-service-00001-abc=50,order-processing-service-00002-xyz=50

# Roll back 100% of traffic to a previous stable revision
gcloud run services update-traffic order-processing-service \
  --region=us-central1 \
  --to-revisions=order-processing-service-00001-abc=100

# Route 100% of traffic to the latest deployed revision
gcloud run services update-traffic order-processing-service \
  --region=us-central1 \
  --to-latest

# ── LOCAL TESTING WITH FUNCTIONS FRAMEWORK ─────────────────────────────────

# Run function locally on port 8080 with Functions Framework
npx @google-cloud/functions-framework --target=processOrder --port=8080

# Invoke locally running HTTP function
curl "http://localhost:8080/?temp=70"

# Execute pre-deployment unit test suite via Mocha
npm test
```

---

## 6. Comprehensive Troubleshooting & Error Matrix

| Symptom / Error Message | Root Cause Analysis | Corrective Resolution Action |
| :--- | :--- | :--- |
| `Cloud Build API has not been used in project...` | The `cloudbuild.googleapis.com` API is disabled. 2nd gen functions strictly require Cloud Build for containerization. | Run `gcloud services enable cloudbuild.googleapis.com` and retry the deployment. |
| `Permission 'iam.serviceaccounts.actAs' denied on service account` | Deploying identity lacks `roles/iam.serviceAccountUser` permission on the runtime service account. | Bind `roles/iam.serviceAccountUser` on the target runtime SA to the deployer identity using `gcloud iam service-accounts add-iam-policy-binding`. |
| `Function failed on loading user code: FileNotFoundError / Cannot find module 'index.js'` | When deploying from a Cloud Storage ZIP archive, files were compressed inside a sub-folder instead of the archive root. | Repackage archive: `cd my-code-dir && zip -r /tmp/source.zip .` ensuring `package.json`/`main.py` is at the root. |
| `AccessDenied: Cloud Run functions service agent does not have permission to read from bucket` | The service agent `service-PROJECT_NUMBER@gcf-admin-robot.iam.gserviceaccount.com` lacks read permissions on the staging bucket. | Run `gcloud storage buckets add-iam-policy-binding gs://BUCKET --member="serviceAccount:service-NUM@gcf-admin-robot..." --role="roles/storage.objectViewer"`. |
| `AccessDenied: Cloud Run functions service agent does not have permission to read repository` | The service agent lacks `roles/source.reader` on the Google Cloud Source Repository. | Run `gcloud projects add-iam-policy-binding PROJECT_ID --member="serviceAccount:service-NUM@gcf-admin-robot..." --role="roles/source.reader"`. |
| `Deployment times out during source upload (> 50 MB - 500 MB uploaded)` | Missing `.gcloudignore` causing `node_modules/`, `.git/`, or large test media to be compressed and uploaded over WAN. | Create a `.gcloudignore` file in the root source directory containing `node_modules/`, `.git/`, `.venv/`. |
| `HTTP 403 Forbidden` on invocation | Function was deployed with `--no-allow-unauthenticated` or omitted `--allow-unauthenticated`. | To allow public traffic, run `gcloud functions add-iam-policy-binding <NAME> --region=<REGION> --member="allUsers" --role="roles/run.invoker"`. |
| `Build failed: buildpack could not determine runtime` | Missing dependency descriptor file (`package.json` for Node, `requirements.txt` for Python, `go.mod` for Go). | Ensure the required manifest file exists at the root of the source directory. |
