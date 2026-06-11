# Kubernetes ConfigMaps, Secrets & Volumes — Session Notes

---

## ConfigMap — store non-sensitive configuration

- ConfigMap stores configuration data as key-value pairs.
- Keeps config OUTSIDE your container image (so you don't rebuild image for config changes).
- Can be consumed as environment variables OR mounted as files inside the pod.

### Create ConfigMap (from session — html-config)

Based on session history: `vi cm.yaml` → `k apply -f cm.yaml` → `k describe cm html-config`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: html-config
data:
  index.html: |
    <html>
      <head><title>Welcome</title></head>
      <body>
        <h1>Hello from ConfigMap!</h1>
        <p>This page is served from a ConfigMap volume.</p>
      </body>
    </html>
```

### Pod using ConfigMap as a volume (from session — volume-demo)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: html-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: html-volume
    configMap:
      name: html-config
```

### Session workflow
```bash
# 1. Create the ConfigMap
kubectl apply -f cm.yaml

# 2. Verify ConfigMap created
kubectl describe cm html-config

# 3. Create the pod that mounts it
kubectl apply -f pod.yaml

# 4. Exec into pod and verify the file is there
kubectl exec -it volume-demo -- /bin/sh
# Inside pod:
cat /usr/share/nginx/html/index.html
# You'll see the HTML from the ConfigMap!

# 5. Key insight: ConfigMap exists independently of the pod
kubectl delete pod volume-demo
kubectl describe cm html-config   # Still exists!
```

### Other ways to create ConfigMap
```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=DB_HOST=postgres \
  --from-literal=DB_PORT=5432

# From a file
kubectl create configmap nginx-conf --from-file=nginx.conf

# From a directory (each file becomes a key)
kubectl create configmap configs --from-file=./config-dir/
```

### Use ConfigMap as environment variables
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo $DB_HOST && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
```

### Use specific keys as env vars
```yaml
    env:
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
```

### Useful commands
```bash
kubectl get configmaps
kubectl describe cm html-config
kubectl get cm html-config -o yaml    # See full content

# Edit live ConfigMap
kubectl edit cm html-config

# Delete
kubectl delete cm html-config
```

### Key points
- ConfigMap changes are reflected in mounted volumes (eventually, ~1 min delay).
- ConfigMap changes are NOT reflected in environment variables (pod restart needed).
- ConfigMap must exist BEFORE the pod starts (unless marked optional).
- Max size: 1 MB per ConfigMap.

---

## Secrets — store sensitive data

- Secrets store sensitive data (passwords, tokens, keys).
- Base64 encoded (NOT encrypted by default!).
- Similar to ConfigMap but intended for sensitive information.
- Kubernetes can restrict access to Secrets via RBAC.

### Create Secret
```bash
# From literals
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t

# From file (e.g., TLS cert)
kubectl create secret tls my-tls \
  --cert=tls.crt \
  --key=tls.key

# From file as generic secret
kubectl create secret generic ssh-key --from-file=id_rsa=~/.ssh/id_rsa
```

### Secret YAML (values must be base64 encoded)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
data:
  username: YWRtaW4=        # echo -n "admin" | base64
  password: czNjcjN0        # echo -n "s3cr3t" | base64
```

### Use `stringData` to avoid manual base64
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
stringData:
  username: admin
  password: s3cr3t
```

### Mount Secret as environment variable
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo $DB_USER && sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
```

### Mount Secret as volume (files)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /etc/secrets/username && sleep 3600"]
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-creds
```

### Secret types
| Type | Use case |
|------|----------|
| `Opaque` | Generic (default) — any key-value data |
| `kubernetes.io/tls` | TLS certificate + key |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/service-account-token` | SA token (auto-created) |

### Useful commands
```bash
kubectl get secrets
kubectl describe secret db-creds        # Shows keys but NOT values
kubectl get secret db-creds -o yaml     # Shows base64-encoded values

# Decode a secret value
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

### ConfigMap vs Secret comparison
| | ConfigMap | Secret |
|--|----------|--------|
| Data type | Non-sensitive config | Sensitive data (passwords, keys) |
| Encoding | Plain text | Base64 encoded |
| Size limit | 1 MB | 1 MB |
| Mounted as | Files or env vars | Files or env vars |
| Access control | Standard RBAC | Can be further restricted |
| In etcd | Plain text | Can be encrypted at rest |

---

## EmptyDir — shared temporary storage between containers

- `emptyDir` is a temporary volume created when a pod is assigned to a node.
- It starts empty and is **deleted when the pod is removed** from the node.
- Main use case: sharing files between containers in the same pod.

### EmptyDir in the session (full-demo pod)
From the session, the `full-demo` pod uses emptyDir to share data between init container, main container, and sidecar:

```yaml
volumes:
- name: workdir
  emptyDir: {}
```

### Simple EmptyDir example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo 'Hello from writer' > /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ["sh", "-c", "sleep 5 && cat /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
```

### EmptyDir with memory backing (RAM disk)
```yaml
volumes:
- name: cache
  emptyDir:
    medium: Memory        # Uses RAM instead of disk (tmpfs)
    sizeLimit: 100Mi      # Limit to 100MB
```

### Key points about EmptyDir
| Feature | Behavior |
|---------|----------|
| Lifetime | Same as the pod (deleted when pod dies) |
| Shared between | All containers in the SAME pod |
| Default storage | Node's disk |
| `medium: Memory` | Uses RAM (faster but limited) |
| Survives container restart | YES (pod is still alive) |
| Survives pod restart | NO (new pod = new emptyDir) |

### Common use cases for EmptyDir
1. **Scratch space** — temp processing area for sorting/caching.
2. **Sharing data between containers** — init container writes, app container reads.
3. **Log aggregation** — app writes logs, sidecar ships them.
4. **Cache** — store intermediate computation results.
