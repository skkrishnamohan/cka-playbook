# Kubernetes API Overview — API Groups, Versions, and Resources

A beginner-friendly guide covering:
- What the Kubernetes API is and how resources are organized
- API Groups (Core group vs Named groups)
- API Versions (Alpha, Beta, Stable)
- How to discover resources in the cluster
- How `apiVersion` in YAML connects to API Groups
- Updating Kubernetes objects (edit, apply, set)

---

## What is the Kubernetes API?

Everything in Kubernetes is an API object. When you run `kubectl`, it is making HTTP requests to the **kube-apiserver** on your behalf.

```
kubectl get pods
  ↓
HTTP GET https://<kube-apiserver>/api/v1/namespaces/default/pods
  ↓
kube-apiserver checks auth → etcd → returns list of pods
```

The API is organized into **API Groups** and **API Versions** to keep things organized and to allow new features to be added without breaking existing ones.

---

## API Versions — Alpha, Beta, Stable

| Version | Example | Stability | Use in production? |
|---------|---------|-----------|-------------------|
| Alpha | `v1alpha1` | Experimental. May be removed. Breaking changes possible. | No |
| Beta | `v1beta1` | Feature-complete. Minor breaking changes possible. | With caution |
| Stable (GA) | `v1`, `v2` | Fully stable. Committed API. | Yes |

How to check available versions:
```bash
kubectl api-versions
```

Sample output:
```
apps/v1
autoscaling/v1
autoscaling/v2
batch/v1
networking.k8s.io/v1
rbac.authorization.k8s.io/v1
v1
...
```

---

## API Groups — How resources are organized

Kubernetes resources are split into two main groups:

### Core API Group (no group name)
- API path: `/api/v1`
- In YAML: `apiVersion: v1`
- Resources: the most fundamental, original Kubernetes objects

| Resource | Kind |
|----------|------|
| Pod | Pod |
| Service | Service |
| Node | Node |
| Namespace | Namespace |
| PersistentVolume | PersistentVolume |
| PersistentVolumeClaim | PersistentVolumeClaim |
| Secret | Secret |
| ConfigMap | ConfigMap |
| ServiceAccount | ServiceAccount |
| Endpoints | Endpoints |
| Event | Event |

### Named API Groups
- API path: `/apis/<group>/<version>`
- In YAML: `apiVersion: <group>/<version>`

| Group | Version | Resources |
|-------|---------|-----------|
| `apps` | `v1` | Deployment, ReplicaSet, StatefulSet, DaemonSet |
| `batch` | `v1` | Job, CronJob |
| `autoscaling` | `v2` | HorizontalPodAutoscaler |
| `networking.k8s.io` | `v1` | Ingress, NetworkPolicy, IngressClass |
| `rbac.authorization.k8s.io` | `v1` | Role, ClusterRole, RoleBinding, ClusterRoleBinding |
| `policy` | `v1` | PodDisruptionBudget |
| `storage.k8s.io` | `v1` | StorageClass, VolumeAttachment |
| `scheduling.k8s.io` | `v1` | PriorityClass |
| `apiextensions.k8s.io` | `v1` | CustomResourceDefinition (CRD) |

So when you write `apiVersion: apps/v1` — it means "use the `apps` group, version `v1`".

---

## Discovering Resources in Your Cluster

### List all resource types
```bash
kubectl api-resources
```

Sample output (partial):
```
NAME                  SHORTNAMES  APIVERSION           NAMESPACED  KIND
pods                  po          v1                   true        Pod
services              svc         v1                   true        Service
deployments           deploy      apps/v1              true        Deployment
replicasets           rs          apps/v1              true        ReplicaSet
statefulsets          sts         apps/v1              true        StatefulSet
daemonsets            ds          apps/v1              true        DaemonSet
jobs                               batch/v1             true        Job
cronjobs              cj          batch/v1             true        CronJob
ingresses             ing         networking.k8s.io/v1 true        Ingress
networkpolicies       netpol      networking.k8s.io/v1 true        NetworkPolicy
horizontalpodautoscalers hpa      autoscaling/v2       true        HorizontalPodAutoscaler
persistentvolumes     pv          v1                   false       PersistentVolume
persistentvolumeclaims pvc        v1                   true        PersistentVolumeClaim
```

