# GCP Core Configuration & Project Management: Decision Trees

This guide provides visual decision trees (Mermaid diagrams & text decision paths) for selecting GCP authentication strategies, project hierarchy structures, and organization policy guardrails.

---

## 1. Authentication Strategy Decision Tree

```mermaid
flowchart TD
    START["Identify Execution Context"] --> IS_CICD{"Is this running in automated CI/CD or production workload?"}

    IS_CICD -- "No (Local Engineer Machine)" --> DEV_AUTH{"Developing application code or using CLI commands?"}
    DEV_AUTH -- "Running gcloud / gsutil / bq CLI" --> GCLOUD_LOGIN["Use 'gcloud auth login' + Named Configurations"]
    DEV_AUTH -- "Developing code with GCP SDKs" --> ADC_LOGIN["Use 'gcloud auth application-default login'"]

    IS_CICD -- "Yes (Automated Pipeline / Workload)" --> HOSTED{"Where is the workload hosted?"}
    HOSTED -- "Inside GCP (GKE, Compute, Cloud Run)" --> GCF_SA["Use Attached Service Account (No keys required)"]
    HOSTED -- "Outside GCP (AWS, GitHub Actions, On-Prem)" --> WIF_CHECK{"Supports OIDC / Workload Identity Federation?"}

    WIF_CHECK -- "Yes (GitHub Actions, AWS, Azure)" --> WIF["Use Workload Identity Federation (Keyless OIDC)"]
    WIF_CHECK -- "No (Legacy On-Prem System)" --> SA_KEY["Use Service Account JSON Key (Vault-encrypted, 90-day rotation)"]

    style GCLOUD_LOGIN fill:#4285F4,color:#fff
    style ADC_LOGIN fill:#34A853,color:#fff
    style GCF_SA fill:#FBBC05,color:#333
    style WIF fill:#34A853,color:#fff
    style SA_KEY fill:#EA4335,color:#fff
```

### ASCII Text Breakdown:
* **CLI User Workflows**: `gcloud auth login` $\rightarrow$ Named Profiles (`~/.config/gcloud/configurations/`).
* **Local SDK Code Execution**: `gcloud auth application-default login` $\rightarrow$ ADC file (`~/.config/gcloud/application_default_credentials.json`).
* **CI/CD Security (GitHub / AWS)**: Workload Identity Federation (WIF) $\rightarrow$ Short-lived STS tokens.
* **Privileged Actions**: Service Account Impersonation (`--impersonate-service-account`) $\rightarrow$ Zero persistent keys.

---

## 2. Project Hierarchy & Isolation Decision Tree

```mermaid
flowchart TD
    START_ENV["Evaluate Project Governance Requirements"] --> COUNT{"How many isolated environments / teams exist?"}

    COUNT -- "Single team, single environment" --> FLAT["Single Project (Baseline)"]
    COUNT -- "Multiple environments (Dev, Staging, Prod)" --> ENV_ISO["Separate Projects per Environment<br/>(e.g., app-dev, app-prod)"]

    ENV_ISO --> DEPT_CHECK{"Multiple business units / strict compliance bounds?"}
    DEPT_CHECK -- "Yes (Finance, Healthcare, Core Engine)" --> FOLDER_STRUCT["Folder-Based Hierarchy:<br/>Org -> Folder (Dept) -> Folder (Env) -> Projects"]
    DEPT_CHECK -- "No (Single App Ecosystem)" --> PROJ_ONLY["Flat Folder Hierarchy:<br/>Org -> Folder (Env) -> Projects"]

    style FOLDER_STRUCT fill:#4285F4,color:#fff
    style ENV_ISO fill:#34A853,color:#fff
```

---

## 3. Organization Policy Enforcement Decision Tree

```mermaid
flowchart TD
    POLICY_START["Evaluate Security & Compliance Constraint"] --> P_TYPE{"Is the constraint binary or restricted value list?"}

    P_TYPE -- "Binary (Enable/Disable Feature)" --> BOOL_POL["Boolean Policy Constraint<br/>(e.g., constraints/compute.disableSerialPortAccess)"]
    P_TYPE -- "Restricted List (Allowed Locations / VPCs)" --> LIST_POL["List Policy Constraint<br/>(e.g., constraints/gcp.resourceLocations: in 'in:us-locations')"]

    BOOL_POL --> SCOPE["Apply Policy at Organization Root with Folder Overrides if exempted"]
    LIST_POL --> SCOPE
```
