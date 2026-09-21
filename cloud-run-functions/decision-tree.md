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

## 4. VPC Ingress & Egress Networking Decision Tree (ASCII)

```
================================================================================
              CLOUD RUN FUNCTIONS VPC NETWORKING DECISION TREE
================================================================================

              What network path is required for your function?
                                       │
        ┌──────────────────────────────┴──────────────────────────────┐
        ▼                                                             ▼
  [ INGRESS (Inbound Traffic) ]                                [ EGRESS (Outbound Traffic) ]
  Who should be allowed to invoke?                             Where does the function call out to?
        │                                                             │
        ├──────────────────────────────┬───────────────┐              ├──────────────────────────────┐
        ▼                              ▼               ▼              ▼                              ▼
  [ Public Internet ]          [ Internal VPC ]  [ Cloud WAF / ALB ] [ Internal VPC Resources ]     [ External APIs / Whitelisting ]
  `--ingress-settings=all`     `--ingress-settings= `--ingress-settings= Cloud SQL, Redis, GKE       Third-party API requiring
  Open webhooks &              internal-only`     internal-and-cloud- `--vpc-egress=                 static public IP address
  developer APIs               Strict private     load-balancing`     private-ranges-only`           `--vpc-egress=all-traffic`
                               microservices      Guarded by Cloud    Direct internal routing via    Routes through Cloud NAT
                                                  Armor security rules Andromeda SDN                  Gateway for fixed public IP
```

---

## 5. Eventarc vs. Cloud Workflows Selection (ASCII Decision Tree)

```
================================================================================
         SERVERLESS ORCHESTRATION VS. CHOREOGRAPHY DECISION TREE
================================================================================

            How are multiple functions / services coordinated?
                                       │
        ┌──────────────────────────────┴──────────────────────────────┐
        ▼                                                             ▼
  [ Asynchronous Choreography ]                                [ Stateful Orchestration ]
  Decoupled, event-driven reactive pipelines                    Sequential steps, conditional business logic,
  "Fire-and-forget" notifications                               human approvals, complex retries & error handling
        │                                                             │
        ▼                                                             ▼
  USE EVENTARC (EVENT-DRIVEN)                                   USE GOOGLE CLOUD WORKFLOWS
  - Single event fan-out (1:N)                                  - Centralized declarative state machine (YAML)
  - Latency-sensitive event routing                             - Zero cold-start stacking (free wait state)
  - Independent microservices without central coordinator       - Native OIDC token injection for private endpoints
  - Direct integration with 90+ GCP event sources               - Up to 1 year execution durability
```

---

## 6. Visual Mermaid Decision Flowcharts

### 6.1 Source Code Location Strategy Flowchart:

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

### 6.2 Deployment Tooling Selection Flowchart:

```mermaid
graph TD
    StartTool["Select Deployment Tooling"] --> ToolEnv{"Developer & Infrastructure Interface?"}

    ToolEnv -->|Interactive Terminal / Shell Automation| GcloudCLI["gcloud CLI<br/>(gcloud functions deploy --gen2 ...)<br/>Best for: Rapid experimentation & admin scripts"]

    ToolEnv -->|Integrated Development Environment| CloudCode["Cloud Code Extension<br/>(VS Code / IntelliJ)<br/>Best for: Local breakpoint debugging & IDE deployments"]

    ToolEnv -->|Automated CI/CD Pipeline| CloudBuild["Google Cloud Build Pipeline<br/>(cloudbuild.yaml & Artifact Registry)<br/>Best for: GitOps, automated test verification & auditable rollouts"]

    ToolEnv -->|Zero-Tooling Web Browser| GCPWebConsole["Google Cloud Web Console<br/>(Cloud Functions UI)<br/>Best for: Beginners, visual troubleshooting & log inspection"]
```

---

### 6.3 IAM Role Assignment Flowchart:

```mermaid
graph TD
    StartIAM["Configure IAM Security & Access"] --> Identity{"Which Identity are you configuring?"}

    Identity -->|Developer / Operator / CI SA| DeployerRoles["Grant Deployer Permissions:<br/>1. roles/cloudfunctions.developer (on Project)<br/>2. roles/iam.serviceAccountUser (on Runtime SA)"]

    Identity -->|Cloud Run Functions Service Agent| AgentRoles{"Which source backend is used?"}
    AgentRoles -->|Cloud Storage ZIP| AgentGCS["Grant roles/storage.objectViewer on bucket"]
    AgentRoles -->|Cloud Source Repositories| AgentCSR["Grant roles/source.reader on repository"]

    Identity -->|Runtime Execution Service Account| RuntimeRoles["Grant Minimum Downstream Roles:<br/>e.g., roles/datastore.user, roles/bigquery.dataEditor"]
```

---

### 6.4 VPC Ingress & Egress Routing Flowchart:

```mermaid
graph TD
    StartVPC["Configure Function Networking"] --> NetDirection{"Direction of Network Traffic?"}

    NetDirection -->|Inbound Ingress| IngressType{"Who invokes the function?"}
    IngressType -->|Public Internet / Any Client| IngAll["--ingress-settings=all<br/>(Public endpoint, direct access)"]
    IngressType -->|Strict Internal Microservices| IngInt["--ingress-settings=internal-only<br/>(Only reachable within VPC / VPN / Interconnect)"]
    IngressType -->|Cloud Armor WAF / HTTPS Load Balancer| IngALB["--ingress-settings=internal-and-cloud-load-balancing<br/>(Blocks direct .run.app URL; requires Load Balancer)"]

    NetDirection -->|Outbound Egress| EgressDest{"Where is the destination located?"}
    EgressDest -->|Private VPC: Cloud SQL / Redis / VMs| EgPriv["--vpc-egress=private-ranges-only<br/>(Routes RFC 1918 internal; internet exits via Google)"]
    EgressDest -->|External Partner needing Static IP| EgAll["--vpc-egress=all-traffic<br/>(Routes 100% traffic through VPC & Cloud NAT Static IP)"]
```

---

### 6.5 Orchestration vs. Event Choreography Flowchart:

```mermaid
graph TD
    StartCoord["Choose Serverless Coordination Model"] --> Pattern{"Coordination Pattern Requirement?"}

    Pattern -->|Event-driven, independent, fire-and-forget| EventarcCoord["Use Eventarc (Choreography)<br/>• Decoupled microservices<br/>• 1:N fan-out on single storage/pubsub event<br/>• Reactive, low-latency execution"]

    Pattern -->|Multi-step workflow, retries, branching logic| WorkflowCoord["Use Cloud Workflows (Orchestration)<br/>• Centralized state machine<br/>• Zero idle compute cost while waiting<br/>• Built-in exponential retries & error handling<br/>• Secure OIDC auth to private functions"]
```
