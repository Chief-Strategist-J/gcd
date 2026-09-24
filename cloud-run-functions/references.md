# Cloud Run Functions: Official Reference Links & Resources

This document provides curated links to official Google Cloud documentation, CLI references, Cloud Buildpack specifications, and production best practices for **Google Cloud Run Functions**.

---

## 1. Official Documentation Links

| Resource Title | URL Link | Description |
| :--- | :--- | :--- |
| **Cloud Run Functions Overview** | [cloud.google.com/functions/docs](https://cloud.google.com/functions/docs) | Official landing page for Google Cloud Run Functions documentation. |
| **Deploying from Source Code** | [cloud.google.com/functions/docs/deploy](https://cloud.google.com/functions/docs/deploy) | Step-by-step guides for deploying from local files, Cloud Storage, and repositories. |
| **`gcloud functions deploy` Reference** | [cloud.google.com/sdk/gcloud/reference/functions/deploy](https://cloud.google.com/sdk/gcloud/reference/functions/deploy) | Complete CLI command specification and parameter dictionary. |
| **Cloud Run Functions IAM Roles** | [cloud.google.com/functions/docs/concepts/iam](https://cloud.google.com/functions/docs/concepts/iam) | Guide on Developer roles, runtime service accounts, and service agent permissions. |
| **Cloud Build Integration & Buildpacks** | [cloud.google.com/build/docs/building-with-buildpacks](https://cloud.google.com/build/docs/building-with-buildpacks) | Details on how Google Cloud Buildpacks automatically compile container images. |
| **Artifact Registry Integration** | [cloud.google.com/artifact-registry/docs/docker/manage-images](https://cloud.google.com/artifact-registry/docs/docker/manage-images) | Managing compiled container images and software artifacts. |
| **Cloud Code IDE Extension** | [cloud.google.com/code/docs](https://cloud.google.com/code/docs) | Using Cloud Code in VS Code and IntelliJ to create, run, and debug functions. |
| **Eventarc Triggers Guide** | [cloud.google.com/eventarc/docs/run/create-trigger](https://cloud.google.com/eventarc/docs/run/create-trigger) | Configuring event-driven CloudEvent triggers across 130+ GCP event sources. |
| **Connecting to Memorystore (Redis)** | [cloud.google.com/functions/docs/networking/connecting-redis](https://cloud.google.com/functions/docs/networking/connecting-redis) | Configuring Serverless VPC Access to securely connect functions to Memorystore Redis and Memcached. |
| **Using Environment Variables** | [cloud.google.com/functions/docs/configuring/env-var](https://cloud.google.com/functions/docs/configuring/env-var) | Setting and reading runtime and buildpack environment variables via CLI and YAML configs. |
| **Firestore Triggers (Native Mode)** | [cloud.google.com/functions/docs/calling/cloud-firestore](https://cloud.google.com/functions/docs/calling/cloud-firestore) | Handling document create, update, delete, and write events via Eventarc CloudEvents. |
| **Using Secrets from Secret Manager** | [cloud.google.com/functions/docs/configuring/secrets](https://cloud.google.com/functions/docs/configuring/secrets) | Mounting secrets as volume files or environment variables with automatic rotation and cross-project access. |
| **Serverless VPC Access Guide** | [cloud.google.com/vpc/docs/configure-serverless-vpc-access](https://cloud.google.com/vpc/docs/configure-serverless-vpc-access) | Dedicated `/28` CIDR allocation, region constraints, and lifecycle management for VPC connectors. |

---

## 2. Production Best Practices Checklist

1. **Security & Identity**:
   - **Never run as Default Service Account**: Always provision a dedicated runtime service account with only the permissions required for that specific function.
   - **Enforce Ingress Authentication**: Avoid `--allow-unauthenticated` in production unless building a public API endpoint. For inter-service calls, use IAM tokens (`roles/run.invoker`).
2. **Build Optimization & Hygiene**:
   - **Maintain `.gcloudignore`**: Always exclude dependencies (`node_modules`, `.venv`), version control (`.git`), tests, and local environment files (`.env`) to minimize upload payload and prevent credential leakage.
   - **ZIP Archive Root Structure**: When deploying via Cloud Storage ZIP files, ensure the manifest and source files reside strictly at the root of the archive.
3. **Performance & Concurrency**:
   - **Tune Multi-Concurrency**: For I/O-bound runtimes (Node.js, Go), set `--concurrency` between 40 and 100 to maximize throughput and drastically lower instance count and billing costs.
   - **Cold Start Elimination**: Set `--min-instances=1` (or higher) on latency-critical endpoints to keep warm instances ready in the pool.
4. **Resilience & Timeouts**:
   - **Set Explicit Timeouts**: Set `--timeout` based on maximum realistic duration (up to 3,600 seconds for HTTP in Gen 2) to guard against hung client connections.
   - **Max Instances Guardrail**: Configure `--max-instances` to prevent runaway billing spikes or accidental Denial-of-Service attacks against downstream databases.
5. **Database & Cache Connection Pooling**:
   - **Client Instantiation Outside Handlers**: Instantiate Redis, Firestore, and SQL clients in global scope outside the request handler function to reuse persistent connection pools across warm container invocations.
   - **Serverless VPC Access Ready State**: Verify the VPC connector is in `READY` status before initiating function deployment.
6. **Secrets & Configuration Management**:
   - **Zero-Trust Secret Injection**: Never hardcode API keys or DB passwords in environment variables or Git commits. Use Google Cloud Secret Manager.
   - **Volume Mount vs Environment Variables**: Prefer mounting secrets as volume files (`--set-secrets=/mount/path=SECRET:latest`) for dynamic rotation of the latest version without requiring container redeployment.
7. **Firestore Trigger Idempotency & Native Mode**:
   - **Native Mode Enforcement**: Verify Firestore operates in Native Mode (Datastore mode does not emit trigger events).
   - **Idempotent Handlers**: Ensure Firestore trigger handlers check event IDs (`cloudEvent.id`) or mutation timestamps to prevent duplicate state processing during at-least-once deliveries.
8. **Observability**:
   - Enable Cloud Logging and Cloud Monitoring to track cold starts, CPU/memory saturation, execution latency, and error rates in real time.
