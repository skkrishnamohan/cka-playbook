# CKA Exam Practice — Day 2: Core Skills 🟡

> **Difficulty**: Medium | **Questions**: 5 | **Target Time**: ~60 minutes
> **Goal**: Build speed on bread-and-butter Kubernetes tasks. These test core workload and service management.

---

## Question 1 — HPA with Stabilization Window

| | |
|---|---|
| **Weightage** | ~7% (estimated) |
| **Difficulty** | 🟡 Medium |
| **CKA Domain** | Workloads & Scheduling (15%) |
| **Estimated Exam Time** | ⏱️ 5–6 minutes |
| **Topics** | HPA, autoscaling, behavior/stabilization |
| **Related Notes** | [k8s-workloads.md](k8s-workloads.md) · [k8s-scheduling-and-affinity.md](k8s-scheduling-and-affinity.md) |

### 📋 Question

- Create a Horizontal Pod Scaler (HPA) named apache-server in the auto-scale namespace. This HPA must target existing deployment called apache-server in the auto-scale namespace.
- Set the HPA to aim for 50% CPU usage per pod. Configure it to have at least 1 pod and at max 4 pods.
- Also set the downscale stabilization window to 30 seconds.

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Create namespace and deployment (already exists in exam)
kubectl create namespace auto-scale
kubectl create deployment apache-server --image=nginx -n auto-scale
```

---

### ✅ Full Solution with Explanations

**Step 1 — Create the HPA using imperative command**

```bash
kubectl autoscale deployment apache-server -n auto-scale \
  --cpu-percent=50 --min=1 --max=4
```

> **WHY imperative first?** — The `kubectl autoscale` command quickly creates the HPA with CPU target, min, and max replicas. However, it doesn't support setting the `behavior` field, so we need to edit it afterwards.

**Step 2 — Verify the HPA was created**

```bash
kubectl get hpa -n auto-scale
```

Expected:
```
NAME            REFERENCE                  TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
apache-server   Deployment/apache-server   cpu: <unknown>/50%   1         4         1          10s
```

> **Note**: `<unknown>` for CPU is normal if metrics-server isn't installed. It won't affect scoring.

**Step 3 — Edit the HPA to add the stabilization window**

```bash
kubectl edit hpa apache-server -n auto-scale
```

Add the `behavior` section under `spec`:

```yaml
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 30
```

> **WHY stabilization window?** — The stabilization window prevents the HPA from rapidly scaling down pods when metrics fluctuate. A 30-second window means the HPA waits 30 seconds of consistently low metrics before removing pods. This prevents "flapping" (scaling up and down repeatedly).

> **WHERE does it go in the YAML?** — Under `spec`, at the same level as `minReplicas`, `maxReplicas`, and `metrics`:
> ```yaml
> spec:
>   minReplicas: 1
>   maxReplicas: 4
>   metrics:
>     - type: Resource
>       resource:
>         name: cpu
>         target:
>           type: Utilization
>           averageUtilization: 50
>   behavior:              # ← Add this block
>     scaleDown:
>       stabilizationWindowSeconds: 30
> ```

**Step 4 — Verify the complete HPA**

```bash
kubectl get hpa apache-server -n auto-scale -o yaml
```

Check that:
- `maxReplicas: 4`
- `minReplicas: 1`
- `averageUtilization: 50`
- `behavior.scaleDown.stabilizationWindowSeconds: 30`

---

### ⚡ Exam Speed Strategy (Target: 3 minutes)

```bash
# Step 1: Create HPA imperatively
kubectl autoscale deployment apache-server -n auto-scale --cpu-percent=50 --min=1 --max=4

# Step 2: Edit to add behavior
kubectl edit hpa apache-server -n auto-scale
# In vi, type /spec to find spec section, then add:
#   behavior:
#     scaleDown:
#       stabilizationWindowSeconds: 30