The `NAMESPACED` column tells you if the resource lives inside a namespace (true) or at cluster-scope (false). For example, Pods are namespaced; Nodes and PersistentVolumes are cluster-scoped.

### Filter resources by API group
```bash
# Only show resources in the apps group
kubectl api-resources --api-group=apps

# Only show cluster-scoped (non-namespaced) resources
kubectl api-resources --namespaced=false

# Only show namespaced resources
kubectl api-resources --namespaced=true
```

### List all API versions available
```bash
kubectl api-versions
```

---

## How apiVersion Works in YAML

The `apiVersion` field in YAML tells Kubernetes which API Group and Version to use:

| apiVersion in YAML | Means | Resources |
|-------------------|-------|-----------|
| `v1` | Core group, version v1 | Pod, Service, Secret, ConfigMap... |
| `apps/v1` | apps group, version v1 | Deployment, StatefulSet, DaemonSet... |
| `batch/v1` | batch group, version v1 | Job, CronJob |
| `autoscaling/v2` | autoscaling group, version v2 | HPA |
| `networking.k8s.io/v1` | networking group, version v1 | Ingress, NetworkPolicy |
| `rbac.authorization.k8s.io/v1` | rbac group, version v1 | Role, ClusterRole, RoleBinding... |
| `scheduling.k8s.io/v1` | scheduling group, version v1 | PriorityClass |

### Full YAML structure reminder

Every Kubernetes object has these four top-level fields:

```yaml
apiVersion: apps/v1        # Which API group + version
kind: Deployment           # What type of object
metadata:                  # Name, labels, namespace, annotations
  name: my-app
  namespace: default
  labels:
    app: my-app
    env: production
  annotations:
    description: "My main web app"
spec:                      # What you want (desired state)
  replicas: 3
  ...
```

Example Pod:
```yaml
apiVersion: v1             # Core group (no group name needed)
kind: Pod
metadata:
  name: my-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

Example Deployment:
```yaml
apiVersion: apps/v1        # apps group, version v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

---

## Calling the API Directly (curl)

You can talk to the kube-apiserver directly. This is useful for understanding how `kubectl` works under the hood.

```bash
# Get all pods in the default namespace (Core group)
curl -k https://<kube-apiserver-ip>/api/v1/namespaces/default/pods

# Get all deployments (Named group: apps/v1)
curl -k https://<kube-apiserver-ip>/apis/apps/v1/namespaces/default/deployments

# Get cluster-level resources (PersistentVolumes — no namespace)
curl -k https://<kube-apiserver-ip>/api/v1/persistentvolumes

# List all API groups
curl -k https://<kube-apiserver-ip>/apis
```

The pattern:
- Core group: `/api/v1/...`
- Named group: `/apis/<group>/<version>/...`

In practice, you always use `kubectl`, but knowing the URL structure helps when troubleshooting or using API clients.

---

## Updating Kubernetes Objects

Three ways to update a running resource:

### 1. kubectl apply (declarative — recommended)
Edit the YAML file, then apply:
```bash
kubectl apply -f nginx-deployment.yaml
```
Kubernetes computes the diff and applies only what changed.

### 2. kubectl edit (live editor)
Opens your default editor (`$EDITOR`, usually vim) with the live YAML:
```bash
kubectl edit deployment nginx-deployment
kubectl edit pod my-pod
```
Save and close the editor — changes apply immediately. Good for quick changes.

### 3. kubectl set (imperative — fast)
Update a specific field without opening a file:
```bash
# Update container image
kubectl set image deployment/nginx-deployment nginx=nginx:1.29

# Update environment variable
kubectl set env deployment/nginx-deployment ENV=production

# Verify the update
kubectl rollout status deployment/nginx-deployment
```

### Choosing which method to use

| Method | When to use |
|--------|-------------|
| `kubectl apply -f` | Production, CI/CD, declarative workflow |
| `kubectl edit` | Quick fixes, learning, debugging |
| `kubectl set image` | Fast image updates during practice/CKA exam |

