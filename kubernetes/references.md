# Kubernetes: Official Reference Links & Resources

This document provides official Kubernetes reference links, best practices guides, and API documentation for `kubectl`.

---

## 1. Official Documentation Links

| Resource Title | URL Link | Description |
| :--- | :--- | :--- |
| **Kubernetes Docs** | [kubernetes.io/docs](https://kubernetes.io/docs/) | Official landing page for Kubernetes documentation. |
| **kubectl Cheat Sheet** | [kubernetes.io/docs/reference/kubectl/cheatsheet/](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) | Complete command reference and syntax examples. |
| **Pod Lifecycle Guide** | [kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) | Detailed state transitions and probe specifications. |
| **Kubernetes Services Guide** | [kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/) | Networking models (ClusterIP, NodePort, LoadBalancer). |
| **Troubleshooting Applications** | [kubernetes.io/docs/tasks/debug/debug-application/](https://kubernetes.io/docs/tasks/debug/debug-application/) | Debugging guide for CrashLoopBackOff, Pending, and network issues. |

---

## 2. Recommended Best Practices

1. **Resource Limits**: Always define explicit `requests` and `limits` for CPU and Memory in every `PodSpec` to prevent `OOMKilled` cascades.
2. **Probes**: Implement `readinessProbe` and `livenessProbe` for all production web services to ensure traffic is only routed to healthy pods.
3. **Security**: Enforce **Pod Security Standards (Restricted)**, run containers as non-root users (`runAsNonRoot: true`), and set `readOnlyRootFilesystem: true`.
