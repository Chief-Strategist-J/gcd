# Kubernetes & Container Lifecycle: Complete Command, Telemetry & Operations Manual

This document is an exhaustive, production-grade manual covering: **Container Build Configurations** $\rightarrow$ **Start, Stop, Restart & Destroy Lifecycle (Images, Pods, Nodes)** $\rightarrow$ **Log Inspection (stdout/stderr, System Logs, Previous Crashes)** $\rightarrow$ **Distributed Tracing (OpenTelemetry, Jaeger, Context Headers)** $\rightarrow$ **Configuration Inspection & Secret Decryption** $\rightarrow$ **Networking & DNS Diagnostics** $\rightarrow$ **Image Memory & Console Output Reading**.

---

## Table of Contents
1. [Section 1: Container Image Building, Optimization & Memory Management](#section-1-container-image-building-optimization--memory-management)
2. [Section 2: Complete Start, Stop, Restart & Destroy Lifecycle (Containers, Pods, Nodes)](#section-2-complete-start-stop-restart--destroy-lifecycle-containers-pods-nodes)
3. [Section 3: Comprehensive Log Inspection & Console Output Reading](#section-3-comprehensive-log-inspection--console-output-reading)
4. [Section 4: Distributed Tracing & Observability (OpenTelemetry, Jaeger, Context Headers)](#section-4-distributed-tracing--observability-opentelemetry-jaeger-context-headers)
5. [Section 5: Reading Configurations, ConfigMaps & Decrypting Secrets](#section-5-reading-configurations-configmaps--decrypting-secrets)
6. [Section 6: Deep Kubernetes Networking & DNS Diagnostics](#section-6-deep-kubernetes-networking--dns-diagnostics)
7. [Section 7: Professional Error Diagnosis & Failure Resolution Matrix](#section-7-professional-failure-diagnosis--resolution-matrix)

---

## Section 1: Container Image Building, Optimization & Memory Management

### 1. Multi-Stage Dockerfile Build Configurations (`docker` / `podman` / `buildah`)
```bash
# Enable Docker BuildKit for parallel stage execution and caching
DOCKER_BUILDKIT=1 docker build \
    --platform linux/amd64 \
    --build-arg APP_VERSION=2.4.0 \
    --build-arg BUILD_ENV=production \
    --target production-stage \
    -t us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0 \
    -f ./Dockerfile .
```

### 2. Image Memory Inspection & Container Resource Monitoring
```bash
# 1. Real-time CPU and Memory consumption across Docker/Podman containers
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}"

# 2. Low-level container runtime memory stats (crictl for containerd)
crictl stats

# 3. Sort Kubernetes pods by active Memory (RAM) consumption
kubectl top pods -A --sort-by=memory

# 4. Display worker node memory capacity vs allocatable vs usage
kubectl top nodes
```

---

## Section 2: Complete Start, Stop, Restart & Destroy Lifecycle (Containers, Pods, Nodes)

### 1. Container & Image Level (`docker` / `podman` / `crictl`)

```bash
# START: Start an existing stopped container
docker start web-container
crictl start CONTAINER_ID

# STOP: Gracefully stop container (SIGTERM followed by SIGKILL)
docker stop --time=30 web-container
crictl stop CONTAINER_ID

# RESTART: Restart running container
docker restart web-container

# DESTROY CONTAINER: Force remove container
docker rm -f web-container
crictl rm CONTAINER_ID

# DESTROY IMAGE: Remove container image from local storage
docker rmi us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0
crictl rmi IMAGE_ID
```

### 2. Kubernetes Pod & Deployment Level (`kubectl`)

```bash
# START / DEPLOY: Apply deployment manifest or scale up replicas
kubectl apply -f ./deployment.yaml -n production
kubectl scale deployment/web-app --replicas=3 -n production

# STOP / PAUSE: Scale deployment replicas to 0 (Pause all pods without deleting spec)
kubectl scale deployment/web-app --replicas=0 -n production

# RESTART: Trigger zero-downtime rolling restart of all pods in deployment
kubectl rollout restart deployment/web-app -n production

# DESTROY POD / DEPLOYMENT: Delete specific pod or entire deployment
kubectl delete pod web-app-74b89-x8q2z -n production
kubectl delete deployment web-app -n production
kubectl delete -f ./deployment.yaml -n production
```

### 3. Kubernetes Worker Node Level

```bash
# STOP / PAUSE NODE: Cordon and gracefully drain pods for maintenance
kubectl cordon worker-node-01
kubectl drain worker-node-01 --ignore-daemonsets --delete-emptydir-data --force --grace-period=60

# START / RESUME NODE: Uncordon node to restore pod scheduling
kubectl uncordon worker-node-01

# DESTROY NODE: Remove worker node object from Kubernetes cluster
kubectl delete node worker-node-01
```

---

## Section 3: Comprehensive Log Inspection & Console Output Reading

### 1. Tailing & Filtering Pod Console Outputs (stdout / stderr)

```bash
# 1. Stream live logs from pod
kubectl logs -f pod/web-app-74b89-x8q2z -n production

# 2. Separate stdout (Standard Output) from stderr (Standard Error) streams
kubectl logs pod/web-app-74b89-x8q2z -n production 1> stdout.log 2> stderr.log

# 3. Stream logs from a specific container in a multi-container pod
kubectl logs -f pod/web-app-74b89-x8q2z -c nginx-sidecar -n production

# 4. Stream logs from ALL containers in pods matching a label
kubectl logs -l app=web-app --all-containers=true -f --tail=100 -n production

# 5. Read logs from PREVIOUS crashed container instance (Crucial for CrashLoopBackOff)
kubectl logs pod/web-app-74b89-x8q2z --previous -c web -n production

# 6. Time-based log filtering (Logs from last 30 minutes or timestamp)
kubectl logs deployment/web-app --since=30m -n production
kubectl logs deployment/web-app --since-time=2026-09-12T10:00:00Z -n production
```

### 2. Node Level & System Logs (`journalctl` / `/var/log`)

```bash
# 1. Stream live kubelet system agent logs on worker node
journalctl -u kubelet -f --no-pager

# 2. Stream containerd container runtime system logs
journalctl -u containerd -f --no-pager

# 3. Direct log files on node filesystem
tail -f /var/log/pods/production_web-app-*/web/0.log
```

---

## Section 4: Distributed Tracing & Observability (OpenTelemetry, Jaeger, Context Headers)

### 1. Inspecting W3C Trace Context Headers (`traceparent`) in HTTP Logs
```bash
# Stream HTTP logs and filter for W3C traceparent headers (Format: 00-TRACE_ID-SPAN_ID-FLAGS)
kubectl logs -f deployment/web-app -n production | grep -E "traceparent|trace_id"
```
* **Header Format**: `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`
  * `4bf92f3577b34da6a3ce929d0e0e4736`: 128-bit global Trace ID (tracks request end-to-end across microservices).
  * `00f067aa0ba902b7`: 64-bit Parent Span ID.

### 2. Inspecting Service Mesh / Envoy Tracing Sidecars (Istio / Linkerd)
```bash
# Query live Envoy sidecar proxy tracing statistics inside pod
kubectl exec -n production pod/web-app-74b89-x8q2z -c istio-proxy -- curl -s localhost:15000/stats | grep -E "tracing|zipkin|jaeger"
```

### 3. Port-Forwarding to Jaeger / OpenTelemetry UI
```bash
# Forward local port 16686 to Jaeger UI in tracing namespace
kubectl port-forward svc/jaeger-query 16686:16686 -n tracing
# Open browser at: http://localhost:16686
```

---

## Section 5: Reading Configurations, ConfigMaps & Decrypting Secrets

### 1. Reading & Inspecting ConfigMaps
```bash
# 1. View ConfigMap structure in YAML
kubectl get configmap web-app-config -n production -o yaml

# 2. Display formatted key-value pairs inside ConfigMap
kubectl describe configmap web-app-config -n production
```

### 2. Reading & Decrypting Secrets
```bash
# 1. View raw base64-encoded secret keys
kubectl get secret web-app-secrets -n production -o json

# 2. Decrypt a specific secret key (e.g. DB_PASSWORD)
kubectl get secret web-app-secrets -n production -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode; echo

# 3. Decrypt ALL secret key-value pairs in a namespace using jq
kubectl get secret web-app-secrets -n production -o json | jq '.data | map_values(@base64d)'
```

### 3. Reading Running Pod Environment & Active Configuration
```bash
# Display active environment variables passed into a running container
kubectl exec -it pod/web-app-74b89-x8q2z -c web -n production -- printenv
```

---

## Section 6: Deep Kubernetes Networking & DNS Diagnostics

### 1. Inter-Pod DNS Resolution Diagnostics (`nslookup` / `dig`)
```bash
# Run DNS query from inside container to test Kubernetes CoreDNS resolution
kubectl exec -it pod/web-app-74b89-x8q2z -n production -- nslookup postgres-service.production.svc.cluster.local

# Query CoreDNS endpoint IPs directly
kubectl get endpoints kube-dns -n kube-system
```

### 2. Service Endpoints & EndpointSlices Inspection
```bash
# Verify which ready Pod IPs are attached to a Service
kubectl get endpoints web-app-service -n production
kubectl get endpointslices -l kubernetes.io/service-name=web-app-service -n production
```

### 3. Inspecting Worker Node IPVS / `iptables` Routing Rules
```bash
# Inspect IPVS virtual server routing table on worker node
sudo ipvsadm -ln

# Inspect iptables KUBE-SERVICES chain on worker node
sudo iptables-save | grep KUBE-SERVICES
```

---

## Section 7: Professional Failure Diagnosis & Resolution Matrix

| Phase | Error State / Status | Root Cause | Diagnosis Command | Immediate Professional Resolution Command |
| :--- | :--- | :--- | :--- | :--- |
| **Build** | **`trivy CVE Exit Code 1`** | High/Critical vulnerability in base OS image | `trivy image IMAGE` | Upgrade base image in Dockerfile (e.g., `alpine:3.19` or `distroless`) and rebuild. |
| **Push** | **`denied: Permission "artifactregistry..." denied`** | Docker credentials not configured for GCP | `gcloud auth list` | Authenticate docker: `gcloud auth configure-docker us-central1-docker.pkg.dev`. |
| **Deploy** | **`CrashLoopBackOff`** | Container process exits on startup (Exit Code 1, 127) | `kubectl logs POD --previous` | Fix application config, entrypoint, or missing env vars. |
| **Deploy** | **`ImagePullBackOff`** | Wrong tag or private registry secret missing | `kubectl describe pod POD` | Create secret: `kubectl create secret docker-registry reg-cred --docker-username=U --docker-password=P` and set `imagePullSecrets` in YAML. |
| **Runtime**| **`OOMKilled` (Exit Code 137)** | Container RAM exceeded `resources.limits.memory` | `kubectl top pod POD` AND `kubectl describe pod POD \| grep -i oom` | Increase memory limit in manifest: `resources.limits.memory: "1Gi"`. |
| **Deploy** | **`Pending` (Unscheduled)** | Insufficient CPU/RAM on worker nodes | `kubectl describe pod POD \| grep -A 5 Events` | Scale worker node pool or reduce `resources.requests` in YAML. |
| **Config** | **`CreateContainerConfigError`** | Referenced ConfigMap or Secret missing | `kubectl describe pod POD` | Create missing Secret/ConfigMap: `kubectl create secret generic S --from-literal=K=V`. |
| **Cluster**| **`Evicted`** | Worker node disk (`DiskPressure`) or RAM pressure | `kubectl top nodes` | Purge failed pods: `kubectl delete pods --field-selector status.phase=Failed -A`. |
| **Cluster**| **`NodeNotReady`** | `kubelet` agent crashed on worker node | `journalctl -u kubelet -f` | Restart kubelet on node: `sudo systemctl restart kubelet`. |
| **Storage**| **`PersistentVolumeClaimPending`** | StorageClass missing or provisioner offline | `kubectl describe pvc PVC` | Verify StorageClass (`kubectl get sc`) or create matching PV. |
| **Ingress**| **`IngressClassNotSpecified`** | Ingress manifest missing `ingressClassName` | `kubectl get ingressclass` | Add `spec.ingressClassName: nginx` to Ingress manifest. |
| **Network**| **`DNS Lookup Timeout (SERVFAIL)`** | CoreDNS pods overloaded or crashed | `kubectl logs -n kube-system -l k8s-app=kube-dns` | Scale CoreDNS deployment: `kubectl scale deployment/coredns -n kube-system --replicas=3`. |
