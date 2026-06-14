# CKA Exam Practice — Day 4: Advanced 🔴

> **Difficulty**: Hard | **Questions**: 5 | **Target Time**: ~90 minutes
> **Goal**: Tackle the hardest questions. These require multi-step solutions with YAML precision. Practice until they feel routine.

---

## Question 4 — PriorityClass + Patch Deployment

| | |
|---|---|
| **Weightage** | 11% |
| **Difficulty** | 🔴 Hard |
| **CKA Domain** | Workloads & Scheduling (15%) |
| **Estimated Exam Time** | ⏱️ 6–8 minutes |
| **Topics** | PriorityClass, preemption, kubectl patch |
| **Related Notes** | [k8s-scheduling-and-affinity.md](k8s-scheduling-and-affinity.md) · [k8s-resource-governance.md](k8s-resource-governance.md) |

### 📋 Question

Perform the following tasks:

Create a new Priority Class named high-priority for user workloads, with a value one less than the highest existing user-defined priority class value.

Patch the existing busybox-logger Deployment in the priority namespace to use the high-priority class.

Note — Pods from other Deployments in the priority namespace should be evicted.

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Create namespace and deployment (already exist in exam)
kubectl create ns priority
kubectl create deployment busybox-logger -n priority --image=busybox:stable -- sh -c "while true; do echo log; sleep 5; done"

# Create an existing priority class (already exists in exam)
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-existing
value: 1000
preemptionPolicy: Never
globalDefault: false
description: "Existing priority class"
EOF

# Create another deployment to be evicted
kubectl create deployment other-app -n priority --image=nginx
```

---

### ✅ Full Solution with Explanations

**Step 1 — Find the highest existing user-defined priority class value**

```bash
kubectl get priorityclass
```

Example output:
```
NAME                      VALUE        GLOBAL-DEFAULT   AGE
high-priority-existing    1000         false            5m
system-cluster-critical   2000000000   false            1h
system-node-critical      2000001000   false            1h
```

> **WHY filter user-defined?** — `system-cluster-critical` and `system-node-critical` are built-in system classes with very high values (2 billion). Ignore these — they're for system components only. The highest **user-defined** priority is `1000` in this example.

**Step 2 — Create the PriorityClass with value = highest - 1**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 999
preemptionPolicy: PreemptLowerPriority
globalDefault: false
description: "High priority class for user workloads"
```

> **WHY each field:**
> - `value: 999` — One less than the highest existing user-defined value (1000). Higher value = higher priority.
> - `preemptionPolicy: PreemptLowerPriority` — **CRITICAL**: This enables preemption. When a pod with this priority can't be scheduled, it will EVICT lower-priority pods to make room. The question says "Pods from other Deployments should be evicted."
> - `globalDefault: false` — Don't make this the default for all pods. Only pods that explicitly reference it get this priority.

> **WHY `PreemptLowerPriority` and NOT `Never`?**
> - `PreemptLowerPriority` — The pod CAN evict lower-priority pods to get scheduled
> - `Never` — The pod waits in queue but never evicts other pods
> - The question says other pods "should be evicted" → we need `PreemptLowerPriority`

```bash
kubectl apply -f priorityclass.yaml
```

**Step 3 — Patch the deployment to use the priority class**

```bash
kubectl patch deployment busybox-logger -n priority --type='merge' \
  -p '{"spec": {"template": {"spec": {"priorityClassName": "high-priority"}}}}'
```

> **WHY `kubectl patch`?** — It's faster than `kubectl edit` for single-field changes. The patch modifies the pod template to include `priorityClassName`.

> **WHY the nested path?** — `priorityClassName` lives in the pod spec:
> ```
> spec (deployment) → template → spec (pod) → priorityClassName
> ```

**Alternative — kubectl edit:**
```bash
kubectl edit deployment busybox-logger -n priority
```
Add under `spec.template.spec`:
```yaml
      priorityClassName: high-priority
```

**Step 4 — Verify the deployment was updated**

```bash
kubectl get deployment busybox-logger -n priority -o yaml | grep priorityClassName
```

Expected: `priorityClassName: high-priority`

**Step 5 — Verify pods from other deployments are evicted**

```bash
kubectl get pods -n priority
```