# Step 3: Quick verify
kubectl get hpa -n auto-scale
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| HPA overview | https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/ |
| HPA walkthrough | https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/ |
| HPA behavior/scaling policies | https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#configurable-scaling-behavior |
| **Search keyword** | `horizontal pod autoscale` or `HPA behavior` |

---

### 💡 Tips & Memory Aids

- 🧠 **Remember**: `behavior` has two sub-fields: `scaleDown` and `scaleUp`. The question asks for `scaleDown`
- ⚠️ **Common mistake**: Setting `stabilizationWindowSeconds` at the wrong YAML level. It goes under `spec.behavior.scaleDown`, NOT under `spec`
- ⚠️ **Original notes had 60 seconds** — the question says **30 seconds**. Always read the question carefully!
- 🔑 **Default stabilization**: scaleDown default is 300 seconds (5 min), scaleUp default is 0 seconds
- 📝 **No imperative way to set behavior** — you must use `kubectl edit` or apply YAML

---

### ✅ Verification Checklist

- [ ] `kubectl get hpa -n auto-scale` shows apache-server targeting 50% CPU, min=1, max=4
- [ ] `kubectl get hpa apache-server -n auto-scale -o yaml | grep stabilization` shows 30
- [ ] HPA targets the correct deployment (apache-server)

---
---

## Question 2 — Expose Deployment via NodePort Service

| | |
|---|---|
| **Weightage** | 10% |
| **Difficulty** | 🟡 Medium |
| **CKA Domain** | Services & Networking (20%) |
| **Estimated Exam Time** | ⏱️ 4–5 minutes |
| **Topics** | Deployments, container ports, Services, NodePort |
| **Related Notes** | [k8s-workloads.md](k8s-workloads.md) · [k8s-ingress-gateway.md](k8s-ingress-gateway.md) |

### 📋 Question

- Reconfigure the existing deployment front-end in namespace spline-reticulator to expose port 80/tcp of the existing container nginx.
- Create a new service named front-end-svc exposing the container port 80/tcp.
- Configure the new service to also expose the individual pod via a NodePort.

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Create namespace and deployment (already exists in exam)
kubectl create ns spline-reticulator
kubectl create deployment front-end --image=nginx -n spline-reticulator
```

---

### ✅ Full Solution with Explanations

**Step 1 — Add containerPort to the existing deployment**

```bash
kubectl edit deployment front-end -n spline-reticulator
```

Find the `containers` section and add the `ports` field:

```yaml
      containers:
      - name: nginx
        image: nginx
        ports:                    # ← Add this block
        - containerPort: 80
          protocol: TCP
```

> **WHY add containerPort?** — While containers can receive traffic on any port they listen on (containerPort is informational), adding it:
> 1. Documents which ports the container uses
> 2. Is required by the question ("expose port 80/tcp of the existing container")
> 3. Helps `kubectl expose` auto-detect the port

**Alternative — Use `kubectl patch`:**

```bash
kubectl patch deployment front-end -n spline-reticulator --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/ports", "value": [{"containerPort": 80, "protocol": "TCP"}]}]'
```

**Step 2 — Create the NodePort service**

Option A — Imperative (fastest):
```bash
kubectl expose deployment front-end -n spline-reticulator \
  --name=front-end-svc --port=80 --target-port=80 --type=NodePort
```

> **WHY `kubectl expose`?** — It's the fastest way to create a service. It automatically sets the selector to match the deployment's labels.

Option B — YAML (if you need more control):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: front-end-svc
  namespace: spline-reticulator
spec:
  type: NodePort
  selector:
    app: front-end
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

> **WHY each field:**
> - `type: NodePort` — Exposes the service on a port on every node (30000–32767 range)
> - `selector: app: front-end` — Must match the deployment's pod labels
> - `port: 80` — The service port (how other pods access it internally)
> - `targetPort: 80` — The container port to forward traffic to

**Step 3 — Verify**

```bash
# Check the service
kubectl get svc front-end-svc -n spline-reticulator

# Check the endpoints (should show pod IPs)
kubectl get endpoints front-end-svc -n spline-reticulator

