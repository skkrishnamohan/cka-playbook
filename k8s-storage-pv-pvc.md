# Kubernetes Storage — PV, PVC, StorageClass & NFS

---

## Storage concepts overview

```
Pod
  ↓
PersistentVolumeClaim (PVC)   ← What the Pod requests
  ↓
PersistentVolume (PV)         ← The actual storage resource
  ↓
Physical Disk / NFS / Cloud disk
```

- **PV** (PersistentVolume): A piece of storage provisioned by an admin or dynamically by a StorageClass.
- **PVC** (PersistentVolumeClaim): A request for storage by a Pod — like asking for 5Gi with ReadWriteOnce.
- **StorageClass**: Defines how storage is dynamically provisioned (which provisioner, what disk type, reclaim policy).

Without StorageClass (static provisioning): Admin manually creates PVs → user creates PVC → K8s binds them.
With StorageClass (dynamic provisioning): User creates PVC → StorageClass auto-creates PV → bound automatically.

---

## PersistentVolume (PV) — the storage resource

- Created by a cluster admin (or automatically by a StorageClass).
- Lives at cluster scope (not namespace-specific).
- Represents actual storage: a disk, NFS share, cloud volume, etc.

### Static PV example (hostPath — for local/dev use)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce             # One node can mount read-write
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/my-pv         # Node-local path (not for production)
```

### Static PV example (NFS)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany             # Multiple nodes can mount read-write
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.56.200    # NFS server IP
    path: /exports/data       # Exported path on NFS server
```

---

## PersistentVolumeClaim (PVC) — the storage request

- Created by the application team / user.
- Lives in a namespace.
- Kubernetes finds a matching PV (by size, access mode, StorageClass) and binds them.

### PVC example
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard    # Must match PV or StorageClass name
```

### Using a PVC in a Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /data          # Where in the container to mount
      name: my-storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-pvc         # Reference to the PVC
```

### PVC commands
```bash
# Check PVC status (should be Bound)
kubectl get pvc
kubectl get pvc -n default

# Describe PVC (shows binding details)
kubectl describe pvc my-pvc

# Check PV status
kubectl get pv

# PVC states:
# Pending  = no matching PV found yet
# Bound    = successfully bound to a PV
# Lost     = PV was deleted while PVC existed
```

---

## Access Modes — how volumes can be mounted

| Mode | Short | Meaning |
|------|-------|---------|
| ReadWriteOnce | RWO | One node mounts read-write (most common) |
| ReadOnlyMany | ROX | Many nodes mount read-only |
| ReadWriteMany | RWX | Many nodes mount read-write (NFS, Azure Files, EFS) |
| ReadWriteOncePod | RWOP | Only one Pod (not just node) can mount read-write |

Storage type comparison:
| Storage | Access Mode |
|---------|------------|
| hostPath | RWO |
| Azure Disk / EBS | RWO |
| NFS | RWX |
| Azure Files / EFS | RWX |

---

## Reclaim Policies — what happens when PVC is deleted

| Policy | Behaviour |
|--------|-----------|
| **Retain** | PV stays, data preserved. Admin must manually clean up. |
| **Delete** | PV and underlying storage are automatically deleted. |
| **Recycle** | (Deprecated) Wipes data and makes PV available again. |

```yaml
# In PV spec:
persistentVolumeReclaimPolicy: Retain   # or Delete
```

---

## StorageClass — dynamic provisioning

- What: A StorageClass tells Kubernetes how to automatically create PVs when a PVC requests them.
- Why: Without it, admins manually create PVs. With it, storage is self-service.
- Provisioner: the plugin that creates the actual disk (cloud-specific or local).

### StorageClass for local storage (bare-metal)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner    # No dynamic provisioning, admin creates PVs
volumeBindingMode: WaitForFirstConsumer      # Don't bind PV until Pod is scheduled
```

> `WaitForFirstConsumer` prevents a PV from being bound to a node before the Pod that needs it is scheduled — avoids storage/scheduling mismatches.

### StorageClass for cloud (e.g. AWS EBS)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### PVC using a StorageClass (dynamic provisioning)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd        # Triggers dynamic PV creation
  resources:
    requests:
      storage: 20Gi
```

### StorageClass commands
```bash
# List storage classes
kubectl get storageclass
kubectl get sc                     # Short form

# Describe a storage class
kubectl describe sc local-storage

# Set a default storage class (PVCs without storageClassName use this)
kubectl patch storageclass local-storage \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## NFS — ReadWriteMany shared storage

- What: Network File System — a shared storage server that multiple nodes can mount simultaneously.
- Why: The only common way to get `ReadWriteMany (RWX)` access mode in on-prem clusters.
- Use cases: Shared config files, media files, data shared across many Pods.

### NFS architecture
```
NFS Server (192.168.56.200)
    ↓  exports /exports/data
