# CKA Exam Practice — Day 3: Networking 🟠

> **Difficulty**: Medium-Hard | **Questions**: 5 | **Target Time**: ~75 minutes
> **Goal**: Master networking questions — NetworkPolicy, CoreDNS, CNI, and TLS. These appear frequently and are worth significant points.

---

## Question 9 — Install CNI with NetworkPolicy Support

| | |
|---|---|
| **Weightage** | 10% |
| **Difficulty** | 🟠 Medium-Hard |
| **CKA Domain** | Cluster Architecture, Installation & Configuration (25%) |
| **Estimated Exam Time** | ⏱️ 3–4 minutes |
| **Topics** | CNI plugins, Calico vs Flannel, NetworkPolicy support |
| **Related Notes** | [docker-vs-containerd.md](docker-vs-containerd.md) · [k8s-network-policy.md](k8s-network-policy.md) |

### 📋 Question

Install and set up a Container Network Interface (CNI) that meets these requirements: pick and install one of the CNI options:

the CNI you choose must statisfy the following requirement:
\<native network policy support\>

Pick and install one of the CNI options:

Flannel version 0.26.1 Manifest — https://github.com/flannel-io/flannel/releases/download/v0.26.1/kube-flanner.yml

Calico version 3.28.2 Manifest — https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# In the exam, you'll have a cluster without CNI installed.
# For practice, you can use a kind cluster without CNI:
# kind create cluster --config kind-no-cni.yaml

# For Killercoda, the cluster may already have a CNI — 
# this question tests your understanding of which CNI to choose.
```

---

### ✅ Full Solution with Explanations

**Step 1 — Understand the requirement: "native network policy support"**

| CNI Plugin | NetworkPolicy Support | Notes |
|-----------|----------------------|-------|
| **Calico** ✅ | **Yes — Native** | Full NetworkPolicy support built-in |
| **Flannel** ❌ | **No** | Only provides networking, no NetworkPolicy enforcement |

> **WHY Calico?** — The question requires "native network policy support". Flannel is a simple overlay network that does NOT support NetworkPolicy. Calico has a built-in policy engine that enforces Kubernetes NetworkPolicy rules. **Always choose Calico when NetworkPolicy is required.**

**Step 2 — Install Calico**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml
```

> **WHY this manifest?** — The `tigera-operator.yaml` installs the Tigera operator which manages Calico components. This is the recommended production installation method.

**Step 3 — Verify installation**

```bash
# Check that Calico pods are running
kubectl get pods -n calico-system
# or
kubectl get pods -n kube-system | grep calico

# Check that nodes become Ready (if they were NotReady without CNI)
kubectl get nodes

# Verify the CNI is working
kubectl get pods --all-namespaces
```

> **WHY check nodes?** — Without a CNI, nodes stay in `NotReady` state because pod networking isn't configured. After installing Calico, nodes should transition to `Ready`.

---

### ⚡ Exam Speed Strategy (Target: 2 minutes)

```bash
# Decision: Need NetworkPolicy? → Calico. Don't need it? → Either works.
# One command:
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml

# Wait and verify
kubectl get nodes    # Should show Ready
kubectl get pods -A  # Calico pods should be Running
```

> **Memory trick**: **C**alico = **C**an do network policies. **F**lannel = **F**lat networking only.

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| Cluster Networking | https://kubernetes.io/docs/concepts/cluster-administration/networking/ |
| Network Plugins | https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/ |
| Calico install | https://docs.tigera.io/calico/latest/getting-started/kubernetes/ |
| **Search keyword** | `network plugin` or `CNI` |

---

### 💡 Tips & Memory Aids

- 🧠 **CNI comparison to remember**:
  - **Calico** — Full L3 networking + NetworkPolicy + BGP routing
  - **Flannel** — Simple L2 overlay only, no policy support
  - **Cilium** — eBPF-based, NetworkPolicy + observability (not in this question)
