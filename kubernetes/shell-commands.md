# Kubernetes & Container Lifecycle: 15-Category Master Reference Manual

This document is an exhaustive, production-grade reference manual for **Kubernetes (`kubectl`) and Container Lifecycle Operations**. It combines container building, image vulnerability scanning, multi-stage Dockerfiles, resource memory monitoring, start/stop/restart/destroy lifecycle control, cluster context switching, GitOps/Kustomize/Helm workflows, advanced JSONPath queries, secret decryption, network DNS diagnostics, distributed tracing, ephemeral container debugging, node maintenance, raw API introspection, and an **Exhaustive 18-Row Failure Resolution Matrix**.

---

## Master Table of Contents
1. [Category 1: Container Image Building, Optimization & Security Scanning](#category-1-container-image-building-optimization--security-scanning)
2. [Category 2: Image & Container Resource Memory Monitoring](#category-2-image--container-resource-memory-monitoring)
3. [Category 3: Complete Start, Stop, Restart & Destroy Lifecycle (Containers, Pods, Nodes)](#category-3-complete-start-stop-restart--destroy-lifecycle-containers-pods-nodes)
4. [Category 4: Cluster Context, Configuration & Multi-Cluster Management (`kubectl config`)](#category-4-cluster-context-configuration--multi-cluster-management-kubectl-config)
5. [Category 5: Declarative Manifest Management, GitOps, Kustomize & Helm](#category-5-declarative-manifest-management-gitops-kustomize--helm)
6. [Category 6: Advanced Output Formatting, JSONPath & Go-Templates](#category-6-advanced-output-formatting-jsonpath--go-templates)
7. [Category 7: Workload Deployment, Rolling Updates, Canary & Blue-Green Releases](#category-7-workload-deployment-rolling-updates-canary--blue-green-releases)
8. [Category 8: ConfigMaps, Secrets, Certificates & Decrypting Configurations](#category-8-configmaps-secrets-certificates--decrypting-configurations)
9. [Category 9: Persistent Volume (PV), PVC & Storage Operations](#category-9-persistent-volume-pv-pvc--storage-operations)
10. [Category 10: Networking, Service Exposure, DNS & Port-Forwarding](#category-10-networking-service-exposure-dns--port-forwarding)
11. [Category 11: Strategic Patching, Labeling & Annotating](#category-11-strategic-patching-labeling--annotating)
12. [Category 12: Comprehensive Log Inspection & Console Output Tailing](#category-12-comprehensive-log-inspection--console-output-tailing)
13. [Category 13: Distributed Tracing & Observability Reference](#category-13-distributed-tracing--observability-reference)
14. [Category 14: Deep Diagnostic, Ephemeral Debugging, Node Maintenance & RBAC Security](#category-14-deep-diagnostic-ephemeral-debugging-node-maintenance--rbac-security)
15. [Category 15: Exhaustive Failure Diagnosis & Resolution Matrix](#category-15-exhaustive-failure-diagnosis--resolution-matrix)

---

## Category 1: Container Image Building, Optimization & Security Scanning

### 1. Multi-Stage Container Build (`docker` / `podman` / `buildah`)
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

### 2. Container Image Vulnerability Scanning (`trivy`)
```bash
# Scan container image for HIGH and CRITICAL CVE vulnerabilities before pushing
trivy image \
    --severity HIGH,CRITICAL \
    --exit-code 1 \
    us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0
```
* **Note**: Exits with code `1` if critical security vulnerabilities are found, halting CI/CD pipeline deployment.

### 3. Authenticate & Push to Google Artifact Registry / Docker Hub
```bash
# 1. Authenticate Docker with Google Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev --quiet

# 2. Push versioned image to container registry
docker push us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0
```

---

## Category 2: Image & Container Resource Memory Monitoring

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

## Category 3: Complete Start, Stop, Restart & Destroy Lifecycle (Containers, Pods, Nodes)

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

## Category 4: Cluster Context, Configuration & Multi-Cluster Management (`kubectl config`)

```bash
# 1. View complete merged kubeconfig file
kubectl config view
kubectl config view --raw

# 2. List all available cluster contexts & display current context
kubectl config get-contexts
kubectl config current-context

# 3. Switch active context to production cluster
kubectl config use-context prod-gke-us-central1

# 4. Set default namespace for active context
kubectl config set-context --current --namespace=production

# 5. Merge multiple kubeconfig files into a single unified file
KUBECONFIG=~/.kube/config1:~/.kube/config2 kubectl config view --flatten > ~/.kube/config
```

---

## Category 5: Declarative Manifest Management, GitOps, Kustomize & Helm

### 1. Inspect Field Schema Specifications (`kubectl explain`)
```bash
# Drill down into exact YAML schema fields and requirements
kubectl explain pod.spec.containers.resources.limits
kubectl explain deployment.spec.strategy --recursive
```

### 2. Declarative Apply & GitOps Manifest Validation
```bash
# Validate local YAML syntax against API server schema without applying
kubectl apply -f ./deployment.yaml --dry-run=server

# Apply full manifest directory recursively
kubectl apply -f ./manifests/ --recursive -n production
```

### 3. Live Cluster Manifest Diff (`kubectl diff`)
```bash
# Compare local Git repository YAML against live cluster state
kubectl diff -f ./manifests/production/
```

### 4. Kustomize Overlay Build & Pipeline Execution
```bash
# Build and apply Kustomize production overlay directly
kubectl kustomize ./overlays/production | kubectl apply -f -
```

### 5. Configuring & Packaging Helm Charts
```bash
# Lint, template, and package Helm chart
helm lint ./charts/web-app
helm template web-release ./charts/web-app -f ./charts/web-app/values-production.yaml
helm package ./charts/web-app
```

### 6. Declarative Garbage Collection (`--prune`)
```bash
# Apply directory and delete cluster resources whose files were removed from Git
kubectl apply -f ./manifests/ --prune --all --selector=app=my-service
```

---

## Category 6: Advanced Output Formatting, JSONPath & Go-Templates

### 1. Extract Base64-Decoded Secret Values in One Command
```bash
kubectl get secret db-credentials -n production -o jsonpath='{.data.password}' | base64 --decode; echo
```

### 2. Extract Container Images Across All Deployments
```bash
kubectl get deployments -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{range .spec.template.spec.containers[*]}{.image}{" "}{end}{"\n"}{end}'
```

### 3. List All Pods with Restart Counts > 0
```bash
kubectl get pods -A -o jsonpath='{range .items[?(@.status.containerStatuses[0].restartCount>0)]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.status.containerStatuses[0].restartCount}{"\n"}{end}'
```

### 4. Custom Column Formatting for Node Memory & Internal IPs
```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,IP:.status.addresses[?(@.type=="InternalIP")].address,CPU:.status.capacity.cpu,MEMORY:.status.capacity.memory
```

### 5. Set-Based Label & Field Selectors
```bash
kubectl get pods -n production -l 'environment in (production, staging),tier!=frontend' --field-selector status.phase=Running
```

---

## Category 7: Workload Deployment, Rolling Updates, Canary & Blue-Green Releases

### 1. Imperative Blueprint Generation
```bash
kubectl create deployment web-app --image=nginx:1.25-alpine --replicas=3 --port=8080 --dry-run=client -o yaml > deployment.yaml
kubectl create job data-migration --image=python:3.11 --dry-run=client -o yaml > job.yaml
kubectl create cronjob hourly-backup --schedule="0 * * * *" --image=busybox --dry-run=client -o yaml > cronjob.yaml
```

### 2. Zero-Downtime Rolling Update Strategy
```bash
# 1. Update container image version
kubectl set image deployment/web-app web=nginx:1.26-alpine -n production --record

# 2. Configure rolling update strategy parameters (maxSurge=25%, maxUnavailable=0)
kubectl patch deployment web-app -n production --type='strategic' -p '
{
  "spec": {
    "strategy": {
      "rollingUpdate": {
        "maxSurge": "25%",
        "maxUnavailable": 0
      }
    }
  }
}'

# 3. Check live rollout progress & history
kubectl rollout status deployment/web-app -n production
kubectl rollout history deployment/web-app -n production

# 4. Instant rollback to previous revision
kubectl rollout undo deployment/web-app -n production
```

### 3. Blue-Green Deployment Cutover
```bash
# Switch Service selector to green deployment (v2)
kubectl patch service web-app-service -n production -p '{"spec":{"selector":{"version":"v2"}}}'
```

### 4. Horizontal Pod Autoscaler (HPA) & Pod Disruption Budget (PDB)
```bash
# Autoscale between 3 and 20 pods at 70% CPU target
kubectl autoscale deployment/web-app --min=3 --max=20 --cpu-percent=70 -n production

# Create PDB enforcing 80% minimum available pods
kubectl create pdb web-app-pdb --selector=app=web-app --min-available=80% -n production
```

---

## Category 8: ConfigMaps, Secrets, Certificates & Decrypting Configurations

### 1. Create ConfigMaps & Secrets
```bash
# Create ConfigMap from env file
kubectl create configmap app-config --from-env-file=./app.env -n production

# Create Secret from literal values
kubectl create secret generic app-secrets --from-literal=DB_PASS='Secret123!' -n production

# Create TLS Secret from certificate & key files
kubectl create secret tls app-tls-cert --cert=./tls.crt --key=./tls.key -n production

# Create Docker Registry pull secret
kubectl create secret docker-registry reg-cred \
    --docker-server=https://index.docker.io/v1/ \
    --docker-username=myuser \
    --docker-password=mypassword \
    --docker-email=myuser@example.com -n production
```

### 2. Reading ConfigMaps & Decrypting Secrets
```bash
# 1. View ConfigMap YAML
kubectl get configmap app-config -n production -o yaml

# 2. Decrypt single secret key
kubectl get secret app-secrets -n production -o jsonpath='{.data.DB_PASS}' | base64 --decode; echo

# 3. Decrypt ALL secret key-value pairs in a namespace using jq
kubectl get secret app-secrets -n production -o json | jq '.data | map_values(@base64d)'

# 4. Read running container environment variables
kubectl exec -it pod/web-app-74b89-x8q2z -c web -n production -- printenv
```

---

## Category 9: Persistent Volume (PV), PVC & Storage Operations

```bash
# List all PersistentVolumeClaims across namespaces
kubectl get pvc -A

# Inspect PV binding status and volume capacity
kubectl get pv -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage,RECLAIM:.spec.persistentVolumeReclaimPolicy,STATUS:.status.phase,CLAIM:.spec.claimRef.name

# Verify StorageClasses available in cluster
kubectl get storageclass
```

---

## Category 10: Networking, Service Exposure, DNS & Port-Forwarding

### 1. Service Exposure & Port Forwarding
```bash
# Expose deployment internally (ClusterIP)
kubectl expose deployment web-app --port=80 --target-port=8080 --name=web-service -n production

# Forward local port 8080 to remote pod port 80
kubectl port-forward pod/web-app-74b89-x8q2z 8080:80 -n production

# Run local HTTP proxy to Kubernetes API server
kubectl proxy --port=8001
```

### 2. Service Endpoints & EndpointSlices Inspection
```bash
# List active endpoint IP addresses attached to services
kubectl get endpoints -A
kubectl get endpointslices -A
```

### 3. Inter-Pod DNS Resolution & Routing Diagnostics
```bash
# Test CoreDNS resolution inside pod
kubectl exec -it pod/web-app-74b89-x8q2z -n production -- nslookup postgres-service.production.svc.cluster.local

# Inspect IPVS routing table on worker node
sudo ipvsadm -ln

# Inspect iptables KUBE-SERVICES chain on worker node
sudo iptables-save | grep KUBE-SERVICES
```

---

## Category 11: Strategic Patching, Labeling & Annotating

```bash
# 1. Strategic Merge Patch deployment env variable
kubectl patch deployment web-app -n production --type='strategic' -p '{"spec":{"template":{"spec":{"containers":[{"name":"web","env":[{"name":"LOG_LEVEL","value":"DEBUG"}]}]}}}}'

# 2. JSON Patch array element removal
kubectl patch deployment web-app -n production --type='json' -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/resources/limits"}]'

# 3. Labeling and Annotating
kubectl label pod web-app-74b89-x8q2z tier=frontend environment=production --overwrite -n production
kubectl annotate deployment web-app description="Production release approved by Ops" -n production
```

---

## Category 12: Comprehensive Log Inspection & Console Output Tailing

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
# Stream live kubelet system agent logs on worker node
journalctl -u kubelet -f --no-pager

# Stream containerd container runtime system logs
journalctl -u containerd -f --no-pager

# Direct log files on node filesystem
tail -f /var/log/pods/production_web-app-*/web/0.log
```

---

## Category 13: Distributed Tracing & Observability Reference

### 1. Inspecting W3C Trace Context Headers (`traceparent`) in HTTP Logs
```bash
# Stream HTTP logs and filter for W3C traceparent headers (Format: 00-TRACE_ID-SPAN_ID-FLAGS)
kubectl logs -f deployment/web-app -n production | grep -E "traceparent|trace_id"
```
* **Header Format**: `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`
  * `4bf92f3577b34da6a3ce929d0e0e4736`: 128-bit global Trace ID.
  * `00f067aa0ba902b7`: 64-bit Parent Span ID.

### 2. Inspecting Service Mesh / Envoy Tracing Sidecars & Port Forwarding
```bash
# Query live Envoy sidecar proxy tracing statistics inside pod
kubectl exec -n production pod/web-app-74b89-x8q2z -c istio-proxy -- curl -s localhost:15000/stats | grep -E "tracing|zipkin|jaeger"

# Port-forward to Jaeger UI in tracing namespace
kubectl port-forward svc/jaeger-query 16686:16686 -n tracing
```

---

## Category 14: Deep Diagnostic, Ephemeral Debugging, Node Maintenance & RBAC Security

### 1. Inject Ephemeral Debug Containers & Node Debugging
```bash
# Inject netshoot diagnostic tools into running Distroless pod
kubectl debug -it pod/web-app-74b89-x8q2z -n production --image=nicolaka/netshoot --target=web -- /bin/bash

# SSH-less root node debugging
kubectl debug node/worker-node-01 -it --image=busybox -- chroot /host

# Live Wireshark packet capture streaming
kubectl exec -n production pod/web-app-74b89-x8q2z -c web -- tcpdump -i any -w - port 80 | wireshark -k -i -
```

### 2. Node Maintenance & Taints
```bash
# Cordon and drain node safely
kubectl cordon worker-node-01
kubectl drain worker-node-01 --ignore-daemonsets --delete-emptydir-data --force --grace-period=60
kubectl uncordon worker-node-01

# Add/remove node taints
kubectl taint nodes worker-node-01 dedicated=gpu:NoSchedule
kubectl taint nodes worker-node-01 dedicated=gpu:NoSchedule-
```

### 3. RBAC Impersonation & Raw API Server Introspection
```bash
# Test permissions via ServiceAccount impersonation
kubectl auth can-i delete secrets -n production --as=system:serviceaccount:production:app-runner-sa

# Query raw API server metrics and health endpoints
kubectl get --raw /metrics | grep apiserver_request_duration_seconds
kubectl get --raw /healthz
kubectl get --raw /livez?verbose
kubectl get --raw /readyz?verbose
kubectl get --raw /openapi/v3
```

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