# Check the deployment has the port
kubectl get deployment front-end -n spline-reticulator -o yaml | grep -A 3 "ports:"
```

---

### ⚡ Exam Speed Strategy (Target: 3 minutes)

```bash
# Step 1: Edit deployment to add port
kubectl edit deploy front-end -n spline-reticulator
# Add ports: [{containerPort: 80}] under the container

# Step 2: Create service imperatively
kubectl expose deployment front-end -n spline-reticulator \
  --name=front-end-svc --port=80 --target-port=80 --type=NodePort

# Step 3: Verify
kubectl get svc -n spline-reticulator
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| Service concepts | https://kubernetes.io/docs/concepts/services-networking/service/ |
| NodePort section | https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport |
| kubectl expose | https://kubernetes.io/docs/reference/kubectl/generated/kubectl_expose/ |
| **Search keyword** | `service` → NodePort section |

---

### 💡 Tips & Memory Aids

- 🧠 **Service types order**: ClusterIP (default) → NodePort → LoadBalancer → ExternalName
- ⚠️ **Common mistake**: Wrong selector — always verify with `kubectl get endpoints` that the service found the pods
- ⚠️ **Don't use `kubectl create service nodeport`** — it sets the selector to `app: front-end-svc` (the service name, not the deployment name!) Use `kubectl expose` instead.
- 🔑 **Quick debug**: If endpoints show `<none>`, the selector doesn't match the pod labels
- 📝 **NodePort range**: 30000–32767 (auto-assigned unless you specify one)

---

### ✅ Verification Checklist

- [ ] Deployment has `containerPort: 80` in the container spec
- [ ] `kubectl get svc -n spline-reticulator` shows `front-end-svc` with type `NodePort`
- [ ] `kubectl get endpoints -n spline-reticulator` shows pod IPs (not `<none>`)

---
---

## Question 3 — Add Sidecar Container with Shared Volume

| | |
|---|---|
| **Weightage** | 15% |
| **Difficulty** | 🟡 Medium |
| **CKA Domain** | Workloads & Scheduling (15%) |
| **Estimated Exam Time** | ⏱️ 5–7 minutes |
| **Topics** | Multi-container pods, sidecar pattern, emptyDir volumes |
| **Related Notes** | [k8s-pod-patterns.md](k8s-pod-patterns.md) · [k8s-configmaps-secrets-volumes.md](k8s-configmaps-secrets-volumes.md) |

### 📋 Question

- Update the existing deployment synergy-leverager, adding a co-located container named sidecar using the busybox:stable image to the existing pod.
- The new co-located container has to run the following command:
  `/bin/sh -c "tail -n+1 -f /var/log/synergy-leverager.log"`
- Use a volume mounted at /var/log to make the log file synergy-leverager.log available to the co-located container.

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Create the deployment (already exists in exam)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergy-leverager
  labels:
    app: synergy-leverager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: synergy-leverager
  template:
    metadata:
      labels:
        app: synergy-leverager
    spec:
      containers:
        - name: synergy-leverager
          image: alpine:latest
          command: ['/bin/sh', '-c', 'while true; do echo "logging" >> /var/log/synergy-leverager.log; sleep 1; done']
          volumeMounts:
            - name: data
              mountPath: /var/log
      volumes:
        - name: data
          emptyDir: {}
EOF
```

---

### ✅ Full Solution with Explanations

**Step 1 — Edit the existing deployment**

```bash
kubectl edit deployment synergy-leverager
```

> **WHY edit instead of apply?** — The deployment already exists. We just need to add a second container and ensure the volume setup is correct.

**Step 2 — Add the sidecar container**

In the `containers` section, add the sidecar after the existing container:

```yaml
    spec:
      containers:
        - name: synergy-leverager          # ← Existing container
          image: alpine:latest
          command: ['/bin/sh', '-c', 'while true; do echo "logging" >> /var/log/synergy-leverager.log; sleep 1; done']
          volumeMounts:
            - name: data
              mountPath: /var/log
        - name: sidecar                    # ← NEW: Add this container
          image: busybox:stable
          command: ['/bin/sh', '-c', 'tail -n+1 -f /var/log/synergy-leverager.log']
          volumeMounts:
            - name: data
              mountPath: /var/log
      volumes:
        - name: data
          emptyDir: {}
