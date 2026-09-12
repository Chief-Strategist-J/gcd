# Cloud Storage: Official Reference Links & Resources

This document provides official Google Cloud reference links, developer guides, and API documentation for Cloud Storage.

---

## 1. Official Documentation Links

| Resource Title | URL Link | Description |
| :--- | :--- | :--- |
| **GCP Cloud Storage Docs** | [cloud.google.com/storage/docs](https://cloud.google.com/storage/docs) | Official landing page for Cloud Storage documentation. |
| **gcloud storage CLI Reference** | [cloud.google.com/sdk/gcloud/reference/storage](https://cloud.google.com/sdk/gcloud/reference/storage) | Complete flag and parameter reference for `gcloud storage`. |
| **Storage Classes Guide** | [cloud.google.com/storage/docs/storage-classes](https://cloud.google.com/storage/docs/storage-classes) | Detailed specs for Standard, Nearline, Coldline, Archive. |
| **Object Lifecycle Management** | [cloud.google.com/storage/docs/lifecycle](https://cloud.google.com/storage/docs/lifecycle) | Rules for automated data transition and object deletion. |
| **Signed URLs Developer Guide** | [cloud.google.com/storage/docs/access-control/signed-urls](https://cloud.google.com/storage/docs/access-control/signed-urls) | Time-limited presigned URL authorization mechanism. |
| **Uniform Bucket-Level Access** | [cloud.google.com/storage/docs/uniform-bucket-level-access](https://cloud.google.com/storage/docs/uniform-bucket-level-access) | IAM security best practice guide. |

---

## 2. Recommended Best Practices

1. **Security**: Always enable **Uniform Bucket-Level Access** and **Public Access Prevention** to enforce centralized IAM security.
2. **Cost Management**: Configure **Object Lifecycle Rules** (`--lifecycle-file`) to automatically transition old logs and backups to `NEARLINE` or `COLDLINE`.
3. **Data Protection**: Enable **Bucket Versioning** (`--versioning`) to protect against accidental file overwrites or deletion.
