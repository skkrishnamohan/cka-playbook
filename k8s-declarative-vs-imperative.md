# Imperative vs Declarative, Kubernetes YAML, apiVersion, and Dry-Run

A beginner-friendly guide that explains:
- Imperative vs Declarative methods for Kubernetes
- What `apiVersion`, `kind`, `metadata`, and `spec` mean in Kubernetes YAML
- YAML basics you need to avoid common mistakes
- How to use `--dry-run` and related practical hacks

This note focuses on clarity for beginners while including useful tips for more experienced users.

---

## 1) Imperative vs Declarative — the core idea

Imperative:
- You tell the system exactly what to do, step by step.
- Commands create resources immediately.
- Fast for quick tasks, experiments, and learning.

Examples (imperative):
```bash
# Create a Pod quickly
kubectl run mypod --image=nginx:1.28

# Create a deployment quickly
kubectl create deployment web --image=nginx:1.28 --replicas=2

# Set image imperatively
kubectl set image deployment/web web=nginx:1.29
```

Pros:
- Fast, low ceremony, good for ad-hoc changes and testing.

Cons:
- Harder to track history and reproduce exact state later.
- Not recommended for production infrastructure as single source of truth.


Declarative:
- You declare desired state in files (YAML/JSON) and apply them.
- Kubernetes controller works to make actual state match desired state.

Examples (declarative):
```yaml
# file: deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.28
```

Apply with:
```bash
kubectl apply -f deployment.yaml
```

Pros:
- Reproducible, versionable (store in git), auditable (config as code).
- Easier to review, test and roll back.

Cons:
- Slightly more setup initially; need to manage file versions.

When to use which:
- Use imperative locally for exploration and debugging.
- Use declarative for production, CI/CD, and collaboration.

---

## 2) What is YAML? Quick primer

- YAML is a human-readable data serialization format used by Kubernetes for manifests.
- Key rules:
  - Indentation matters (use spaces, not tabs).
  - Mappings use `key: value`.
  - Lists use `- item` under an indented key.

Example:
```yaml
fruits:
  - apple
  - banana

person:
  name: Alice
  age: 30
```

Common YAML mistakes in k8s manifests:
- Using tabs instead of spaces — parser errors.
- Wrong indentation for `- name:` under containers.
- Missing `-` for list elements (containers, volumeClaimTemplates).

Tip: Use `kubectl apply -f file.yaml --dry-run=client` to catch syntax issues quickly.

---

## 3) Core Kubernetes YAML keys: `apiVersion`, `kind`, `metadata`, `spec`

Every Kubernetes manifest usually contains these core fields. Think of them as the minimum contract.

- `apiVersion`: The API group and version for the resource. Examples: `v1`, `apps/v1`, `batch/v1`.
  - Determines the schema and fields the server expects.
  - Use `kubectl api-resources` and `kubectl explain` to find valid `apiVersion`/`kind` combinations.
  - Example: Deployments live under `apps/v1` in modern clusters.

- `kind`: The resource type (Pod, Deployment, ReplicaSet, Service, StatefulSet, Job, etc.).

- `metadata`: Identifying information about the object.
  - `name` (required for most resources), `namespace` (if not default), `labels`, `annotations`.
  - Labels are key/value pairs used for selection. Annotations store non-identifying metadata.

- `spec`: The desired state specification of the object.
  - The content of `spec` differs for each `kind` (Deployment.spec, Pod.spec, Service.spec).
  - Example: `replicas`, `selector`, `template` for a Deployment.spec.

Minimal example explained:
```yaml
apiVersion: v1        # core group v1
kind: Pod             # resource type
metadata:
  name: simple-pod    # object name
spec:
  containers:
  - name: web
    image: nginx:1.28
```

Using `kubectl explain`:
```bash
kubectl explain deployment --recursive  # view fields and types for deployment
kubectl explain deployment.spec.template.spec.containers
```

---

## 4) `apiVersion` deeper: why it matters

- API groups evolve; multiple versions may exist (v1beta1, v1, etc.). A missing or incorrect `apiVersion` will cause validation errors.
- To discover current versions and resources:
```bash
kubectl api-resources
kubectl api-versions
```
- Always prefer stable `v1` APIs when available (e.g., `apps/v1` for Deployments).

---

## 5) Dry-run: what it is and why it is useful

Dry-run allows you to ask the API server (or client) to validate and show what would happen without persisting changes.

Modes:
- `--dry-run=client` (local validation only): kubectl performs client-side validation and does not contact the API server.
- `--dry-run=server` (server-side validation): the request reaches the API server; it performs admission controllers, defaulting, and validation, but no object is persisted.

Examples:
```bash
# Client dry-run (fast, local)
kubectl apply -f deployment.yaml --dry-run=client

# Server dry-run (recommended to see server defaults and admission decisions)
kubectl apply -f deployment.yaml --dry-run=server

# Output the would-be object YAML (server-side defaults applied)
kubectl apply -f deployment.yaml --dry-run=server -o yaml

# Create imperatively but only print the manifest (client-side)
kubectl create deployment web --image=nginx:1.28 --dry-run=client -o yaml

# Create imperatively and get server-validated manifest
kubectl create deployment web --image=nginx:1.28 --dry-run=server -o yaml
```

Why `--dry-run=server` is powerful:
- It shows server-side default values (like securityContext defaults, image pull policy) and admission controller effects.
- Good for validating CRDs and policies before applying.