```

> **WHY this works — the Sidecar Pattern:**
> 1. Both containers share the same `emptyDir` volume named `data`
> 2. Main container writes logs to `/var/log/synergy-leverager.log`
> 3. Sidecar container tails (reads) the same file from the shared volume
> 4. `emptyDir` is perfect here — it's ephemeral storage shared between containers in the same pod

> **WHY `tail -n+1 -f`?**
> - `-n+1` — Start from line 1 (beginning of file)
> - `-f` — Follow the file (keep watching for new lines)
> - Together: read all existing content then continue watching for new content

> **WHY `emptyDir`?**
> - Created when a Pod is assigned to a Node
> - Exists as long as the Pod runs on that Node
> - Shared between all containers in the same Pod
> - No data persistence needed — this is just for log streaming

**Step 3 — Verify the deployment rolled out**

```bash
kubectl rollout status deployment synergy-leverager
```

**Step 4 — Verify both containers are running**

```bash
kubectl get pods -l app=synergy-leverager
```

Expected: `READY 2/2`

**Step 5 — Verify the sidecar can read logs**

```bash
# Check sidecar logs — should show "logging" messages
kubectl logs deployment/synergy-leverager -c sidecar

# Or exec into the pod to verify
kubectl exec -it $(kubectl get pod -l app=synergy-leverager -o name) -c synergy-leverager -- cat /var/log/synergy-leverager.log
```

---

### ⚡ Exam Speed Strategy (Target: 4 minutes)

```bash
# Step 1: Edit the deployment
kubectl edit deployment synergy-leverager

# In vi: Go to the containers section, position cursor after the last volumeMount of the existing container
# Type 'o' to add a new line and paste the sidecar container:
#     - name: sidecar
#       image: busybox:stable
#       command: ['/bin/sh', '-c', 'tail -n+1 -f /var/log/synergy-leverager.log']
#       volumeMounts:
#         - name: data
#           mountPath: /var/log

# Make sure volumes section has emptyDir (add if missing):
#   volumes:
#     - name: data
#       emptyDir: {}

# Save and verify
kubectl get pods -l app=synergy-leverager
# Should show 2/2 Ready
```

> **Memory trick**: Sidecar needs 3 things — **N**ame, **I**mage, **C**ommand + same **V**olume mount = **NICV**

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| Sidecar containers | https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/ |
| Init containers | https://kubernetes.io/docs/concepts/workloads/pods/init-containers/ |
| Volumes - emptyDir | https://kubernetes.io/docs/concepts/storage/volumes/#emptydir |
| **Search keyword** | `sidecar containers` or `multi-container pod` |

---

### 💡 Tips & Memory Aids

- 🧠 **Sidecar = same pod, same volumes, different container** — they share the filesystem via volumes
- ⚠️ **Common mistake**: Forgetting to add `volumeMounts` to the sidecar — without it, the sidecar can't see the log file
- ⚠️ **Common mistake**: Using a different volume name in the sidecar — must match exactly (`data` in this case)
- ⚠️ **YAML indentation**: The sidecar container must be at the same indent level as the main container (under `containers:` list)
- 🔑 **Verify with logs**: `kubectl logs <pod> -c sidecar` should show the log content
- 📝 **If volume doesn't exist yet**: Add both the `volumes` section AND `volumeMounts` to both containers

---

### ✅ Verification Checklist

- [ ] `kubectl get pods` shows `2/2` Ready for the synergy-leverager pod
- [ ] `kubectl logs <pod> -c sidecar` shows "logging" messages
- [ ] Both containers mount the same volume at `/var/log`

---
---

## Question 10 — Fix Pod Resources for WordPress

| | |
|---|---|
| **Weightage** | 9% |
| **Difficulty** | 🟡 Medium |
| **CKA Domain** | Workloads & Scheduling (15%) |
| **Estimated Exam Time** | ⏱️ 6–8 minutes |
| **Topics** | Resource requests/limits, node capacity, deployment scaling |
| **Related Notes** | [k8s-resource-governance.md](k8s-resource-governance.md) · [k8s-scheduling-and-affinity.md](k8s-scheduling-and-affinity.md) |

### 📋 Question

You manage a WordPress application. Some pods not up and running. Adjust all Pod resource requests:

Divide node resources evenly across 3 pods.

Give each pod fair share of CPU and memory.

Add overhead to keep node stable.

Use exact same requests for both containers and init containers.

Scale down WordPress deployment to 0 replicas while updating resource requests.

After updates:

WordPress keeps 3 replicas.

All pods are running and ready.

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Create the WordPress deployment (already exists in exam)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app.kubernetes.io/name: wordpress
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: wordpress
  template:
    metadata:
      labels:
        app.kubernetes.io/name: wordpress
    spec:
      initContainers:
        - name: init-wordpress
          image: busybox:1.28
          command: ['sh', '-c', "echo initializing wordpress..."]
      containers:
        - name: wordpress
          image: busybox:1.28
          command: ['sh', '-c', 'echo WordPress is running! && sleep 3600']
EOF
```