- ⚠️ **Common mistake**: Installing Flannel when NetworkPolicy is required → zero points
- ⚠️ **The exam gives you the exact manifest URL** — just copy-paste and `kubectl apply -f`
- 🔑 **Quick decision**: If the question mentions "network policy" → Calico. Always.

---

### ✅ Verification Checklist

- [ ] Calico pods are running (`kubectl get pods -n calico-system` or `kubectl get pods -A | grep calico`)
- [ ] All nodes are in `Ready` state
- [ ] NetworkPolicy can be created (`kubectl create -f` a test NetworkPolicy)

---
---

## Question 13 — Choose Least-Permissive NetworkPolicy

| | |
|---|---|
| **Weightage** | 10% |
| **Difficulty** | 🟠 Medium-Hard |
| **CKA Domain** | Services & Networking (20%) |
| **Estimated Exam Time** | ⏱️ 5–6 minutes |
| **Topics** | NetworkPolicy analysis, least privilege, namespace selectors |
| **Related Notes** | [k8s-network-policy.md](k8s-network-policy.md) |

### 📋 Question

There are two deployments: frontend and backend deployment.

Frontend will be in frontend NS, backend in backend NS.

Apply the least permissive policy to have interaction between frontend and backend deployment.

Choose one of the 3 YAML files with the least permission:

YAML — pod selector of all, type is ingress, and pod selector with all

YAML — pod selector + namespace selector

YAML — pod selector, namespace selector, POD CIDR

3 YAML will be given we have to choose which opt for the requirement

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Create namespaces and deployments
kubectl create ns frontend
kubectl create ns backend
kubectl create deployment frontend --image=nginx -n frontend
kubectl create deployment backend --image=nginx -n backend
kubectl expose deployment backend -n backend --port=8080 --target-port=80
```

---

### ✅ Full Solution with Explanations

**Understanding the 3 options:**

**Policy 1 — WIDE OPEN (Most Permissive) ❌**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: open-backend-access
  namespace: backend
spec:
  podSelector: {}           # Applies to ALL pods in backend namespace
  policyTypes:
  - Ingress
  ingress:
  - from: []                # Allows traffic from EVERYWHERE
    ports:
    - protocol: TCP
      port: 8080
```

> **WHY NOT this one?** — `podSelector: {}` selects ALL pods, and `from: []` means allow from ANY source. This is the **most permissive** — any pod in the cluster can access backend pods on port 8080. **Least privilege violation!**

---

**Policy 2 — TARGETED (Least Permissive) ✅ CORRECT ANSWER**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend          # Only applies to backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
      podSelector:
        matchLabels:
          app: frontend     # Only frontend pods in frontend namespace
    ports:
    - protocol: TCP
      port: 8080
```

> **WHY this one is correct?** — This is the **least permissive** because:
> 1. `podSelector` with `app: backend` — Only applies to pods labeled `app: backend` (not all pods)
> 2. `namespaceSelector` + `podSelector` together (AND logic) — Only allows pods labeled `app: frontend` that are in the `frontend` namespace
> 3. Only port 8080 is allowed
> 4. **No extra CIDR block** — doesn't open access to any IP range

> **CRITICAL: AND vs OR in NetworkPolicy**
> ```yaml
> # AND logic (both conditions must match):
> - from:
>   - namespaceSelector: ...
>     podSelector: ...        # Same list item = AND
>
> # OR logic (either condition matches):
> - from:
>   - namespaceSelector: ...  # Separate list items = OR
>   - podSelector: ...
> ```

---

**Policy 3 — TARGETED + EXTRA CIDR (More Permissive) ❌**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend-with-cidr
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
      podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  - from:                    # ← EXTRA rule!
    - ipBlock:
        cidr: 192.168.1.0/24
    ports:
    - protocol: TCP
      port: 8080
```

> **WHY NOT this one?** — It has the same frontend-to-backend rule as Policy 2, BUT it adds an **extra `ipBlock` rule** that allows any IP in `192.168.1.0/24` to access backend. This is more permissive than Policy 2.

---

**Step — Apply the correct policy**

