# CKA Exam Practice — Master Index

> **Goal**: Clear CKA with a great score and finish **30 minutes early**
> **Strategy**: 5 days of focused practice — easy → hard — building speed and muscle memory
> **Exam**: 2 hours | 66% to pass | K8s v1.34 | `kubernetes.io/docs` allowed

---

## 5-Day Study Schedule

| Day | File | Difficulty | Questions | Focus | Time |
|-----|------|-----------|-----------|-------|------|
| 1 | [exam-day1-warmup.md](exam-day1-warmup.md) | 🟢 Easy | Q5, Q6, Q12, Q17, Q18 | StorageClass, Helm, CRDs, Node count, Kubelet | ~45 min |
| 2 | [exam-day2-core-skills.md](exam-day2-core-skills.md) | 🟡 Medium | Q1, Q2, Q3, Q10, Q21 | HPA, Services, Sidecars, Resources, Ingress | ~60 min |
| 3 | [exam-day3-networking.md](exam-day3-networking.md) | 🟠 Medium-Hard | Q9, Q13, Q14, Q16, Q20 | CNI, NetworkPolicy, CoreDNS, TLS config | ~75 min |
| 4 | [exam-day4-advanced.md](exam-day4-advanced.md) | 🔴 Hard | Q4, Q7, Q8, Q11, Q15 | PriorityClass, PV/PVC, Gateway API, cri-dockerd, crictl | ~90 min |
| 5 | [exam-day5-cluster-ops.md](exam-day5-cluster-ops.md) | 🔴 Hard | Q19, Q22 + Mock Review | kubeadm upgrade, Cluster troubleshooting | ~60 min |

---

## All 22 Questions at a Glance

| # | Question Summary | Weight | Difficulty | CKA Domain | Day | Key Skill |
|---|-----------------|--------|-----------|------------|-----|-----------|
| 1 | HPA with stabilization window | — | 🟡 | Workloads & Scheduling | 2 | `kubectl autoscale` + edit behavior |
| 2 | Expose deployment via NodePort | 10% | 🟡 | Services & Networking | 2 | `kubectl expose` + Service YAML |
| 3 | Add sidecar container with shared volume | 15% | 🟡 | Workloads & Scheduling | 2 | Multi-container pods, emptyDir |
| 4 | PriorityClass + patch deployment | 11% | 🔴 | Workloads & Scheduling | 4 | PriorityClass, preemption, kubectl patch |
| 5 | Create default StorageClass | 8% | 🟢 | Storage | 1 | StorageClass YAML, annotations |
| 6 | Helm template ArgoCD (skip CRDs) | 7% | 🟢 | Cluster Architecture | 1 | `helm repo add`, `helm template` |
| 7 | Restore MariaDB with PVC | 13% | 🔴 | Storage | 4 | PV/PVC binding, Deployment volumes |
| 8 | Migrate Ingress to Gateway API | 12% | 🔴 | Services & Networking | 4 | Gateway, HTTPRoute YAML |
| 9 | Install CNI with NetworkPolicy support | 10% | 🟠 | Cluster Architecture | 3 | Calico vs Flannel, `kubectl apply` |
| 10 | Fix pod resources for WordPress | 9% | 🟡 | Workloads & Scheduling | 2 | Resource requests/limits, `kubectl top node` |
| 11 | Install cri-dockerd + sysctl params | 10% | 🔴 | Cluster Architecture | 4 | `dpkg`, `systemctl`, sysctl |
| 12 | List cert-manager CRDs + subject field | 7% | 🟢 | Cluster Architecture | 1 | `kubectl get crd`, jsonpath |
| 13 | Choose least-permissive NetworkPolicy | 10% | 🟠 | Services & Networking | 3 | NetworkPolicy analysis |
| 14 | CoreDNS custom domain configuration | — | 🟠 | Services & Networking | 3 | ConfigMap edit, CoreDNS Corefile |
| 15 | Create pod + crictl inspect on node | — | 🔴 | Troubleshooting | 4 | `crictl ps`, `crictl inspect`, SSH |
| 16 | NetworkPolicy allow from namespace | — | 🟠 | Services & Networking | 3 | namespaceSelector, port filtering |
| 17 | Count schedulable nodes | — | 🟢 | Troubleshooting | 1 | Taints, `kubectl describe node` |
| 18 | Fix NotReady worker node | — | 🟢 | Troubleshooting | 1 | `systemctl start/enable kubelet` |
| 19 | kubeadm upgrade master node | — | 🔴 | Cluster Architecture | 5 | `kubeadm upgrade`, drain, uncordon |
| 20 | Enforce TLS 1.3 only via ConfigMap | — | 🟠 | Services & Networking | 3 | ConfigMap edit, `rollout restart` |
| 21 | Create Ingress for echo service | — | 🟡 | Services & Networking | 2 | Ingress YAML, path-based routing |
| 22 | Troubleshoot broken cluster | — | 🔴 | Troubleshooting | 5 | Static pods, etcd, kubelet, systemd |

---

## Domain Coverage

| CKA Domain | Weight | Questions | Count |
|------------|--------|-----------|-------|
| Cluster Architecture, Installation & Configuration (25%) | 25% | Q5, Q6, Q9, Q11, Q12, Q19 | 6 |
| Workloads & Scheduling (15%) | 15% | Q1, Q3, Q4, Q10 | 4 |
| Services & Networking (20%) | 20% | Q2, Q8, Q13, Q14, Q16, Q20, Q21 | 7 |
| Storage (10%) | 10% | Q5, Q7 | 2 |
| Troubleshooting (30%) | 30% | Q15, Q17, Q18, Q22 | 4 |

---

## Exam Day Strategy

### Time Budget (120 minutes)

| Phase | Time | Action |
|-------|------|--------|
| First pass | 0–70 min | Do all easy + medium questions first (≤5 min each) |
| Second pass | 70–100 min | Tackle hard questions |
| Review | 100–110 min | Verify answers, check namespaces |
| Buffer | 110–120 min | Fix any remaining issues |

### Golden Rules
1. **Always check context first** — `kubectl config use-context <name>`
2. **Always check namespace** — add `-n <namespace>` to every command
3. **Use imperative commands** for speed — `kubectl create`, `kubectl run`, `kubectl expose`
4. **Use `--dry-run=client -o yaml`** to generate YAML fast
5. **Don't waste time on one question** — skip and come back
6. **Verify with `kubectl get`** after every answer