---

### ✅ Full Solution with Explanations

**Step 1 — Check node capacity**

```bash
kubectl describe node | grep -A 6 "Allocatable:"
```

> **WHY?** — To divide resources evenly across 3 pods, you need to know how much CPU and memory the node has. Look at `Allocatable` (not `Capacity`) — this is what's actually available for pods.

Example output:
```
Allocatable:
  cpu:                2
  memory:             4Gi
  ephemeral-storage:  48Gi
  pods:               110
```

**Step 2 — Calculate per-pod resources**

```
Total CPU: 2000m (2 cores)
Total Memory: 4096Mi (4Gi)

Per pod (3 pods): 
  CPU: 2000m / 3 ≈ 666m → use ~200m (with overhead for system)
  Memory: 4096Mi / 3 ≈ 1365Mi → use ~128Mi (with overhead for system)
```

> **WHY leave overhead?** — The node runs system components (kubelet, kube-proxy, containerd, OS) that need resources. If you allocate everything to pods, the node becomes unstable. **Reserve ~30-40% for system overhead.**

> **Exam tip**: The exam may give you hints about exact values. Use values that let all 3 pods schedule without exceeding node capacity.

**Step 3 — Scale down to 0 replicas**

```bash
kubectl scale deployment wordpress --replicas=0
```

> **WHY scale down first?** — The question explicitly asks to scale down while updating. This prevents rolling update issues when changing resource requests.

**Step 4 — Edit the deployment to add resources**

```bash
kubectl edit deployment wordpress
```

Add `resources` to BOTH the init container AND the main container:

```yaml
    spec:
      initContainers:
        - name: init-wordpress
          image: busybox:1.28
          command: ['sh', '-c', "echo initializing wordpress..."]
          resources:
            requests:
              cpu: "100m"
              memory: "100Mi"
            limits:
              cpu: "200m"
              memory: "200Mi"
      containers:
        - name: wordpress
          image: busybox:1.28
          command: ['sh', '-c', 'echo WordPress is running! && sleep 3600']
          resources:
            requests:
              cpu: "100m"
              memory: "100Mi"
            limits:
              cpu: "200m"
              memory: "200Mi"
```

> **WHY same requests for init and main containers?** — The question explicitly requires this. In Kubernetes, the effective resource request for a pod is `max(sum of all containers, max of all init containers)`. Using the same values simplifies calculation.

> **WHY requests AND limits?**
> - `requests` — Minimum guaranteed resources. Scheduler uses this to place pods.
> - `limits` — Maximum allowed. Container gets OOM-killed if it exceeds memory limit.