> **Note**: Eviction happens automatically when the scheduler needs resources. If the node has enough capacity, pods might not be evicted (they coexist). The priority class still matters if resources become scarce.

---

### ⚡ Exam Speed Strategy (Target: 4 minutes)

```bash
# Step 1: Check existing priority classes
kubectl get priorityclass | grep -v system

# Step 2: Create new priority class (value = highest - 1)
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 999
preemptionPolicy: PreemptLowerPriority
globalDefault: false
description: "High priority for user workloads"
EOF

# Step 3: Patch the deployment
kubectl patch deploy busybox-logger -n priority --type='merge' \
  -p '{"spec":{"template":{"spec":{"priorityClassName":"high-priority"}}}}'

# Step 4: Verify
kubectl get pods -n priority
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| Pod Priority | https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/ |
| PriorityClass | https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#priorityclass |
| kubectl patch | https://kubernetes.io/docs/reference/kubectl/generated/kubectl_patch/ |
| **Search keyword** | `priority class` or `pod priority preemption` |

---

### 💡 Tips & Memory Aids

- 🧠 **Priority value**: Higher number = higher priority. System classes are ~2 billion, user classes are much lower.
- 🧠 **Preemption policy**: `PreemptLowerPriority` = CAN evict, `Never` = WON'T evict
- ⚠️ **Common mistake**: Using `preemptionPolicy: Never` when the question says "pods should be evicted"
- ⚠️ **Common mistake**: Setting value equal to (not one less than) the existing highest
- 🔑 **Ignore system priority classes**: `system-cluster-critical` and `system-node-critical` are built-in — focus only on user-defined ones
- 📝 **`kubectl patch` JSON format**: `'{"spec":{"template":{"spec":{"priorityClassName":"name"}}}}'`

---

### ✅ Verification Checklist

- [ ] `kubectl get priorityclass high-priority` exists with correct value
- [ ] `preemptionPolicy` is `PreemptLowerPriority`
- [ ] `kubectl get deploy busybox-logger -n priority -o yaml | grep priorityClassName` shows `high-priority`
- [ ] Pods are running in the `priority` namespace

---
---

## Question 7 — Restore MariaDB Deployment with PVC

| | |
|---|---|
| **Weightage** | 13% |
| **Difficulty** | 🔴 Hard |
| **CKA Domain** | Storage (10%) |
| **Estimated Exam Time** | ⏱️ 7–9 minutes |
| **Topics** | PV, PVC, Deployment volumes, data persistence |
| **Related Notes** | [k8s-storage-pv-pvc.md](k8s-storage-pv-pvc.md) · [k8s-configmaps-secrets-volumes.md](k8s-configmaps-secrets-volumes.md) |

### 📋 Question

A MariaDB Deployment in the mariadb namespace has been deleted. Restore it ensuring data persistence:

Create a PVC named mariadb in the mariadb namespace: Storage 250Mi.

Note — Use the existing retained PersistentVolume.

Edit the mariadb-deployment.yaml to use the new PVC.

Apply the updated Deployment to the cluster.

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Create namespace (already exists in exam)
kubectl create namespace mariadb

# Create PV with Retain policy (already exists in exam)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mariadb-pv
spec:
  capacity:
    storage: 250Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /tmp/mariadb
EOF
```

---

### ✅ Full Solution with Explanations

**Step 1 — Check the existing PV**

```bash
kubectl get pv
```

> **WHY?** — You need to match your PVC to the existing PV. Check:
> - `storageClassName` — your PVC must use the same class
> - `capacity` — your PVC must request ≤ the PV capacity
> - `accessModes` — must match
> - `STATUS` — should be `Available` or `Released`

Example output:
```
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      STORAGECLASS   AGE
mariadb-pv  250Mi      RWO            Retain           Available   standard       5m
```

> **If PV status is `Released`** (previously bound to a deleted PVC):
> ```bash
> # Remove the old claim reference to make it Available again
> kubectl patch pv mariadb-pv -p '{"spec":{"claimRef": null}}'
> ```
> **WHY?** — A `Released` PV still has a reference to the old PVC. Removing `claimRef` makes it `Available` for binding.

