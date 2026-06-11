# CKA Curriculum v1.35 — Study Navigator

> **Source**: [CKA_Curriculum_v1.35.pdf](https://github.com/cncf/curriculum/blob/master/CKA_Curriculum_v1.35.pdf)
> **Exam duration**: 2 hours | **Passing score**: 66% | **K8s version**: v1.34
> **Certification valid**: 2 years

---

## Domains & Weightage

| # | Domain | Weight | My Progress |
|---|--------|--------|-------------|
| 1 | Cluster Architecture, Installation & Configuration | 25% | 🟢 Complete |
| 2 | Workloads & Scheduling | 15% | 🟢 Complete |
| 3 | Services & Networking | 20% | 🟢 Complete |
| 4 | Storage | 10% | 🟢 Complete |
| 5 | Troubleshooting | 30% | � Complete |

---

## 1. Cluster Architecture, Installation & Configuration — 25%

| Topic | My Notes | Status |
|-------|----------|--------|
| Manage Role-Based Access Control (RBAC) | [k8s-rbac.md](k8s-rbac.md) | ✅ |
| Prepare underlying infrastructure for installing a K8s cluster | [create-cluster.md](create-cluster.md) | ✅ |
| Create and manage clusters using kubeadm | [create-cluster.md](create-cluster.md) | ✅ |
| Manage the lifecycle of Kubernetes clusters (upgrades) | [create-cluster.md](create-cluster.md) | ✅ |
| Implement and configure a highly-available control plane | [k8s-architecture.md](k8s-architecture.md) | ✅ |
| Use Helm and Kustomize to install cluster components | [k8s-helm.md](k8s-helm.md) | ✅ |
| Understand extension interfaces (CNI, CSI, CRI, etc.) | [docker-vs-containerd.md](docker-vs-containerd.md) | ✅ |
| Understand CRDs, install and configure operators | [k8s-api-overview.md](k8s-api-overview.md) | ✅ |

### Supporting notes for this domain:
- [k8s-architecture.md](k8s-architecture.md) — Control plane components, etcd, API server, scheduler, etcd backup/restore
- [controller-manager.md](controller-manager.md) — Controller Manager deep dive
- [k8s-etc-kubernetes-deep-dive.md](k8s-etc-kubernetes-deep-dive.md) — /etc/kubernetes directory, static pods, certs, etcd backup/restore
- [docker-vs-containerd.md](docker-vs-containerd.md) — Container runtimes, CRI, CNI plugins, CoreDNS
- [k8s-api-overview.md](k8s-api-overview.md) — API Groups, API Versions, apiVersion in YAML, resource discovery
- [create-cluster.md](create-cluster.md) — kubeadm cluster creation, cluster upgrade process
- [k8s-helm.md](k8s-helm.md) — Helm charts, repos, OCI registry, dependencies
- [k8s-resource-governance.md](k8s-resource-governance.md) — LimitRange, ResourceQuota, Pod Security Admission, Node Authorization

---

## 2. Workloads & Scheduling — 15%

| Topic | My Notes | Status |
|-------|----------|--------|
| Understand deployments and perform rolling updates & rollbacks | [k8s-workloads.md](k8s-workloads.md) | ✅ |
| Use ConfigMaps and Secrets to configure applications | [k8s-configmaps-secrets-volumes.md](k8s-configmaps-secrets-volumes.md) | ✅ |
| Configure workload autoscaling | [k8s-workloads.md](k8s-workloads.md) | ✅ |
| Understand primitives for robust, self-healing deployments | [k8s-workloads.md](k8s-workloads.md) | ✅ |
| Configure Pod admission and scheduling (limits, node affinity, etc.) | [k8s-scheduling-and-affinity.md](k8s-scheduling-and-affinity.md), [k8s-resource-governance.md](k8s-resource-governance.md) | ✅ |

### Supporting notes for this domain:
- [k8s-workloads.md](k8s-workloads.md) — Deployments, ReplicaSets, DaemonSets, StatefulSets, Jobs, HPA, VPA, Services, Labels & Selectors, ResourceQuota
- [k8s-api-overview.md](k8s-api-overview.md) — API Groups, YAML structure, updating objects
- [k8s-scheduling-and-affinity.md](k8s-scheduling-and-affinity.md) — Node/Pod affinity, taints, tolerations, priority
- [k8s-configmaps-secrets-volumes.md](k8s-configmaps-secrets-volumes.md) — ConfigMaps, Secrets, EmptyDir
- [k8s-pod-patterns.md](k8s-pod-patterns.md) — Init containers, sidecars, multi-container pods
- [k8s-resource-governance.md](k8s-resource-governance.md) — LimitRange, ResourceQuota, Pod Security Admission

---

## 3. Services & Networking — 20%

| Topic | My Notes | Status |
|-------|----------|--------|
| Understand connectivity between Pods | [docker-vs-containerd.md](docker-vs-containerd.md), [k8s-network-policy.md](k8s-network-policy.md) | ✅ |
| Define and enforce Network Policies | [k8s-network-policy.md](k8s-network-policy.md) | ✅ |
| Use ClusterIP, NodePort, LoadBalancer service types and endpoints | [k8s-workloads.md](k8s-workloads.md) | ✅ |
| Use the Gateway API to manage Ingress traffic | [k8s-ingress-gateway.md](k8s-ingress-gateway.md) | ✅ |
| Know how to use Ingress controllers and Ingress resources | [k8s-ingress-gateway.md](k8s-ingress-gateway.md) | ✅ |
| Understand and use CoreDNS | [docker-vs-containerd.md](docker-vs-containerd.md) | ✅ |

### Supporting notes for this domain:
- [k8s-workloads.md](k8s-workloads.md) — Services (ClusterIP, NodePort, LoadBalancer, Headless)
- [k8s-network-policy.md](k8s-network-policy.md) — NetworkPolicy ingress/egress, pod isolation
- [k8s-ingress-gateway.md](k8s-ingress-gateway.md) — Ingress (NGINX), Gateway API, MetalLB
- [create-cluster.md](create-cluster.md) — CoreDNS setup, DNS testing, CNI plugin
- [docker-vs-containerd.md](docker-vs-containerd.md) — CNI plugins (Calico, Flannel, Cilium), CIDR, CoreDNS explanation

---

## 4. Storage — 10%

| Topic | My Notes | Status |
|-------|----------|--------|
| Implement storage classes and dynamic volume provisioning | [k8s-storage-pv-pvc.md](k8s-storage-pv-pvc.md) | ✅ |
| Configure volume types, access modes and reclaim policies | [k8s-storage-pv-pvc.md](k8s-storage-pv-pvc.md) | ✅ |
| Manage persistent volumes and persistent volume claims | [k8s-storage-pv-pvc.md](k8s-storage-pv-pvc.md) | ✅ |

### Supporting notes for this domain:
- [k8s-storage-pv-pvc.md](k8s-storage-pv-pvc.md) — PV, PVC, StorageClass, NFS, access modes, reclaim policies
- [k8s-configmaps-secrets-volumes.md](k8s-configmaps-secrets-volumes.md) — EmptyDir volumes, ConfigMap/Secret volume mounts

---

## 5. Troubleshooting — 30%

| Topic | My Notes | Status |
|-------|----------|--------|
| Troubleshoot clusters and nodes | [k8s-troubleshooting.md](k8s-troubleshooting.md) | ✅ |
| Troubleshoot cluster components | [k8s-troubleshooting.md](k8s-troubleshooting.md) | ✅ |
| Monitor cluster and application resource usage | [k8s-troubleshooting.md](k8s-troubleshooting.md) | ✅ |
| Manage and evaluate container output streams | [k8s-troubleshooting.md](k8s-troubleshooting.md) | ✅ |
| Troubleshoot services and networking | [k8s-troubleshooting.md](k8s-troubleshooting.md) | ✅ |

### Supporting notes for this domain:
- [k8s-troubleshooting.md](k8s-troubleshooting.md) — Node NotReady, kubelet, cordon/drain, static pod fixes, kubectl logs, container exit codes, Service endpoints debugging, DNS failures, NetworkPolicy, common exam scenarios
- [k8s-pod-patterns.md](k8s-pod-patterns.md) — CrashLoopBackOff diagnosis, container states, restart policies, termination sequence
- [k8s-etc-kubernetes-deep-dive.md](k8s-etc-kubernetes-deep-dive.md) — Static pod manifests, kubelet cert paths, /etc/kubernetes structure

---

## General / Cross-cutting Notes

| File | Topics covered |
|------|---------------|
| [k8s-declarative-vs-imperative.md](k8s-declarative-vs-imperative.md) | kubectl apply vs create, declarative workflow |
| [cka-practice-tasks.md](cka-practice-tasks.md) | Hands-on practice scenarios |
| [k8s-hands-on-projects.md](k8s-hands-on-projects.md) | **10 POC projects** — real-world end-to-end scenarios on Killercoda |

---

## Topics to create notes for (priority order)

> Sorted by: domain weight × topic gap

1. ~~**🔴 Troubleshooting** (30%) — Entire domain needs notes — HIGHEST PRIORITY~~ ✅ Done
   - ~~Troubleshoot clusters, nodes, components~~
   - ~~Application debugging (`kubectl logs`, `describe`, `exec`)~~
   - ~~Service/networking troubleshooting~~
   - Container output streams (stdout/stderr)
   - Monitoring (`kubectl top`, metrics-server)

2. **✅ Cluster Architecture** (25%) — Fully covered
   - HA control plane → [k8s-architecture.md](k8s-architecture.md) ✅
   - CRDs & Operators → [k8s-api-overview.md](k8s-api-overview.md) ✅
   - Kustomize → [k8s-helm.md](k8s-helm.md) ✅

3. **✅ Services & Networking** (20%) — Fully covered
   - Ingress + Gateway API → [k8s-ingress-gateway.md](k8s-ingress-gateway.md) ✅
   - Network Policies → [k8s-network-policy.md](k8s-network-policy.md) ✅
   - Services (ClusterIP/NodePort/LoadBalancer) → [k8s-workloads.md](k8s-workloads.md) ✅

4. **✅ Storage** (10%) — Fully covered
   - PV, PVC, StorageClass, NFS, access modes, reclaim policies → [k8s-storage-pv-pvc.md](k8s-storage-pv-pvc.md) ✅

5. **✅ Workloads** (15%) — Fully covered
   - HPA, VPA, ResourceQuota → [k8s-workloads.md](k8s-workloads.md) ✅
   - LimitRange, Pod Security Admission → [k8s-resource-governance.md](k8s-resource-governance.md) ✅

---

## Exam tips (from reviews)

- Hands-on, performance-based (not multiple choice)
- You can use `kubernetes.io/docs` during the exam
- Practice speed — 2 hours is tight
- `kubectl explain` is your friend
- Master imperative commands for speed (`kubectl run`, `kubectl create`, `kubectl expose`)
- Troubleshooting is 30% — don't skip it!