**Step 5 — Scale back up to 3 replicas**

```bash
kubectl scale deployment wordpress --replicas=3
```

**Step 6 — Verify all pods are running**

```bash
kubectl get pods -l app.kubernetes.io/name=wordpress
```

Expected: All 3 pods should be `Running` and `1/1 Ready`.

If pods are `Pending`, check:
```bash
kubectl describe pod <pending-pod-name>
```

> Look for `Insufficient cpu` or `Insufficient memory` → reduce the resource values.

---

### ⚡ Exam Speed Strategy (Target: 5 minutes)

```bash
# Check node resources
kubectl top node    # or kubectl describe node | grep -A5 Allocatable

# Scale down
kubectl scale deploy wordpress --replicas=0

# Edit and add resources
kubectl edit deploy wordpress
# Add resources block to both init and main containers
# Use conservative values: 100m CPU, 100Mi memory

# Scale back up
kubectl scale deploy wordpress --replicas=3

# Verify
kubectl get pods
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| Resource Management | https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/ |
| LimitRange | https://kubernetes.io/docs/concepts/policy/limit-range/ |
| **Search keyword** | `manage resources containers` |

---

### 💡 Tips & Memory Aids

- 🧠 **Resources format**: CPU in millicores (`100m` = 0.1 core), Memory in Mi (`128Mi`)
- ⚠️ **Common mistake**: Setting resources too high — pods become `Pending` because the scheduler can't find a node with enough capacity
- ⚠️ **Common mistake**: Forgetting init container resources — question says "exact same requests for both"
- 🔑 **Quick formula**: `Node Allocatable / (number of pods + 1 for overhead)` ≈ per-pod request
- 📝 **If pods are Pending**: Reduce requests. Start low (100m/100Mi) and increase if needed

---

### ✅ Verification Checklist

- [ ] `kubectl get pods` shows all 3 WordPress pods as `Running` and `Ready`
- [ ] Both init container and main container have the same resource requests
- [ ] `kubectl describe pod <pod>` shows no `Insufficient` events

---
---

## Question 21 — Create Ingress for Echo Service

| | |
|---|---|
| **Weightage** | ~6% (estimated) |
| **Difficulty** | 🟡 Medium |
| **CKA Domain** | Services & Networking (20%) |
| **Estimated Exam Time** | ⏱️ 5–6 minutes |
| **Topics** | Ingress, path-based routing, nginx annotations |
| **Related Notes** | [k8s-ingress-gateway.md](k8s-ingress-gateway.md) · [k8s-workloads.md](k8s-workloads.md) |

### 📋 Question

Namespace: sound repeater
Exposing Service Echoserver-service on http://example.org/echo using service port 8080

Expose the service echo-server in the given name space Create an nginx ingress for aabove deployment and service for the host http://www.example.org/echo over port 80 curl -o /dev/null -s -w "%{http_code}\n" http://example.org/echo.

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Install ingress-nginx controller (already installed in exam)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=120s

# Create namespace
kubectl create namespace sound-repeater

# Create the echo-server deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server
  namespace: sound-repeater
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo-server
  template:
    metadata:
      labels:
        app: echo-server
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo
        args:
          - "-text=echo"
        ports:
          - containerPort: 5678
EOF

# Create the service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: echo-server
  namespace: sound-repeater
spec:
  selector:
    app: echo-server
  ports:
    - name: http
      port: 8080
      targetPort: 5678
EOF

# Add hosts entry
echo "127.0.0.1 example.org" | sudo tee -a /etc/hosts
```

---

### ✅ Full Solution with Explanations

**Step 1 — Create the Ingress resource**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress
  namespace: sound-repeater
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.org
    http:
      paths:
      - path: /echo
        pathType: Prefix
        backend:
          service:
            name: echo-server
            port:
              number: 8080
