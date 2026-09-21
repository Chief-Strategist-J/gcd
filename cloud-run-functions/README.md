# Feature 9: Cloud Run Functions (Serverless Compute & Event-Driven Workloads)

Welcome to the dedicated documentation index for **Google Cloud Run Functions** (formerly Cloud Functions 2nd gen). This feature module provides architectural blueprints, build and deployment mechanics, decision trees for source locations and tooling, exhaustive `gcloud functions` CLI manuals, failure recovery runbooks, and IAM security matrices.

---

## Module Documentation Index

| Section | Document File | Description |
| :--- | :--- | :--- |
| **Architecture (HLD & LLD)** | [`hld-lld-design.md`](./hld-lld-design.md) | High-Level Architecture (Serverless Event-Driven Topology, Cloud Build, Artifact Registry, Cloud Run runtime) & Low-Level Design (Buildpack containerization, Gen 1 vs Gen 2 differences, IAM Service Agent delegation, and Cloud Logging build pipelines). |
| **Decision Trees** | [`decision-tree.md`](./decision-tree.md) | ASCII & Visual Mermaid decision trees for Deployment Source Strategy (Local Directory vs Cloud Storage ZIP vs Cloud Source Repositories/GitHub vs Console Inline Editor), Deployment Tooling, and Trigger Architectures (HTTP vs Eventarc). |
| **CLI Commands & Error Matrix** | [`shell-commands.md`](./shell-commands.md) | Comprehensive `gcloud functions deploy` command manual, deep flag breakdown (`--gen2`, `--source`, `--entry-point`, `--runtime`, `--stage-bucket`), `.gcloudignore` syntax, IAM role assignments, and an exhaustive Troubleshooting Error Matrix. |
| **Triggers, VPC & Workflows** | [`triggers-vpc-workflows.md`](./triggers-vpc-workflows.md) | In-depth manual covering HTTP and 90+ Eventarc event triggers (Pub/Sub, Storage, Firestore document-level rules, Firebase), 1:1 trigger binding vs 1:N fanout, VPC Ingress/Egress, and Google Cloud Workflows orchestration. |
| **Security, IAM & Zero-Trust** | [`security-and-iam.md`](./security-and-iam.md) | Comprehensive security guide covering Identity-Based vs Network-Based controls, OAuth 2.0 vs OIDC ID tokens, Service-to-Service authentication, Runtime Service Accounts, and VPC Service Controls (VPC-SC). |
| **Case Studies & Hands-On Labs** | [`casestudy/README.md`](./casestudy/README.md) | End-to-end hands-on lab blueprint covering HTTP functions, Cloud Storage event triggers via Eventarc, Functions Framework unit testing with Mocha/Sinon, and immutable revision traffic management. |
| **Official References** | [`references.md`](./references.md) | Official Google Cloud Run Functions documentation links, Cloud Buildpack specs, Eventarc integrations, IAM security guides, and runtime lifecycle references. |

---

## Key Technical & Operational Concepts Covered

1. **Dual-Generation Architecture (Gen 1 vs Gen 2)**: Cloud Run functions (2nd gen) built natively on top of Google Cloud Run and Eventarc, delivering up to 60-minute execution timeouts, multi-concurrency (up to 1,000 concurrent requests per instance), and direct integration with Artifact Registry.
2. **Automated Source-to-Container Build Lifecycle**:
   - Source code uploaded to Google Cloud Storage (either default staging bucket or explicit `--stage-bucket`).
   - Cloud Build automates buildpack compilation into standard OCI-compliant container images without requiring a custom Dockerfile.
   - Built images are pushed to Google Cloud Artifact Registry within the user project.
   - Cloud Run functions schedules and manages the container lifecycle to execute requests on demand.
3. **Flexible Source Code Ingestion Models**:
   - **Local Machine**: Direct directory upload with `.gcloudignore` filtering to strip `node_modules`, `.git`, temporary artifacts, and secrets.
   - **Cloud Storage**: Ingesting zip archives (`gs://bucket/source.zip`), requiring source code files to reside strictly at the archive root.
   - **Cloud Source Repositories**: Integrated with Git revisions, branches, tags, and sub-path directories (`/revisions/<rev>/paths/<path>`), linking directly to GitHub or Bitbucket mirrors.
   - **Cloud Console Inline Editor**: Real-time browser-based development with dual-pane navigation (file tree + source code editor).
4. **IAM Privilege Delegation & Service Agents**:
   - Developer permissions: `roles/cloudfunctions.developer` + `roles/iam.serviceAccountUser` on the runtime service account.
   - Service Agent authorization: Granting `roles/source.reader` for Source Repositories, and bucket read permissions for Cloud Storage ingestion.
5. **Multi-Channel Deployment Tooling**:
   - Automated deployment workflows via `gcloud CLI`, declarative Cloud Build configs (`cloudbuild.yaml`), Cloud Code IDE extensions (VS Code & IntelliJ), and the Cloud Console.
6. **Unified Observability & Auditability**:
   - Cloud Build build-time logs streamed directly to Cloud Logging in the user project.
   - Runtime execution stdout/stderr logs and latency traces exported to Cloud Logging and Cloud Trace.

---

## Quick Navigation

1. [View High-Level & Low-Level Design (`hld-lld-design.md`)](./hld-lld-design.md)
2. [View Decision Trees (`decision-tree.md`)](./decision-tree.md)
3. [View Shell Command Reference & Error Matrix (`shell-commands.md`)](./shell-commands.md)
4. [View Triggers, VPC Networking & Workflows (`triggers-vpc-workflows.md`)](./triggers-vpc-workflows.md)
5. [View Security, IAM & Zero-Trust Manual (`security-and-iam.md`)](./security-and-iam.md)
6. [View Case Studies & Hands-On Labs (`casestudy/README.md`)](./casestudy/README.md)
7. [View Official Reference Links (`references.md`)](./references.md)
