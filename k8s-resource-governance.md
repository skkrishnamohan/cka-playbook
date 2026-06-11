# Kubernetes Resource Governance — LimitRange, ResourceQuota, Pod Security Admission & Node Authorization

---

## Governance hierarchy

```
Cluster
   │
   ├── Node Authorization    ← Controls what kubelets can access
   │
   └── Namespace
          │
          ├── Pod Security Admission   ← Controls what Pods can do (security)
          │
          ├── ResourceQuota            ← Total budget for the whole namespace
          │
          └── LimitRange               ← Per-Pod / per-container rules
                    ↓
               Individual Containers
```

---

## LimitRange — per-Pod and per-container resource controls

- What: LimitRange sets minimum, maximum, and default resource values for individual Pods and containers within a namespace.
- Why: Without defaults, Pods with no `resources:` block bypass ResourceQuota (quota requires requests to be set). LimitRange auto-assigns defaults.
- Scope: Each container individually, not the whole namespace.

### LimitRange example — defaults + min/max
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:                    # Applied if container doesn't specify limits
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:             # Applied if container doesn't specify requests
      cpu: "100m"
      memory: "128Mi"
    max:                        # No container can exceed these
      cpu: "2"
      memory: "1Gi"
    min:                        # No container can be below these
      cpu: "50m"
      memory: "64Mi"
```

### LimitRange for Pods (total across all containers)
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limits
  namespace: dev
spec:
  limits:
  - type: Pod
    max:
      cpu: "4"
      memory: "4Gi"
```

### LimitRange commands
```bash
# List LimitRanges in a namespace
kubectl get limitrange -n dev

# Describe (shows current limits)
kubectl describe limitrange container-limits -n dev
```

---

## ResourceQuota — namespace-level total budget

- What: ResourceQuota caps the total resource consumption (CPU, memory, object counts) for an entire namespace.
- Why: Prevents one team or application from consuming all cluster resources.
- How it works: When a Pod is created, Kubernetes sums up all existing resource requests in the namespace and checks if adding the new Pod would exceed the quota. If yes → rejected.
- Scope: The whole namespace combined, not individual pods.

### LimitRange vs ResourceQuota — key difference
| | LimitRange | ResourceQuota |
|-|-----------|---------------|
| Scope | Per Pod / Per Container | Entire Namespace |
| Controls | Individual resource limits | Aggregate namespace budget |
| Sets defaults | Yes | No |
| Min/Max per container | Yes | No |
| Total pod count | No | Yes |

### Basic ResourceQuota example (CPU + memory)
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"          # All pods combined can request up to 4 CPUs
    requests.memory: "8Gi"     # All pods combined can request up to 8Gi memory
    limits.cpu: "6"            # All pods combined cannot exceed 6 CPU limit
    limits.memory: "16Gi"      # All pods combined cannot exceed 16Gi limit
```

### What ResourceQuota can limit
```yaml
spec:
  hard:
    # Compute
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "6"
    limits.memory: "16Gi"

    # Object counts
    pods: "20"
    services: "10"
    deployments.apps: "10"
    jobs.batch: "5"
    persistentvolumeclaims: "10"

    # Other
    secrets: "20"
    configmaps: "20"
```

### ResourceQuota arithmetic example (from class)
```
Namespace quota:  requests.cpu = "4"

Current pods:
  app1 = 500m
  app2 = 1000m (1 CPU)
  app3 = 1500m (1.5 CPU)
  Total = 3 CPUs used

Remaining: 1 CPU available

New pod requests: cpu = "2"
  3 + 2 = 5 CPUs → EXCEEDS quota of 4 → REJECTED ❌
```

### Common error when quota is exceeded
```
Error from server (Forbidden): error when creating "deployment.yaml":
pods "my-pod-xyz" is forbidden:
exceeded quota: team-quota,
requested: requests.cpu=1,
used: requests.cpu=4,
limited: requests.cpu=4
```

### Resolving quota exceeded errors
```bash
# Option 1: Increase the quota
kubectl edit resourcequota team-quota -n dev
# or update quota.yaml and:
kubectl apply -f quota.yaml

# Option 2: Delete unused workloads to free quota
kubectl get pods -n dev
kubectl delete pod unused-pod -n dev
```

### ResourceQuota commands
```bash
# List ResourceQuotas in a namespace
kubectl get resourcequota -n dev
kubectl get quota -n dev              # Short form

# Describe (shows Used vs Hard - most useful!)
kubectl describe resourcequota team-quota -n dev
# Example output:
# Resource          Used   Hard
# requests.cpu      3      4
# limits.cpu        4      6
# pods              8      20

# Check namespace resource summary
kubectl describe namespace dev
```

---

## Pod Security Admission (PSA) — securing what Pods can do

- What: PSA is a built-in Kubernetes admission controller that enforces security profiles on Pods at the namespace level.
- Why: Prevents insecure Pod configurations (running as root, host networking, privileged containers).
- Replaced: Pod Security Policy (PSP) — deprecated and removed in K8s 1.25.

### Three security levels

| Level | Description | Allows |
|-------|-------------|--------|
| **privileged** | No restrictions | Everything (for system tools like CNI, storage drivers) |
| **baseline** | Moderate security | Standard workloads, blocks most dangerous settings |
| **restricted** | Maximum security | Blocks privileged containers, root user, host networking, etc. |

### Blocked by `restricted` level:
- `securityContext.privileged: true`
- Running as root (`runAsNonRoot: false`)
- Host namespaces (`hostNetwork`, `hostPID`, `hostIPC`)
- Unsafe capabilities (e.g. `NET_ADMIN`, `SYS_ADMIN`)
- `hostPath` volumes

### Three enforcement modes
- **enforce** — Reject Pods that violate the policy.
- **warn** — Allow but print a warning.
- **audit** — Allow but log a violation (for monitoring).

### Applying PSA labels to a namespace
```bash
# Enforce restricted policy (most secure — rejects violating pods)
kubectl label namespace dev \
  pod-security.kubernetes.io/enforce=restricted

