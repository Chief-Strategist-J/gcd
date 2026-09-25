# Feature 4: Kubernetes (k8s) & kubectl (Container Orchestration)

Welcome to the dedicated documentation index for **Kubernetes Container Orchestration & kubectl**. This feature module provides comprehensive architectural blueprints, low-level state machine mechanics, decision trees, CLI command manuals, failure recovery runbooks, and a unified 3-layer networking manual bridging Docker, Kubernetes CNI, and Google Cloud VPC underlay infrastructure.

---

## Module Documentation Index

| Section | Document File | Description |
| :--- | :--- | :--- |
| **Architecture (HLD & LLD)** | [`hld-lld-design.md`](./hld-lld-design.md) | High-Level Architecture (Control Plane vs Worker Nodes) & Low-Level Mechanics (Pod Lifecycle State Machine, `etcd` Raft quorum, `kubelet` startup/liveness/readiness probes, and API server admission control). |
| **Decision Trees** | [`decision-tree.md`](./decision-tree.md) | ASCII & Visual Mermaid decision trees for Workload API selection (Deployment vs StatefulSet vs DaemonSet vs Job vs CronJob) and Service exposure strategy (ClusterIP vs NodePort vs LoadBalancer vs Headless). |
| **CLI Commands & Error Matrix** | [`shell-commands.md`](./shell-commands.md) | In-depth `kubectl` command manual, custom JSONPath projections, resource debugging (`logs`, `exec`, `debug`), exhaustive Error Resolution Matrix, and **Generative AI & LLM Inference on GKE (NVIDIA L4 GPU, Hugging Face TGI & Gradio)**. |
| **Unified Networking & GCP VPC Underlay** | [`networking-shell-command.md`](./networking-shell-command.md) | Exhaustive 27-section networking manual connecting Docker, Kubernetes CNI (Cilium eBPF Datapath v2), and Google Cloud VPC (Subnets, Firewalls, Cloud NAT, Peering, HA VPN, Interconnect, Gateway API, Cloud Armor WAF, and Workload Identity). |
| **Official References** | [`references.md`](./references.md) | Official Kubernetes documentation links, kubectl cheat sheet, API conventions, CIS security benchmarks, and **GKE AI/ML & NVIDIA GPU Best Practices**. |

---

## Key Technical & Operational Concepts Covered

1. **Control Plane & Worker Node Topology**: Separation of concerns between API Server, etcd distributed key-value store, Scheduler, Controller Manager, and Node agents (`kubelet`, container runtime, and CNI).
2. **Pod Lifecycle & State Transitions**: Deterministic state progression across `Pending`, `Running`, `Succeeded`, `Failed`, and `Unknown`, regulated by container startup, liveness, and readiness probes.
3. **Workload API Controllers**: Tailored selection between stateless Deployments, stateful ordered sets (StatefulSet), node-local daemons (DaemonSet), run-to-completion Jobs, and scheduled CronJobs.
4. **Declarative Service Abstractions**: Internal East-West ClusterIP virtual routing, NodePort static port binding, Headless direct DNS resolution, and Cloud LoadBalancer external ingress.
5. **Unified 3-Layer Networking Architecture**: End-to-end packet datapath linking Layer 3 Docker container namespaces, Layer 2 CNI host kernel hooks, and Layer 1 Google Cloud Andromeda SDN fabric.
6. **VPC-Native Cluster Datapath**: Direct Pod IP allocation from secondary VPC subnet alias ranges, completely eliminating VXLAN/Geneve overlay encapsulation overhead.
7. **Cilium eBPF Datapath v2**: Sub-nanosecond $O(1)$ BPF hash map lookups, bypassing iptables sequential chains and Netfilter connection tracking race conditions.
8. **Layer 3/4 Micro-Segmentation**: Zero-trust default-deny NetworkPolicies isolating backend microservices with granular ingress and egress rules.
9. **Next-Gen Gateway API & Traffic Splitting**: Role-oriented ingress architecture separating NetOps (`Gateway`) from Developers (`HTTPRoute`), with native canary percentage weighting and header rewrites.
10. **Enterprise Security & Perimeter Defense**: Defense-in-depth enforcement utilizing Cloud Armor WAF policies, VPC Service Controls (VPC-SC), Workload Identity federation, and Binary Authorization image attestation.
11. **Hybrid Interconnect & HA VPN Failover**: Dynamic BGP route exchange with BFD sub-second failover, deterministic MED metrics, and TCP MSS clamping over IPsec tunnels.
12. **High-Throughput DNS & IP Capacity Planning**: NodeLocal DNSCache daemons intercepting UDP queries to eliminate conntrack drops, paired with dynamic secondary CIDR expansion.
13. **Generative AI & LLM Inference on GKE**: Provisioning NVIDIA L4 Tensor Core GPU nodes (G2 machines), Hugging Face Text Generation Inference (TGI) containerization, POSIX shared memory `/dev/shm` mounts, tensor parallelism sharding, Google Cloud Managed Prometheus scraping, and Gradio chat frontend integration.

---

## Quick Navigation

1. [View High-Level & Low-Level Design (`hld-lld-design.md`)](./hld-lld-design.md)
2. [View Decision Trees (`decision-tree.md`)](./decision-tree.md)
3. [View Shell Command Reference & Error Matrix (`shell-commands.md`)](./shell-commands.md)
4. [View Unified Networking Manual (`networking-shell-command.md`)](./networking-shell-command.md)
5. [View Official Reference Links (`references.md`)](./references.md)
