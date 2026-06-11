# Kubernetes RBAC — Session Notes

---

## What is RBAC?

- RBAC = Role-Based Access Control.
- Controls WHO can do WHAT on WHICH resources in the cluster.
- Enabled by default in Kubernetes (`--authorization-mode=RBAC` on API server).
- Four key objects: **Role**, **ClusterRole**, **RoleBinding**, **ClusterRoleBinding**.

---

## RBAC building blocks

```
WHO (Subject)          +  WHAT (Role)           +  WHERE (Binding)
───────────────────       ──────────────────       ─────────────────────
• User                    • Role (namespace)       • RoleBinding (namespace)
• Group                   • ClusterRole (cluster)  • ClusterRoleBinding (cluster)
• ServiceAccount
```

### Key formula from session:
> **Role + RoleBinding = Namespace scope**
> **ClusterRole + ClusterRoleBinding = Cluster-wide scope**

---

## Scope comparison

| | Role + RoleBinding | ClusterRole + ClusterRoleBinding |
|--|-------------------|----------------------------------|
| Scope | Single namespace | Entire cluster |
| Resources | Namespaced resources (pods, services, etc.) | All resources including non-namespaced (nodes, PVs, namespaces) |
| Use case | Dev team access to their namespace | Cluster admin, viewing nodes |

---

## Session Demo — Role for namespace `dev`

### Step 1: Create namespace and user context
```bash
# Create namespace
kubectl create namespace dev
```

### Step 2: Create Role (from session — dev-role)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: dev-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
```

### Understanding the Role YAML:
| Field | Meaning |
|-------|---------|
| `namespace: dev` | This role only works in namespace `dev` |
| `apiGroups: [""]` | Core API group (pods, services, configmaps, etc.) |
| `apiGroups: ["apps"]` | Apps API group (deployments, replicasets, statefulsets) |
| `resources` | Which resources this role can access |
| `verbs` | What actions are allowed |

### Common verbs:
| Verb | Meaning |
|------|---------|
| `get` | Read a specific resource |
| `list` | List all resources of a type |
| `watch` | Watch for changes (streaming) |
| `create` | Create new resources |
| `update` | Modify existing resources |
| `patch` | Partially modify resources |
| `delete` | Delete resources |
| `*` | All verbs (full access) |

### Step 3: Create RoleBinding (from session — dev-rolebinding)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-rolebinding
  namespace: dev
subjects:
- kind: User
  name: cka1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
```

### Understanding the RoleBinding YAML:
| Field | Meaning |
|-------|---------|
| `subjects` | WHO gets the permissions |
| `subjects[].kind: User` | A human user (or `ServiceAccount`, `Group`) |
| `subjects[].name: cka1` | The specific user named "cka1" |
| `roleRef` | WHICH role to attach |
| `roleRef.kind: Role` | Linking to a Role (namespace-scoped) |
| `roleRef.name: dev-role` | The specific role to bind |

### Step 4: Test with `kubectl auth can-i` (from session)
```bash
# Can user cka1 delete pods in dev namespace?
kubectl auth can-i delete pods --as=cka1 -n dev
# Output: no ← Because dev-role only has get/list/watch

# Can user cka1 list pods in dev namespace?
kubectl auth can-i list pods --as=cka1 -n dev
# Output: yes ✅

# Can user cka1 list pods in default namespace?
kubectl auth can-i list pods --as=cka1 -n default
# Output: no ← Role only applies in dev namespace

# Can user cka1 create deployments in dev?
kubectl auth can-i create deployments --as=cka1 -n dev
# Output: no ← Only get/list/watch allowed
```

---

## ClusterRole + ClusterRoleBinding (cluster-wide)

### ClusterRole — can view nodes (non-namespaced resource)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

### ClusterRoleBinding — bind to user
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-viewer-binding
subjects:
- kind: User
  name: cka1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
```

### Test
```bash
kubectl auth can-i list nodes --as=cka1
# Output: yes ✅ (cluster-wide, no namespace needed)
```

---

## ServiceAccount RBAC

Pods use ServiceAccounts (not Users) to authenticate to the API server.

### Create ServiceAccount and bind a role
```bash
# Create SA
kubectl create serviceaccount deploy-bot -n dev

# Create role for deployments (full CRUD)
kubectl create role deploy-manager \
  --verb=get,list,create,update,delete \
  --resource=deployments \
  -n dev

# Bind it
kubectl create rolebinding deploy-bot-binding \
  --role=deploy-manager \
  --serviceaccount=dev:deploy-bot \
  -n dev
```

### Use SA in a pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: deployer-pod
  namespace: dev
spec:
  serviceAccountName: deploy-bot
  containers:
  - name: kubectl
    image: bitnami/kubectl
    command: ["sleep", "3600"]
```

---

## Imperative commands (quick for CKA exam)

```bash
# Create Role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n dev

# Create ClusterRole
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# Create RoleBinding
kubectl create rolebinding dev-read \
  --role=pod-reader \
  --user=cka1 \
  -n dev

# Create ClusterRoleBinding
kubectl create clusterrolebinding global-node-read \
  --clusterrole=node-reader \
  --user=cka1

# Bind a ClusterRole at namespace level (reuse ClusterRole in a namespace)
kubectl create rolebinding dev-admin \
  --clusterrole=admin \
  --user=cka1 \
  -n dev
```

---

## Special: ClusterRole + RoleBinding combo

You CAN bind a ClusterRole using a RoleBinding — this limits the ClusterRole to a single namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-admin-binding
  namespace: dev
subjects:
- kind: User
  name: cka1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole        # Using a ClusterRole
  name: admin              # Built-in admin ClusterRole
  apiGroup: rbac.authorization.k8s.io
```
This gives `cka1` admin-level access **ONLY in namespace dev**.

---

## Built-in ClusterRoles

| ClusterRole | Permissions |
|-------------|-------------|
| `cluster-admin` | Full access to everything (superuser) |
| `admin` | Full access within a namespace (no quota/namespace modification) |
| `edit` | Read/write most resources in a namespace (no roles/bindings) |
| `view` | Read-only access to most resources in a namespace |

```bash
# See all cluster roles
kubectl get clusterroles

# See what a built-in role allows
kubectl describe clusterrole view
```

---

## Useful commands
```bash
# List roles/bindings
kubectl get roles -n dev
kubectl get rolebindings -n dev
kubectl get clusterroles
kubectl get clusterrolebindings

# Check permissions
kubectl auth can-i list pods --as=cka1 -n dev
kubectl auth can-i '*' '*' --as=system:admin   # Check if full admin

# Check YOUR own permissions
kubectl auth can-i create pods
kubectl auth can-i --list    # List ALL your permissions

# Who can do what?
kubectl auth can-i delete pods --as=cka1 -n dev --list
```

---

## CKA exam tips

1. `kubectl auth can-i` is your best friend for verifying RBAC.
2. Use imperative commands to create roles/bindings quickly in exam.
3. Remember: Role → namespace, ClusterRole → cluster-wide.
4. A RoleBinding can reference a ClusterRole (limits it to one namespace).
5. `apiGroups: [""]` = core API (pods, services, nodes, configmaps, secrets, PVs, PVCs).
6. `apiGroups: ["apps"]` = deployments, replicasets, statefulsets, daemonsets.
7. ServiceAccounts format in bindings: `--serviceaccount=<namespace>:<sa-name>`.
