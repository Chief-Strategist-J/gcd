# Cloud Run Functions: Decision Trees (ASCII & Visual)

This document provides structured decision logic trees for selecting source code locations, deployment tooling, runtime triggers, and IAM permission strategies for **Google Cloud Run Functions**.

---

## 1. Source Location Selection (ASCII Decision Tree)

```
================================================================================
              CLOUD RUN FUNCTIONS SOURCE LOCATION DECISION TREE
================================================================================

            Where does your function source code currently reside?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
  [ Local Machine ]            [ Object Storage ]            [ Git Repository ]
   Workstation, Laptop          Pre-packaged Builds,          Cloud Source Repositories,
   or Local CI Runner           Automated CI Artifacts        GitHub, Bitbucket Mirrors
        │                              │                              │
        ├────────────────┐             ▼                              ▼
        ▼                ▼      USE CLOUD STORAGE               USE SOURCE REPO
  Quick testing?   Multi-file   `--source=gs://.../app.zip`    `--source=https://...`
        │          production?         │                              │
        │                │             ▼                              ▼
        ▼                ▼       CRITICAL RULE:                 CRITICAL RULE:
  CONSOLE INLINE   USE LOCAL     Files MUST reside at root      Grant Cloud Run Functions
  EDITOR           DIRECTORY     of the ZIP archive (no         Service Agent `roles/`
  (Console UI)     `--source=.`  nested top-level directory).   `source.reader` role.
        │                │
        ▼                ▼
  Use for rapid    CRITICAL RULE:
  prototypes &     Always configure `.gcloudignore`
  syntax tweaks    to exclude `node_modules` & secrets.
```

---

## 2. Deployment Tooling Selection (ASCII Decision Tree)

```
================================================================================
             CLOUD RUN FUNCTIONS DEPLOYMENT TOOLING DECISION TREE
================================================================================

              What is your primary interface / operational environment?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
  [ Local Terminal / CLI ]      [ Inside the IDE ]             [ CI/CD Pipeline ]
   Scripting, One-Off Tasks,     VS Code or IntelliJ           GitOps, Production
   Terminal automation           Developer Experience           Build & Automated Tests
        │                              │                              │
        ▼                              ▼                              ▼
  USE GCLOUD CLI                 USE CLOUD CODE                 USE CLOUD BUILD
  `gcloud functions deploy`      IDE Extension                  `cloudbuild.yaml`
  - Granular flag control        - Run & debug locally          - Ephemeral build worker
  - Scriptable shell commands    - Visual deployment wizard     - Direct Artifact Registry
  - Fast feedback loops          - Built-in breakpoint tooling    integration & audit logs
```

---

## 3. Runtime Trigger & Architecture Selection (ASCII Decision Tree)

```
================================================================================
           CLOUD RUN FUNCTIONS TRIGGER & GENERATION DECISION TREE
================================================================================

                 How will your function be invoked by clients?
                                       │
        ┌──────────────────────────────┴──────────────────────────────┐
        ▼                                                             ▼
  [ Synchronous Invocations ]                                  [ Asynchronous Invocations ]
  REST APIs, Webhooks, Microservices                            Event-Driven, Storage, Pub/Sub
        │                                                             │
        ▼                                                             ▼
  USE HTTP TRIGGER                                             USE EVENTARC TRIGGER
  `--trigger-http`                                             `--trigger-event-filters=...`
        │                                                             │
        ├───────────────────────────────┐                             ├───────────────────────┐
        ▼                               ▼                             ▼                       ▼
  Public Endpoint?               Internal / Auth?               Pub/Sub Topic?          Cloud Storage?
  `--allow-unauthenticated`      `--no-allow-unauthenticated`   `--trigger-topic=...`   `--trigger-event-filters=`
  (API Gateway / Web Frontend)   (Requires OIDC token auth)                             `type=google.cloud.storage...`
```

---

## 4. Visual Mermaid Decision Flowcharts

### 4.1 Source Code Location Strategy Flowchart:

```mermaid
graph TD
    StartSrc["Select Function Source Code Location"] --> DevContext{"Where is your source code maintained?"}

    DevContext -->|Local Workstation / Laptop| LocalCheck{"Is this a quick test or production code?"}
    LocalCheck -->|Quick browser prototype| ConsoleUI["Google Cloud Console Inline Editor<br/>(Dual-pane file navigator + editor)"]
    LocalCheck -->|Structured local repository| LocalDir["Local Directory Ingestion<br/>Flag: --source=.<br/>Requires: .gcloudignore"]

    DevContext -->|Automated Build / Staging Bucket| GCSBucket["Cloud Storage ZIP Archive<br/>Flag: --source=gs://bucket/file.zip<br/>Requires: Files at root of zip<br/>Requires: Service Agent Storage Viewer"]

    DevContext -->|Version-Controlled Git Repo| GitRepo["Cloud Source Repositories<br/>Flag: --source=https://source.developers.google.com/...<br/>Supports: GitHub / Bitbucket mirrors<br/>Requires: Service Agent roles/source.reader"]
```

---

### 4.2 Deployment Tooling Selection Flowchart:

```mermaid
graph TD
    StartTool["Select Deployment Tooling"] --> ToolEnv{"Developer & Infrastructure Interface?"}

    ToolEnv -->|Interactive Terminal / Shell Automation| GcloudCLI["gcloud CLI<br/>(gcloud functions deploy --gen2 ...)<br/>Best for: Rapid experimentation & admin scripts"]

    ToolEnv -->|Integrated Development Environment| CloudCode["Cloud Code Extension<br/>(VS Code / IntelliJ)<br/>Best for: Local breakpoint debugging & IDE deployments"]

    ToolEnv -->|Automated CI/CD Pipeline| CloudBuild["Google Cloud Build Pipeline<br/>(cloudbuild.yaml & Artifact Registry)<br/>Best for: GitOps, automated test verification & auditable rollouts"]

    ToolEnv -->|Zero-Tooling Web Browser| GCPWebConsole["Google Cloud Web Console<br/>(Cloud Functions UI)<br/>Best for: Beginners, visual troubleshooting & log inspection"]
```

---

### 4.3 IAM Role Assignment Flowchart:

```mermaid
graph TD
    StartIAM["Configure IAM Security & Access"] --> Identity{"Which Identity are you configuring?"}

    Identity -->|Developer / Operator / CI SA| DeployerRoles["Grant Deployer Permissions:<br/>1. roles/cloudfunctions.developer (on Project)<br/>2. roles/iam.serviceAccountUser (on Runtime SA)"]

    Identity -->|Cloud Run Functions Service Agent| AgentRoles{"Which source backend is used?"}
    AgentRoles -->|Cloud Storage ZIP| AgentGCS["Grant roles/storage.objectViewer on bucket"]
    AgentRoles -->|Cloud Source Repositories| AgentCSR["Grant roles/source.reader on repository"]

    Identity -->|Runtime Execution Service Account| RuntimeRoles["Grant Minimum Downstream Roles:<br/>e.g., roles/datastore.user, roles/bigquery.dataEditor"]
```