Node1  ←── mounts ──→  Node2  ←── mounts ──→  Node3
  ↓                     ↓                       ↓
Pod A              Pod B & C               Pod D & E
(all read/write the same files)
```

### NFS PV + PVC example
```yaml
# Step 1: Admin creates PV pointing to NFS
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.56.200
    path: /exports/data
---
# Step 2: App team creates PVC requesting NFS storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: ""              # Empty = use static PV, not dynamic
---
# Step 3: Pod uses the PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shared-app
spec:
  replicas: 3                       # All 3 replicas share the same NFS volume
  selector:
    matchLabels:
      app: shared
  template:
    metadata:
      labels:
        app: shared
    spec:
      containers:
      - name: app
        image: nginx
        volumeMounts:
        - mountPath: /shared-data
          name: nfs-volume
      volumes:
      - name: nfs-volume
        persistentVolumeClaim:
          claimName: nfs-pvc
```

---

## StatefulSet Storage — stable per-Pod volumes

StatefulSets use `volumeClaimTemplates` — each Pod gets its own PVC automatically:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: db
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: db
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
# Creates: data-database-0, data-database-1, data-database-2
```

---

## Storage decision guide

| Requirement | Solution |
|------------|---------|
| Single Pod needs persistent storage | PV + PVC with RWO |
| Multiple Pods share the same data | NFS with RWX |
| Auto-provision storage on demand | StorageClass (dynamic) |
| Bare-metal cluster, no cloud | Local StorageClass or NFS |
| StatefulSet with per-pod storage | volumeClaimTemplates |
| Preserve data after PVC delete | reclaimPolicy: Retain |

---

## One-line takeaways
- PV = the actual storage; PVC = the claim/request for that storage; Pod uses PVC.
- StorageClass eliminates manual PV creation — just create a PVC and storage appears.
- `ReadWriteOnce (RWO)` = one node; `ReadWriteMany (RWX)` = many nodes (NFS/Azure Files/EFS).
- `WaitForFirstConsumer` on StorageClass prevents scheduling vs. storage location conflicts.
- NFS is the go-to RWX solution for on-prem clusters — multiple pods on different nodes share the same files.
- `reclaimPolicy: Retain` keeps data safe after PVC deletion; `Delete` removes everything automatically.

---

## local PV — node-specific persistent storage

- What: A `local` PV uses a specific directory on a specific node. Unlike `hostPath`, it requires `nodeAffinity` and integrates properly with StorageClass schedulers.
- Why not hostPath: `hostPath` is informal — no scheduling awareness, no PVC binding, unsafe in production. `local` PV works with `WaitForFirstConsumer` to ensure pods are only scheduled to the node where data actually lives.
- Use case: SSDs physically attached to a bare-metal node (databases with high I/O requirements).

### hostPath vs local PV comparison

| Feature | hostPath | local PV |
|---------|---------|----------|
| Node pinning | No — pod can land on any node | Yes — enforced by `nodeAffinity` |
| Works with PVC | No | Yes |
| StorageClass support | No | Yes (`WaitForFirstConsumer`) |
| Production safe | No | Yes (with scheduler awareness) |
| Survives pod restart | Only if pod lands on same node | Always (affinity guarantees same node) |

### local PV YAML (mandatory nodeAffinity)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage     # Must match the StorageClass
  local:
    path: /mnt/disks/ssd1             # Real path on the specific node
  nodeAffinity:                        # MANDATORY — ties this PV to one node
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node1                       # Only node1 has /mnt/disks/ssd1
```

Matching StorageClass (already covered above, reuse `local-storage`):
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner   # Manual (admin creates PVs)
volumeBindingMode: WaitForFirstConsumer     # Don't bind until pod is scheduled
```

PVC to claim local PV:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 10Gi
```

Pod using local PV (Kubernetes schedules it to node1 automatically):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  containers:
  - name: db
    image: postgres:15
    volumeMounts:
    - name: db-data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: db-data
    persistentVolumeClaim:
      claimName: local-pvc
# Kubernetes sees nodeAffinity on the PV → schedules pod to node1
```

---

## CSI Drivers — the modern plugin standard

- What: CSI (Container Storage Interface) is the standard API for storage plugins in Kubernetes. All major cloud and on-prem storage systems now provide CSI drivers.
- Why CSI: Before CSI, storage code was compiled directly into Kubernetes (called "in-tree plugins"). This made updates slow and risky. CSI drivers are external — installed separately, updated independently.
- Status: In-tree plugins (e.g., `kubernetes.io/aws-ebs`) are being deprecated. CSI replacements are now the standard.

