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
| **GKE GPUs Overview** | [cloud.google.com/kubernetes-engine/docs/concepts/gpus](https://cloud.google.com/kubernetes-engine/docs/concepts/gpus) | Architecture guide for deploying and scheduling GPUs (NVIDIA L4, A100, H100) on GKE. |
| **Serve Gemma with TGI on GKE** | [cloud.google.com/kubernetes-engine/docs/tutorials/serve-gemma-gpu-tgi](https://cloud.google.com/kubernetes-engine/docs/tutorials/serve-gemma-gpu-tgi) | Official Google Cloud tutorial on serving Gemma models using Hugging Face TGI on NVIDIA L4 GPUs. |
| **GKE Autopilot Workload Separation & GPUs** | [cloud.google.com/kubernetes-engine/docs/how-to/autopilot-gpus](https://cloud.google.com/kubernetes-engine/docs/how-to/autopilot-gpus) | Auto-provisioning GPU hardware, driver installation, and node autoscaling in Autopilot. |
| **Hugging Face Text Generation Inference** | [huggingface.co/docs/text-generation-inference](https://huggingface.co/docs/text-generation-inference) | Official documentation for TGI deployment, model sharding, tensor parallelism, and Prometheus metrics. |
| **Google Cloud Managed Prometheus (GMP)** | [cloud.google.com/stackdriver/docs/managed-prometheus](https://cloud.google.com/stackdriver/docs/managed-prometheus) | Deploying PodMonitoring resources to scrape GKE container metrics into Cloud Monitoring. |
| **GKE AI/ML GitHub Repository** | [github.com/gke-demos/serving-gemma-2b](https://github.com/gke-demos/serving-gemma-2b) | Reference repository containing TGI Gemma 2B manifests, Gradio web UI, and monitoring specs. |

---

## 2. Recommended Best Practices

### General Kubernetes & Workload Practices
1. **Resource Limits**: Always define explicit `requests` and `limits` for CPU and Memory in every `PodSpec` to prevent `OOMKilled` cascades.
2. **Probes**: Implement `readinessProbe` and `livenessProbe` for all production web services to ensure traffic is only routed to healthy pods.
3. **Security**: Enforce **Pod Security Standards (Restricted)**, run containers as non-root users (`runAsNonRoot: true`), and set `readOnlyRootFilesystem: true`.

### Generative AI & GPU-Accelerated LLM Serving (NVIDIA L4, TGI, Gemma)
4. **Equal GPU Requests & Limits**: Kubernetes does not support fractional GPU oversubscription without Multi-Instance GPU (MIG) or Time-Slicing. In your Pod spec, GPU requests must exactly equal limits:
   ```yaml
   resources:
     limits:
       nvidia.com/gpu: "1"
     requests:
       nvidia.com/gpu: "1"
   ```
5. **Shared Memory (`/dev/shm`) Volume Mounting**: PyTorch and Hugging Face TGI utilize POSIX shared memory for tensor operations. By default, Docker/Kubernetes allocates only 64MB to `/dev/shm`, which causes bus errors under high batch concurrency. Mount an `emptyDir` backed by RAM (`medium: Memory`) at `/dev/shm`:
   ```yaml
   volumeMounts:
     - mountPath: /dev/shm
       name: dshm
   volumes:
     - name: dshm
       emptyDir:
         medium: Memory
   ```
6. **Autopilot Driver Auto-Injection**: In GKE Autopilot clusters, requesting an NVIDIA accelerator (`nvidia.com/gpu: 1` or `cloud.google.com/gke-accelerator: nvidia-l4`) automatically triggers GKE to provision the GPU node and inject the appropriate NVIDIA drivers without manual DaemonSet installation.
7. **Model Sharding & Tensor Parallelism**:
   - For single-GPU models with <24 GB VRAM requirements (such as **Gemma 2B**, **Gemma 7B**, **Falcon 7B**), a single **NVIDIA L4 (24GB)** provides optimal cost-to-performance.
   - For larger models exceeding 24GB VRAM (e.g., **Falcon 40B**, **Llama-3-70B**), configure multi-GPU sharding by allocating multiple GPUs (`nvidia.com/gpu: "2"` or `"4"`) and setting `NUM_SHARD: "2"` in the TGI container environment to enable tensor parallelism.
8. **Hugging Face Token Protection**: Never hardcode Hugging Face API tokens in container images or plain-text YAML manifests. Provision a Kubernetes Secret (`kubectl create secret generic hf-secret --from-literal=hf_api_token=...`) and inject it securely via `valueFrom.secretKeyRef`.
9. **Observability via Managed Service for Prometheus**: TGI exposes standard Prometheus metrics (token latency, batch size, throughput, time-to-first-token) on its default port. Deploy a `PodMonitoring` custom resource to scrape metrics every 30 seconds into Google Cloud Monitoring without managing Prometheus servers.