**Step 2 — Create the PVC**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb
  namespace: mariadb
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 250Mi
```

> **WHY each field must match the PV:**
> - `accessModes: ReadWriteOnce` — Must match the PV's access mode
> - `storageClassName: standard` — Must match the PV's storageClassName for binding
> - `storage: 250Mi` — Must be ≤ the PV's capacity (250Mi)

```bash
kubectl apply -f pvc.yaml
```

**Step 3 — Verify PVC is bound**

```bash
kubectl get pvc -n mariadb
```

Expected:
```
NAME      STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mariadb   Bound    mariadb-pv   250Mi      RWO            standard       5s
```

> **WHY check?** — If the PVC is `Pending`, the PV didn't bind. Common reasons:
> - StorageClass mismatch
> - Access mode mismatch
> - PV status is `Released` (need to remove `claimRef`)

**Step 4 — Create/Edit the Deployment to use the PVC**

The question says to edit `mariadb-deployment.yaml`. In the exam, this file likely exists on disk:

```bash
# Find the file
ls ~/mariadb-deployment.yaml
# or
find / -name "mariadb-deployment.yaml" 2>/dev/null
```

Edit it to include the volume and volumeMount:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
  namespace: mariadb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
        - name: mariadb
          image: mariadb:10.5
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: rootpassword
          ports:
            - containerPort: 3306
          volumeMounts:              # ← Add this
            - name: mariadb-storage
              mountPath: /var/lib/mysql
      volumes:                       # ← Add this
        - name: mariadb-storage
          persistentVolumeClaim:
            claimName: mariadb
```

> **WHY `/var/lib/mysql`?** — This is MariaDB/MySQL's default data directory. Mounting the PVC here ensures database files are stored on the persistent volume.

> **WHY `claimName: mariadb`?** — This references the PVC we created in Step 2. The volume name (`mariadb-storage`) is internal and can be anything — just must match between `volumes` and `volumeMounts`.

**Step 5 — Apply the deployment**

```bash
kubectl apply -f mariadb-deployment.yaml
```

**Step 6 — Verify everything is working**

```bash
# Check deployment
kubectl get deployment -n mariadb

# Check pod is running
kubectl get pods -n mariadb

# Check PVC is still bound
kubectl get pvc -n mariadb

# Check detailed deployment info
kubectl describe deployment mariadb -n mariadb
```

---

### ⚡ Exam Speed Strategy (Target: 5 minutes)

