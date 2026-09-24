# Official Google Cloud Firestore References & Best Practices

This document contains curated official documentation links, technical references, and architectural guidance for **Google Cloud Firestore**.

---

## 1. Official Documentation & Guides

* **[Google Cloud Firestore Documentation Overview](https://cloud.google.com/firestore/docs)**
  Official landing page for Google Cloud Firestore product features, documentation, and releases.
* **[Firebase Firestore Documentation](https://firebase.google.com/docs/firestore)**
  Comprehensive client SDK documentation for Web, iOS, Android, Unity, and Flutter.
* **[Choosing Between Native and Datastore Mode](https://cloud.google.com/firestore/docs/chosen-firestore-mode)**
  Official architectural comparison guide detailing feature differences between Native Mode and Datastore Mode.
* **[Firestore Locations & Multi-Region Support](https://cloud.google.com/firestore/docs/locations)**
  Complete list of available regional and multi-region locations, availability SLAs, and pricing boundaries.
* **[Firestore Triggers for Cloud Run Functions](https://cloud.google.com/functions/docs/calling/cloud-firestore)**
  Detailed guide on triggering Cloud Run functions (2nd gen) from Firestore Native Mode document mutation events via Eventarc.

---

## 2. Security & Access Control References

* **[Firebase Security Rules Language Reference](https://firebase.google.com/docs/firestore/security/rules-structure)**
  Detailed reference for constructing security rules, handling user authorization, and evaluating request payload schema.
* **[Firestore IAM Roles & Permissions](https://cloud.google.com/firestore/docs/iam)**
  Google Cloud IAM roles (`roles/datastore.user`, `roles/datastore.viewer`, `roles/datastore.owner`, `roles/datastore.importExportAdmin`) for server environments.

---

## 3. Data Operations, Indexing & Quotas

* **[Firestore Quotas & Limits](https://cloud.google.com/firestore/docs/quotas)**
  Authoritative reference for document size limits (max 1 MB), write rate limits, query limits, and transaction execution constraints.
* **[Index Types & Composite Indexing](https://cloud.google.com/firestore/docs/concepts/index-overview)**
  In-depth guide covering single-field indexes, composite index definitions, and automated indexing configurations.
* **[Managed Export and Import Guide](https://cloud.google.com/firestore/docs/manage-data)**
  Instructions for setting up automated exports to Cloud Storage buckets for disaster recovery, analytics, or database cloning.
* **[Point-in-Time Recovery (PITR) & Backups](https://cloud.google.com/firestore/docs/pitr)**
  Documentation on enabling continuous backup snapshots and point-in-time recovery for audit compliance and operational protection.

---

## 4. Architectural Best Practices

1. **Avoid Sequential Document IDs in High Write Throughput Workloads**:
   Sequential keys (e.g. `doc_0001`, `doc_0002`, or timestamps as IDs) create hot spots on single Paxos storage split keys. Use auto-generated UUIDs or hash prefixes for uniform data distribution.
2. **Utilize Counter Sharding for High-Frequency Updates**:
   A single document can sustain approximately 1 write per second under heavy contention. For counters updated frequently (e.g. view counts, likes), aggregate across multiple sub-counter documents.
3. **Limit Transaction Scope**:
   Keep transactions small and fast to minimize lock contention and avoid `ABORTED` errors under high concurrent traffic.
4. **Use Collection Group Queries for Deep Schemas**:
   When querying subcollections across parent documents, register Collection Group indexes rather than duplicating data into root collections.
5. **Strict Native Mode Requirement for Cloud Functions Triggers**:
   Firestore event triggers via Eventarc strictly require **Firestore in Native mode**; Datastore mode does not emit document mutation events. Ensure trigger document paths omit trailing slashes.