---

## Useful API Discovery Commands

```bash
# List all resource types with their API version
kubectl api-resources -o wide

# Explain a resource (shows all fields with documentation)
kubectl explain pod
kubectl explain deployment.spec
kubectl explain deployment.spec.strategy

# Explain a specific field recursively
kubectl explain pod.spec.containers --recursive

# Get API version for a resource
kubectl api-resources | grep deployment
```

`kubectl explain` is your best friend on the CKA exam — it shows you the YAML structure without needing internet access.

---

## Quick reference — which apiVersion to use?

| Kind | apiVersion |
|------|-----------|
| Pod | `v1` |
| Service | `v1` |
| ConfigMap | `v1` |
| Secret | `v1` |
| PersistentVolume | `v1` |
| PersistentVolumeClaim | `v1` |
| Namespace | `v1` |
| ServiceAccount | `v1` |
| Deployment | `apps/v1` |
| ReplicaSet | `apps/v1` |
| StatefulSet | `apps/v1` |
| DaemonSet | `apps/v1` |
| Job | `batch/v1` |
| CronJob | `batch/v1` |
| HorizontalPodAutoscaler | `autoscaling/v2` |
| Ingress | `networking.k8s.io/v1` |
| NetworkPolicy | `networking.k8s.io/v1` |
| Role | `rbac.authorization.k8s.io/v1` |
| ClusterRole | `rbac.authorization.k8s.io/v1` |
| RoleBinding | `rbac.authorization.k8s.io/v1` |
| ClusterRoleBinding | `rbac.authorization.k8s.io/v1` |
| PriorityClass | `scheduling.k8s.io/v1` |
| StorageClass | `storage.k8s.io/v1` |

---

## CRDs — CustomResourceDefinitions

### What is a CRD?

Kubernetes ships with built-in resources like `Pod`, `Deployment`, and `Service`. A **CustomResourceDefinition (CRD)** lets you teach Kubernetes about **your own resource types**.

**Analogy:** Kubernetes is a database. Built-in resources are pre-built tables (pods, services). A CRD is a `CREATE TABLE` statement — you're adding a new table with your own schema, and then Kubernetes will store and serve objects of that type via its API.

Once you install a CRD, you can do:
```bash
kubectl get <your-custom-resource>
kubectl apply -f <your-custom-resource>.yaml
kubectl describe <your-custom-resource> <name>
```

### Discovering installed CRDs

```bash
# List all CRDs installed in the cluster
kubectl get crd
# NAME                                      CREATED AT
# certificaterequests.cert-manager.io       2024-01-01T10:00:00Z
# certificates.cert-manager.io             2024-01-01T10:00:00Z
# gateways.gateway.networking.k8s.io       2024-01-01T10:05:00Z

# Get details of a specific CRD
kubectl describe crd certificates.cert-manager.io

# Use kubectl explain to explore CRD schema (same as built-in resources!)
kubectl explain certificates.cert-manager.io
kubectl explain certificates.cert-manager.io.spec
kubectl explain certificates.cert-manager.io.spec.secretName
```

### What does a CRD YAML look like?

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.example.com   # must be: <plural>.<group>
spec:
  group: example.com           # API group — used in apiVersion field
  versions:
    - name: v1
      served: true             # this version is active
      storage: true            # this is the version stored in etcd
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                color:
                  type: string
                size:
                  type: integer
  scope: Namespaced            # or Cluster
  names:
    plural: widgets
    singular: widget
    kind: Widget               # the Kind used in YAML
    shortNames:
      - wg
```

```bash
# Install the CRD
kubectl apply -f widget-crd.yaml

# Verify it exists
kubectl get crd widgets.example.com

# Now create a Widget custom resource
kubectl apply -f - <<EOF
apiVersion: example.com/v1
kind: Widget
metadata:
  name: my-widget
spec:
  color: blue
  size: 3
EOF

