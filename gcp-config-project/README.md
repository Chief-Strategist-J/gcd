# GCP Core Configuration & Project Management

Welcome to the **GCP Core Configuration & Project Management** module. This folder contains production-grade architecture, decision matrices, CLI manuals, and reference guides for managing Google Cloud Platform identity, authentication, project hierarchies, configurations, organization policies, and quota limits.

---

## 📂 Module Sitemap

| Document | Description |
| :--- | :--- |
| 🏗️ [**HLD & LLD Architecture Design**](file:///home/btpl-lap-22/live/gcd/gcp-config-project/hld-lld-design.md) | High-Level & Low-Level Design diagrams for GCP Resource Hierarchy, Authentication models (ADC, Service Account Impersonation, Keys), and IAM Policy Inheritance. |
| 🌳 [**Decision Tree Guide**](file:///home/btpl-lap-22/live/gcd/gcp-config-project/decision-tree.md) | Visual decision flowcharts for selecting Authentication Mechanisms, Project Hierarchy structures, and Organization Policy enforcement. |
| 💻 [**CLI Shell Commands & Operations Manual**](file:///home/btpl-lap-22/live/gcd/gcp-config-project/shell-commands.md) | Exhaustive command manual covering `gcloud auth`, `gcloud config`, `gcloud projects`, `gcloud billing`, `gcloud organizations`, and `gcloud quotas` with **Expected Terminal Outputs** and **Verification Checks**. |
| 📚 [**Official References**](file:///home/btpl-lap-22/live/gcd/gcp-config-project/references.md) | Links to official GCP SDK documentation, IAM security best practices, and quota management docs. |

---

## 🎯 Key Operational Capabilities Covered

1. **Authentication & Identity**: User logins, Application Default Credentials (ADC), Service Account Key management, and Service Account Impersonation without key downloading.
2. **Named `gcloud` Configurations**: Switching context between multiple environments (dev, staging, prod), setting default projects, regions, and zones.
3. **Project Lifecycle Management**: Project creation, labeling, deletion, soft-delete restoration, and billing account linking.
4. **Resource Hierarchy & Organization Policies**: Organization folders, boolean and list constraints, and IAM policy bindings.
5. **Quota & API Management**: Enabling GCP APIs, checking project API quota consumption, and requesting quota increases.
6. **Error Diagnosis Matrix**: Exhaustive troubleshooting matrix for `Unauthenticated`, `PermissionDenied`, `QuotaExceeded`, and `BillingDisabled` errors.
