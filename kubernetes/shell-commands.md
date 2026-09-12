# Kubernetes: Complete & Exhaustive `kubectl` Command Line Reference Manual

This document is the **definitive master encyclopedia for `kubectl`**, covering every single operational category: Cluster Context Management, Declarative Manifest & GitOps Workflows, Advanced JSONPath & Go-Templates, Workload Lifecycle Operations, Storage/Secrets/ConfigMaps, Network Exposure & Ingress, Strategic Patching, Low-Level Ephemeral Debugging, Node Maintenance & Drain, RBAC & Security Audit, Raw API Server Calls, and an **Exhaustive Failure Resolution Matrix**.

---

## Table of Contents
1. [Category 1: Cluster Context, Configuration & Multi-Cluster Management (`kubectl config`)](#category-1-cluster-context-configuration--multi-cluster-management-kubectl-config)
2. [Category 2: Declarative Manifest Management, GitOps & Field Schema Inspection](#category-2-declarative-manifest-management-gitops--field-schema-inspection)
3. [Category 3: Advanced Output Formatting, JSONPath & Go-Templates](#category-3-advanced-output-formatting-jsonpath--go-templates)
4. [Category 4: Workload Deployment, Rolling Updates & Canary Lifecycle](#category-4-workload-deployment-rolling-updates--canary-lifecycle)
5. [Category 5: ConfigMaps, Secrets, Certificates & Storage Operations](#category-5-configmaps-secrets-certificates--storage-operations)
6. [Category 6: Networking, Service Exposure, Port-Forwarding & Ingress](#category-6-networking-service-exposure-port-forwarding--ingress)
7. [Category 7: Strategic Patching, Labeling & Annotating](#category-7-strategic-patching-labeling--annotating)
8. [Category 8: Deep Diagnostic, Logging, Ephemeral Containers & Telemetry](#category-8-deep-diagnostic-logging-ephemeral-containers--telemetry)
9. [Category 9: Node Maintenance, Eviction, Taints & Tolerations](#category-9-node-maintenance-eviction-taints--tolerations)
10. [Category 10: RBAC Security, ServiceAccount Impersonation & Security Audit](#category-10-rbac-security-serviceaccount-impersonation--security-audit)
11. [Category 11: Raw API Server Introspection & Low-Level Calls](#category-11-raw-api-server-introspection--low-level-calls)
12. [Category 12: Exhaustive Error Diagnosis & Failure Resolution Matrix](#category-12-exhaustive-error-diagnosis--failure-resolution-matrix)

---

## Category 1: Cluster Context, Configuration & Multi-Cluster Management (`kubectl config`)

### 1. View Active Kubeconfig & Merged Configuration
```bash
# View complete merged kubeconfig file
kubectl config view

# Display raw sensitive credentials and auth tokens
kubectl config view --raw
```

### 2. Multi-Cluster Context Switching
```bash
# List all available cluster contexts
kubectl config get-contexts

# Display current active context
kubectl config current-context

# Switch active context to production cluster
kubectl config use-context prod-gke-us-central1
```

### 3. Change Default Namespace for Active Context
```bash
# Permanently set default namespace to 'production' for current context
kubectl config set-context --current --namespace=production
```

### 4. Merge Multiple Kubeconfig Files
```bash
# Combine config1 and config2 into a single unified kubeconfig
KUBECONFIG=~/.kube/config1:~/.kube/config2 kubectl config view --flatten > ~/.kube/config
```

---

## Category 2: Declarative Manifest Management, GitOps & Field Schema Inspection

### 1. Inspect Field Schema Specifications (`kubectl explain`)
```bash
# Drill down into exact YAML schema fields and requirements
kubectl explain pod.spec.containers.resources.limits

# Recursively list all fields for a resource spec
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

### 5. Declarative Garbage Collection (`--prune`)
```bash
# Apply directory and delete cluster resources whose files were removed from Git
kubectl apply -f ./manifests/ --prune --all --selector=app=my-service
```

---

## Category 3: Advanced Output Formatting, JSONPath & Go-Templates

### 1. Extract Base64-Decoded Secret Values in One Command
```bash
# Decrypt database password secret payload directly in terminal
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

### 5. Sort Resources by Creation Time or CPU Usage
```bash
# Sort pods by restart count descending
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'

# Sort nodes by CPU capacity
kubectl get nodes --sort-by='.status.capacity.cpu'
```

### 6. Set-Based Label & Field Selectors
```bash
# Query running pods matching multiple labels excluding frontend
kubectl get pods -n production -l 'environment in (production, staging),tier!=frontend' --field-selector status.phase=Running
```

---

## Category 4: Workload Deployment, Rolling Updates & Canary Lifecycle

### 1. Imperative Resource Generation (Dry-Run Blueprints)
```bash
# Generate Deployment manifest blueprint
kubectl create deployment web-app --image=nginx:1.25-alpine --replicas=3 --port=8080 --dry-run=client -o yaml > deployment.yaml

# Generate Job blueprint
kubectl create job data-migration --image=python:3.11 --dry-run=client -o yaml > job.yaml

# Generate CronJob blueprint running every hour
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

# 3. Check live rollout progress
kubectl rollout status deployment/web-app -n production

# 4. View complete deployment revision history
kubectl rollout history deployment/web-app -n production

# 5. Rollback to specific revision
kubectl rollout undo deployment/web-app --to-revision=2 -n production

# 6. Restart all pods in deployment (Trigger rolling restart)
kubectl rollout restart deployment/web-app -n production
```

### 3. Dynamic Scaling & Horizontal Pod Autoscaling (HPA)
```bash
# Scale deployment replicas manually
kubectl scale deployment/web-app --replicas=10 -n production

# Configure HPA to autoscale between 3 and 20 pods at 70% CPU threshold
kubectl autoscale deployment/web-app --min=3 --max=20 --cpu-percent=70 -n production
```

---

## Category 5: ConfigMaps, Secrets, Certificates & Storage Operations

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

### 2. Persistent Volume (PV) & PVC Management
```bash
# List all PersistentVolumeClaims across namespaces
kubectl get pvc -A

# Inspect PV binding status and volume capacity
kubectl get pv -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage,RECLAIM:.spec.persistentVolumeReclaimPolicy,STATUS:.status.phase,CLAIM:.spec.claimRef.name
```

---

## Category 6: Networking, Service Exposure, Port-Forwarding & Ingress

### 1. Expose Deployments via Services
```bash
# Expose deployment internally (ClusterIP)
kubectl expose deployment web-app --port=80 --target-port=8080 --name=web-service -n production

# Expose deployment via Cloud LoadBalancer
kubectl expose deployment web-app --type=LoadBalancer --port=443 --target-port=8080 --name=web-lb -n production
```

### 2. Port Forwarding & API Proxy
```bash
# Forward local port 8080 to remote pod port 80
kubectl port-forward pod/web-app-74b89-x8q2z 8080:80 -n production

# Forward local port 5432 to remote service
kubectl port-forward svc/postgres-service 5432:5432 -n production

# Run local HTTP proxy to Kubernetes API server
kubectl proxy --port=8001
```

### 3. Service Endpoint Inspection
```bash
# List active endpoint IP addresses attached to services
kubectl get endpoints -A
```

---

## Category 7: Strategic Patching, Labeling & Annotating

### 1. Strategic Merge Patching
```bash
# Hot-patch deployment environment variable
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

### 2. JSON Patching (`application/json-patch+json`)
```bash
# Remove container resource limit via exact JSON path
kubectl patch deployment web-app -n production --type='json' -p='[
  {"op": "remove", "path": "/spec/template/spec/containers/0/resources/limits"}
]'
```

### 3. Managing Labels & Annotations
```bash
# Add label to pod with overwrite permission
kubectl label pod web-app-74b89-x8q2z tier=frontend environment=production --overwrite -n production

# Remove label from pod (append hyphen)
kubectl label pod web-app-74b89-x8q2z tier- -n production

# Annotate deployment with deployment release note
kubectl annotate deployment web-app description="Production release v2.4.0 approved by Ops" -n production
```

---

## Category 8: Deep Diagnostic, Logging, Ephemeral Containers & Telemetry

### 1. Real-Time Multi-Container Log Tailing
```bash
# Stream logs from previous crashed instance across all containers
kubectl logs -f deployment/web-app -n production --all-containers=true --previous --tail=200 --timestamps
```

### 2. Inject Ephemeral Debug Containers (Packet Sniffing)
```bash
# Inject netshoot diagnostic tools into running Distroless pod
kubectl debug -it pod/web-app-74b89-x8q2z -n production \
    --image=nicolaka/netshoot \
    --target=web \
    -- /bin/bash
```

### 3. SSH-Less Node Root Terminal Debugging
```bash
# Launch a privileged root shell on worker node host filesystem
kubectl debug node/worker-node-01 -it --image=busybox -- chroot /host
```

### 4. Stream Remote Pod Packet Capture to Local Wireshark
```bash
kubectl exec -n production pod/web-app-74b89-x8q2z -c web -- tcpdump -i any -w - port 80 | wireshark -k -i -
```

### 5. Cluster Event Stream & Resource Usage (`top`)
```bash
# Stream cluster events sorted by creation timestamp
kubectl get events -A --sort-by='.metadata.creationTimestamp'

# Display CPU and RAM consumption for nodes and pods
kubectl top nodes
kubectl top pods -A --sort-by=memory
```

---

## Category 9: Node Maintenance, Eviction, Taints & Tolerations

### 1. Cordon & Uncordon Node
```bash
# Prevent new pods from scheduling onto node
kubectl cordon worker-node-01

# Restore scheduling availability
kubectl uncordon worker-node-01
```

### 2. Graceful Node Drain (OS Upgrade / Maintenance)
```bash
# Safely evict pods enforcing PodDisruptionBudgets
kubectl drain worker-node-01 \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --force \
    --grace-period=60
```

### 3. Node Taints & Tolerations
```bash
# Add NoSchedule taint to node
kubectl taint nodes worker-node-01 dedicated=gpu:NoSchedule

# Remove taint from node (append hyphen)
kubectl taint nodes worker-node-01 dedicated=gpu:NoSchedule-
```

---

## Category 10: RBAC Security, ServiceAccount Impersonation & Security Audit

### 1. Test RBAC Permissions via Impersonation (`--as`)
```bash
# Test if specific ServiceAccount can delete secrets in production
kubectl auth can-i delete secrets -n production \
    --as=system:serviceaccount:production:app-runner-sa
```

### 2. Certificate Signing Request (CSR) Approval Workflow
```bash
# List and approve pending certificate signing requests
kubectl get csr
kubectl certificate approve csr-nx92k
```

---

## Category 11: Raw API Server Introspection & Low-Level Calls

```bash
# Query raw API server Prometheus metrics
kubectl get --raw /metrics | grep apiserver_request_duration_seconds

# Query API health endpoints
kubectl get --raw /healthz
kubectl get --raw /livez?verbose
kubectl get --raw /readyz?verbose
kubectl get --raw /openapi/v3
```

---

## Category 12: Exhaustive Error Diagnosis & Failure Resolution Matrix

| Error State / Status | Root Cause | Low-Level Diagnosis Command | Immediate Resolution Command |
| :--- | :--- | :--- | :--- |
| **`CrashLoopBackOff`** | App process crashes on startup (Exit Code 1, 2, 127) | `kubectl logs POD --previous --tail=100` AND `kubectl describe pod POD \| grep -A 10 "Last State"` | Inspect previous logs & exit code. Fix application config, environment variables, or container entrypoint in YAML. |
| **`ImagePullBackOff` / `ErrImagePull`** | Wrong tag, rate limit, or private registry credentials missing | `kubectl describe pod POD \| grep -A 5 Events` | Verify image tag or create imagePullSecret: `kubectl create secret docker-registry reg-cred --docker-server=REG --docker-username=USER --docker-password=PASS -n NS` and add `imagePullSecrets` to PodSpec. |
| **`OOMKilled` (Exit Code 137)** | Container RAM usage exceeded `resources.limits.memory` | `kubectl describe pod POD \| grep -i -E "oom\|exit code 137"` | Increase memory limits in manifest: `resources.limits.memory: "1Gi"` or profile app memory leaks. |
| **`Pending` (Unscheduled Pod)** | Insufficient CPU/RAM on worker nodes or NodeAffinity mismatch | `kubectl describe pod POD \| grep -A 5 Events` | Add worker node capacity, scale down node requests, or fix `nodeSelector`/`tolerations` in PodSpec. |
| **`CreateContainerConfigError`** | Referenced ConfigMap or Secret key missing in cluster | `kubectl describe pod POD \| grep -i "configmap\|secret"` | Create missing Secret/ConfigMap: `kubectl create secret generic SECRET_NAME --from-literal=KEY=VAL -n NS`. |
| **`Evicted`** | Worker node disk (`DiskPressure`) or memory (`MemoryPressure`) threshold breached | `kubectl get nodes -o custom-columns=NAME:.metadata.name,DISKPRESSURE:.status.conditions[?(@.type=="DiskPressure")].status` | Purge failed/evicted pods: `kubectl delete pods --field-selector status.phase=Failed -A` and clean unused node images (`crictl rmi --prune`). |
| **`NodeNotReady`** | `kubelet` agent crashed or container runtime (containerd) unresponsive | `kubectl describe node NODE_NAME \| grep -A 10 Conditions` | SSH to node and restart container runtime & kubelet: `sudo systemctl restart containerd kubelet`. |
| **`PersistentVolumeClaimPending`** | StorageClass missing, PV affinity mismatch, or provisioner offline | `kubectl describe pvc PVC_NAME` | Verify StorageClass: `kubectl get sc`. Ensure dynamic provisioner controller is running. |
| **`CreateContainerError`** | Volume mount path collision or invalid entrypoint binary path | `kubectl describe pod POD \| grep -A 5 Events` | Fix `volumeMounts.mountPath` or correct `command: ["/bin/sh", "-c"]` in PodSpec. |
| **`IngressClassNotSpecified`** | Ingress manifest missing `ingressClassName` field | `kubectl get ingressclass` | Add `spec.ingressClassName: nginx` to Ingress YAML manifest. |
| **`403 Unauthorized (API Server)`** | Kubeconfig authentication token expired or missing RBAC permissions | `kubectl auth can-i get pods --as=USER` | Re-authenticate `gcloud container clusters get-credentials CLUSTER` or bind required Role/ClusterRole. |
| **`Cannot eviction pod: PodDisruptionBudget violation`** | `kubectl drain` blocked because eviction would breach PDB minimum available pods | `kubectl get pdb -A` | Temporarily adjust PDB `minAvailable` or force drain: `kubectl drain NODE --ignore-daemonsets --delete-emptydir-data --force`. |
| **`ServiceAccountNotFound`** | PodSpec specifies a `serviceAccountName` that does not exist in namespace | `kubectl get sa -n NS` | Create ServiceAccount: `kubectl create sa SA_NAME -n NS` or fix spelling in PodSpec. |
| **`TaintTolerationMismatch`** | Pod lacks matching toleration for node taint | `kubectl describe node NODE \| grep Taints` | Add matching `tolerations` array to PodSpec in YAML. |
| **`KubeletReadOnlyPortForbidden`** | Port 10255 disabled for security | `curl -k https://NODE_IP:10250/metrics` | Use authenticated port 10250 with bearer token authentication for node metrics scraper. |