### Common CSI drivers table

| Storage | CSI Driver Name | Use Case |
|---------|----------------|----------|
| AWS EBS | `ebs.csi.aws.com` | RWO block storage on AWS |
| GCE Persistent Disk | `pd.csi.storage.gke.io` | RWO block storage on GCP |
| Azure Disk | `disk.csi.azure.com` | RWO block storage on Azure |
| Azure Files | `file.csi.azure.com` | RWX file storage on Azure |
| OpenEBS Local | `openebs.io/local` | RWO on bare-metal (hostpath-based) |
| NFS CSI | `nfs.csi.k8s.io` | RWX NFS in any cluster |
| Ceph RBD | `rbd.csi.ceph.com` | RWO block storage (on-prem) |
| Longhorn | `driver.longhorn.io` | RWO distributed storage (on-prem) |

### How CSI driver appears in a StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com          # ← This is the CSI driver name
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### How CSI works (high level)

```
kubectl apply -f pvc.yaml
       ↓
StorageClass provisioner (CSI driver) called
       ↓
CSI driver calls cloud API (e.g. AWS CreateVolume)
       ↓
PV created automatically + PVC bound
       ↓
Pod scheduled → CSI driver attaches + mounts volume
```

---

## OpenEBS — dynamic provisioning on bare-metal (full exercise)

OpenEBS is a popular open-source storage solution for Kubernetes clusters running on bare metal (no cloud storage). It dynamically provisions local PVs using the node's disk.

- `openebs.io/local` driver uses actual node directories (like hostPath but managed by CSI)
- Perfect for CKA lab clusters (kubeadm, kind, minikube)
- PVC → CSI driver creates directory on node → Pod gets it as a volume

> **Killercoda prerequisite:** OpenEBS is NOT pre-installed. Install it first:
> ```bash
> kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
> # Wait for all pods to reach Running state (takes 2-3 minutes)
> kubectl get pods -n openebs -w
> # Press Ctrl+C when all pods are Running, then verify the StorageClass was created
> kubectl get sc
> # You should see: openebs-hostpath   openebs.io/local   Delete   WaitForFirstConsumer ...
> ```

### Step 1 — StorageClass (provisioner = OpenEBS local)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-hostpath
provisioner: openebs.io/local           # OpenEBS CSI driver
reclaimPolicy: Delete                   # Delete volume when PVC deleted
volumeBindingMode: WaitForFirstConsumer # Don't provision until pod is scheduled
```

### Step 2 — PVC (triggers dynamic PV creation)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: html-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: openebs-hostpath    # Links to the StorageClass above
  resources:
    requests:
      storage: 1Gi
# PV is NOT created yet — WaitForFirstConsumer means wait until a pod requests it
```

### Step 3 — Pod that uses the PVC
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-storage
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html  # Nginx serves files from here
      name: html-vol
  volumes:
  - name: html-vol
    persistentVolumeClaim:
      claimName: html-pvc               # References the PVC above
# When this pod is scheduled → OpenEBS creates the PV on that node
```

### Step 4 — Expose via NodePort Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-storage-svc
spec:
  type: NodePort
  selector:
    app: nginx-storage       # Labels don't apply here — wrong, pod has no selector
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30090
```

> If your pod has no app label, add one:
> ```yaml
> metadata:
>   name: nginx-storage
>   labels:
>     app: nginx-storage
> ```

### Step 5 — Write content and verify
```bash
# Write HTML into the mounted volume
kubectl exec -it nginx-storage -- /bin/sh -c \
  'echo "<h1>Hello from PersistentVolume</h1>" > /usr/share/nginx/html/index.html'

# Get node IP
kubectl get nodes -o wide

# Access the page
curl http://<NODE-IP>:30090
# Output: <h1>Hello from PersistentVolume</h1>
```

### What happens when you delete and recreate the pod
```bash
# Delete pod
kubectl delete pod nginx-storage

# Recreate pod (same PVC name)
kubectl apply -f nginx-storage-pod.yaml

# Exec and check file — still there!
kubectl exec -it nginx-storage -- cat /usr/share/nginx/html/index.html
# The data survived because it's in the PV, not in the container
```

### Verify PV and PVC
```bash
# PVC should be Bound
kubectl get pvc html-pvc

# PV is auto-created by OpenEBS
kubectl get pv

# Describe PV to see the actual directory on the node
kubectl describe pv <pv-name>
# Look for: Source.Path: /var/openebs/local/...
```
