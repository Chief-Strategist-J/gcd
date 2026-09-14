# Kubernetes & Container Lifecycle: Complete Command, Verification & Operations Manual

This document is an exhaustive, production-grade manual for **Kubernetes (`kubectl`) and Container Lifecycle Operations**. Each section provides:
1. **Command to Execute**3. **Verification & Correctness Check (How to confirm if your configuration is correct or broken)**

---

## Table of Contents
1. [Category 1: Container Image Building & Security Verification](#category-1-container-image-building--security-verification)
2. [Category 2: Image & Container Resource Memory Monitoring](#category-2-image--container-resource-memory-monitoring)
3. [Category 3: Complete Start, Stop, Restart & Destroy Lifecycle (Containers, Pods, Nodes)](#category-3-complete-start-stop-restart--destroy-lifecycle-containers-pods-nodes)
4. [Category 4: Cluster Context & Multi-Cluster Management (`kubectl config`)](#category-4-cluster-context--multi-cluster-management-kubectl-config)
5. [Category 5: Configuration Validation, Dry-Runs, Kustomize & Helm Verification](#category-5-configuration-validation-dry-runs-kustomize--helm-verification)
6. [Category 6: Advanced Output Formatting & JSONPath Queries](#category-6-advanced-output-formatting--jsonpath-queries)
7. [Category 7: Workload Deployment, Rolling Updates & Canary Verification](#category-7-workload-deployment-rolling-updates--canary-verification)
8. [Category 8: ConfigMaps, Secrets, Certificates & Secret Decryption](#category-8-configmaps-secrets-certificates--secret-decryption)
9. [Category 9: Persistent Volume (PV) & PVC Storage Verification](#category-9-persistent-volume-pv--pvc-storage-verification)
10. [Category 10: Networking, Service Exposure, DNS & Routing Verification](#category-10-networking-service-exposure-dns--routing-verification)
11. [Category 11: Strategic Patching, Labeling & Annotating](#category-11-strategic-patching-labeling--annotating)
12. [Category 12: Comprehensive Log Inspection & Console Output Tailing](#category-12-comprehensive-log-inspection--console-output-tailing)
13. [Category 13: Distributed Tracing & Observability Verification](#category-13-distributed-tracing--observability-verification)
14. [Category 14: Ephemeral Container Debugging, Node Maintenance & Security Audit](#category-14-ephemeral-container-debugging-node-maintenance--security-audit)
15. [Category 15: Exhaustive Failure Diagnosis & Resolution Matrix](#category-15-exhaustive-failure-diagnosis--resolution-matrix)

---

## Category 1: Container Image Building & Security Verification

### 1. Multi-Stage Container Build (`docker` / `podman`)

```bash
# Build multi-stage container image for targeted architecture
DOCKER_BUILDKIT=1 docker build \
    --platform linux/amd64 \
    --build-arg APP_VERSION=2.4.0 \
    -t us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0 \
    -f ./Dockerfile .
```

#### How to Verify Configuration Correctness:
```bash
# Check built image size and inspect layers
docker images us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0
```

---

### 2. Vulnerability Scan Verification (`trivy`)

```bash
trivy image \
    --severity HIGH,CRITICAL \
    --exit-code 1 \
    us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0
```

#### Expected Output (Passed Security Check):
```text
us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0 (alpine 3.19.1)
=================================================================================
Total: 0 (HIGH: 0, CRITICAL: 0)
```

---

### 3. Production Docker Container Instance Generator (`docker run`)

Generates and runs an isolated container instance with explicit port mapping, environment variables, restart policies, and named volume mounts.

```bash
docker run -d \
    --name web-app-container \
    -p 8080:8080 \
    --env DB_HOST=10.1.0.25 \
    --env DB_PORT=5432 \
    --restart=unless-stopped \
    --memory=512m \
    --cpus=1.0 \
    us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0
```

#### How to Verify Configuration Correctness:
```bash
docker ps --filter "name=web-app-container" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

---

### 4. GCP Artifact Registry Authentication & Container Push Generator (`docker push`)

Authenticates local Docker CLI against Google Cloud Artifact Registry and pushes image artifacts.

```bash
# Step 1: Configure Docker credential helper for GCP Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev --quiet

# Step 2: Push container image to GCP Artifact Registry
docker push us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0
```

#### How to Verify Configuration Correctness:
```bash
gcloud artifacts docker images list us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo
```

---

### 5. Multi-Architecture Container Build & Registry Push Generator (`docker buildx`)

Generates cross-platform container builds (`linux/amd64` and `linux/arm64`) using BuildKit and pushes them directly to Artifact Registry.

```bash
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --build-arg APP_VERSION=2.4.0 \
    -t us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0 \
    --push \
    -f ./Dockerfile .
```

#### How to Verify Multi-Arch Manifest:
```bash
docker buildx imagetools inspect us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0
```

---

## Category 2: Image & Container Resource Memory Monitoring

### 1. Monitor Pod Memory & CPU Usage (`kubectl top`)

```bash
kubectl top pods -n production --sort-by=memory
```

#### How to Verify Memory Configuration Correctness:
```bash
# Compare current usage (142Mi) against configured requests/limits
kubectl get pod web-app-74b89-x8q2z -n production -o jsonpath='{.spec.containers[0].resources}'
```
* **Correctness Analysis**: Active usage (`142Mi`) exceeds `requests` (`128Mi`) and remains safely under `limits` (`512Mi`). Configuration is optimal and not at risk of `OOMKilled`.

---

## Category 3: Complete Start, Stop, Restart & Destroy Lifecycle (Containers, Pods, Nodes)

### 1. Container Level (`docker` / `crictl`)

```bash
# START
docker start web-container
# Expected Output: web-container

# STOP
docker stop --time=30 web-container
# Expected Output: web-container

# RESTART
docker restart web-container
# Expected Output: web-container

# DESTROY CONTAINER
docker rm -f web-container
# Expected Output: web-container
```

---

### 2. Kubernetes Deployment Level (`kubectl`)

```bash
# 1. START / DEPLOY
kubectl apply -f ./deployment.yaml -n production
```

#### How to Verify Deployment Correctness:
```bash
kubectl get deployment web-app -n production
```
* **What to Read**: `READY` must show `3/3` matching `AVAILABLE` `3`. If `0/3`, pods are failing health probes.

---

```bash
# 2. STOP / PAUSE (Scale to 0)
kubectl scale deployment/web-app --replicas=0 -n production
```
#### How to Verify:
```bash
kubectl get deployment web-app -n production
# Output: NAME web-app READY 0/0 AVAILABLE 0
```

---

```bash
# 3. RESTART (Zero-Downtime Rolling Restart)
kubectl rollout restart deployment/web-app -n production
```
#### How to Verify Restart Status:
```bash
kubectl rollout status deployment/web-app -n production
```

---

```bash
# 4. DESTROY DEPLOYMENT
kubectl delete deployment web-app -n production
```

---

### 3. Worker Node Maintenance (`cordon` / `drain` / `uncordon`)

```bash
# 1. CORDON (Stop scheduling new pods)
kubectl cordon worker-node-01
```
#### How to Verify Node Correctness:
```bash
kubectl get node worker-node-01
```

---

```bash
# 2. DRAIN (Evict running pods safely)
kubectl drain worker-node-01 --ignore-daemonsets --delete-emptydir-data --force --grace-period=60
```

---

```bash
# 3. UNCORDON (Restore scheduling availability)
kubectl uncordon worker-node-01
# Expected Output: node/worker-node-01 uncordoned
```

---

## Category 4: Cluster Context & Multi-Cluster Management (`kubectl config`)

```bash
# Switch active context to production
kubectl config use-context prod-gke-us-central1
```

#### How to Verify Active Context Correctness:
```bash
kubectl config current-context
# Expected Output: prod-gke-us-central1
```

---

## Category 5: Configuration Validation, Dry-Runs, Kustomize & Helm Verification

### 1. Server Dry-Run Validation (`--dry-run=server`)

```bash
# Validate YAML syntax and server API schema without persisting change
kubectl apply -f ./deployment.yaml --dry-run=server
```
#### Expected Output (Invalid Config / Broken Schema):
```text
error: error validating "./deployment.yaml": error validating data: ValidationError(Deployment.spec): missing required field "selector"; if you choose to ignore these errors, turn off validation with --validate=false
```

---

### 2. Preview Manifest Differences (`kubectl diff`)

```bash
kubectl diff -f ./deployment.yaml
```
#### Expected Output (Shows Exact GitOps Changes):
```diff
--- /tmp/LIVE-192837/deployment.yaml
+++ /tmp/LOCAL-902183/deployment.yaml
@@ -18,3 +18,3 @@
       containers:
       - name: web
-        image: nginx:1.25-alpine
+        image: nginx:1.26-alpine
```

---

### 3. Imperative Kubernetes YAML Specification Generators (--dry-run=client -o yaml)

Instead of writing YAML manifests by hand, `kubectl` imperative generators create complete, valid resource specifications instantly without applying them to the cluster.

```bash
# 1. Generate Deployment Manifest with Replicas, Port, and Image
kubectl create deployment web-api \
    --image=us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0 \
    --replicas=3 \
    --port=8080 \
    --dry-run=client -o yaml > deployment.yaml

# 2. Generate ClusterIP Service Exposing Deployment
kubectl expose deployment web-api \
    --name=web-api-svc \
    --port=80 \
    --target-port=8080 \
    --type=ClusterIP \
    --dry-run=client -o yaml > service.yaml

# 3. Generate ConfigMap from Literals
kubectl create configmap app-config \
    --from-literal=DB_HOST=10.1.0.25 \
    --from-literal=DB_PORT=5432 \
    --dry-run=client -o yaml > configmap.yaml

# 4. Generate Secret from Literals
kubectl create secret generic db-secret \
    --from-literal=username=admin \
    --from-literal=password=SuperSecretPass123! \
    --dry-run=client -o yaml > secret.yaml

# 5. Generate Ingress Routing Specification
kubectl create ingress web-ingress \
    --rule="app.example.com/*=web-api-svc:80" \
    --dry-run=client -o yaml > ingress.yaml
```

#### How to Verify Generated Specs:
```bash
kubectl apply -f ./deployment.yaml --dry-run=server
```

---

## Category 6: Advanced Output Formatting & JSONPath Queries

### 1. Extract & Decrypt Secret Password in One Command

```bash
kubectl get secret db-credentials -n production -o jsonpath='{.data.password}' | base64 --decode; echo
```

---

### 2. Query Pods Restart Counts

```bash
kubectl get pods -n production -o jsonpath='{range .items[*]}{.metadata.name}{"\tRestarts: "}{.status.containerStatuses[0].restartCount}{"\n"}{end}'
```

---

## Category 7: Workload Deployment, Rolling Updates & Canary Verification

### 1. Rolling Image Update (`kubectl set image`)

```bash
kubectl set image deployment/web-app web=nginx:1.26-alpine -n production --record
```

#### How to Verify Rolling Update Correctness:
```bash
# Check rollout status
kubectl rollout status deployment/web-app -n production
```

---

### 2. Immediate Rollback (`kubectl rollout undo`)

```bash
kubectl rollout undo deployment/web-app -n production
```

---

## Category 8: ConfigMaps, Secrets, Certificates & Secret Decryption

### 1. Create Secret from Literals

```bash
kubectl create secret generic app-secrets \
    --from-literal=DB_PASS='Secret123!' \
    -n production
```

#### How to Verify Secret Creation & Decrypt All Keys:
```bash
kubectl get secret app-secrets -n production -o json | jq '.data | map_values(@base64d)'
```

---

## Category 9: Persistent Volume (PV) & PVC Storage Verification

```bash
kubectl get pvc -n production
```
* **What to Read**: `STATUS` MUST read `Bound`. If `Pending`, storage provisioner failed.

---

## Category 10: Networking, Service Exposure, DNS & Routing Verification

### 1. Expose Service via LoadBalancer

```bash
kubectl expose deployment web-app --type=LoadBalancer --port=80 --target-port=8080 --name=web-service -n production
```

#### How to Verify Service & External IP Assignment:
```bash
kubectl get service web-service -n production
```
* **What to Read**: Verify `EXTERNAL-IP` changes from `<pending>` to valid IP (`35.202.110.42`).

---

### 2. Verify Service Endpoint Attachments

```bash
kubectl get endpoints web-service -n production
```
* **Correctness Analysis**: IP addresses listed in `ENDPOINTS` match active Pod IPs. If `<none>`, readiness probes are failing.

---

### 3. DNS Resolution Diagnostics

```bash
kubectl exec -it pod/web-app-74b89-x8q2z -n production -- nslookup postgres-service.production.svc.cluster.local
```

---

## Category 11: Strategic Patching, Labeling & Annotating

```bash
kubectl patch deployment web-app -n production --type='strategic' -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "web",
            "env": [{"name": "LOG_LEVEL", "value": "DEBUG"}]
          }
        ]
      }
    }
  }
}'
```

---

## Category 12: Comprehensive Log Inspection & Console Output Tailing

### 1. Separate stdout and stderr Streams

```bash
kubectl logs pod/web-app-74b89-x8q2z -n production 1> stdout.log 2> stderr.log
```
* **What to Read**: `stdout.log` contains application access logs; `stderr.log` contains warnings and exception stack traces.

---

### 2. Read Logs from Previous Crashed Container (`--previous`)

```bash
kubectl logs pod/web-app-74b89-x8q2z -c web -n production --previous --tail=50
```

---

## Category 13: Distributed Tracing & Observability Verification

### 1. Tailing W3C Trace Context Headers (`traceparent`)

```bash
kubectl logs -f deployment/web-app -n production | grep -E "traceparent|trace_id"
```

---

## Category 14: Ephemeral Container Debugging, Node Maintenance & Security Audit

### 1. Ephemeral Container Injection (`kubectl debug`)

```bash
kubectl debug -it pod/web-app-74b89-x8q2z -n production \
    --image=nicolaka/netshoot \
    --target=web \
    -- /bin/bash
```

---

### 2. Test RBAC Permissions (`kubectl auth can-i`)

```bash
kubectl auth can-i delete secrets -n production --as=system:serviceaccount:production:app-runner-sa
```
*(If unauthorized: returns `no`)*

---

## Category 15: Exhaustive Failure Diagnosis & Resolution Matrix

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
| **Rollout**| **`Rollout Stalled / Timeout`** | New replica pods fail readiness probes | `kubectl rollout status deploy/APP` | Undo rollout immediately: `kubectl rollout undo deploy/APP`. |
| **RBAC**   | **`403 Forbidden (API Server)`** | Kubeconfig token expired or missing RoleBinding | `kubectl auth can-i get pods` | Re-authenticate `gcloud container clusters get-credentials CLUSTER`. |
| **Drain**  | **`Cannot evict pod: PDB violation`** | `kubectl drain` blocked by PodDisruptionBudget | `kubectl get pdb -A` | Force drain: `kubectl drain NODE --ignore-daemonsets --delete-emptydir-data --force`. |
| **SA**     | **`ServiceAccountNotFound`** | PodSpec specifies non-existent `serviceAccountName` | `kubectl get sa -n NS` | Create ServiceAccount: `kubectl create sa SA_NAME -n NS`. |
| **Taint**  | **`TaintTolerationMismatch`** | Pod lacks matching toleration for node taint | `kubectl describe node NODE \| grep Taints` | Add matching `tolerations` array to PodSpec in YAML. |
| **Kubelet**| **`KubeletReadOnlyPortForbidden`** | Port 10255 disabled for security | `curl -k https://NODE_IP:10250/metrics` | Use authenticated port 10250 with bearer token authentication. |