```bash
kubectl apply -f policy2.yaml

# Verify
kubectl get networkpolicy -n backend
kubectl describe networkpolicy frontend-to-backend -n backend
```

---

### ⚡ Exam Speed Strategy (Target: 3 minutes)

```
Quick Decision Framework:
1. Read all 3 policies
2. Eliminate: podSelector: {} with from: [] → too open
3. Eliminate: any policy with ipBlock/CIDR → extra access
4. Choose: the one with ONLY namespaceSelector + podSelector (AND logic)
```

> **Memory trick**: **Least permissive = most specific selectors + no CIDR**

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| NetworkPolicy | https://kubernetes.io/docs/concepts/services-networking/network-policies/ |
| **Search keyword** | `network policy` |

---

### 💡 Tips & Memory Aids

- 🧠 **Least permissive ranking**: `podSelector + namespaceSelector` < `podSelector + namespaceSelector + CIDR` < `podSelector: {} + from: []`
- 🧠 **AND vs OR**: Same YAML list item = AND, Different list items = OR
- ⚠️ **Common mistake**: Confusing AND/OR in NetworkPolicy `from` rules — the YAML indentation matters!
- 🔑 **`kubernetes.io/metadata.name`** — Every namespace automatically gets this label set to the namespace name. No need to manually label namespaces.
- 📝 **In exam**: Just READ the 3 policies, choose the most restrictive one, and apply it. Don't write YAML from scratch.

---

### ✅ Verification Checklist

- [ ] `kubectl get netpol -n backend` shows the selected policy
- [ ] The policy uses `podSelector` with specific labels (not `{}`)
- [ ] The policy uses BOTH `namespaceSelector` AND `podSelector` in the `from` field
- [ ] No CIDR/ipBlock rules

---
---

## Question 14 — CoreDNS Custom Domain Configuration

| | |
|---|---|
| **Weightage** | ~7% (estimated) |
| **Difficulty** | 🟠 Medium-Hard |
| **CKA Domain** | Services & Networking (20%) |
| **Estimated Exam Time** | ⏱️ 6–8 minutes |
| **Topics** | CoreDNS, ConfigMap, DNS resolution, Corefile |
| **Related Notes** | [docker-vs-containerd.md](docker-vs-containerd.md) · [k8s-troubleshooting.md](k8s-troubleshooting.md) |

### 📋 Question

Solve this question on: ssh cka5774

The CoreDNS configuration in the cluster needs to be updated:

Make a backup of the existing configuration Yaml and store it at /opt/course/16/coredns_backup.yaml.
You should be able to fast recover from the backup.

Update the CoreDNS configuration in the cluster so that DNS resolution for SERVICE.NAMESPACE.custom-domain will work exactly like and in addition to SERVICE.NAMESPACE.cluster.local.

Test your configuration for example from a Pod with busybox:1 image. These commands should result in an IP address:

nslookup kubernetes.default.svc.cluster.local
nslookup kubernetes.default.svc.custom-domain

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# SSH to the correct cluster (in exam)
# ssh cka5774

# No special setup needed — CoreDNS is already running in every cluster
# Just ensure CoreDNS pods are running:
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

---

### ✅ Full Solution with Explanations

**Step 1 — Create the backup directory and take backup**

```bash
mkdir -p /opt/course/16/

kubectl -n kube-system get configmap coredns -o yaml > /opt/course/16/coredns_backup.yaml
```

> **WHY backup?** — The question requires it, and it's a safety net. If your CoreDNS edit breaks DNS, you can recover with:
> ```bash
> kubectl apply -f /opt/course/16/coredns_backup.yaml
> ```

**Step 2 — Verify the current CoreDNS config**

```bash
kubectl -n kube-system get configmap coredns -o yaml
```

