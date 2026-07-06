# CKA Study Hub

**Certified Kubernetes Administrator** — Personal Study Notes & Hands-On Labs

> Exam: 2 hours · Passing score: 66% · K8s v1.34 · Valid 2 years  
> Practice environment: [Killercoda](https://killercoda.com) (2-node kubeadm cluster)  
> Curriculum source: [CKA_Curriculum_v1.35.pdf](https://github.com/cncf/curriculum/blob/master/CKA_Curriculum_v1.35.pdf)

---

## Domain Progress

| # | Domain | Weight | Status |
|---|--------|:------:|--------|
| 1 | Cluster Architecture, Installation & Configuration | 25% | 🟢 Complete |
| 2 | Workloads & Scheduling | 15% | 🟢 Complete |
| 3 | Services & Networking | 20% | 🟢 Complete |
| 4 | Storage | 10% | 🟢 Complete |
| 5 | Troubleshooting | 30% | 🟢 Complete |

---

## All Notes — Quick Navigation

| File | What's Inside |
|------|--------------|
| [k8s-architecture.md](k8s-architecture.md) | Control plane components, etcd, API server, scheduler, HA setup |
| [k8s-api-overview.md](k8s-api-overview.md) | API Groups, apiVersion, resource discovery, `kubectl explain` |
| [controller-manager.md](controller-manager.md) | Controller Manager deep-dive, built-in controllers |
| [k8s-etc-kubernetes-deep-dive.md](k8s-etc-kubernetes-deep-dive.md) | `/etc/kubernetes` structure, static pods, certs, etcd backup/restore |
| [create-cluster.md](create-cluster.md) | kubeadm cluster creation, upgrades, CoreDNS, CNI |
| [docker-vs-containerd.md](docker-vs-containerd.md) | CRI, containerd, CNI plugins (Calico/Flannel/Cilium), CoreDNS |
| [k8s-helm.md](k8s-helm.md) | Helm charts, repos, OCI registry, dependencies, Kustomize |
| [k8s-resource-governance.md](k8s-resource-governance.md) | LimitRange, ResourceQuota, Pod Security Admission, Node Authorization |
| [k8s-workloads.md](k8s-workloads.md) | Pod → ReplicaSet → Deployment → StatefulSet → DaemonSet → Job → CronJob → HPA → Services |
| [k8s-scheduling-and-affinity.md](k8s-scheduling-and-affinity.md) | Node/Pod affinity, taints, tolerations, priority classes |
| [k8s-pod-patterns.md](k8s-pod-patterns.md) | Init containers, sidecar, ambassador, adapter, CrashLoopBackOff, probes |
| [k8s-configmaps-secrets-volumes.md](k8s-configmaps-secrets-volumes.md) | ConfigMaps, Secrets, emptyDir, volume mounts |
| [k8s-storage-pv-pvc.md](k8s-storage-pv-pvc.md) | PV, PVC, StorageClass, access modes, reclaim policies, CSI, OpenEBS |
| [k8s-network-policy.md](k8s-network-policy.md) | NetworkPolicy ingress/egress, pod isolation, deny-all patterns |
| [k8s-ingress-gateway.md](k8s-ingress-gateway.md) | Ingress (NGINX), Gateway API, MetalLB, TLS |
| [k8s-rbac.md](k8s-rbac.md) | Role, ClusterRole, RoleBinding, ClusterRoleBinding, ServiceAccounts |
| [k8s-declarative-vs-imperative.md](k8s-declarative-vs-imperative.md) | `kubectl apply` vs `kubectl create`, speed commands |
| [k8s-troubleshooting.md](k8s-troubleshooting.md) | Node NotReady, kubelet, logs, container exit codes, DNS failures, networking debug |
| [cka-practice-tasks.md](cka-practice-tasks.md) | 26 timed practice tasks with real CLI output |
| [k8s-hands-on-projects.md](k8s-hands-on-projects.md) | **10 POC projects** — full real-world scenarios, copy-paste on Killercoda |

---

## Study Notes by Domain

### 1. Cluster Architecture, Installation & Configuration — 25%

| Topic | Notes | Status |
|-------|-------|--------|
| Manage RBAC | [k8s-rbac.md](k8s-rbac.md) | ✅ |
| Prepare infrastructure for K8s | [create-cluster.md](create-cluster.md) | ✅ |
| Create & manage clusters (kubeadm) | [create-cluster.md](create-cluster.md) | ✅ |
| Cluster lifecycle & upgrades | [create-cluster.md](create-cluster.md) | ✅ |
| High-availability control plane | [k8s-architecture.md](k8s-architecture.md) | ✅ |
| Helm and Kustomize | [k8s-helm.md](k8s-helm.md) | ✅ |
| Extension interfaces (CNI, CSI, CRI) | [docker-vs-containerd.md](docker-vs-containerd.md) | ✅ |
| CRDs and Operators | [k8s-api-overview.md](k8s-api-overview.md) | ✅ |

---

### 2. Workloads & Scheduling — 15%

| Topic | Notes | Status |
|-------|-------|--------|
| Deployments, rolling updates, rollbacks | [k8s-workloads.md](k8s-workloads.md) | ✅ |
| ConfigMaps and Secrets | [k8s-configmaps-secrets-volumes.md](k8s-configmaps-secrets-volumes.md) | ✅ |
| Workload autoscaling (HPA/VPA) | [k8s-workloads.md](k8s-workloads.md) | ✅ |
| Self-healing deployments | [k8s-workloads.md](k8s-workloads.md) | ✅ |
| Pod scheduling (limits, affinity) | [k8s-scheduling-and-affinity.md](k8s-scheduling-and-affinity.md) · [k8s-resource-governance.md](k8s-resource-governance.md) | ✅ |

---

### 3. Services & Networking — 20%

| Topic | Notes | Status |
|-------|-------|--------|
| Pod-to-pod connectivity | [docker-vs-containerd.md](docker-vs-containerd.md) · [k8s-network-policy.md](k8s-network-policy.md) | ✅ |
| Network Policies | [k8s-network-policy.md](k8s-network-policy.md) | ✅ |
| ClusterIP, NodePort, LoadBalancer | [k8s-workloads.md](k8s-workloads.md) | ✅ |
| Gateway API | [k8s-ingress-gateway.md](k8s-ingress-gateway.md) | ✅ |
| Ingress controllers & resources | [k8s-ingress-gateway.md](k8s-ingress-gateway.md) | ✅ |
| CoreDNS | [docker-vs-containerd.md](docker-vs-containerd.md) | ✅ |

---

### 4. Storage — 10%

| Topic | Notes | Status |
|-------|-------|--------|
| StorageClasses & dynamic provisioning | [k8s-storage-pv-pvc.md](k8s-storage-pv-pvc.md) | ✅ |
| Volume types, access modes, reclaim policies | [k8s-storage-pv-pvc.md](k8s-storage-pv-pvc.md) | ✅ |
| PersistentVolumes & PersistentVolumeClaims | [k8s-storage-pv-pvc.md](k8s-storage-pv-pvc.md) | ✅ |

---

### 5. Troubleshooting — 30%

| Topic | Notes | Status |
|-------|-------|--------|
| Troubleshoot clusters and nodes | [k8s-troubleshooting.md](k8s-troubleshooting.md) | ✅ |
| Troubleshoot cluster components | [k8s-troubleshooting.md](k8s-troubleshooting.md) | ✅ |
| Monitor resource usage (`kubectl top`) | [k8s-troubleshooting.md](k8s-troubleshooting.md) | ✅ |
| Container output streams (logs) | [k8s-troubleshooting.md](k8s-troubleshooting.md) | ✅ |
| Troubleshoot services and networking | [k8s-troubleshooting.md](k8s-troubleshooting.md) | ✅ |

---

## Hands-On Projects

Full end-to-end scenarios on Killercoda — copy-paste ready → [k8s-hands-on-projects.md](k8s-hands-on-projects.md)

| # | Project | Skills |
|---|---------|--------|
| 1 | Multi-Tier Web App (frontend + backend + DB) | Deployments, Services, DNS, ConfigMaps |
| 2 | Zero-Downtime Update & Emergency Rollback | RollingUpdate, rollout history, `rollout undo` |
| 3 | Auto-Scaling Under Traffic Load | HPA, Metrics Server, load generation |
| 4 | Stateful Database with Persistent Storage | StatefulSet, PVC, data survival test |
| 5 | RBAC Multi-Team Cluster Security | Role, ClusterRole, RoleBinding, `auth can-i` |
| 6 | Network Policy Zero-Trust Microservices | deny-all, selective ingress, connectivity test |
| 7 | Node Drain & Pod Rescheduling | cordon, drain, taint, uncordon |
| 8 | Self-Healing App — Probes & Auto-Restart | livenessProbe, readinessProbe, CrashLoopBackOff |
| 9 | Persistent Storage — Data Survival Test | PV, PVC, emptyDir contrast |
| 10 | Ingress Controller — API Gateway Routing | Nginx Ingress, path-based + host-based routing |

---

## Exam Tips

- Performance-based exam — no multiple choice, real cluster tasks
- `kubernetes.io/docs` is allowed during the exam — know how to search fast
- **Troubleshooting is 30%** — the single highest-weight domain
- Master imperative commands for speed:
  ```bash
  kubectl run nginx --image=nginx:1.28 --dry-run=client -o yaml > pod.yaml
  kubectl create deployment web --image=nginx --replicas=3
  kubectl expose deployment web --port=80 --type=NodePort
  kubectl create configmap app-config --from-literal=ENV=prod
  kubectl create secret generic db-creds --from-literal=password=s3cr3t
  kubectl create serviceaccount my-sa
  kubectl create role dev-role --verb=get,list,create --resource=pods
  kubectl create rolebinding dev-bind --role=dev-role --serviceaccount=default:my-sa
  ```
- `kubectl explain <resource>.<field>` — look up any field without leaving the terminal
- Always `kubectl rollout status` after applying a deployment
- 2 hours is tight — skip hard questions and come back