```bash
# Step 1: Check existing PV
kubectl get pv

# Step 2: If PV is Released, clear the claim
kubectl patch pv mariadb-pv -p '{"spec":{"claimRef": null}}'

# Step 3: Create PVC (match PV's storageClass and accessModes)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb
  namespace: mariadb
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 250Mi
EOF

# Step 4: Edit the deployment file and add volumes + volumeMounts
vi ~/mariadb-deployment.yaml
# Add:
#   volumeMounts:
#     - name: data
#       mountPath: /var/lib/mysql
# And:
#   volumes:
#     - name: data
#       persistentVolumeClaim:
#         claimName: mariadb

# Step 5: Apply
kubectl apply -f ~/mariadb-deployment.yaml

# Step 6: Verify
kubectl get pods -n mariadb
kubectl get pvc -n mariadb
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| PersistentVolumeClaim | https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims |
| Volumes in Pods | https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes |
| **Search keyword** | `persistent volume claim` → find the PVC YAML example |

---

### 💡 Tips & Memory Aids

- 🧠 **PVC must match PV on**: `storageClassName`, `accessModes`, and `capacity` (≤ PV)
- 🧠 **Volume binding**: `volumes` section references the PVC → `volumeMounts` mounts it into the container
- ⚠️ **Common mistake**: PV is `Released` and won't bind — patch to remove `claimRef`
- ⚠️ **Common mistake**: Wrong `storageClassName` — always check the PV first with `kubectl get pv`
- ⚠️ **Common mistake**: Forgetting to add BOTH `volumes` and `volumeMounts` in the deployment
- 🔑 **MariaDB/MySQL data path**: Always `/var/lib/mysql`
- 📝 **Reclaim policies**: `Retain` (keeps data), `Delete` (removes PV), `Recycle` (deprecated)

---

### ✅ Verification Checklist

- [ ] `kubectl get pv` shows PV bound to the PVC
- [ ] `kubectl get pvc -n mariadb` shows `Bound` status
- [ ] `kubectl get pods -n mariadb` shows pod as `Running`
- [ ] Deployment YAML has `volumes` and `volumeMounts` sections

---
---

## Question 8 — Migrate Ingress to Gateway API

| | |
|---|---|
| **Weightage** | 12% |
| **Difficulty** | 🔴 Hard |
| **CKA Domain** | Services & Networking (20%) |
| **Estimated Exam Time** | ⏱️ 8–10 minutes |
| **Topics** | Gateway API, Gateway, HTTPRoute, TLS, migration from Ingress |
| **Related Notes** | [k8s-ingress-gateway.md](k8s-ingress-gateway.md) |

### 📋 Question

Migrate an existing web application from Ingress to Gateway API. Maintain HTTPS access.

Create a Gateway named web-gateway with hostname gateway.web.k8s.local (existing TLS and listener configuration).

Create an HTTPRoute named web-route with hostname gateway.web.k8s.local (maintain existing routing rules).

Note — GatewayClass nginx is installed.

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml

# Install NGINX Gateway Fabric
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.6.2/deploy/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.6.2/deploy/default/deploy.yaml

# Verify GatewayClass
kubectl get gatewayclass

# Create webapp namespace, deployment, service
kubectl create ns webapp
kubectl create deployment web -n webapp --image=nginx --port=80
kubectl expose deployment web -n webapp --name=web-service --port=80

# Create TLS secret
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -subj "/CN=gateway.web.k8s.local" \
  -keyout tls.key -out tls.crt
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key -n webapp

# Create existing Ingress (what we're migrating FROM)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: webapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
    - hosts:
        - gateway.web.k8s.local
      secretName: web-tls
  rules:
  - host: gateway.web.k8s.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

echo "127.0.0.1 gateway.web.k8s.local" | sudo tee -a /etc/hosts
```

---

### ✅ Full Solution with Explanations

**Step 1 — Review the existing Ingress to understand what to migrate**

```bash
kubectl get ingress web -n webapp -o yaml
```

Key information to extract:
- **Host**: `gateway.web.k8s.local`
- **TLS**: Uses `web-tls` secret
- **Backend**: `web-service` on port `80`
- **Path**: `/` with `Prefix` pathType

**Step 2 — Check the GatewayClass name**

```bash
kubectl get gatewayclass
```

Expected:
```
NAME    CONTROLLER                                   ACCEPTED   AGE
nginx   gateway.nginx.org/nginx-gateway-controller   True       5m
```

> **WHY?** — The Gateway resource must reference a GatewayClass. The question tells us it's `nginx`.

**Step 3 — Create the Gateway resource**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: webapp
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: gateway.web.k8s.local
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: web-tls
  - name: http
    protocol: HTTP
    port: 80
    hostname: gateway.web.k8s.local
```

> **WHY each field — mapping from Ingress:**
>
> | Ingress field | Gateway API equivalent |
> |--------------|----------------------|
> | `spec.tls[].hosts` | `listeners[].hostname` |
> | `spec.tls[].secretName` | `listeners[].tls.certificateRefs[].name` |
> | `spec.rules[].host` | `listeners[].hostname` |
> | Ingress Controller | `gatewayClassName: nginx` |
>
> - `protocol: HTTPS` + `tls.mode: Terminate` — The Gateway terminates TLS (decrypts HTTPS traffic)
> - `certificateRefs` — Points to the same TLS secret that the Ingress used
> - Two listeners (HTTPS on 443, HTTP on 80) — Maintains both HTTP and HTTPS access

```bash
kubectl apply -f gateway.yaml
```

**Step 4 — Create the HTTPRoute**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: webapp
spec:
  parentRefs:
  - name: web-gateway
  hostnames:
  - gateway.web.k8s.local
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-service
      port: 80
```