```

> **WHY each field:**
> - `host: example.org` — Only matches requests to this hostname
> - `path: /echo` — Only matches requests to `/echo` path
> - `pathType: Prefix` — Matches `/echo`, `/echo/`, `/echo/anything`
> - `service.name: echo-server` — The backend service to route traffic to
> - `service.port.number: 8080` — The service port (NOT the container port)
> - `nginx.ingress.kubernetes.io/rewrite-target: /` — Rewrites `/echo` to `/` before forwarding to the backend. Without this, the backend would receive `/echo` as the path.

**Step 2 — Apply the Ingress**

```bash
kubectl apply -f ingress.yaml
```

**Step 3 — Verify**

```bash
# Check the ingress resource
kubectl get ingress -n sound-repeater

# Check the ingress details
kubectl describe ingress echo-ingress -n sound-repeater

# Test with curl (verification command from question)
curl -o /dev/null -s -w "%{http_code}\n" http://example.org/echo
```

Expected: HTTP status `200`

> **WHY the curl command works this way:**
> - `-o /dev/null` — Discard response body
> - `-s` — Silent mode (no progress bar)
> - `-w "%{http_code}\n"` — Print only the HTTP status code
> - If you get `200` — success! If `404` or `503` — check service/ingress configuration

---

### ⚡ Exam Speed Strategy (Target: 3 minutes)

```bash
# Generate Ingress YAML fast
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress
  namespace: sound-repeater
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.org
    http:
      paths:
      - path: /echo
        pathType: Prefix
        backend:
          service:
            name: echo-server
            port:
              number: 8080
EOF

# Verify
curl -o /dev/null -s -w "%{http_code}\n" http://example.org/echo
```

> **Tip**: The Ingress YAML structure is in the K8s docs under "Ingress" → "Simple fanout" example. Copy and modify.

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| Ingress concepts | https://kubernetes.io/docs/concepts/services-networking/ingress/ |
| Ingress NGINX annotations | https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/ |
| **Search keyword** | `ingress` → find the "Simple fanout" or "Name-based virtual hosting" example |

---

### 💡 Tips & Memory Aids

- 🧠 **Ingress YAML structure**: `rules` → `host` → `http` → `paths` → `path` + `backend` → `service`
- ⚠️ **Common mistake**: Using `targetPort` instead of service `port` in the backend — Ingress references the **service port**, not the container port
- ⚠️ **Common mistake**: Wrong namespace — Ingress must be in the SAME namespace as the service
- ⚠️ **Common mistake**: Forgetting the `rewrite-target` annotation — without it, the backend receives `/echo` instead of `/`
- 🔑 **`pathType` options**: `Prefix` (matches path prefix), `Exact` (exact match), `ImplementationSpecific`
- 📝 **Port confusion**: Container listens on 5678, Service exposes 8080→5678, Ingress uses service port 8080

---

### ✅ Verification Checklist

- [ ] `kubectl get ingress -n sound-repeater` shows the ingress with correct host and path
- [ ] `kubectl describe ingress -n sound-repeater echo-ingress` shows correct backend
- [ ] `curl` returns status code 200

---
---

## 📊 Day 2 Summary

| Question | Topic | Target Time | Key Command |
|----------|-------|-------------|-------------|
| Q1 | HPA + stabilization | 3 min | `kubectl autoscale` + `kubectl edit` |
| Q2 | Deployment port + NodePort | 3 min | `kubectl expose --type=NodePort` |
| Q3 | Sidecar + shared volume | 4 min | `kubectl edit deploy` + add container |
| Q10 | Pod resource requests | 5 min | `kubectl describe node` + `kubectl edit` |
| Q21 | Ingress path routing | 3 min | Ingress YAML with rewrite-target |
| **Total** | | **~18 min** | |

> 💪 **These 5 questions cover workloads, services, and resource management — the bread-and-butter of CKA!**

---

**Previous**: [Day 1 — Warmup 🟢](exam-day1-warmup.md) | **Next**: [Day 3 — Networking 🟠](exam-day3-networking.md)
