# GCP Core Configuration & Project Management: Complete Operations & Verification Manual

This document is an exhaustive operational manual for **GCP Core Configuration, Project Lifecycle, Authentication, and Organization Governance**.

Every command snippet includes:
1. **Command to Execute**
2. **Expected Terminal Output (What to read & look for in terminal)**
3. **How to Verify Configuration Correctness & Expected Verification Output**

---

## Table of Contents
1. [Category 1: Identity, Authentication & ADC Management](#category-1-identity-authentication--adc-management)
2. [Category 2: Service Account Lifecycle, Scopes, IAM Roles & Key Management](#category-2-service-account-lifecycle-scopes-iam-roles--key-management)
3. [Category 3: SDK Named Configurations Management](#category-3-sdk-named-configurations-management)
4. [Category 4: Project Lifecycle & Billing Account Linkage](#category-4-project-lifecycle--billing-account-linkage)
5. [Category 5: Resource Hierarchy, IAM Governance & Organization Policies](#category-5-resource-hierarchy-iam-governance--organization-policies)
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

## Category 2: Service Account Lifecycle, Scopes, IAM Roles & Key Management

### 1. Identify Service Account Types & Create Custom Service Accounts

GCP provides three distinct Service Account types:
- **Custom Service Account** (User-Created, recommended for microservices): `custom-app-sa@PROJECT_ID.iam.gserviceaccount.com`
- **Default Compute Engine SA** (Built-in, Editor role): `PROJECT_NUMBER-compute@developer.gserviceaccount.com`
- **Google Cloud APIs SA** (System managed, Editor role): `PROJECT_NUMBER@cloudservices.gserviceaccount.com`

```bash
# 1. Create a Custom Service Account for Microservice Workloads
gcloud iam service-accounts create backend-storage-sa \
    --display-name="Backend Cloud Storage Runner" \
    --description="Isolated identity for backend application reading Cloud Storage" \
    --project=prod-core-api-01-9921

# 2. Assign Granular IAM Role (Modern Security Pattern - Replaces Legacy Scopes)
gcloud projects add-iam-policy-binding prod-core-api-01-9921 \
    --member="serviceAccount:backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com" \
    --role="roles/storage.objectAdmin"
```

#### Expected Terminal Output:
```text
Created service account [backend-storage-sa].
Updated IAM policy for project [prod-core-api-01-9921].
```

#### How to Verify Configuration Correctness:
```bash
gcloud iam service-accounts describe backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com
```

---

### 2. Attach Service Accounts & Access Scopes to Compute Engine Instances

```bash
# Option A: Provision Instance with Custom SA & full cloud-platform scope (Permissions governed by IAM)
gcloud compute instances create microservice-vm1 \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --service-account=backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com \
    --scopes=cloud-platform

# Option B: Provision Instance overriding default SA with legacy Access Scopes (Read-Only Storage)
gcloud compute instances create legacy-app-vm \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --service-account=backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com \
    --scopes=storage-ro,logging-write

# Option C: Disable Service Account on Instance (No GCP API Credentials)
gcloud compute instances create unauthenticated-vm \
    --zone=us-central1-a \
    --no-service-account \
    --no-scopes
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/prod-core-api-01-9921/zones/us-central1-a/instances/microservice-vm1].
```

---

### 3. Grant Service Account User Role (`roles/iam.serviceAccountUser`)

To allow a developer or group to deploy VMs running as a specific Service Account, grant `Service Account User` on the SA resource itself:

```bash
# Grant Service Account User role to developer group on target Service Account
gcloud iam service-accounts add-iam-policy-binding backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com \
    --member="group:developers@company.com" \
    --role="roles/iam.serviceAccountUser"
```

#### Expected Terminal Output:
```text
Updated IAM policy for service account [backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com].
bindings:
- members:
  - group:developers@company.com
  role: roles/iam.serviceAccountUser
```

#### How to Verify Configuration Correctness:
```bash
gcloud iam service-accounts get-iam-policy backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com
```

---

### 4. Service Account Key Management (User-Managed External Keys vs Google-Managed)

> [!WARNING]
> Google-managed keys are automatically rotated every 2 weeks and stored securely in escrow. User-managed keys (JSON/P12 files) should be used as a **last resort** for external non-GCP workloads (Max 10 keys per SA).

```bash
# 1. Create a User-Managed Private Key file (External Key)
gcloud iam service-accounts keys create ./sa-key.json \
    --iam-account=backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com

# 2. List all User-Managed and Google-Managed Keys for Service Account
gcloud iam service-accounts keys list \
    --iam-account=backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com \
    --format="table(name.basename(), keyType, validAfterTime, validBeforeTime)"

# 3. Delete an Compromised or Stale User-Managed Service Account Key
gcloud iam service-accounts keys delete KEY_ID_HASH \
    --iam-account=backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com \
    --quiet
```

#### Expected Terminal Output:
```text
created key [a1b2c3d4e5f67890] of type [json] as [./sa-key.json]
KEY_ID            KEY_TYPE      VALID_AFTER           VALID_BEFORE
a1b2c3d4e5f67890  USER_MANAGED  2026-09-13T10:00:00Z  2036-09-13T10:00:00Z
```

---

### 5. Service Account Impersonation (Keyless Best Practice)

Avoid storing persistent JSON keys by using short-lived Service Account token impersonation:

```bash
# 1. Execute CLI commands impersonating Service Account directly
gcloud compute instances list \
    --impersonate-service-account=backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com \
    --project=prod-core-api-01-9921

# 2. Generate short-lived OAuth2 access token for code SDK testing (Valid for 1 Hour)
gcloud auth print-access-token \
    --impersonate-service-account=backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com
```

#### Expected Terminal Output:
```text
WARNING: This command is using service account impersonation. All API calls will be executed as [backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com].
NAME              ZONE           MACHINE_TYPE   STATUS
microservice-vm1  us-central1-a  e2-medium      RUNNING
```

#### How to Verify Configuration Correctness:
```bash
# Confirm user identity holds Service Account Token Creator role on SA
gcloud iam service-accounts get-iam-policy backend-storage-sa@prod-core-api-01-9921.iam.gserviceaccount.com \
    --flatten="bindings[].members" \
    --filter="bindings.role:roles/iam.serviceAccountTokenCreator"
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

## Category 5: Resource Hierarchy, IAM Governance & Organization Policies

### 1. Grant Role Bindings for All 5 IAM Member Types (User, SA, Group, Workspace Domain, Cloud Identity)

```bash
# 1. Member Type 1: Google Account (Individual User)
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="user:engineer@company.com" \
    --role="roles/viewer"

# 2. Member Type 2: Service Account (Application Identity)
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="serviceAccount:backend-sa@gcd-prod-analytics-8812.iam.gserviceaccount.com" \
    --role="roles/storage.objectAdmin"

# 3. Member Type 3: Google Group (Group Collection)
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="group:devops-lead-team@company.com" \
    --role="roles/editor"

# 4. Member Type 4: Google Workspace Domain (Organization Virtual Group)
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="domain:company.com" \
    --role="roles/browser"

# 5. Member Type 5: Cloud Identity Domain (Identity-Only Management)
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="domain:cloudidentity.company.com" \
    --role="roles/resourcemanager.organizationViewer"
```

#### Expected Terminal Output:
```text
Updated IAM policy for project [gcd-prod-analytics-8812].
bindings:
- members:
  - user:engineer@company.com
  role: roles/viewer
- members:
  - serviceAccount:backend-sa@gcd-prod-analytics-8812.iam.gserviceaccount.com
  role: roles/storage.objectAdmin
- members:
  - group:devops-lead-team@company.com
  role: roles/editor
- members:
  - domain:company.com
  role: roles/browser
```

#### How to Verify Configuration Correctness:
```bash
gcloud projects get-iam-policy gcd-prod-analytics-8812 --format="json"
```

---

### 2. Configure IAM Conditional Role Bindings (Attribute & Time-Based Access)

```bash
# Grant temporary Compute Admin role expiring at a specific timestamp
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="user:contractor@company.com" \
    --role="roles/compute.admin" \
    --condition='title="Temporary Incident Access",description="Expires end of month",expression="request.time < timestamp(\"2026-12-31T23:59:59Z\")"'
```

#### Expected Terminal Output:
```text
Updated IAM policy for project [gcd-prod-analytics-8812].
bindings:
- condition:
    description: Expires end of month
    expression: request.time < timestamp("2026-12-31T23:59:59Z")
    title: Temporary Incident Access
  members:
  - user:contractor@company.com
  role: roles/compute.admin
```

#### How to Verify Configuration Correctness:
```bash
gcloud projects get-iam-policy gcd-prod-analytics-8812 \
    --flatten="bindings[].members" \
    --filter="bindings.condition:*" \
    --format="table(bindings.role, bindings.members, bindings.condition.title, bindings.condition.expression)"
```

---

### 3. Configure IAM Deny Policies & Guardrails (Overrides Allow Policies)

```bash
# 1. Create Deny Policy definition file (deny-policy.yaml)
cat << 'EOF' > deny-policy.yaml
name: organizations/981273918234/denyPolicies/deny-service-account-key-creation
rules:
- denyRule:
    deniedPrincipals:
    - "principalSet://googlegroups.com/external-contractors@company.com"
    deniedPermissions:
    - "iam.serviceAccountKeys.create"
    title: "Deny SA key creation to external contractors"
EOF

# 2. Apply Deny Policy at Organization level
gcloud iam deny-policies create deny-service-account-key-creation \
    --organization=981273918234 \
    --deny-rule-file=deny-policy.yaml
```

#### Expected Terminal Output:
```text
Created deny policy [deny-service-account-key-creation].
```

#### How to Verify Configuration Correctness:
```bash
gcloud iam deny-policies describe deny-service-account-key-creation \
    --organization=981273918234
```

---

### 4. Query Policy Insights & Recommender (Enforcing Least Privilege)

```bash
# 1. List ML-driven IAM Role Recommendations to strip excess permissions
gcloud recommender recommendations list \
    --project=gcd-prod-analytics-8812 \
    --location=global \
    --recommender=google.iam.policy.Recommender \
    --format="table(name, description, stateInfo.state)"

# 2. Query Policy Insights regarding permission usage
gcloud recommender insights list \
    --project=gcd-prod-analytics-8812 \
    --location=global \
    --insight-type=google.iam.policy.Insight
```

#### Expected Terminal Output:
```text
NAME                                                                                               DESCRIPTION                                                                 STATE
projects/123/locations/global/recommenders/google.iam.policy.Recommender/recommendations/rec-1  Replace Editor with Storage Object Viewer for user user:engineer@company.com ACTIVE
```

---

### 5. Organization Policies: Boolean Constraints & Location Restrictions

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

### 6. Provision Service Accounts for Application Workloads

```bash
# Create Service Account for application identity
gcloud iam service-accounts create prod-app-sa \
    --display-name="Production App Service Account" \
    --description="Service account for backend microservice execution" \
    --project=gcd-prod-analytics-8812
```

#### Expected Terminal Output:
```text
Created service account [prod-app-sa].
```

#### How to Verify Configuration Correctness:
```bash
gcloud iam service-accounts describe prod-app-sa@gcd-prod-analytics-8812.iam.gserviceaccount.com
```

---

### 7. Configure Organization Restrictions (Egress Proxy Data Exfiltration Guardrails)

The **Organization Restrictions** feature prevents data exfiltration by injecting specific HTTP request headers at the corporate Egress Proxy for managed devices. GCP inspects incoming API requests and denies access if the target resource belongs to an unauthorized external GCP organization.

#### Egress Proxy Header Format:
Header key: `X-Goog-Allowed-Resources-Authorized-Organizations`  
Header value: Comma-separated list of authorized GCP Organization IDs (e.g., `981273918234,481920491823`)

```bash
# 1. Simulate Egress Proxy Request to Authorized Organization (Allowed Access - 200 OK)
curl -v -H "X-Goog-Allowed-Resources-Authorized-Organizations: 981273918234" \
     -H "Authorization: Bearer $(gcloud auth print-access-token)" \
     "https://storage.googleapis.com/storage/v1/b/my-authorized-company-bucket/o"

# 2. Simulate Egress Proxy Request targeting Unauthorized External Organization (Denied Access - 403 Forbidden)
curl -v -H "X-Goog-Allowed-Resources-Authorized-Organizations: 981273918234" \
     -H "Authorization: Bearer $(gcloud auth print-access-token)" \
     "https://storage.googleapis.com/storage/v1/b/unauthorized-personal-external-bucket/o"
```

#### Expected Terminal Output (Authorized Access):
```text
< HTTP/2 200
{
  "kind": "storage#objects"
}
```

#### Expected Terminal Output (Unauthorized Access Denied by Org Restrictions):
```text
< HTTP/2 403 Forbidden
{
  "error": {
    "code": 403,
    "message": "Request denied by Organization Restrictions policy. Target resource belongs to an unauthorized organization.",
    "status": "PERMISSION_DENIED"
  }
}
```

#### How to Verify Configuration Correctness:
```bash
# Verify Organization Policy restricting allowed GCP resource locations
gcloud resource-manager org-policies describe constraints/gcp.restrictResourceLocations \
    --organization=981273918234
```

---

### 8. IAM Security Best Practices & Audit Operations Manual

#### Practice A: Grant Roles to Google Groups (Role Assignment Pattern)
Instead of binding roles to individual user accounts, bind roles to purpose-built Google Groups (e.g. `net-admins@company.com`, `storage-writers@company.com`).

```bash
# 1. Bind Role to Role-Assignment Google Group
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="group:network-admins-group@company.com" \
    --role="roles/compute.networkAdmin"

# 2. Audit Group Memberships via Cloud Identity API
gcloud identity groups memberships list \
    --group-email="network-admins-group@company.com" \
    --format="table(preferredMemberKey.id, roles[0].name)"
```

#### Practice B: Service Account Naming Conventions & Key Auditing
Always specify descriptive `--display-name` values using standard naming conventions (`ENV-App-Role-SA`), and audit user-managed key inventory:

```bash
# 1. Provision Custom SA with Descriptive Display Name & Purpose
gcloud iam service-accounts create prod-payment-processor-sa \
    --display-name="PROD Payment Processing Microservice SA" \
    --description="Identity for payment gateway microservice reading Cloud SQL & KMS" \
    --project=gcd-prod-analytics-8812

# 2. Audit Service Account Keys (Detect Stale External Keys)
gcloud iam service-accounts keys list \
    --iam-account=prod-payment-processor-sa@gcd-prod-analytics-8812.iam.gserviceaccount.com \
    --format="table(name.basename(), keyType, validAfterTime, validBeforeTime)"
```

#### Practice C: Audit IAM Policy Changes via Cloud Audit Logs
Query Cloud Logging for `SetIamPolicy` API calls to detect unauthorized permission grants across the hierarchy:

```bash
gcloud logging read \
    'protoPayload.serviceName="iam.googleapis.com" OR protoPayload.methodName:"SetIamPolicy"' \
    --project=gcd-prod-analytics-8812 \
    --limit=10 \
    --format="table(timestamp, protoPayload.authenticationInfo.principalEmail, protoPayload.methodName, protoPayload.resourceName)"
```

#### Practice D: Deploy Identity-Aware Proxy (IAP) for Application & SSH Access
Replace network-level VPNs and open firewall ports with Identity-Aware Proxy (IAP) application-level authorization:

```bash
# 1. Enable IAP Access for HTTPS Web Applications (App Engine / GKE Ingress / Compute LB)
gcloud iap web add-iam-policy-binding \
    --resource-type=app-engine \
    --member="group:authorized-finance-users@company.com" \
    --role="roles/iap.httpsResourceAccessor" \
    --project=gcd-prod-analytics-8812

# 2. Grant IAP Tunnel Access for Secure Bastion-less SSH / RDP Access
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="group:sysadmins@company.com" \
    --role="roles/iap.tunnelResourceAccessor"
```

#### How to Verify IAP Configuration:
```bash
gcloud iap web get-iam-policy --resource-type=app-engine --project=gcd-prod-analytics-8812
```

---

### 9. Lab Operations Manual: Service Account User (`roles/iam.serviceAccountUser`), GCS Roles & VM Access Control

This section details all `gcloud` CLI commands required to execute the complete IAM & Service Account User lab end-to-end.

#### Step 1: Manage User Access (Grant & Remove Project Roles)
```bash
# 1. Grant Project Viewer Role to User 2
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="user:username2@domain.com" \
    --role="roles/viewer"

# 2. Inspect IAM Policy for Project
gcloud projects get-iam-policy gcd-prod-analytics-8812

# 3. Remove Project Viewer Role from User 2 (Revoke Project Access)
gcloud projects remove-iam-policy-binding gcd-prod-analytics-8812 \
    --member="user:username2@domain.com" \
    --role="roles/viewer"

# 4. Grant Narrow Storage Object Viewer Role to User 2 (Resource-Specific Access)
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="user:username2@domain.com" \
    --role="roles/storage.objectViewer"
```

#### Step 2: Prepare Storage Bucket & Sample File
```bash
# 1. Create Cloud Storage Bucket
gcloud storage buckets create gs://gcd-lab-bucket-9901 \
    --project=gcd-prod-analytics-8812 \
    --location=US

# 2. Upload Sample File & Rename
gcloud storage cp sample.txt gs://gcd-lab-bucket-9901/sample.txt

# 3. Verify Bucket File Listing (As Storage Object Viewer)
gcloud storage ls gs://gcd-lab-bucket-9901
```

#### Step 3: Provision Service Account & Assign Service Account User Role
```bash
# 1. Create Custom Service Account (read-bucket-objects)
gcloud iam service-accounts create read-bucket-objects \
    --display-name="read-bucket-objects" \
    --project=gcd-prod-analytics-8812

# 2. Grant Storage Object Viewer Role to Service Account
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="serviceAccount:read-bucket-objects@gcd-prod-analytics-8812.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

# 3. Grant Service Account User Role (roles/iam.serviceAccountUser) on the SA Resource
gcloud iam service-accounts add-iam-policy-binding read-bucket-objects@gcd-prod-analytics-8812.iam.gserviceaccount.com \
    --member="domain:altostrat.com" \
    --role="roles/iam.serviceAccountUser"

# 4. Grant Compute Instance Admin Role (roles/compute.instanceAdmin.v1) to User/Domain
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="domain:altostrat.com" \
    --role="roles/compute.instanceAdmin.v1"
```

#### Step 4: Create VM Attached to Service Account
```bash
gcloud compute instances create demoiam \
    --zone=us-central1-a \
    --machine-type=e2-micro \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --service-account=read-bucket-objects@gcd-prod-analytics-8812.iam.gserviceaccount.com \
    --scopes=storage-rw
```

#### Step 5: Test & Validate Service Account Identity Inside VM Terminal (SSH)
```bash
# 1. SSH into the VM instance
gcloud compute ssh demoiam --zone=us-central1-a

# --- INSIDE LINUX VM TERMINAL ---

# 2. Attempt to list VM instances (Fails with 403 / Scope/Permission error: SA lacks compute.instances.list)
gcloud compute instances list

# 3. Download object from bucket (Succeeds because SA has Storage Object Viewer role)
gcloud storage cp gs://gcd-lab-bucket-9901/sample.txt .

# 4. Local rename file
mv sample.txt sample2.txt

# 5. Attempt to upload renamed file back to bucket (Fails with 403 Forbidden: SA lacks Storage Object Creator)
gcloud storage cp sample2.txt gs://gcd-lab-bucket-9901
```

#### Step 6: Dynamically Update Service Account Permissions (Elevate SA Role)
```bash
# 1. Grant Storage Object Creator role to Service Account (From local terminal)
gcloud projects add-iam-policy-binding gcd-prod-analytics-8812 \
    --member="serviceAccount:read-bucket-objects@gcd-prod-analytics-8812.iam.gserviceaccount.com" \
    --role="roles/storage.objectCreator"

# 2. Retry upload command inside VM SSH terminal (Succeeds now)
gcloud storage cp sample2.txt gs://gcd-lab-bucket-9901
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
