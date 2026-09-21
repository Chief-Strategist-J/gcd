# Google Cloud Pub/Sub (`gcloud pubsub`) Production Command Reference

A comprehensive, production-grade CLI reference manual for administering **Google Cloud Pub/Sub**. Every command is organized into logical architectural sections (`# ── SECTION ──`), with explicit breakdowns of valid values, data formats, enums, units, and enterprise trade-offs.

---

## 1. Topic Creation & Policy Configuration

Topics serve as the ingestion channels where publishers submit messages. They support data retention, regional storage compliance, KMS customer-managed encryption, and schema contracts.

```bash
gcloud pubsub topics create enterprise-order-events \
  # ── MESSAGE RETENTION & BACKLOG ──────────────────────────────────────────
  --message-retention-duration=7d \
  # Time string: Duration to retain messages on topic before automatic expiration.
  # Values: "10m" to "31d" (e.g., "1h", "24h", "7d"). Default: No retention on topic.
  # Benefit: Enables new subscriptions to seek back in time to messages published before creation.

  # ── DATA RESIDENCY & STORAGE REGIONS ─────────────────────────────────────
  --message-storage-policy-allowed-regions=us-central1,us-east1 \
  # Comma-separated strings: Specific GCP regions where message data can be stored at rest.
  # Default: Google chooses globally.
  # Compliance: Enforces EU GDPR or US data sovereignty mandates.

  # ── CUSTOMER-MANAGED ENCRYPTION KEYS (CMEK) ──────────────────────────────
  --topic-encryption-kms-key=projects/my-prod-project/locations/us-central1/keyRings/pubsub-ring/cryptoKeys/order-key \
  # Resource URI: Cloud KMS key for envelope encryption of messages at rest.

  # ── DATA CONTRACT & SCHEMA ATTACHMENT ────────────────────────────────────
  --schema=projects/my-prod-project/schemas/order-event-schema \
  # Resource URI / Name: Attached Avro or Protocol Buffer schema.
  --message-encoding=JSON
  # Enum: Format expected for published messages.
  # Values: "JSON" | "BINARY".
```

---

## 2. Subscription Architectures

### 2.1 Pull Subscription (High-Throughput Worker Cluster)

Pull subscriptions allow subscriber workers (GKE pods, Compute Engine, Dataflow) to lease messages on demand with fine-grained concurrency control.

```bash
gcloud pubsub subscriptions create order-events-worker-sub \
  # ── TOPIC BINDING ────────────────────────────────────────────────────────
  --topic=enterprise-order-events \
  # String: Name of the parent Pub/Sub topic.

  # ── ACKNOWLEDGMENT LEASE DEADLINE ────────────────────────────────────────
  --ack-deadline=60 \
  # Integer (seconds): Time consumer has to ACK or NACK a message before redelivery.
  # Values: 10 to 600 (seconds). Default: 10s.
  # Recommendation: Set to 2x-3x average consumer processing time.

  # ── BACKLOG RETENTION & TIME-TRAVEL ──────────────────────────────────────
  --message-retention-duration=7d \
  # Time string: Retention duration for unacknowledged messages.
  # Values: "10m" to "31d". Default: "7d".
  --retain-acked-messages \
  # Flag: Retains messages even AFTER they are acknowledged.
  # Required for: Replaying historical events using the "seek" command.

  # ── ADVANCED RELIABILITY & ORDERING ──────────────────────────────────────
  --enable-message-ordering \
  # Flag: Enforces strict sequential delivery for messages sharing an orderingKey.
  --enable-exactly-once-delivery \
  # Flag: Guarantees no duplicate deliveries while acknowledgment lease is valid.

  # ── DEAD-LETTER TOPIC (DLQ) CONFIGURATION ────────────────────────────────
  --dead-letter-topic=projects/my-prod-project/topics/order-events-dlq \
  # Resource URI: Destination topic for messages exceeding retry limits.
  --max-delivery-attempts=5 \
  # Integer: Number of failed delivery attempts before diverting to DLQ.
  # Values: 5 to 100. Default: 5.

  # ── SERVER-SIDE ATTRIBUTE FILTER EXPRESSION ──────────────────────────────
  --message-filter='attributes.priority = "HIGH" AND attributes.region = "US"'
  # SQL-like expression: Discards non-matching messages server-side at zero egress cost.
```

---

### 2.2 Push Subscription (Cloud Run / Cloud Functions / Webhooks)

Push subscriptions deliver messages via HTTPS POST requests directly to serverless container endpoints with automatic OIDC authentication.

