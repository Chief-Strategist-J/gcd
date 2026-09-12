# GCP Core Configuration & Project Management: Complete Operations & Verification Manual

This document is an exhaustive operational manual for **GCP Core Configuration, Project Lifecycle, Authentication, and Organization Governance**.

Every command snippet includes:
1. **Command to Execute**
2. **Expected Terminal Output (What to read & look for in terminal)**
3. **How to Verify Configuration Correctness & Expected Verification Output**

---

## Table of Contents
1. [Category 1: Identity, Authentication & ADC Management](#category-1-identity-authentication--adc-management)
2. [Category 2: Service Account Impersonation & Key Management](#category-2-service-account-impersonation--key-management)
3. [Category 3: SDK Named Configurations Management](#category-3-sdk-named-configurations-management)
4. [Category 4: Project Lifecycle & Billing Account Linkage](#category-4-project-lifecycle--billing-account-linkage)
5. [Category 5: Resource Hierarchy & Organization Policies](#category-5-resource-hierarchy--organization-policies)
6. [Category 6: API Services Management & Quota Inspection](#category-6-api-services-management--quota-inspection)
7. [Category 7: Exhaustive Failure Diagnosis & Resolution Matrix](#category-7-exhaustive-failure-diagnosis--resolution-matrix)

---

## Category 1: Identity, Authentication & ADC Management

### 1. Authenticate User Account via Web Browser OAuth2

```bash
gcloud auth login --launch-browser
```

#### Expected Terminal Output:
```text
Your browser has been opened to visit:

    https://accounts.google.com/o/oauth2/auth?response_type=code&client_id=32555940559.apps.googleusercontent.com&redirect_uri=http%3A%2F%2Flocalhost%3A8085%2F&scope=openid+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform&state=A98f7d6s87f6s87

You are now logged in as [admin@company.com].
Your current project is [None]. You can set your project using:
  $ gcloud config set project PROJECT_ID
```

#### How to Verify Configuration Correctness:
```bash
gcloud auth list
```

#### Expected Verification Output:
```text
  Credentialed Accounts
ACTIVE  ACCOUNT
*       admin@company.com

To set the active account, run:
    $ gcloud config set account `ACCOUNT`
```

---

### 2. Generate Application Default Credentials (ADC) for Code SDKs

```bash
gcloud auth application-default login
```

#### Expected Terminal Output:
```text
Credentials saved to file: [/home/user/.config/gcloud/application_default_credentials.json]

These credentials will be used by any library that requests Application Default Credentials (ADC).
```

#### How to Verify Configuration Correctness:
```bash
# Print active ADC account details from credentials JSON file
gcloud auth application-default print-access-token --format="value(token)" | cut -c 1-20
```

#### Expected Verification Output:
```text
ya29.a0AfB_byC1a9zK...
```

---

## Category 2: Service Account Impersonation & Key Management

### 1. Impersonate Service Account for Execution (Keyless Security)

```bash
gcloud compute instances list \
    --impersonate-service-account=dev-ops-sa@prod-core-api-01-9921.iam.gserviceaccount.com \
    --project=prod-core-api-01-9921
```

#### Expected Terminal Output:
```text
WARNING: This command is using service account impersonation. All API calls will be executed as [dev-ops-sa@prod-core-api-01-9921.iam.gserviceaccount.com].
NAME           ZONE           MACHINE_TYPE   PREEMPTIBLE  STATUS
prod-web-vm1   us-central1-a  n2-standard-4               RUNNING
prod-web-vm2   us-central1-b  n2-standard-4               RUNNING
```

#### How to Verify Configuration Correctness:
```bash
# Verify user identity has the Service Account Token Creator role
gcloud iam service-accounts get-iam-policy dev-ops-sa@prod-core-api-01-9921.iam.gserviceaccount.com \
    --format="yaml(bindings)"
```

#### Expected Verification Output:
```yaml
bindings:
- members:
  - user:admin@company.com
  role: roles/iam.serviceAccountTokenCreator
```

---

## Category 3: SDK Named Configurations Management

### 1. Create and Activate Named Profile for Production Environment

```bash
# 1. Create named configuration profile
gcloud config configurations create prod-environment

# 2. Set default active properties
gcloud config set project prod-core-api-01-9921
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
```

#### Expected Terminal Output:
```text
Created [prod-environment].
Activated [prod-environment].
Updated property [core/project].
Updated property [compute/region].
Updated property [compute/zone].
```

#### How to Verify Configuration Correctness:
```bash
gcloud config configurations list
```

#### Expected Verification Output:
```text
NAME              IS_ACTIVE  ACCOUNT            PROJECT                 COMPUTE_DEFAULT_ZONE  COMPUTE_DEFAULT_REGION
default           False      dev@company.com    dev-app-123             us-east1-b            us-east1
prod-environment  True       admin@company.com  prod-core-api-01-9921   us-central1-a         us-central1
```

---

## Category 4: Project Lifecycle & Billing Account Linkage

### 1. Provision New GCP Project with Organization & Folder Placement

```bash
gcloud projects create gcd-prod-analytics-8812 \
    --name="GCD Production Analytics" \
    --folder=481920491823 \
    --labels=environment=production,team=analytics
```

#### Expected Terminal Output:
```text
Create in progress for [https://cloudresourcemanager.googleapis.com/v1/projects/gcd-prod-analytics-8812].
Waiting for [operations/cp.481920491823] to finish...done.
Enabling service [cloudapis.googleapis.com] on project [gcd-prod-analytics-8812]...
```

#### How to Verify Configuration Correctness:
```bash
gcloud projects describe gcd-prod-analytics-8812 --format="yaml(projectId, name, lifecycleState, parent)"
```

#### Expected Verification Output:
```yaml
lifecycleState: ACTIVE
name: GCD Production Analytics
parent:
  id: '481920491823'
  type: folder
projectId: gcd-prod-analytics-8812
```

---

### 2. Link Billing Account to Project

```bash
gcloud billing projects link gcd-prod-analytics-8812 \
    --billing-account=01A2B3-4C5D6E-7F8G9H
```

#### Expected Terminal Output:
```text
billingAccountName: billingAccounts/01A2B3-4C5D6E-7F8G9H
billingEnabled: true
name: projects/gcd-prod-analytics-8812/billingInfo
projectId: gcd-prod-analytics-8812
```

#### How to Verify Configuration Correctness:
```bash
gcloud billing projects describe gcd-prod-analytics-8812 --format="value(billingEnabled)"
```

#### Expected Verification Output:
```text
True
```

---

## Category 5: Resource Hierarchy & Organization Policies

### 1. List Organization Policies & Enforce Restricted Locations Constraint

```bash
# Enforce serial port access restriction on organization level
gcloud resource-manager org-policies enable-enforce \
    --organization=981273918234 \
    constraints/compute.disableSerialPortAccess
```

#### Expected Terminal Output:
```text
Updated org policy for organization [981273918234].
```

#### How to Verify Configuration Correctness:
```bash
gcloud resource-manager org-policies describe \
    constraints/compute.disableSerialPortAccess \
    --organization=981273918234
```

#### Expected Verification Output:
```yaml
booleanPolicy:
  enforced: true
constraint: constraints/compute.disableSerialPortAccess
```

---

## Category 6: API Services Management & Quota Inspection

### 1. Enable Required GCP APIs for Deployment

```bash
gcloud services enable \
    compute.googleapis.com \
    container.googleapis.com \
    bigquery.googleapis.com \
    storage.googleapis.com \
    --project=gcd-prod-analytics-8812
```

#### Expected Terminal Output:
```text
Operation "operations/acf.p2-gcd-prod-analytics-8812-78192" finished successfully.
```

#### How to Verify Configuration Correctness:
```bash
gcloud services list --enabled --project=gcd-prod-analytics-8812 --filter="NAME:(compute container bigquery storage)"
```

#### Expected Verification Output:
```text
NAME                    TITLE
bigquery.googleapis.com  BigQuery API
compute.googleapis.com   Compute Engine API
container.googleapis.com Kubernetes Engine API
storage.googleapis.com   Cloud Storage API
```

---

## Category 7: Exhaustive Failure Diagnosis & Resolution Matrix

| Error Code / Symptom | Root Cause | Diagnosis Command | Resolution Command |
| :--- | :--- | :--- | :--- |
| **`401 Unauthenticated` / `Reauthentication Required`** | OAuth token expired or revoked. | `gcloud auth list` | `gcloud auth login` |
| **`403 PermissionDenied` (iam.serviceAccounts.actAs)** | User lacks `roles/iam.serviceAccountUser` on target SA. | `gcloud iam service-accounts get-iam-policy SA_EMAIL` | `gcloud iam service-accounts add-iam-policy-binding SA_EMAIL --member="user:EMAIL" --role="roles/iam.serviceAccountUser"` |
| **`403 BillingNotEnabled`** | Project lacks an active linked GCP billing account. | `gcloud billing projects describe PROJECT_ID` | `gcloud billing projects link PROJECT_ID --billing-account=ACCOUNT_ID` |
| **`409 ProjectAlreadyExists`** | Project ID is globally unique and already in use. | `gcloud projects describe PROJECT_ID` | Create project with unique suffix: `gcloud projects create PROJECT_ID-$(date +%s)` |
| **`QuotaExceeded` (CPUs/IPs)** | Project hit compute quota limit in target region. | `gcloud compute project-info describe --project=PROJECT_ID` | Request quota increase via Console or `gcloud alpha quotas requests create` |