# List Widgets — works just like kubectl get pods
kubectl get widgets
kubectl get widget my-widget -o yaml
kubectl delete widget my-widget
```

---

## Operators

### What is an Operator?

An **Operator** is a Kubernetes application that:
1. Defines one or more **CRDs** (your custom resource type)
2. Runs a **controller** (a process watching for changes to those custom resources)
3. **Automates** complex operational tasks that a human admin would otherwise do manually

**Analogy:** A CRD is a job description ("We need a Database"). An Operator is the expert employee who reads that job description and actually sets everything up, monitors it, and fixes problems.

```
User creates:         Operator sees it and:
┌──────────────┐      ┌──────────────────────────────────────────────────┐
│  kind: Kafka │  →   │  Creates StatefulSet + Services + ConfigMaps     │
│  name: prod  │      │  Waits for pods to be ready                      │
│  replicas: 3 │      │  Configures inter-broker communication           │
└──────────────┘      │  Sets up replication topics                      │
                      │  Monitors health and auto-heals                  │
                      └──────────────────────────────────────────────────┘
```

### Operator vs plain Helm chart

| | Helm Chart | Operator |
|---|---|---|
| Installs resources | ✅ | ✅ |
| Understands app state | ❌ | ✅ |
| Auto-heals failures | ❌ | ✅ |
| Handles upgrades | Manual rollout | Automated |
| Examples | nginx, prometheus | cert-manager, Postgres Operator |

### Installing an Operator (cert-manager example)

cert-manager is a popular Operator that automatically provisions TLS certificates.

```bash
# Method 1: kubectl apply (simplest)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.0/cert-manager.yaml

# What this installs:
# - CRDs: Certificate, CertificateRequest, Issuer, ClusterIssuer, Challenge, Order
# - Namespace: cert-manager
# - Deployments: cert-manager, cert-manager-webhook, cert-manager-cainjector
# - RBAC: ClusterRoles and ClusterRoleBindings

# Wait for the operator to be ready
kubectl wait --for=condition=Available deployment --all -n cert-manager --timeout=120s

# Verify CRDs are installed
kubectl get crd | grep cert-manager
# certificaterequests.cert-manager.io    ...
# certificates.cert-manager.io          ...
# clusterissuers.cert-manager.io        ...
# issuers.cert-manager.io               ...

# Now you can use the custom resources the operator understands
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF

kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-cert
  namespace: default
spec:
  secretName: my-cert-tls
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  commonName: my-app.example.com
  dnsNames:
    - my-app.example.com
EOF

# The operator sees this Certificate resource and automatically creates a TLS Secret
kubectl get certificate my-cert
kubectl get secret my-cert-tls
```

```bash
# Method 2: Helm (more config options)
helm repo add cert-manager https://charts.jetstack.io
helm repo update
helm install cert-manager cert-manager/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

# Uninstall
helm uninstall cert-manager -n cert-manager
kubectl delete namespace cert-manager
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.0/cert-manager.yaml
```

### Inspect any operator's resources

```bash
# See all resource types an operator added
kubectl api-resources | grep cert-manager
# certificates      cert      cert-manager.io/v1   true    Certificate
# issuers                     cert-manager.io/v1   true    Issuer
# clusterissuers              cert-manager.io/v1   false   ClusterIssuer

# Explore the schema of a custom resource
kubectl explain certificate
kubectl explain certificate.spec
kubectl explain certificate.spec.issuerRef

# Get all custom resources across all CRDs
kubectl get $(kubectl api-resources --verbs=list --namespaced -o name | paste -s -d,) -A 2>/dev/null
```

### CKA exam — what you need to know

```bash
# List all CRDs in the cluster
kubectl get crd

# Check what CRDs an operator installed
kubectl get crd | grep <operator-name>

# Inspect a custom resource schema
kubectl explain <crd-kind>
kubectl explain <crd-kind>.spec.<field>

# Get all instances of a custom resource
kubectl get <crd-kind> -A

# Install/uninstall an operator via kubectl apply
kubectl apply -f <operator-manifest-url>
kubectl delete -f <operator-manifest-url>

# Check operator pod status
kubectl get pods -n <operator-namespace>
kubectl logs -n <operator-namespace> <operator-pod>
```