> **WHY each field — mapping from Ingress:**
>
> | Ingress field | HTTPRoute equivalent |
> |--------------|---------------------|
> | `spec.rules[].host` | `hostnames[]` |
> | `spec.rules[].http.paths[].path` | `rules[].matches[].path.value` |
> | `spec.rules[].http.paths[].pathType` | `rules[].matches[].path.type` |
> | `spec.rules[].http.paths[].backend` | `rules[].backendRefs[]` |
> | (Ingress resource itself) | `parentRefs` → points to Gateway |
>
> - `parentRefs` — Links this route to the Gateway. In Ingress, routes were embedded in the Ingress object. In Gateway API, they're separate.
> - `backendRefs` — Same backend service and port as the Ingress rules.

```bash
kubectl apply -f httproute.yaml
```

**Step 5 — Verify the migration**

```bash
# Check Gateway status
kubectl get gateway web-gateway -n webapp

# Check HTTPRoute status
kubectl get httproute web-route -n webapp

# Check the Gateway is accepted and programmed
kubectl describe gateway web-gateway -n webapp

# Test HTTPS access
kubectl get svc -n nginx-gateway    # Find the NodePort
curl -k https://gateway.web.k8s.local:<nodeport>
```

---

### ⚡ Exam Speed Strategy (Target: 6 minutes)

```bash
# Step 1: Check existing ingress for reference
kubectl get ingress web -n webapp -o yaml

# Step 2: Check GatewayClass name
kubectl get gatewayclass

# Step 3: Create Gateway (copy from docs, modify)
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: webapp
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: gateway.web.k8s.local
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: web-tls
  - name: http
    protocol: HTTP
    port: 80
    hostname: gateway.web.k8s.local
EOF

# Step 4: Create HTTPRoute
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: webapp
spec:
  parentRefs:
  - name: web-gateway
  hostnames:
  - gateway.web.k8s.local
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-service
      port: 80
EOF

# Step 5: Verify
kubectl get gateway,httproute -n webapp
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| Gateway API | https://kubernetes.io/docs/concepts/services-networking/gateway/ |
| Gateway API reference | https://gateway-api.sigs.k8s.io/ |
| Migrating from Ingress | https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/ |
| **Search keyword** | `gateway api` → find Gateway and HTTPRoute examples |

---

### 💡 Tips & Memory Aids

- 🧠 **Gateway API has 3 resources**: GatewayClass (infra), Gateway (listeners), HTTPRoute (routing rules)
- 🧠 **Ingress → Gateway API mapping**:
  - Ingress → Gateway + HTTPRoute (split into 2 resources)
  - IngressClass → GatewayClass
  - Ingress rules → HTTPRoute rules
  - Ingress TLS → Gateway listeners with TLS
- ⚠️ **Common mistake**: Wrong `gatewayClassName` — always check with `kubectl get gatewayclass`
- ⚠️ **Common mistake**: Forgetting `parentRefs` in HTTPRoute — without it, the route isn't attached to any gateway
- ⚠️ **Common mistake**: Wrong namespace — Gateway and HTTPRoute should be in the same namespace as the backend service
- 🔑 **API version**: `gateway.networking.k8s.io/v1` (not `v1beta1` in newer K8s)
- 📝 **TLS mode**: `Terminate` (gateway decrypts) vs `Passthrough` (gateway forwards encrypted)

---

### ✅ Verification Checklist

- [ ] `kubectl get gateway web-gateway -n webapp` shows the gateway
- [ ] `kubectl get httproute web-route -n webapp` shows the route
- [ ] Gateway has both HTTP and HTTPS listeners
- [ ] HTTPRoute references the correct gateway and backend service
- [ ] HTTPS access works via the nginx-gateway service

---
---

## Question 11 — Install cri-dockerd + System Parameters

| | |
|---|---|
| **Weightage** | 10% |
| **Difficulty** | 🔴 Hard |
| **CKA Domain** | Cluster Architecture, Installation & Configuration (25%) |
| **Estimated Exam Time** | ⏱️ 5–7 minutes |
| **Topics** | cri-dockerd, systemd, sysctl, CRI |
| **Related Notes** | [docker-vs-containerd.md](docker-vs-containerd.md) · [k8s-architecture.md](k8s-architecture.md) |

### 📋 Question

Complete these tasks to prepare the system for Kubernetes: Set up cri-dockerd:

Install the Debian package: ~/cri-dockerd_0.3.9-3.0.ubuntu-focal_amd64.deb

Start the cri-dockerd service.

Enable and start the systemd service for cri-dockerd.

Configure these system parameters:

Set net.bridge.bridge-nf-call-iptables to 1.

Set net.ipv6.conf.all.forwarding to 1.

Set net.ipv4.ip_forward to 1.

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Download the cri-dockerd package (already in ~/ in exam)
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.17/cri-dockerd_0.3.17.3-0.ubuntu-focal_amd64.deb -O ~/cri-dockerd_0.3.9-3.0.ubuntu-focal_amd64.deb
```