Quick hacks using dry-run:
- Generate manifest from an imperative command and save it to a file:
  ```bash
  kubectl create deployment web --image=nginx:1.28 --dry-run=client -o yaml > web.yaml
  ```
- Preview server-side validated manifest (shows defaults and admission modifications):
  ```bash
  kubectl apply -f web.yaml --dry-run=server -o yaml
  ```
- Use `kubectl diff -f file.yaml` to show what would change on the server compared to current state.

---

## 6) Practical workflow recommendations (beginner → expert)

Beginner workflow (safe & repeatable):
1. Write a manifest `deployment.yaml`.
2. Validate locally: `kubectl apply -f deployment.yaml --dry-run=client`.
3. Validate server-side: `kubectl apply -f deployment.yaml --dry-run=server -o yaml`.
4. Preview changes: `kubectl diff -f deployment.yaml`.
5. Apply: `kubectl apply -f deployment.yaml`.

Improved (CI/CD) workflow:
- Keep manifests in Git. Use CI to run `kubectl apply --dry-run=server -f` (or helm template + dry-run) against a test cluster.
- Use `kubectl rollout status` and health probes to verify after apply.

Expert tips & hacks:
- Use `kubectl create ... --dry-run=client -o yaml` to generate skeleton manifests from imperative commands.
- Use `kubectl apply --server-side` or `--force` carefully when migrating resources between servers.
- Use `kubectl explain` to learn schema for any resource (great for CRDs and unfamiliar API versions).
- For complex changes, use small incremental changes and `kubectl rollout undo` on failures.

---

## 7) Quick reference commands

```bash
# Discover resources and versions
kubectl api-versions
kubectl api-resources

# Inspect resource schema
kubectl explain deployment --recursive

# Validate locally (client dry-run)
kubectl apply -f file.yaml --dry-run=client

# Validate on server (server dry-run) and print object
kubectl apply -f file.yaml --dry-run=server -o yaml

# Preview changes between local file and live cluster
kubectl diff -f file.yaml

# Generate manifest from imperative command
kubectl create deployment web --image=nginx:1.28 --dry-run=client -o yaml > web.yaml
```

---

## Additional CKA-relevant concepts

### kubectl apply vs kubectl replace vs kubectl create
| Command | Behavior | Use case |
|---------|----------|----------|
| `kubectl create` | Creates resource. Fails if already exists. | First-time creation only |
| `kubectl apply` | Creates OR updates. Uses three-way merge. | Declarative management (preferred) |
| `kubectl replace` | Replaces entire object. Fails if doesn't exist. | Full replacement (destructive) |

### Three-way merge in `kubectl apply`
- When you run `kubectl apply`, Kubernetes compares THREE things:
  1. Your local YAML file (what you want)
  2. The live object in the cluster (current state)
  3. The `last-applied-configuration` annotation (what you applied last time)
- This allows `kubectl apply` to know which fields YOU changed vs fields changed by other controllers (like HPA).
- The annotation is stored at: `metadata.annotations.kubectl.kubernetes.io/last-applied-configuration`

### kubectl patch — partial updates without full YAML
```bash
# Strategic merge patch (merge into existing)
kubectl patch deployment web -p '{"spec":{"replicas":5}}'

# JSON patch (precise operations)
kubectl patch deployment web --type='json' -p='[{"op":"replace","path":"/spec/replicas","value":5}]'

# Patch to add a label
kubectl patch pod mypod -p '{"metadata":{"labels":{"env":"prod"}}}'

# Patch to change container image
kubectl patch deployment web -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.29"}]}}}}'
```

### kubectl edit — live editing
- Opens the resource in your default editor (`$EDITOR` or vi).
- Changes are applied immediately when you save and exit.
- Useful for quick one-off changes but NOT recommended for production (no audit trail).
```bash
kubectl edit deployment web
kubectl edit svc my-service

# Use a specific editor
KUBE_EDITOR="nano" kubectl edit deployment web
```

### kubectl set — quick imperative updates
```bash
# Update image (most common in CKA exam)
kubectl set image deployment/web nginx=nginx:1.29

# Update resources
kubectl set resources deployment/web -c=nginx --limits=cpu=200m,memory=512Mi

# Update service account
kubectl set serviceaccount deployment/web my-sa
```

### Field selectors and label selectors
```bash
# Field selectors (filter on object fields)
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector spec.nodeName=worker1

# Label selectors
kubectl get pods -l app=web
kubectl get pods -l 'app in (web,api)'
kubectl get pods -l app=web,tier=frontend

# Show labels as columns
kubectl get pods --show-labels
kubectl get pods -L app,tier   # Show specific labels as columns
```

### Output formatting (critical for CKA speed)
```bash
# JSON path — extract specific fields
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# Sort by field
kubectl get pods --sort-by=.metadata.creationTimestamp

# Get specific field quickly
kubectl get pod mypod -o jsonpath='{.spec.containers[0].image}'
```

### CKA exam tip: fastest workflow
1. Generate YAML with `--dry-run=client -o yaml > file.yaml`
2. Edit the YAML to add what you need
3. Apply with `kubectl apply -f file.yaml`
4. Verify with `kubectl get` / `kubectl describe`

This is faster than writing YAML from scratch and less error-prone than pure imperative commands for complex objects.
