# Kubernetes & Container Lifecycle: End-to-End Build, Configure & Deploy Manual

This document provides a production-grade, end-to-end reference for the complete application deployment lifecycle: **Phase 1: Building & Scanning Container Images** $\rightarrow$ **Phase 2: Configuring Manifests & Helm/Kustomize** $\rightarrow$ **Phase 3: Deployment & Release Strategies (Rolling, Canary, Blue-Green)** $\rightarrow$ **Phase 4: Auto-Scaling & HA** $\rightarrow$ **Phase 5: Professional Failure Diagnosis & Resolution Matrix**.

---

## Table of Contents
1. [Phase 1: Building, Scanning & Pushing Container Images](#phase-1-building-scanning--pushing-container-images)
2. [Phase 2: Configuring Kubernetes Manifests, Secrets & Helm/Kustomize](#phase-2-configuring-kubernetes-manifests-secrets--helmkustomize)
3. [Phase 3: Deployment & Zero-Downtime Release Strategies](#phase-3-deployment--zero-downtime-release-strategies)
4. [Phase 4: Auto-Scaling (HPA), HA (PDB) & Network Isolation](#phase-4-auto-scaling-hpa-ha-pdb--network-isolation)
5. [Phase 5: Low-Level Diagnostics, Telemetry & Ephemeral Debugging](#phase-5-low-level-diagnostics-telemetry--ephemeral-debugging)
6. [Phase 6: Professional Failure Diagnosis & Resolution Matrix](#phase-6-professional-failure-diagnosis--resolution-matrix)

---

## Phase 1: Building, Scanning & Pushing Container Images

### 1. Multi-Stage Container Build (`docker` / `podman`)
```bash
# Build multi-stage container image for targeted architecture
docker build \
    --platform linux/amd64 \
    --build-arg APP_VERSION=2.4.0 \
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

## Phase 2: Configuring Kubernetes Manifests, Secrets & Helm/Kustomize

### 1. Generating Dry-Run Manifest Blueprints (`kubectl create --dry-run`)
```bash
# Generate clean Deployment YAML blueprint without executing on API server
kubectl create deployment web-app \
    --image=us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0 \
    --replicas=3 \
    --port=8080 \
    --dry-run=client -o yaml > ./base/deployment.yaml
```

### 2. Configuring Secure Secrets & ConfigMaps
```bash
# Create ConfigMap from environment file
kubectl create configmap web-app-config \
    --from-env-file=./configs/production.env \
    --dry-run=client -o yaml > ./base/configmap.yaml

# Create Secret from literal credentials
kubectl create secret generic web-app-secrets \
    --from-literal=DB_PASSWORD='SuperSecretPass123!' \
    --from-literal=API_KEY='live_key_902183' \
    --dry-run=client -o yaml > ./base/secrets.yaml
```

### 3. Configuring Kustomize Multi-Environment Overlays
```bash
# Directory Structure:
# overlays/production/
# ├── kustomization.yaml
# └── patch-replicas.yaml

# Build and validate production Kustomize overlay manifest
kubectl kustomize ./overlays/production
```

### 4. Configuring & Packaging Helm Charts
```bash
# 1. Lint Helm chart for syntax errors
helm lint ./charts/web-app

# 2. Render template locally with custom values.yaml
helm template web-release ./charts/web-app -f ./charts/web-app/values-production.yaml

# 3. Package Helm chart into tar.gz release artifact
helm package ./charts/web-app
```

---

## Phase 3: Deployment & Zero-Downtime Release Strategies

### 1. Declarative Apply & GitOps Deployment
```bash
# Apply full directory of configured manifests
kubectl apply -f ./manifests/production/ --namespace=production

# Helm Upgrade / Install with wait condition
helm upgrade --install web-release ./charts/web-app \
    --namespace=production \
    --values=./charts/web-app/values-production.yaml \
    --wait \
    --timeout=5m
```

### 2. Zero-Downtime Rolling Update Strategy
```bash
# 1. Update container image version
kubectl set image deployment/web-app web=us-central1-docker.pkg.dev/YOUR_PROJECT/app-repo/web-app:v2.4.0 -n production --record

# 2. Configure zero-downtime rolling update parameters (maxSurge=25%, maxUnavailable=0)
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

# 3. Monitor live rolling update status
kubectl rollout status deployment/web-app -n production
```

### 3. Blue-Green Deployment Cutover
```bash
# Deploy green version deployment (v2), then switch Service selector to green
kubectl patch service web-app-service -n production -p '{"spec":{"selector":{"version":"v2"}}}'
```

### 4. Emergency Instant Rollback Commands
```bash
# Rollback Kubernetes deployment to previous working revision
kubectl rollout undo deployment/web-app -n production

# Rollback Helm release to revision 2
helm rollback web-release 2 --namespace=production
```

---

## Phase 4: Auto-Scaling (HPA), HA (PDB) & Network Isolation

### 1. Configure Horizontal Pod Autoscaler (HPA)
```bash
# Auto-scale pods between 3 and 20 replicas based on 70% target CPU utilization
kubectl autoscale deployment web-app \
    --min=3 \
    --max=20 \
    --cpu-percent=70 \
    --namespace=production
```

### 2. Configure Pod Disruption Budget (PDB) for High Availability
```bash
# Ensure at least 80% of pods remain online during voluntary node maintenance
kubectl create pdb web-app-pdb \
    --selector=app=web-app \
    --min-available=80% \
    --namespace=production
```

### 3. Configure NetworkPolicy for Microservice Isolation
```bash
# Create NetworkPolicy denying all unapproved ingress traffic to production pods
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```

---

## Phase 5: Low-Level Diagnostics, Telemetry & Ephemeral Debugging

### 1. Extract Base64 Secrets in One Command
```bash
kubectl get secret web-app-secrets -n production -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode; echo
```

### 2. Inject Ephemeral Debug Container into Running Pod
```bash
kubectl debug -it pod/web-app-74b89-x8q2z -n production \
    --image=nicolaka/netshoot \
    --target=web \
    -- /bin/bash
```

### 3. Graceful Node Drain & Maintenance
```bash
# Cordon and drain node safely
kubectl cordon worker-node-01
kubectl drain worker-node-01 --ignore-daemonsets --delete-emptydir-data --force --grace-period=60
kubectl uncordon worker-node-01
```

---

## Phase 6: Professional Failure Diagnosis & Resolution Matrix

| Phase | Error State / Status | Root Cause | Diagnosis Command | Immediate Resolution Command |
| :--- | :--- | :--- | :--- | :--- |
| **Build** | **`trivy CVE Exit Code 1`** | High/Critical vulnerability in base OS image | `trivy image IMAGE` | Upgrade base image in Dockerfile (e.g., `alpine:3.19` or `distroless`) and rebuild. |
| **Push** | **`denied: Permission "artifactregistry.repositories.download" denied`** | Docker credentials not configured for GCP | `gcloud auth list` | Authenticate docker: `gcloud auth configure-docker us-central1-docker.pkg.dev`. |
| **Deploy** | **`CrashLoopBackOff`** | Container process exits on startup (Exit Code 1, 127) | `kubectl logs POD --previous` | Fix application config, entrypoint, or missing env vars. |
| **Deploy** | **`ImagePullBackOff`** | Wrong tag or private registry secret missing | `kubectl describe pod POD` | Create secret: `kubectl create secret docker-registry reg-cred --docker-username=U --docker-password=P` and set `imagePullSecrets` in YAML. |
| **Runtime**| **`OOMKilled` (Exit Code 137)** | Container RAM exceeded `resources.limits.memory` | `kubectl describe pod POD \| grep -i oom` | Increase memory limit in manifest: `resources.limits.memory: "1Gi"`. |
| **Deploy** | **`Pending` (Unscheduled)** | Insufficient CPU/RAM on worker nodes | `kubectl describe pod POD \| grep -A 5 Events` | Scale worker node pool or reduce `resources.requests` in YAML. |
| **Config** | **`CreateContainerConfigError`** | Referenced ConfigMap or Secret missing | `kubectl describe pod POD` | Create missing Secret/ConfigMap: `kubectl create secret generic S --from-literal=K=V`. |
| **Cluster**| **`Evicted`** | Worker node disk (`DiskPressure`) or RAM pressure | `kubectl get nodes` | Purge failed pods: `kubectl delete pods --field-selector status.phase=Failed -A`. |
| **Cluster**| **`NodeNotReady`** | `kubelet` agent crashed on worker node | `kubectl describe node NODE` | Restart kubelet on node: `sudo systemctl restart kubelet`. |
| **Storage**| **`PersistentVolumeClaimPending`** | StorageClass missing or provisioner offline | `kubectl describe pvc PVC` | Verify StorageClass (`kubectl get sc`) or create matching PV. |
| **Ingress**| **`IngressClassNotSpecified`** | Ingress manifest missing `ingressClassName` | `kubectl get ingressclass` | Add `spec.ingressClassName: nginx` to Ingress manifest. |
| **Rollout**| **`Rollout Stalled / Timeout`** | New replica pods fail readiness probes | `kubectl rollout status deploy/APP` | Undo rollout immediately: `kubectl rollout undo deploy/APP`. |
| **RBAC**   | **`403 Forbidden (API Server)`** | Kubeconfig token expired or missing RoleBinding | `kubectl auth can-i get pods` | Re-authenticate `gcloud container clusters get-credentials CLUSTER`. |