---

### ✅ Full Solution with Explanations

#### Part A — Install and Configure cri-dockerd

**Step 1 — Install the .deb package**

```bash
sudo dpkg -i ~/cri-dockerd_0.3.9-3.0.ubuntu-focal_amd64.deb
```

> **WHY `dpkg -i`?** — `dpkg` is the Debian package manager. The `-i` flag installs a `.deb` package file. The exam provides the package in the home directory.

**Step 2 — Start the cri-dockerd service**

```bash
sudo systemctl start cri-docker.service
```

> **WHY?** — After package installation, the service is registered but may not be running. `start` launches it immediately.

> **Note**: The service name is `cri-docker` (with a hyphen), NOT `cri-dockerd` (with a 'd'). The socket is `cri-docker.socket`.

**Step 3 — Enable the service (persist across reboots)**

```bash
sudo systemctl enable cri-docker.service
sudo systemctl enable cri-docker.socket
```

> **WHY both?** — The `.service` is the main daemon. The `.socket` enables socket activation (starts the service on-demand when a connection arrives). Enabling both ensures cri-dockerd is always available.

**Step 4 — Verify the service is running**

```bash
sudo systemctl status cri-docker.service
sudo systemctl status cri-docker.socket
```

Look for:
- `Active: active (running)` for the service
- `Active: active (listening)` for the socket
- `enabled` in the Loaded line

```bash
# Alternative verification
systemctl list-units | grep cri
```

---

#### Part B — Configure System Parameters

**Step 5 — Set the sysctl parameters**

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
EOF
```

> **WHY each parameter:**
> - `net.bridge.bridge-nf-call-iptables = 1` — Makes bridged traffic go through iptables rules. Required for Kubernetes networking (kube-proxy) to work correctly. Without this, pod traffic bypasses iptables and services won't work.
> - `net.ipv6.conf.all.forwarding = 1` — Enables IPv6 packet forwarding. Required for dual-stack (IPv4+IPv6) Kubernetes support.
> - `net.ipv4.ip_forward = 1` — Enables IPv4 packet forwarding. **Absolutely essential** — without this, pods can't communicate across nodes.

> **WHY `/etc/sysctl.d/k8s.conf`?** — Files in `/etc/sysctl.d/` persist across reboots. Any `.conf` file here is automatically loaded at boot. This is the K8s-recommended approach.

**Step 6 — Apply the parameters immediately (without reboot)**

```bash
sudo sysctl --system
```

> **WHY `--system`?** — Loads settings from ALL config files in `/etc/sysctl.d/`, `/run/sysctl.d/`, and `/usr/lib/sysctl.d/`. This applies our new settings immediately.

**Step 7 — Verify the parameters**

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv6.conf.all.forwarding
sysctl net.ipv4.ip_forward
```

All should show `= 1`.

---

### ⚡ Exam Speed Strategy (Target: 3 minutes)

```bash
# Part A: Install cri-dockerd
sudo dpkg -i ~/cri-dockerd_0.3.9-3.0.ubuntu-focal_amd64.deb
sudo systemctl start cri-docker.service
sudo systemctl enable cri-docker.service
sudo systemctl enable cri-docker.socket

# Part B: Set sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

# Quick verify
systemctl status cri-docker.service --no-pager
sysctl net.ipv4.ip_forward
```

> **Memory trick**: **D-S-E** for cri-dockerd = Dpkg → Start → Enable
> **Memory trick**: **B-6-4** for sysctl = Bridge-nf, IPv6-forward, IPv4-forward

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| Container Runtimes | https://kubernetes.io/docs/setup/production-environment/container-runtimes/ |
| Dual-stack sysctl | https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/dual-stack-support/ |
| Prerequisite sysctl | https://kubernetes.io/docs/setup/production-environment/container-runtimes/#prerequisite-ipv4-forwarding-optional |
| **Search keyword** | `container runtime` → prerequisites section |

