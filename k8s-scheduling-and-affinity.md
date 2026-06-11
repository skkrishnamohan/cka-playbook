# Kubernetes Scheduling, Affinity & Taints — Session Notes

---

## Labels — the foundation of scheduling

- Labels are key-value pairs attached to any Kubernetes object.
- Scheduling rules (affinity, taints, network policies, services) all use labels to select targets.
- Without labels, none of the scheduling features below will work.

### Labeling nodes (session command)
```bash
# Add a label to a node
kubectl label node <node-name> <key>=<value>

# Example from session:
kubectl label nodes node01 node-role.kubernetes.io/workernode1=true

# View labels on nodes
kubectl get nodes --show-labels

# Remove a label (trailing -)
kubectl label nodes node01 node-role.kubernetes.io/workernode1-
```

### Labeling pods
```bash
# Add label to a pod
kubectl label pod mypod app=web

# Overwrite existing label
kubectl label pod mypod app=api --overwrite

# Filter by labels
kubectl get pods -l app=web
kubectl get pods -l 'app in (web,api)'
```

---

## Priority Classes — who gets evicted first

- PriorityClass assigns importance to pods.
- Higher value = higher priority.
- When cluster runs out of resources, lower-priority pods get **preempted** (evicted) to make room for higher-priority pods.

### High Priority Class (from session)
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
preemptionPolicy: PreemptLowerPriority
globalDefault: false
description: "This is for critical applications."
```

### Low Priority Class (from session)
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 5000
preemptionPolicy: Never
globalDefault: false
description: "This is for non-essential applications."
```

