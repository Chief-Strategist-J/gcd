# GCP Core Configuration & Project Management: High-Level & Low-Level Design Architecture

This document presents the **High-Level Design (HLD)** and **Low-Level Design (LLD)** for Google Cloud Platform (GCP) Organization Resource Hierarchy, Authentication Mechanics, Named SDK Configurations, and Policy Inheritance.

---

## 1. High-Level Design (HLD) Architecture

The GCP Resource Hierarchy governs how cloud resources are organized, how security policies (IAM) inherit downward, and how billing accounts aggregate infrastructure costs.

```mermaid
graph TD
    subgraph GCP Organization Node
        ORG["Google Cloud Organization<br/>(e.g., company.com)"]
    end

    subgraph Org Policy & IAM Root
        POLICY["Organization Policies<br/>(e.g., disableSerialPortAccess, restrictVpcPeering)"]
        IAM_ORG["Root IAM Bindings<br/>(Organization Admin, Security Admin)"]
    end

    ORG --> POLICY
    ORG --> IAM_ORG

    subgraph Folder Hierarchy
        F_DEV["Folder: Development"]
        F_PROD["Folder: Production"]
    end

    ORG --> F_DEV
    ORG --> F_PROD

    subgraph Projects
        P_DEV1["Project: dev-web-app-01<br/>ID: dev-web-app-01-4829"]
        P_PROD1["Project: prod-core-api-01<br/>ID: prod-core-api-01-9921"]
    end

    F_DEV --> P_DEV1
    F_PROD --> P_PROD1

    subgraph Services & Resources
        GKE["GKE Cluster"]
        GCS["Cloud Storage Bucket"]
        BQ["BigQuery Dataset"]
        GCE["Compute Engine VMs"]
    end

    P_DEV1 --> GKE
    P_DEV1 --> GCS
    P_PROD1 --> BQ
    P_PROD1 --> GCE

    subgraph Billing Architecture
        BILLING["GCP Billing Account<br/>(Credit Card / Invoice Billing)"]
    end

    BILLING -.->|Linked Billing| P_DEV1
    BILLING -.->|Linked Billing| P_PROD1

    style ORG fill:#4285F4,stroke:#333,stroke-width:2px,color:#fff
    style POLICY fill:#EA4335,stroke:#333,stroke-width:1px,color:#fff
    style P_DEV1 fill:#34A853,stroke:#333,stroke-width:1px,color:#fff
    style P_PROD1 fill:#FBBC05,stroke:#333,stroke-width:1px,color:#333
```

---

## 2. Low-Level Design (LLD) Architecture: Authentication & `gcloud` Profiles

The LLD illustrates how local SDK authentication, named configurations, Application Default Credentials (ADC), and Service Account Impersonation interact with Google APIs.

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Engineer / CI/CD pipeline
    participant CLI as gcloud CLI (~/.config/gcloud)
    participant ADC as Application Default Credentials (ADC)
    participant IAM as GCP IAM & STS Token Service
    participant API as GCP Resource API (e.g., GKE / Compute / BQ)

    rect rgb(240, 248, 255)
    Note over Dev, CLI: Scenario A: User Authentication & Named Configurations
    Dev->>CLI: gcloud auth login / gcloud config set project prod-core-api-01
    CLI->>CLI: Store Refresh Token in ~/.config/gcloud/credentials.db
    CLI->>CLI: Activate profile in ~/.config/gcloud/configurations/config_prod
    end

    rect rgb(255, 245, 238)
    Note over Dev, API: Scenario B: Service Account Impersonation (No Stored Keys)
    Dev->>CLI: gcloud compute instances list --impersonate-service-account=ops-sa@prod.iam.gserviceaccount.com
    CLI->>IAM: Call generateAccessToken(target_sa, scopes) using User OAuth Token
    IAM-->>CLI: Return Short-Lived OIDC OAuth2 Token (1 Hour Lifetime)
    CLI->>API: GET /compute/v1/projects/prod-core-api-01/zones/us-central1-a/instances
    API-->>CLI: 200 OK (Instance List Data Returned)
    end

    rect rgb(245, 255, 245)
    Note over Dev, ADC: Scenario C: Application Default Credentials (ADC)
    Dev->>CLI: gcloud auth application-default login
    CLI->>ADC: Write ~/.config/gcloud/application_default_credentials.json
    Note over ADC, API: Client Libraries (Go, Python, Java) automatically load credentials from ADC path
    end
```

---

## 3. Key Architecture Components

1. **GCP Resource Hierarchy**:
   - **Organization**: Root node tied to Google Workspace or Cloud Identity domain. Sets baseline organization policies and root IAM guardrails.
   - **Folders**: Organizational units (e.g., by environment `dev/prod` or department `finance/engineering`). Permissions applied at folder level inherit down to all contained projects.
   - **Projects**: The fundamental quota, billing, and resource isolation boundary. Every GCP resource belongs to exactly one project.

2. **`gcloud` Named Configuration Engine**:
   - Manages isolated SDK profiles stored under `~/.config/gcloud/configurations/`.
   - Allows instant context switching (`gcloud config configurations activate dev-profile`) without re-authenticating.

3. **Authentication Mechanics**:
   - **User Credentials**: Short-lived OAuth2 tokens generated via browser login (`gcloud auth login`).
   - **Application Default Credentials (ADC)**: Credential fallback standard used by code SDKs (`google-cloud-storage`, `google-cloud-bigquery`) during local development.
   - **Service Account Impersonation**: Enterprise security standard where users assume short-lived SA identities via `iam.serviceAccounts.getAccessToken` without creating or leaking persistent `.json` key files.