---

### 💡 Tips & Memory Aids

- 🧠 **3 sysctl params to remember**: bridge-nf-call-iptables, ipv6-forwarding, ipv4-forward — all set to 1
- ⚠️ **Service name gotcha**: `cri-docker` not `cri-dockerd` — the service drops the trailing 'd'
- ⚠️ **Don't forget `sysctl --system`** — without it, parameters only take effect after reboot
- ⚠️ **Don't forget `systemctl enable`** — without it, cri-dockerd won't survive a reboot
- 🔑 **Use `tee` not `>>`** — `tee` with `sudo` correctly writes to root-owned files. `sudo echo ... >>` doesn't work because `>>` runs as the current user.
- 📝 **br_netfilter module**: If `bridge-nf-call-iptables` fails, you may need: `sudo modprobe br_netfilter`

---

### ✅ Verification Checklist

- [ ] `systemctl status cri-docker.service` shows `active (running)` and `enabled`
- [ ] `systemctl status cri-docker.socket` shows `active (listening)` and `enabled`
- [ ] `sysctl net.bridge.bridge-nf-call-iptables` returns `1`
- [ ] `sysctl net.ipv6.conf.all.forwarding` returns `1`
- [ ] `sysctl net.ipv4.ip_forward` returns `1`
- [ ] Config persists in `/etc/sysctl.d/k8s.conf`

---
---

## Question 15 — Create Pod + crictl Inspect on Node

| | |
|---|---|
| **Weightage** | ~7% (estimated) |
| **Difficulty** | 🔴 Hard |
| **CKA Domain** | Troubleshooting (30%) |
| **Estimated Exam Time** | ⏱️ 7–9 minutes |
| **Topics** | Pod creation, crictl, container runtime, SSH, labels |
| **Related Notes** | [k8s-troubleshooting.md](k8s-troubleshooting.md) · [docker-vs-containerd.md](docker-vs-containerd.md) |

### 📋 Question

Solve this question on: ssh cka2556

In Namespace project-tiger create a Pod named tigers-reunite of image httpd:2-alpine with labels pod=container and container=pod. Find out on which node the Pod is scheduled. SSH into that node and find the containerd container belonging to that Pod.

Using command crictl:

Write the ID of the container and the info.runtimeType in /course/17/pod-container.txt

Write the logs of the container into /opt/course/17/pod-container.log

You can connect to a worker node using ssh cka2556-node1 or ssh cka2556-node2 from cka2556

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Create namespace (may not exist)
kubectl create ns project-tiger
```

---

### ✅ Full Solution with Explanations

**Step 1 — SSH into the exam cluster**

```bash
ssh cka2556
```

**Step 2 — Create the Pod with labels**

```bash
kubectl run tigers-reunite -n project-tiger \
  --image=httpd:2-alpine \
  --labels="pod=container,container=pod"
```

> **WHY imperative?** — `kubectl run` is the fastest way to create a single pod. The `--labels` flag accepts comma-separated key=value pairs.

Alternative YAML:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tigers-reunite
  namespace: project-tiger
  labels:
    pod: container
    container: pod
spec:
  containers:
  - name: webserver
    image: httpd:2-alpine
```

**Step 3 — Find which node the Pod is running on**

```bash
kubectl get pod tigers-reunite -n project-tiger -o wide
```

Example output:
```
NAME             READY   STATUS    IP          NODE
tigers-reunite   1/1     Running   10.244.1.5  cka2556-node1
```

Note the `NODE` column (e.g., `cka2556-node1`).

**Step 4 — SSH into that node**

```bash
ssh cka2556-node1
```

**Step 5 — Find the container using crictl**

```bash
crictl ps | grep tigers-reunite
```

> **WHY `crictl`?** — `crictl` is the CLI tool for CRI-compatible container runtimes (containerd, CRI-O). It's the standard way to interact with containers at the runtime level, replacing `docker` commands.

Example output:
```
abc123def456   httpd:2-alpine   Running   tigers-reunite   ...
```

Note the container ID (first column): `abc123def456`

**Step 6 — Get the runtimeType using crictl inspect**