### Pod using Priority Class (from session)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: busybox
    image: busybox
    args: ["sh", "-c", "sleep 100000"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "500m"
```

### Key concepts
| Field | Meaning |
|-------|---------|
| `value` | Integer priority (higher = more important). Range: -2 billion to 1 billion |
| `preemptionPolicy: PreemptLowerPriority` | Can evict lower-priority pods to get scheduled |
| `preemptionPolicy: Never` | Will wait in queue but won't evict others |
| `globalDefault: true` | This becomes the default for ALL pods without a priorityClassName |

### How preemption works
1. High-priority pod can't be scheduled (no resources available).
2. Scheduler looks for nodes where evicting lower-priority pods would free enough resources.
3. Lower-priority pods get evicted (graceful termination).
4. High-priority pod gets scheduled on that node.

### Useful commands
```bash
kubectl get priorityclasses
kubectl describe priorityclass high-priority

# Check which priority a pod is using
kubectl get pod high-priority-pod -o yaml | grep priority
```

---

## Node Affinity — schedule pods on specific nodes

- Node Affinity = "I want to run on nodes that match these labels."
- It's a more powerful replacement for `nodeSelector`.

### Two types
| Type | Behavior |
|------|----------|
| `requiredDuringSchedulingIgnoredDuringExecution` | HARD rule — pod stays Pending if no node matches |
| `preferredDuringSchedulingIgnoredDuringExecution` | SOFT rule — scheduler tries but can fallback |

### Example: Pod that requires SSD nodes
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: app
    image: nginx
```

### Example: Prefer zone-a but can run anywhere
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: zone-preferred-pod
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - zone-a
  containers:
  - name: app
    image: nginx
```

### Operators for matchExpressions
| Operator | Meaning |
|----------|---------|
| `In` | Label value must be one of the listed values |
| `NotIn` | Label value must NOT be in the list |
| `Exists` | Label key must exist (value doesn't matter) |
| `DoesNotExist` | Label key must NOT exist |
| `Gt` | Label value > specified value (numeric) |
| `Lt` | Label value < specified value (numeric) |

---

## Pod Affinity — schedule pods together

- Pod Affinity = "I want to run on the same node/zone as pods that match this label."
- Useful when two services need low latency between them (e.g., app + cache).

### Pod Affinity (from session)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: friend-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'echo I love my neighbors && sleep 3600']
```

**What this does:** `friend-pod` will ONLY be scheduled on a node where a pod with label `app: myapp` is already running.

---

## Pod Anti-Affinity — keep pods apart

- Anti-Affinity = "I do NOT want to run on the same node/zone as pods matching this label."
- Useful for spreading replicas across nodes for high availability.

### Example: Spread web pods across different nodes
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx
    image: nginx
```

**What this does:** No two pods with label `app: web` will be placed on the same node.

### Preferred Anti-Affinity (soft rule)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx
    image: nginx
```

---

## TopologyKey — what "same place" means

- `topologyKey` defines the scope of "co-located" or "spread apart."
- It refers to a **node label** that groups nodes together.

| TopologyKey | Meaning |
|-------------|---------|
| `kubernetes.io/hostname` | Same **node** (each node has unique hostname) |
| `topology.kubernetes.io/zone` | Same **availability zone** (multiple nodes in a zone) |
| `topology.kubernetes.io/region` | Same **region** |

### How it works with affinity
```yaml
# "Run on the same NODE as pods with app=myapp"
topologyKey: kubernetes.io/hostname

# "Run in the same ZONE as pods with app=myapp"
topologyKey: topology.kubernetes.io/zone
```

---

## MaxSkew — Topology Spread Constraints

- MaxSkew controls how evenly pods are distributed across topology domains.
- It's more fine-grained than anti-affinity for spreading pods.
- `maxSkew` = maximum allowed difference in pod count between any two topology domains.

### Example: Spread pods evenly across nodes
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: spread-pod
  labels:
    app: web
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
  containers:
  - name: nginx
    image: nginx
```

### Example: Spread across zones with soft constraint
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web
      containers:
      - name: nginx
        image: nginx
```

### Key fields
| Field | Meaning |
|-------|---------|
| `maxSkew: 1` | Max difference of 1 pod between any two domains |
| `maxSkew: 2` | Max difference of 2 pods (more relaxed) |
| `whenUnsatisfiable: DoNotSchedule` | Hard — pod stays Pending if can't satisfy |
| `whenUnsatisfiable: ScheduleAnyway` | Soft — schedule somewhere, try to minimize skew |

### Visual example (maxSkew: 1, topologyKey: hostname)
```
Node1: [web-1] [web-2]     ← 2 pods
Node2: [web-3] [web-4]     ← 2 pods
Node3: [web-5]             ← 1 pod

Difference between max(2) and min(1) = 1 ≤ maxSkew(1) ✅

If you try to add another pod to Node1:
Node1: [web-1] [web-2] [web-6]  ← 3 pods
Difference: 3 - 1 = 2 > maxSkew(1) ❌ NOT ALLOWED
```

---

## Taints and Tolerations — repel pods from nodes

- **Taint** = applied to a NODE to say "don't schedule here unless you tolerate me."
- **Toleration** = applied to a POD to say "I can handle that taint."
- Taints push pods away; tolerations allow specific pods through.

### Taint effects (from session)
| Effect | Behavior |
|--------|----------|
| `NoSchedule` | Pod will NOT be scheduled unless it tolerates the taint |
| `PreferNoSchedule` | Kubernetes will TRY to avoid scheduling here (soft) |
| `NoExecute` | Pod will be EVICTED if already running and doesn't tolerate it |

### Apply a taint (from session)
```bash
# Taint a node
kubectl taint node node01 type=maintenance:NoSchedule

# Another example
kubectl taint node node01 special=dedicated:NoSchedule
```

### Pod with toleration (from session)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-toleration
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "special"
    operator: "Equal"
    value: "dedicated"
    effect: "NoSchedule"
```

### Untaint — remove a taint from a node
```bash
# Remove taint (add a minus - at the end)
kubectl taint node node01 type=maintenance:NoSchedule-

# Remove taint by key only (removes all effects for that key)
kubectl taint node node01 special-

# Verify taint is removed
kubectl describe node node01 | grep Taint
```

### Toleration operators
| Operator | Meaning |
|----------|---------|
| `Equal` | key, value, and effect must all match the taint |
| `Exists` | key and effect must match (value is ignored) |

### Tolerate ALL taints (use with caution)
```yaml
tolerations:
- operator: "Exists"    # No key = matches everything
```

### Common use cases
| Scenario | Taint | Who tolerates |
|----------|-------|---------------|
| Master node | `node-role.kubernetes.io/control-plane:NoSchedule` | Only system pods |
| GPU node | `gpu=true:NoSchedule` | Only GPU workloads |
| Maintenance | `maintenance=true:NoExecute` | Nothing (evicts all pods) |
| Dedicated node | `team=data:NoSchedule` | Only data team's pods |

### Check existing taints
```bash
kubectl describe node node01 | grep -A5 Taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

---

## Quick comparison: Affinity vs Taints

| Feature | Affinity | Taints & Tolerations |
|---------|----------|---------------------|
| Applied to | Pod spec | Taints on Node, Tolerations on Pod |
| Direction | Pod says "I want to go here" | Node says "Stay away unless allowed" |
| Use case | Attract pods to nodes | Repel pods from nodes |
| Scope | Node labels / Pod labels | Node taints |

---

## Scheduling decision flow (how scheduler decides)

```
1. Filter nodes (taints, resource requests, node affinity hard rules)
         ↓
2. Score remaining nodes (preferences, anti-affinity, spread constraints)
         ↓
3. Pick highest-scoring node
         ↓
4. Bind pod to node
```