```bash
gcloud pubsub subscriptions create order-events-push-sub \
  --topic=enterprise-order-events \
  --push-endpoint=https://order-processor-service-xyz.a.run.app/handle-event \
  --ack-deadline=60 \
  # ── OIDC AUTHENTICATION HEADERS ──────────────────────────────────────────
  --push-auth-service-account=pubsub-invoker-sa@my-prod-project.iam.gserviceaccount.com \
  # Service Account: Generates signed Google OIDC Bearer token injected in Authorization header.
  --push-auth-token-audience=https://order-processor-service-xyz.a.run.app
  # String: Target audience expected in token (defaults to endpoint URL if omitted).
```

---

### 2.3 BigQuery Direct Subscription (Zero-Code Streaming Ingestion)

BigQuery direct subscriptions stream messages straight into a BigQuery table without writing or running Dataflow pipelines.

```bash
gcloud pubsub subscriptions create order-events-bigquery-sub \
  --topic=enterprise-order-events \
  --bigquery-table=my-prod-project:ecommerce_analytics.orders_raw \
  --use-topic-schema \
  # Flag: Maps message fields directly into table columns using the topic's schema.
  --write-metadata \
  # Flag: Writes pubsub metadata columns (subscription_name, message_id, publish_time, attributes).
  --drop-unknown-fields
  # Flag: Discards fields present in message that do not exist in BigQuery table schema.
```

---

### 2.4 Cloud Storage Direct Subscription (Data Lake Archival)

Batches streaming messages directly into Cloud Storage buckets as raw text or structured Avro records.

```bash
gcloud pubsub subscriptions create order-events-gcs-archive-sub \
  --topic=enterprise-order-events \
  --cloud-storage-bucket=my-prod-data-lake \
  --cloud-storage-file-prefix=events/orders/year=%Y/month=%m/ \
  --cloud-storage-file-suffix=.avro \
  --cloud-storage-max-duration=5m \
  # Time string: Batches messages up to 5 minutes before flushing file to GCS.
  --cloud-storage-max-bytes=100MiB \
  # Byte size: Maximum batch size before flushing file. Values: 1KiB to 10GiB.
  --cloud-storage-output-format-avro \
  # Flag: Writes Avro binary format (alternative: text format).
  --cloud-storage-write-metadata
  # Flag: Includes pubsub message attributes and timestamp in Avro record.
```

---

## 3. Publishing Messages to Topics

### Pattern 1: Publish Plain Message with Custom Attributes
```bash
gcloud pubsub topics publish enterprise-order-events \
  --message='{"orderId": "ORD-1002", "amount": 99.50, "currency": "USD"}' \
  --attribute=priority=HIGH,region=US,eventType=ORDER_CREATED
```

### Pattern 2: Publish with Ordering Key (Strict FIFO Partitioning)
```bash
# Messages with the exact same ordering-key are delivered sequentially in order
gcloud pubsub topics publish enterprise-order-events \
  --message='{"step": 1, "action": "CREATE_ACCOUNT"}' \
  --ordering-key=customer-account-98821

gcloud pubsub topics publish enterprise-order-events \
  --message='{"step": 2, "action": "DEPOSIT_FUNDS"}' \
  --ordering-key=customer-account-98821
```

---

## 4. Consuming & Acknowledging Messages (CLI Inspection)

```bash
# ── PULL MESSAGES SYNCHRONOUSLY ────────────────────────────────────────────

# Pull up to 5 messages and automatically acknowledge them (Testing only!)
gcloud pubsub subscriptions pull order-events-worker-sub \
  --limit=5 \
  --auto-ack

# Pull up to 5 messages WITHOUT auto-acknowledging (Extracts Ack IDs)
gcloud pubsub subscriptions pull order-events-worker-sub \
  --limit=5 \
  --format="table(ackId,message.messageId,message.data.decode(base64),message.attributes)"

# ── EXPLICIT ACKNOWLEDGMENT & LEASE EXTENSION ──────────────────────────────

# Manually acknowledge a message using its Ack ID
gcloud pubsub subscriptions ack order-events-worker-sub \
  --ack-ids="ACK_ID_STRING_FROM_PULL_OUTPUT"

# Extend the acknowledgment deadline for a slow-running batch task
gcloud pubsub subscriptions modify-ack-deadline order-events-worker-sub \
  --ack-ids="ACK_ID_STRING" \
  --ack-deadline=120
```

---

## 5. Schema Management (Apache Avro & Protocol Buffers)

```bash
# ── CREATE AVRO SCHEMA ─────────────────────────────────────────────────────

# Define Avro schema file
cat << 'EOF' > order_schema.avsc
{
  "type": "record",
  "name": "OrderEvent",
  "namespace": "com.company.events",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "status", "type": "string"}
  ]
}
EOF

# Register schema in Google Cloud Pub/Sub
gcloud pubsub schemas create order-event-schema \
  --type=AVRO \
  --definition-file=order_schema.avsc

# Validate message payload against registered schema before publishing
gcloud pubsub schemas validate-message \
  --message='{"orderId": "ORD-101", "amount": 49.99, "status": "APPROVED"}' \
  --message-encoding=JSON \
  --schema-name=order-event-schema
```

