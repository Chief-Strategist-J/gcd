# Kubernetes: Decision Trees (ASCII & Visual)

This document provides decision logic trees for selecting Kubernetes Workload APIs, Service Types, and Persistent Storage strategies.

---

## 1. Workload API Selection (ASCII Decision Tree)

```
================================================================================
                    KUBERNETES WORKLOAD API DECISION TREE
================================================================================

                    What type of application are you deploying?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
 [ Long-Running Workload ]      [ Batch / Task Execution ]    [ Node Infrastructure ]
  APIs, Web Servers, Workers    Data Processing, Migrations    Log Collectors, Monitoring
        │                              │                       (Fluentd, Promtail, NodeExporter)
        │                              │                              │
        ▼                              ▼                              ▼
  Stateless or Stateful?        One-time or Scheduled?         USE DAEMONSET
        │                              │                       Runs 1 Pod per Node
        ├───────────────┐              ├───────────────┐
        ▼               ▼              ▼               ▼
 [ Stateless ]     [ Stateful ]     [ One-Time ]    [ Scheduled ]
  DEPLOYMENT        STATEFULSET      JOB             CRONJOB
  Replicas can      Unique Network   Runs to         Runs on cron
  restart anywhere   ID & Persistent  Completion     schedule
                    Storage (PVC)
```

---

## 2. Kubernetes Service Type Selection (ASCII Decision Tree)

```
================================================================================
                    KUBERNETES SERVICE TYPE DECISION TREE
================================================================================

                 How should your Pods be exposed to traffic?
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
 [ Cluster Internal Only ]      [ External via Node Port ]    [ External via Cloud LB ]
  Inter-pod microservice         Testing / Dev environments   Production Web App / Traffic
  communication                  (Ports 30000-32767)          Exposed to Internet
        │                              │                              │
        ▼                              ▼                              ▼
  ClusterIP                      NodePort                       LoadBalancer / Ingress
  Internal Virtual IP            Exposes port on Node IP        Provisions Cloud L7/L4 LB
```

---

## 3. Visual Mermaid Decision Flowcharts

### Workload Selection Flowchart:
```mermaid
graph TD
    StartWorkload["Select Workload Controller"] --> Nature{"Workload Execution Nature?"}
    
    Nature -->|Stateless Service / Microservice| Dep["Deployment + ReplicaSet"]
    Nature -->|Stateful Database / Redis Cluster| STS["StatefulSet (Stable Hostnames & PVCs)"]
    Nature -->|Node Infrastructure Agent| DS["DaemonSet (Exactly 1 Pod per Worker Node)"]
    Nature -->|One-Time Batch Execution| Job["Job (Runs to completion)"]
    Nature -->|Recurring Cron Task| CronJob["CronJob (Scheduled execution)"]
```

### Service Exposure Flowchart:
```mermaid
graph TD
    StartSvc["Expose Workload Traffic"] --> TrafficType{"Access Requirement?"}
    
    TrafficType -->|Internal Cluster Traffic Only| ClusterIP["ClusterIP (Default internal IP)"]
    TrafficType -->|Direct High-Port Node Exposure| NodePort["NodePort (Port range 30000-32767)"]
    TrafficType -->|Production External HTTP/HTTPS| Ingress["Ingress Controller + LoadBalancer"]
```
