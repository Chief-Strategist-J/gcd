# Google Cloud Pub/Sub: Official Reference Links & Resources

This document provides curated links to official Google Cloud Pub/Sub documentation, API references, architecture guides, and production reliability best practices.

---

## 1. Official Documentation Links

| Resource Title | URL Link | Description |
| :--- | :--- | :--- |
| **Pub/Sub Documentation Hub** | [cloud.google.com/pubsub/docs](https://cloud.google.com/pubsub/docs) | Official landing page for Google Cloud Pub/Sub documentation. |
| **Architectural Overview** | [cloud.google.com/pubsub/docs/overview](https://cloud.google.com/pubsub/docs/overview) | Deep-dive guide into the global distributed Pub/Sub data plane. |
| **`gcloud pubsub` Command Reference** | [cloud.google.com/sdk/gcloud/reference/pubsub](https://cloud.google.com/sdk/gcloud/reference/pubsub) | Complete CLI specification for topics, subscriptions, schemas, and snapshots. |
| **Subscriber Types & Ingestion** | [cloud.google.com/pubsub/docs/subscriber](https://cloud.google.com/pubsub/docs/subscriber) | Guide on choosing and configuring Pull, Push, BigQuery, and Storage subscriptions. |
| **Message Ordering Guide** | [cloud.google.com/pubsub/docs/ordering](https://cloud.google.com/pubsub/docs/ordering) | Configuring and managing ordering keys, partitions, and recovery. |
| **Exactly-Once Delivery** | [cloud.google.com/pubsub/docs/exactly-once-delivery](https://cloud.google.com/pubsub/docs/exactly-once-delivery) | Architectural specifications for lease-based deduplication. |
| **Dead-Letter Topics (DLQ)** | [cloud.google.com/pubsub/docs/dead-letter-topics](https://cloud.google.com/pubsub/docs/dead-letter-topics) | Isolating poison pill messages and handling delivery retries. |
| **Schemas (Avro & Protobuf)** | [cloud.google.com/pubsub/docs/schemas](https://cloud.google.com/pubsub/docs/schemas) | Defining and validating Apache Avro and Protocol Buffer data contracts. |
| **Pub/Sub Quotas & Limits** | [cloud.google.com/pubsub/quotas](https://cloud.google.com/pubsub/quotas) | Official throughput quotas, message size limits (10 MB), and rate ceilings. |

---

## 2. Production Best Practices Checklist

1. **Reliability & Poison Pill Isolation**:
   - **Always Bind Dead-Letter Topics (DLQ)**: When using message ordering, a single malformed message will block all subsequent messages for that ordering key. Configure a DLQ with `max-delivery-attempts=5` to isolate poison pills automatically.
   - **Configure Reasonable Ack Deadlines**: Default 10 seconds is often too aggressive for complex tasks. Set `--ack-deadline=60` and leverage client library automatic lease extension (heartbeating).
2. **Cost & Network Optimization**:
   - **Leverage Server-Side Message Filtering**: Filter messages on the Pub/Sub server using `--message-filter`. Discarded messages incur zero network egress cost and zero consumer compute cycles.
   - **Adopt Direct Subscriptions**: Replace custom Cloud Run/Dataflow batch pipelines with **BigQuery Direct Subscriptions** or **Cloud Storage Direct Subscriptions** to lower operational complexity and total cost of ownership.
3. **Security & Identity**:
   - **Enforce OIDC Authentication on Push Subscriptions**: Always specify `--push-auth-service-account` when pushing to Cloud Run or Cloud Functions, and restrict the target service to `--no-allow-unauthenticated`.
   - **Enforce Least Privilege on Service Agents**: Grant the Google-managed Pub/Sub service agent only the specific roles needed on downstream resources (`roles/bigquery.dataEditor`, `roles/storage.objectCreator`, `roles/pubsub.publisher` on DLQ).
4. **Disaster Recovery & Rollback**:
   - **Retain Acknowledged Messages**: Enable `--retain-acked-messages` on mission-critical subscriptions to allow time-travel replay via `gcloud pubsub subscriptions seek` in the event of subscriber code regressions.
   - **Pre-Deployment Snapshots**: Take a snapshot before rolling out major consumer updates (`gcloud pubsub snapshots create`) to enable instantaneous rollback if consumer bugs occur.
