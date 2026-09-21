# Cloud Run Functions: Case Studies & Hands-On Engineering Labs

Welcome to the **Cloud Run Functions Case Studies & Hands-On Labs** directory. This section provides detailed implementation blueprints, lab runbooks, architectural analyses, and verification workflows for real-world serverless workloads on Google Cloud.

---

## Lab & Case Study Index

| Lab / Case Study | Documentation File | Description & Architecture Focus |
| :--- | :--- | :--- |
| **Lab 1: HTTP & Cloud Storage Event Functions with Revisions** | [`http-and-cloud-storage-event-functions-lab.md`](./http-and-cloud-storage-event-functions-lab.md) | Full hands-on lab covering Functions Framework HTTP endpoints, Cloud Storage event triggers via Eventarc, unit testing with Mocha/Sinon, and immutable revision traffic splitting. |

---

## Architectural Highlights

1. **Functions Framework Integration**:
   - Building lightweight, portable serverless handlers with `@google-cloud/functions-framework`.
   - `functions.http()` for RESTful webhooks and APIs with IAM OIDC Bearer authentication.
   - `functions.cloudEvent()` for asynchronous, event-driven integrations responding to Google Cloud infrastructure events.
2. **Eventarc Routing & Service Agents**:
   - Decoupling event producers (Cloud Storage) from event consumers (Cloud Run functions) using Google Cloud Eventarc.
   - Securing service identities with `roles/pubsub.publisher`, `roles/eventarc.eventReceiver`, and `roles/eventarc.serviceAgent`.
3. **Automated Unit Testing & Mocking**:
   - Isolating serverless handlers from cloud infrastructure using `@google-cloud/functions-framework/testing`.
   - Mocking Express `req` and `res` objects with `sinon.stub()`.
4. **Immutable Revision Lifecycle**:
   - Utilizing Cloud Run's native container revision mechanics to perform canary deployments, 50/50 A/B testing, and instant zero-downtime rollbacks.