The current Corefile looks like this:
```
.:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

**Step 3 — Edit the CoreDNS ConfigMap**

```bash
kubectl -n kube-system edit configmap coredns
```

Add `custom-domain` to the `kubernetes` plugin line:

```
.:53 {
    errors
    health
    ready
    kubernetes custom-domain cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

> **WHY just add `custom-domain` to the kubernetes line?**
> - The `kubernetes` plugin in CoreDNS takes a list of zones it should serve
> - By adding `custom-domain` alongside `cluster.local`, CoreDNS will resolve services using BOTH domain suffixes
> - `kubernetes.default.svc.custom-domain` → will resolve the same way as `kubernetes.default.svc.cluster.local`
> - Both are treated as valid service DNS names

> **CRITICAL**: Put `custom-domain` BEFORE `cluster.local` in the list. This ensures it's recognized as a valid zone.

**Step 4 — Restart CoreDNS to apply changes**

```bash
kubectl -n kube-system rollout restart deployment coredns
```

> **WHY restart?** — CoreDNS has a `reload` plugin that auto-reloads the Corefile, but it can take up to 30 seconds. Restarting the deployment forces an immediate reload.

**Step 5 — Wait for CoreDNS to be ready**

```bash
kubectl -n kube-system rollout status deployment coredns
```

**Step 6 — Test DNS resolution**

```bash
# Test the original domain (should still work)
kubectl run -it --rm dns-test --image=busybox:1 --restart=Never -- nslookup kubernetes.default.svc.cluster.local

# Test the custom domain (should now work too)
kubectl run -it --rm dns-test2 --image=busybox:1 --restart=Never -- nslookup kubernetes.default.svc.custom-domain
```

Both should return the same IP address (the Kubernetes API service ClusterIP, typically `10.96.0.1`).

> **WHY use `--rm`?** — The `--rm` flag automatically deletes the pod after it exits, keeping the cluster clean.

> **WHY `--restart=Never`?** — Creates a standalone Pod (not a Deployment). Combined with `--rm`, it's a quick one-shot test.

---

### ⚡ Exam Speed Strategy (Target: 4 minutes)

```bash
# Backup
mkdir -p /opt/course/16
kubectl -n kube-system get cm coredns -o yaml > /opt/course/16/coredns_backup.yaml

# Edit — just add "custom-domain" to the kubernetes line
kubectl -n kube-system edit cm coredns
# Change: kubernetes cluster.local in-addr.arpa ip6.arpa
# To:     kubernetes custom-domain cluster.local in-addr.arpa ip6.arpa

# Restart
kubectl -n kube-system rollout restart deploy coredns

# Test (nslookup commands are usually given in the question)
kubectl run -it --rm test --image=busybox:1 --restart=Never -- nslookup kubernetes.default.svc.custom-domain
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| CoreDNS | https://kubernetes.io/docs/tasks/administer-cluster/coredns/ |
| DNS for Services | https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/ |
| CoreDNS kubernetes plugin | https://coredns.io/plugins/kubernetes/ |
| **Search keyword** | `coredns` or `dns service` |

---

### 💡 Tips & Memory Aids

- 🧠 **CoreDNS config is in a ConfigMap** named `coredns` in `kube-system` namespace
- 🧠 **Adding a custom domain = just add it to the `kubernetes` plugin line** — it's a space-separated list of zones
- ⚠️ **Common mistake**: Creating a whole new server block — you don't need to! Just add the domain name to the existing `kubernetes` line
- ⚠️ **Common mistake**: Forgetting to restart CoreDNS — without restart, changes may take 30+ seconds to apply
- 🔑 **Quick recovery**: If DNS breaks → `kubectl apply -f /opt/course/16/coredns_backup.yaml`
- 📝 **DNS format**: `<service>.<namespace>.svc.<zone>` — e.g., `kubernetes.default.svc.custom-domain`

---

### ✅ Verification Checklist

- [ ] Backup exists at `/opt/course/16/coredns_backup.yaml`
- [ ] `nslookup kubernetes.default.svc.cluster.local` returns an IP
- [ ] `nslookup kubernetes.default.svc.custom-domain` returns the SAME IP
- [ ] CoreDNS pods are running (`kubectl get pods -n kube-system -l k8s-app=kube-dns`)

---
---

## Question 16 — NetworkPolicy Allow From Namespace

| | |
|---|---|
| **Weightage** | ~7% (estimated) |
| **Difficulty** | 🟠 Medium-Hard |
| **CKA Domain** | Services & Networking (20%) |
| **Estimated Exam Time** | ⏱️ 4–5 minutes |
| **Topics** | NetworkPolicy, namespace selectors, port filtering |
| **Related Notes** | [k8s-network-policy.md](k8s-network-policy.md) |

### 📋 Question

Create a new NetworkPolicy named "allow-port-from-namespace" in the existing namespace foobar. In the foobar namespace, nginx pod was running already.
Ensure that the new NetworkPolicy allows only Pods in namespace internal-ns to connect to port 9000 of Pods in namespace foobar.

Further ensure that the new NetworkPolicy:
does not allow access to Pods, which don't listen on port 9000
does not allow access from Pods, which are not in namespace internal-ns

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Create namespaces (already exist in exam)
kubectl create ns foobar
kubectl create ns internal-ns

# Create a test nginx pod in foobar
kubectl run nginx --image=nginx -n foobar

# Label the internal-ns namespace (auto-labeled in modern K8s)
kubectl label ns internal-ns project=my-app
```

---

### ✅ Full Solution with Explanations

**Step 1 — Understand the requirements**

| Requirement | Implementation |
|-------------|---------------|
| Only pods in `internal-ns` can connect | `namespaceSelector` matching `internal-ns` |
| Only port 9000 is allowed | `ports` field with port `9000` |
| No access to pods not on port 9000 | Covered by specifying port `9000` only |
| No access from other namespaces | Covered by `namespaceSelector` targeting only `internal-ns` |

**Step 2 — Create the NetworkPolicy**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-from-namespace
  namespace: foobar
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: internal-ns
    ports:
    - protocol: TCP
      port: 9000
```

> **WHY each field:**
> - `podSelector: {}` — Applies to ALL pods in the `foobar` namespace. This means the policy protects every pod.
> - `policyTypes: [Ingress]` — We're controlling incoming traffic only.
> - `namespaceSelector` with `kubernetes.io/metadata.name: internal-ns` — Only allows traffic from pods in the `internal-ns` namespace. This label is automatically set on every namespace in modern Kubernetes.
> - `ports: [{protocol: TCP, port: 9000}]` — Only allows traffic to port 9000.

> **WHY `podSelector: {}` (empty)?** — The question says "Pods in namespace foobar" generally — it doesn't specify particular pods. An empty selector means "all pods in this namespace are protected by this policy."

> **WHY `kubernetes.io/metadata.name` instead of a custom label?** — This is a built-in label automatically applied to every namespace. Its value equals the namespace name. Using it means you don't need to manually label the `internal-ns` namespace.

**Step 3 — Apply and verify**

```bash
kubectl apply -f networkpolicy.yaml

# Verify the policy
kubectl get networkpolicy -n foobar
kubectl describe networkpolicy allow-port-from-namespace -n foobar
```

---

### ⚡ Exam Speed Strategy (Target: 3 minutes)

```bash
# Copy the NetworkPolicy template from docs, modify it:
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-from-namespace
  namespace: foobar
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: internal-ns
    ports:
    - protocol: TCP
      port: 9000
EOF

kubectl get netpol -n foobar
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| NetworkPolicy | https://kubernetes.io/docs/concepts/services-networking/network-policies/ |
| **Search keyword** | `network policy` → find the "Behavior of to and from selectors" section |
| **Specific example** | Look for "Allow ingress from a namespace" pattern |

---

### 💡 Tips & Memory Aids

- 🧠 **NetworkPolicy template**: `podSelector` (who is protected) + `from` (who can access) + `ports` (which ports)
- 🧠 **`kubernetes.io/metadata.name`** — built-in label on every namespace, no need to manually label
- ⚠️ **Common mistake**: Putting the policy in the wrong namespace — it must be in `foobar` (the namespace being protected)
- ⚠️ **Common mistake**: Using `from: []` (empty array means allow ALL) vs `from: [<selector>]` (allow specific)
- 🔑 **Default deny**: Once a NetworkPolicy selects a pod, it's denied all traffic except what's explicitly allowed
- 📝 **Short alias**: `kubectl get netpol` instead of `kubectl get networkpolicy`

---

### ✅ Verification Checklist

- [ ] `kubectl get netpol -n foobar` shows `allow-port-from-namespace`
- [ ] `kubectl describe netpol -n foobar allow-port-from-namespace` shows correct namespace selector and port
- [ ] Only `internal-ns` pods can reach port 9000 in `foobar`

---
---

## Question 20 — Enforce TLS 1.3 Only via ConfigMap

| | |
|---|---|
| **Weightage** | ~8% (estimated) |
| **Difficulty** | 🟠 Medium-Hard |
| **CKA Domain** | Services & Networking (20%) |
| **Estimated Exam Time** | ⏱️ 5–7 minutes |
| **Topics** | ConfigMap, nginx TLS configuration, rollout restart |
| **Related Notes** | [k8s-configmaps-secrets-volumes.md](k8s-configmaps-secrets-volumes.md) · [k8s-workloads.md](k8s-workloads.md) |

### 📋 Question

An nginx deployment configured via a configMap containing the Nginx config file.
The task is to ensure that TLS 1.2 should not be supported or permitted. And it should only work with the TLS V1.3 permitted. And it should only work with the TLS1.3 Do the required steps and verify that it works with the TLS 1.3.

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Create namespace
kubectl create ns webapp

# Generate self-signed TLS certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -subj "/CN=webapp.local" \
  -keyout tls.key -out tls.crt

# Create TLS secret
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key -n webapp

# Create the ConfigMap with TLSv1.2 + TLSv1.3 (the "broken" state)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: webapp
data:
  nginx.conf: |
    server {
        listen 443 ssl;
        server_name webapp.local;
        ssl_certificate /etc/nginx/tls/tls.crt;
        ssl_certificate_key /etc/nginx/tls/tls.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        location / {
            return 200 "Hello Secure World!";
        }
    }
EOF

# Create the Deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      volumes:
        - name: config-vol
          configMap:
            name: nginx-config
        - name: tls-vol
          secret:
            secretName: web-tls
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 443
          volumeMounts:
            - name: config-vol
              mountPath: /etc/nginx/conf.d
            - name: tls-vol
              mountPath: /etc/nginx/tls
EOF

# Create the Service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
  namespace: webapp
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
    - port: 443
      targetPort: 443
      nodePort: 30443
EOF
```

---

### ✅ Full Solution with Explanations

**Step 1 — Understand the current state**

In the exam, the deployment, ConfigMap, and service already exist. First, examine the ConfigMap:

```bash
kubectl get configmap nginx-config -n webapp -o yaml
```

> **WHY?** — You need to see what the current `ssl_protocols` line says. It likely has `TLSv1.2 TLSv1.3`.

You can also check the deployment to understand how the ConfigMap is mounted:

```bash
kubectl get deployment webapp -n webapp -o yaml | grep -A 10 "volumes:"
```

**Step 2 — Edit the ConfigMap to remove TLS 1.2**

```bash
kubectl edit configmap nginx-config -n webapp
```

Change:
```
ssl_protocols TLSv1.2 TLSv1.3;
```
To:
```
ssl_protocols TLSv1.3;
```

> **WHY?** — The `ssl_protocols` directive in nginx controls which TLS versions are accepted. By removing `TLSv1.2`, only TLS 1.3 connections will be permitted. TLS 1.2 connections will be rejected with a protocol version error.

**Step 3 — Restart the deployment** ⚠️ CRITICAL STEP

```bash
kubectl rollout restart deployment webapp -n webapp
```

> **WHY is this mandatory?** — ConfigMap updates are NOT automatically picked up by running containers. Even though the ConfigMap is updated in the cluster, nginx inside the container still has the old config loaded in memory. **You MUST restart the deployment** to force nginx to reload the new config.
>
> **Common trap in exam**: Editing the ConfigMap and thinking you're done. Without a restart, TLS 1.2 will still work!

**Step 4 — Wait for the rollout to complete**

```bash
kubectl rollout status deployment webapp -n webapp
```

**Step 5 — Verify TLS 1.3 works and TLS 1.2 is blocked**

```bash
# Test TLS 1.2 — should FAIL
curl -k --tls-max 1.2 https://localhost:30443
# Expected: curl: (35) OpenSSL: error:... tlsv1 alert protocol version

# Test TLS 1.3 — should SUCCEED
curl -k --tls-max 1.3 https://localhost:30443
# Expected: Hello Secure World!
```

> **WHY `-k`?** — The `-k` flag skips certificate verification. We use this because the cert is self-signed (not trusted by a CA). In the exam, this is fine.
>
> **WHY `--tls-max 1.2`?** — This forces curl to use TLS 1.2 or lower. If the server rejects it, TLS 1.2 is properly disabled.

---

### ⚡ Exam Speed Strategy (Target: 3 minutes)

```bash
# Step 1: Look at the ConfigMap
kubectl get cm nginx-config -n webapp -o yaml

# Step 2: Edit ConfigMap — change ssl_protocols
kubectl edit cm nginx-config -n webapp
# Change: ssl_protocols TLSv1.2 TLSv1.3;
# To:     ssl_protocols TLSv1.3;

# Step 3: RESTART the deployment (CRITICAL!)
kubectl rollout restart deploy webapp -n webapp

# Step 4: Verify (curl commands usually given in question)
curl -k --tls-max 1.2 https://localhost:30443   # Should fail
curl -k --tls-max 1.3 https://localhost:30443   # Should succeed
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| ConfigMaps | https://kubernetes.io/docs/concepts/configuration/configmap/ |
| Secrets (TLS) | https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets |
| **Search keyword** | `configmap` |

---

### 💡 Tips & Memory Aids

- 🧠 **Key insight**: Edit ConfigMap → MUST restart deployment. ConfigMap changes are NOT live-reloaded by default.
- ⚠️ **#1 trap**: Forgetting `kubectl rollout restart` after editing the ConfigMap
- ⚠️ **Common mistake**: Editing the deployment YAML instead of the ConfigMap — the TLS config is in the ConfigMap, not the deployment spec
- 🔑 **Verify both sides**: Test that TLS 1.2 FAILS and TLS 1.3 WORKS — both checks are needed for full score
- 📝 **nginx ssl_protocols syntax**: `ssl_protocols TLSv1.3;` — note the semicolon at the end
- 📝 **Immutable ConfigMap**: If the question also asks to make the ConfigMap immutable, add `immutable: true` at the root level of the ConfigMap YAML (same level as `data:`)

---

### ✅ Verification Checklist

- [ ] ConfigMap has `ssl_protocols TLSv1.3;` (no TLSv1.2)
- [ ] Deployment was restarted (`kubectl rollout status` shows complete)
- [ ] `curl -k --tls-max 1.2` returns a TLS error (connection refused)
- [ ] `curl -k --tls-max 1.3` returns the expected response

---
---

## 📊 Day 3 Summary

| Question | Topic | Target Time | Key Action |
|----------|-------|-------------|------------|
| Q9 | CNI selection | 2 min | Choose Calico for NetworkPolicy support |
| Q13 | Least-permissive policy | 3 min | Analyze 3 policies → pick most restrictive |
| Q14 | CoreDNS custom domain | 4 min | Add domain to `kubernetes` plugin line |
| Q16 | NetworkPolicy from namespace | 3 min | `namespaceSelector` + `ports` |
| Q20 | TLS 1.3 enforcement | 3 min | Edit ConfigMap + rollout restart |
| **Total** | | **~15 min** | |

> 💪 **Networking is 20% of the exam. Master these and you've secured a huge chunk of points!**

---

**Previous**: [Day 2 — Core Skills 🟡](exam-day2-core-skills.md) | **Next**: [Day 4 — Advanced 🔴](exam-day4-advanced.md)