---

## 6. Snapshots & Time-Travel Seek Operations

Snapshots capture the acknowledgment state of a subscription at a specific point in time, enabling instantaneous rollbacks of message streams.

```bash
# ── CREATE SNAPSHOT ────────────────────────────────────────────────────────

# Create a snapshot capturing current unacknowledged backlog
gcloud pubsub snapshots create pre-deployment-snapshot \
  --subscription=order-events-worker-sub

# ── TIME-TRAVEL SEEK OPERATIONS ───────────────────────────────────────────

# Roll back subscription to the snapshot state (Re-delivers messages acked after snapshot)
gcloud pubsub subscriptions seek order-events-worker-sub \
  --snapshot=pre-deployment-snapshot

# Seek backwards in time by absolute ISO-8601 timestamp
gcloud pubsub subscriptions seek order-events-worker-sub \
  --time="2026-09-21T10:00:00Z"
```

---

## 7. IAM Permissions & Service Agent Setup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)")
export PUBSUB_SERVICE_AGENT="service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com"

# ── DEAD-LETTER QUEUE (DLQ) SERVICE AGENT BINDINGS ─────────────────────────

# 1. Allow Pub/Sub Service Agent to publish failed messages to the DLQ topic
gcloud pubsub topics add-iam-policy-binding order-events-dlq \
  --member="serviceAccount:${PUBSUB_SERVICE_AGENT}" \
  --role="roles/pubsub.publisher"

# 2. Allow Pub/Sub Service Agent to acknowledge forwarded messages on primary sub
gcloud pubsub subscriptions add-iam-policy-binding order-events-worker-sub \
  --member="serviceAccount:${PUBSUB_SERVICE_AGENT}" \
  --role="roles/pubsub.subscriber"

# ── BIGQUERY DIRECT SUBSCRIPTION SERVICE AGENT BINDING ─────────────────────

# Allow Pub/Sub Service Agent to insert rows into BigQuery dataset
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${PUBSUB_SERVICE_AGENT}" \
  --role="roles/bigquery.dataEditor"

# ── CLOUD STORAGE DIRECT SUBSCRIPTION SERVICE AGENT BINDING ────────────────

# Allow Pub/Sub Service Agent to write batch objects into Cloud Storage bucket
gcloud storage buckets add-iam-policy-binding gs://my-prod-data-lake \
  --member="serviceAccount:${PUBSUB_SERVICE_AGENT}" \
  --role="roles/storage.objectCreator"
```

---

## 8. Comprehensive Troubleshooting & Error Matrix

| Symptom / Error Message | Root Cause Analysis | Corrective Resolution Action |
| :--- | :--- | :--- |
| `Message redelivered repeatedly despite successful processing` | Consumer processing duration exceeds `--ack-deadline`. Pub/Sub reclaims un-acked message and redelivers. | 1. Increase `--ack-deadline` (e.g. from 10s to 60s).<br>2. Enable `--enable-exactly-once-delivery`.<br>3. Use background heartbeating to send `modifyAckDeadline` extensions. |
| `Delivery halted for specific ordering-key` | An earlier message for that ordering key failed or timed out. Message ordering blocks subsequent deliveries. | 1. Ensure subscriber acknowledges or NACKs failed message.<br>2. Bind a **Dead-Letter Topic** with `max-delivery-attempts` to purge poison pill messages. |
| `Permission denied on DeadLetterTopic` | The Google-managed Pub/Sub Service Agent lacks `roles/pubsub.publisher` on the DLQ topic. | Run: `gcloud pubsub topics add-iam-policy-binding DLQ_TOPIC --member="serviceAccount:service-NUM@gcp-sa-pubsub..." --role="roles/pubsub.publisher"`. |
| `BigQuery direct subscription fails with INCOMPATIBLE_SCHEMA` | Published message JSON does not match BigQuery table schema, or new fields were added without `--drop-unknown-fields`. | 1. Add `--drop-unknown-fields` to subscription.<br>2. Align table schema with topic's registered Avro/Protobuf schema. |
| `Push endpoint returns HTTP 403 Forbidden` | Target Cloud Run / Cloud Functions service requires authentication, but push subscription lacks OIDC service account. | Add `--push-auth-service-account=SA_EMAIL` to the push subscription, and grant `roles/run.invoker` to that SA. |
| `Seek fails with "Cannot seek to time before subscription retention"` | Subscription was not configured with `--retain-acked-messages`, or requested timestamp is older than topic/subscription retention. | Enable `--retain-acked-messages` and ensure `--message-retention-duration` encompasses the desired seek window. |
