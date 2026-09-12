# Compute Engine: Official Reference Links & Resources

This document provides official Google Cloud reference links, best practices guides, and API documentation for Compute Engine.

---

## 1. Official Documentation Links

| Resource Title | URL Link | Description |
| :--- | :--- | :--- |
| **GCP Compute Engine Docs** | [cloud.google.com/compute/docs](https://cloud.google.com/compute/docs) | Official landing page for Compute Engine documentation. |
| **gcloud compute CLI Reference** | [cloud.google.com/sdk/gcloud/reference/compute](https://cloud.google.com/sdk/gcloud/reference/compute) | Complete flag and parameter reference for `gcloud compute`. |
| **Machine Types Guide** | [cloud.google.com/compute/docs/machine-types](https://cloud.google.com/compute/docs/machine-types) | Detailed specs for E2, N2, C2, M2 machine series. |
| **IAP TCP Forwarding Guide** | [cloud.google.com/iap/docs/using-tcp-forwarding](https://cloud.google.com/iap/docs/using-tcp-forwarding) | Setup guide for SSH tunneling into private VMs without public IPs. |
| **Spot VMs Developer Guide** | [cloud.google.com/compute/docs/instances/spot](https://cloud.google.com/compute/docs/instances/spot) | Best practices for cost-optimized batch computing on GCP. |
| **Persistent Disk Options** | [cloud.google.com/compute/docs/disks](https://cloud.google.com/compute/docs/disks) | Storage performance comparison (pd-standard, pd-balanced, pd-ssd). |

---

## 2. Recommended Best Practices

1. **Security**: Always restrict public SSH access (Port 22) and use **Identity-Aware Proxy (IAP)** (`--tunnel-through-iap`) for private VMs.
2. **Cost Optimization**: Use **Spot VMs** (`--provisioning-model=SPOT`) for non-critical workloads to save 60%–90%.
3. **Resiliency**: Distribute VM instances across multiple zones in a region to ensure high availability.