```bash
crictl inspect <container-id> | grep runtimeType
```

Example output:
```
"runtimeType": "io.containerd.runc.v2",
```

**Step 7 — Save container ID and runtimeType to file**

```bash
# Create the directory
mkdir -p /course/17

# Write container ID and runtimeType
echo "<container-id>" > /course/17/pod-container.txt
crictl inspect <container-id> | grep runtimeType >> /course/17/pod-container.txt
```

> **WHY `>` then `>>`?** — First `>` creates/overwrites the file with the container ID. Then `>>` appends the runtimeType to the same file.

**Step 8 — Save container logs to file**

```bash
mkdir -p /opt/course/17

crictl logs <container-id> > /opt/course/17/pod-container.log
```

> **WHY `crictl logs` not `kubectl logs`?** — We're on the worker node, not the control plane. `kubectl` may not be available/configured on worker nodes. `crictl` works directly with the container runtime.

**Step 9 — Exit back to the control plane**

```bash
exit    # Exit worker node
exit    # Exit exam cluster (if needed for next question)
```

---

### ⚡ Exam Speed Strategy (Target: 5 minutes)

```bash
# On control plane
ssh cka2556
kubectl run tigers-reunite -n project-tiger --image=httpd:2-alpine --labels="pod=container,container=pod"
kubectl get pod tigers-reunite -n project-tiger -o wide    # Note the NODE

# SSH to the node
ssh cka2556-node1    # Use the node from above
crictl ps | grep tigers

# Save results (replace CONTAINER_ID)
CONTAINER_ID=$(crictl ps | grep tigers | awk '{print $1}')
mkdir -p /course/17 /opt/course/17
echo "$CONTAINER_ID" > /course/17/pod-container.txt
crictl inspect $CONTAINER_ID | grep runtimeType >> /course/17/pod-container.txt
crictl logs $CONTAINER_ID > /opt/course/17/pod-container.log

exit    # Back to control plane
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| crictl docs | https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/ |
| Container runtimes | https://kubernetes.io/docs/setup/production-environment/container-runtimes/ |
| **Search keyword** | `crictl` |

---

### 💡 Tips & Memory Aids

- 🧠 **crictl commands** map to docker commands:
  - `crictl ps` = `docker ps` (list containers)
  - `crictl inspect` = `docker inspect` (container details)
  - `crictl logs` = `docker logs` (container logs)
- ⚠️ **Common mistake**: Running `crictl` on the control plane — you must SSH to the worker node where the pod runs
- ⚠️ **Common mistake**: Forgetting to exit back — always `exit` after finishing on a worker node
- ⚠️ **Common mistake**: Wrong output file path — the question has TWO different paths (`/course/17/` and `/opt/course/17/`)
- 🔑 **Quick container ID**: `crictl ps | grep <pod-name> | awk '{print $1}'`
- 📝 **`--labels` syntax**: Comma-separated, no spaces: `--labels="key1=val1,key2=val2"`

---

### ✅ Verification Checklist

- [ ] Pod `tigers-reunite` is running in `project-tiger` namespace with correct labels
- [ ] `/course/17/pod-container.txt` contains container ID and runtimeType
- [ ] `/opt/course/17/pod-container.log` contains container logs
- [ ] You have exited back to the control plane / bastion

---
---

## 📊 Day 4 Summary

| Question | Topic | Target Time | Key Action |
|----------|-------|-------------|------------|
| Q4 | PriorityClass + patch | 4 min | Create PriorityClass + `kubectl patch` |
| Q7 | PV/PVC + Deployment | 5 min | Match PVC to PV + add volumes |
| Q8 | Gateway API migration | 6 min | Gateway + HTTPRoute YAML |
| Q11 | cri-dockerd + sysctl | 3 min | `dpkg` + `systemctl` + `sysctl` |
| Q15 | crictl on worker node | 5 min | SSH + `crictl ps/inspect/logs` |
| **Total** | | **~23 min** | |

> 💪 **These are the hardest questions. If you can do these in under 25 minutes, you'll crush the exam!**

---

**Previous**: [Day 3 — Networking 🟠](exam-day3-networking.md) | **Next**: [Day 5 — Cluster Ops 🔴](exam-day5-cluster-ops.md)