# Enforce baseline policy (standard apps)
kubectl label namespace dev2 \
  pod-security.kubernetes.io/enforce=baseline

# No restrictions (for system namespaces like kube-system)
kubectl label namespace kube-system \
  pod-security.kubernetes.io/enforce=privileged

# Add warn mode alongside enforce (shows warnings without blocking)
kubectl label namespace dev \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted

# Verify labels
kubectl get ns dev --show-labels
```

### What happens to a bad Pod
```yaml
# This Pod runs as privileged — violates restricted policy
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  namespace: dev                  # namespace has enforce=restricted
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: true            # ← BLOCKED in restricted namespace
```

```bash
kubectl apply -f bad-pod.yaml
# Error from server (Forbidden): pods "bad-pod" is forbidden:
# violates PodSecurity "restricted:latest": ...
```

### Result by namespace label (from class demo)
| Namespace | PSA label | Pod result |
|-----------|-----------|-----------|
| dev | restricted | Rejected ❌ |
| dev1 | privileged | Allowed ✅ |
| dev2 | baseline | Rejected ❌ (if privileged container) |

### Good Pod (passes restricted)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: dev
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

---

## Node Authorization — kubelet least privilege

- What: A built-in Kubernetes authorization mode that restricts what each kubelet (on a worker node) can read/write in the API server.
- Why: Without Node authorization, a compromised worker node could read secrets from ALL nodes. With it, each node can only access its own resources.
- Built-in by default on kubeadm clusters.

### The problem Node Authorization solves
```
Without Node Authorization:
  worker-node-1 compromised
        ↓
  Can read Secrets from all nodes ❌ (security risk)

With Node Authorization:
  worker-node-1 compromised
        ↓
  Can ONLY read Secrets assigned to worker-node-1's pods ✅
```

### API server configuration
```bash
# Check API server flags (kubeadm cluster)
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep authorization-mode
# Output: --authorization-mode=Node,RBAC
```

Both `Node` and `RBAC` must be enabled together. Order matters: `Node` runs first.

### Kubelet identity — how the API server recognizes a node
```bash
# Kubelets authenticate using client certificates with this identity:
# CN (Common Name): system:node:<node-name>
# O  (Organization): system:nodes

# Example certificate subject:
# CN=system:node:worker01
# O=system:nodes

# Verify kubelet certificate
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -text | grep -A2 Subject

# Or check CSR
kubectl get csr
```

### What Node Authorization allows each kubelet
```
Allowed for its OWN node:
  ✅ Read Pods assigned to this node
  ✅ Read Secrets referenced by this node's Pods
  ✅ Read ConfigMaps referenced by this node's Pods
  ✅ Read PV attachments for this node
  ✅ Update own node status

Not allowed:
  ❌ Read Pods on other nodes
  ❌ Read Secrets from other nodes' Pods
  ❌ Modify cluster-level resources
```

### Interview questions (from session)
**Q: Why use Node Authorization?**
> Restricts kubelets to only access resources associated with their own node — enforces least-privilege at the node level.

**Q: What API server flag enables it?**
> `--authorization-mode=Node,RBAC`

**Q: What identity must a kubelet use?**
> A client certificate with `CN=system:node:<node-name>` and `O=system:nodes`

**Q: What's the difference between LimitRange and ResourceQuota?**
> LimitRange = per-Pod/container min/max + defaults. ResourceQuota = total namespace budget.

---

## Complete governance picture (production best practice)
```
Namespace (dev)
│
├── Pod Security Admission
│     enforce: restricted        ← Security: blocks unsafe pod configs
│
├── ResourceQuota
│     requests.cpu: "4"          ← Total CPU budget for whole namespace
│     requests.memory: "8Gi"
│     pods: "20"
│
└── LimitRange
      default cpu: 100m          ← Ensures every pod has sane defaults
      max cpu: 2                 ← No single container goes wild
      defaultRequest: 50m
```

---

## Quick command reference
```bash
# LimitRange
kubectl get limitrange -n dev
kubectl describe limitrange container-limits -n dev

# ResourceQuota
kubectl get resourcequota -n dev
kubectl describe resourcequota team-quota -n dev

# Pod Security Admission
kubectl label namespace dev pod-security.kubernetes.io/enforce=restricted
kubectl get ns dev --show-labels

# Node Authorization (check API server config)
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep authorization-mode
kubectl get csr
```

---

## One-line takeaways
- LimitRange = rules for each individual container (defaults, min, max). Needed so pods have requests set (required by ResourceQuota).
- ResourceQuota = total budget for the entire namespace (CPU, memory, pod count). Rejects new pods when budget exhausted.
- Pod Security Admission replaces PSP — label namespaces with `privileged`, `baseline`, or `restricted` to control what Pods can do.
- Node Authorization ensures a compromised worker node can't read secrets from other nodes — each kubelet gets least-privilege access.
- Production pattern: ResourceQuota + LimitRange + PSA restricted = secure, governed namespace.
