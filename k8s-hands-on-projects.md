# Kubernetes Hands-On POC Projects — Real-World Scenarios

> **Environment**: Killercoda — 2-node kubeadm cluster (`controlplane` + `node01`), Calico CNI  
> **Goal**: Each project simulates a real production scenario. Follow steps in order. Add your own comments as you go.  
> **Images used**: All real Docker Hub images — copy-paste ready.

---

## Project Index

| # | Project | Core Skills Demonstrated |
|---|---------|--------------------------|
| 1 | [Multi-Tier Web App](#project-1) | Deployments, Services, ConfigMaps, DNS |
| 2 | [Zero-Downtime Update & Emergency Rollback](#project-2) | RollingUpdate, rollout history, rollback |
| 3 | [Auto-Scaling Under Traffic Load](#project-3) | HPA, Metrics Server, load generation |
| 4 | [Stateful Database with Persistent Storage](#project-4) | StatefulSet, PVC, data survival |
| 5 | [RBAC — Multi-Team Cluster Security](#project-5) | ServiceAccount, Role, RoleBinding, audit |
| 6 | [Network Policy — Zero-Trust Microservices](#project-6) | NetworkPolicy, deny-all, selective allow |
| 7 | [Node Drain & Pod Rescheduling](#project-7) | cordon, drain, taint, pod eviction |
| 8 | [Self-Healing App — Probes & Auto-Restart](#project-8) | livenessProbe, readinessProbe, events |
| 9 | [Persistent Storage — Data Survival Test](#project-9) | PV, PVC, hostPath, emptyDir comparison |
| 10 | [Ingress Controller — API Gateway Routing](#project-10) | Ingress, path-based routing, TLS |

---

<a name="project-1"></a>
## Project 1: Multi-Tier Web Application Deployment

### Real-World Scenario
A startup needs to deploy their web application to Kubernetes for the first time. The stack has a public-facing **frontend**, an internal **backend API**, and a **PostgreSQL database**. Only the frontend should be reachable from the internet. The backend and DB must be internal-only.

### Architecture
```
Internet
    ↓
NodePort :30080
    ↓
frontend-svc (ClusterIP)
    ↓                       [cluster-internal only]
frontend pods (nginx:1.28)
    ↓ DNS: backend-svc
backend pods (nginx:1.28)
    ↓ DNS: db-svc
postgres pod (postgres:15)
```

### What you prove
- Deployments, Services, ConfigMaps, Secrets
- Internal DNS-based service discovery (`backend-svc`, `db-svc`)
- NodePort for external access
- Namespace isolation

---

```bash
# ============================================================
# STEP 1: Create a dedicated namespace for this project
# ============================================================
kubectl create namespace webapp

# Verify it was created
kubectl get namespaces | grep webapp


# ============================================================
# STEP 2: Deploy the Database tier (PostgreSQL)
# ============================================================
# Best practice: never hardcode passwords — use a Secret
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: webapp
type: Opaque
stringData:
  POSTGRES_PASSWORD: "webapp123"
  POSTGRES_DB: "appdb"
  POSTGRES_USER: "appuser"
EOF

# Deploy PostgreSQL
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-db
  namespace: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres-db
      tier: database
  template:
    metadata:
      labels:
        app: postgres-db
        tier: database
    spec:
      containers:
      - name: postgres
        image: postgres:15
        envFrom:
        - secretRef:
            name: db-secret
        ports:
        - containerPort: 5432
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "appuser", "-d", "appdb"]
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: db-svc
  namespace: webapp
spec:
  selector:
    app: postgres-db
  ports:
  - port: 5432
    targetPort: 5432
  # No type: ClusterIP is needed — it's the default
  # This service is reachable ONLY inside the cluster as "db-svc"
EOF


# ============================================================
# STEP 3: Deploy the Backend tier
# ============================================================
# We simulate the backend API using nginx serving custom HTML
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-html
  namespace: webapp
data:
  index.html: |
    <h1>Backend API v1</h1>
    <p>Tier: backend</p>
    <p>DB endpoint: db-svc:5432/appdb</p>
    <p>Status: OK</p>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: api
  template:
    metadata:
      labels:
        app: backend
        tier: api
    spec:
      containers:
      - name: backend
        image: nginx:1.28
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: html
        configMap:
          name: backend-html
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: webapp
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
EOF


# ============================================================
# STEP 4: Deploy the Frontend tier
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-html
  namespace: webapp
data:
  index.html: |
    <h1>MyApp Frontend v1.0</h1>
    <p>Welcome to the public-facing frontend.</p>
    <p>Backend URL (internal): http://backend-svc/</p>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: web
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: frontend
        image: nginx:1.28
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: html
        configMap:
          name: frontend-html
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: webapp
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080         # Fixed port so you know where to hit it
EOF


# ============================================================
# STEP 5: Verify everything is running
# ============================================================
kubectl get all -n webapp

# Wait until all pods are Running and READY 2/2 or 1/1
kubectl wait --for=condition=Ready pods --all -n webapp --timeout=120s


# ============================================================
# STEP 6: Test external access (from Killercoda terminal)
# ============================================================
# Get the controlplane node IP
NODE_IP=$(kubectl get node controlplane -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
echo "Node IP: $NODE_IP"

# Hit the frontend
curl -s http://$NODE_IP:30080
# Expected: <h1>MyApp Frontend v1.0</h1>


# ============================================================
# STEP 7: Test internal DNS resolution (key concept!)
# ============================================================
# Launch a debug pod and test service-to-service communication
kubectl run debug -n webapp --image=busybox:1.28 --restart=Never -it --rm -- sh -c "
  echo '=== Frontend response ==='
  wget -qO- http://frontend-svc/ 2>/dev/null
  echo
  echo '=== Backend response ==='
  wget -qO- http://backend-svc/ 2>/dev/null
  echo
  echo '=== DB DNS resolution ==='
  nslookup db-svc
  echo
  echo '=== Full DNS name ==='
  nslookup db-svc.webapp.svc.cluster.local
"
# KEY INSIGHT: Services are reachable by short name within same namespace
# Cross-namespace: use db-svc.webapp.svc.cluster.local


# ============================================================
# STEP 8: Explore — show labels and selectors at work
# ============================================================
kubectl get pods -n webapp --show-labels
kubectl get pods -n webapp -l tier=database
kubectl get pods -n webapp -l tier=api
kubectl get pods -n webapp -l tier=web


# ============================================================
# CLEANUP
# ============================================================
kubectl delete namespace webapp
```

**Key Learnings**:
- ClusterIP services are reachable by DNS name only inside the cluster
- NodePort exposes a port on every node's IP
- Labels + selectors wire services to pods — the glue of Kubernetes networking
- Readiness probes prevent traffic going to pods that aren't ready yet

---

<a name="project-2"></a>
## Project 2: Zero-Downtime Rolling Update & Emergency Rollback

### Real-World Scenario
Your team just pushed a critical bug fix to production (v1 → v2 update). Halfway through the rollout, you discover the new image has a runtime error. You need to roll back to the previous version **immediately** without any downtime. This is the most common production emergency in Kubernetes.

### What you prove
- Rolling update strategy (maxSurge, maxUnavailable)
- Rollout monitoring with `kubectl rollout status`
- Rollout history and revision management
- Emergency rollback in under 30 seconds
- How Kubernetes prevents a fully-broken deploy from replacing everything

---

```bash
# ============================================================
# STEP 1: Deploy v1 of the application
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 4                    # 4 replicas for realistic demo
  revisionHistoryLimit: 5        # Keep last 5 versions for rollback
  selector:
    matchLabels:
      app: web-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1          # At most 1 pod unavailable during update
      maxSurge: 1                # At most 1 extra pod during update
  template:
    metadata:
      labels:
        app: web-app
        version: v1
    spec:
      containers:
      - name: web
        image: nginx:1.27        # Version 1
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30081
  type: NodePort
EOF

# Wait for v1 to be fully up
kubectl rollout status deployment/web-app
kubectl get pods -l app=web-app
# Note: all pods show image nginx:1.27


# ============================================================
# STEP 2: In a SECOND terminal — watch pods during the update
# ============================================================
# Open another terminal and run this BEFORE triggering the update:
# watch kubectl get pods -l app=web-app
# This shows pods being replaced one by one (never all at once)


# ============================================================
# STEP 3: Trigger v2 rolling update (good update)
# ============================================================
# Update to nginx:1.28 — this simulates deploying a new app version
kubectl set image deployment/web-app web=nginx:1.28

# Watch it roll out (in the main terminal)
kubectl rollout status deployment/web-app
# You'll see: "Waiting for deployment to finish: 1 out of 4 new replicas have been updated..."
# Eventually: "deployment "web-app" successfully rolled out"

# Verify all pods are now on the new image
kubectl get pods -l app=web-app -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
# All should show: nginx:1.28

# Check rollout history — v1 is preserved!
kubectl rollout history deployment/web-app
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>


# ============================================================
# STEP 4: Simulate a BAD deployment (broken image)
# ============================================================
# This is what happens when someone deploys the wrong tag to production
kubectl set image deployment/web-app web=nginx:THIS-TAG-DOESNT-EXIST

# Watch it get stuck (in second terminal or here)
kubectl rollout status deployment/web-app
# It will hang — the new pods can't pull the image

# Check what's happening
kubectl get pods -l app=web-app
# You'll see: some pods in ErrImagePull or ImagePullBackOff
# But OLD pods are still running! maxUnavailable=1 protects production

kubectl describe pods -l app=web-app | grep -A5 "Events:"
# Look for: Failed to pull image... "not found"

# The KEY insight: old pods are still serving traffic because
# Kubernetes won't kill them until the new ones are Ready


# ============================================================
# STEP 5: EMERGENCY ROLLBACK (simulate real incident response)
# ============================================================
# Time is critical — rollback immediately
kubectl rollout undo deployment/web-app

# Monitor the rollback
kubectl rollout status deployment/web-app

# Verify all pods are healthy and on nginx:1.28
kubectl get pods -l app=web-app
kubectl rollout history deployment/web-app
# Now you have revision 4 (rollback to nginx:1.28)

# Confirm the service is serving correctly
NODE_IP=$(kubectl get node controlplane -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://$NODE_IP:30081
# HTTP Status: 200


# ============================================================
# STEP 6: Rollback to a specific revision (bonus)
# ============================================================
kubectl rollout history deployment/web-app

# Go back to revision 1 (nginx:1.27)
kubectl rollout undo deployment/web-app --to-revision=1
kubectl rollout status deployment/web-app

# Verify image is nginx:1.27
kubectl get pods -l app=web-app -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'


# ============================================================
# CLEANUP
# ============================================================
kubectl delete deployment web-app
kubectl delete service web-app-svc
```

**Key Learnings**:
- `maxUnavailable: 1` is your safety net — old pods keep serving during a bad deploy
- `kubectl rollout undo` is your fastest path out of a production incident
- `revisionHistoryLimit` controls how many versions you can roll back to
- Always watch `kubectl rollout status` during deploys — don't just apply and walk away

---

<a name="project-3"></a>
## Project 3: Auto-Scaling Under Traffic Load (HPA)

### Real-World Scenario
Your web application is handling normal traffic fine with 2 pods. A marketing campaign goes live and traffic spikes 10x. You need Kubernetes to automatically scale out pods to handle the load, then scale back in when traffic drops to save money.

### What you prove
- Horizontal Pod Autoscaler (HPA)
- Metrics Server installation and configuration
- CPU-based auto-scaling
- Load generation and real-time scaling observation

---

```bash
# ============================================================
# STEP 1: Install Metrics Server (required for HPA)
# Killercoda does NOT have this pre-installed
# ============================================================
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for Killercoda (kubeadm cluster uses self-signed certs)
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Wait for it to be ready (takes ~30-60 seconds)
kubectl -n kube-system rollout status deployment metrics-server

# Test it works
kubectl top nodes
# Should show CPU and Memory usage for controlplane and node01


# ============================================================
# STEP 2: Deploy the target application
# CRITICAL: Container MUST have resource requests defined for HPA to work
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-web
spec:
  replicas: 1                    # Start with 1 — HPA will scale up
  selector:
    matchLabels:
      app: hpa-web
  template:
    metadata:
      labels:
        app: hpa-web
    spec:
      containers:
      - name: web
        image: nginx:1.28
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"          # HPA uses this as the 100% baseline
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: hpa-web-svc
spec:
  selector:
    app: hpa-web
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl rollout status deployment/hpa-web


# ============================================================
# STEP 3: Create the HPA
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-web
  minReplicas: 1                 # Never scale below 1
  maxReplicas: 6                 # Never scale above 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50   # Scale up if avg CPU > 50% of request (>50m per pod)
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60  # Wait 60s before scaling down (avoid flapping)
EOF

# Check HPA status (takes a minute for metrics to populate)
kubectl get hpa hpa-web
# TARGETS shows <current>/<target> — wait for it to show a real number, not <unknown>

kubectl describe hpa hpa-web


# ============================================================
# STEP 4: Generate load (open a separate terminal for this)
# ============================================================
# Run a load generator pod — it hammers the nginx service continuously
kubectl run load-gen \
  --image=busybox:1.28 \
  --restart=Never \
  -- sh -c "while true; do wget -q -O- http://hpa-web-svc; done"

# Now watch the HPA respond (run in MAIN terminal)
# Watch every 5 seconds
watch -n 5 kubectl get hpa hpa-web

# Also watch the pods scaling
watch -n 5 kubectl get pods -l app=hpa-web

# You should see replicas increase from 1 → 2 → 3... (may take 60-90s to respond)


# ============================================================
# STEP 5: Check detailed HPA metrics
# ============================================================
kubectl describe hpa hpa-web
# Look for:
#   Metrics: cpu resource utilization (percentage of request): XX% / 50%
#   Events:  SuccessfulRescale — Scaled up from 1 to 3

kubectl top pods -l app=hpa-web
# See actual CPU usage per pod


# ============================================================
# STEP 6: Stop the load and watch scale-down
# ============================================================
kubectl delete pod load-gen

# HPA won't scale down immediately (stabilizationWindowSeconds: 60)
# After ~60-90 seconds it will scale back to 1 replica
watch -n 10 kubectl get hpa hpa-web
# Watch replicas count drop back to 1


# ============================================================
# CLEANUP
# ============================================================
kubectl delete hpa hpa-web
kubectl delete deployment hpa-web
kubectl delete service hpa-web-svc
```

**Key Learnings**:
- HPA requires **Metrics Server** AND **resource requests** on containers — both are mandatory
- Scale-up is fast (15-30s), scale-down is slow by design (stabilizationWindow prevents thrashing)
- `<unknown>` in HPA TARGETS means Metrics Server isn't working or requests aren't set
- The `averageUtilization` target is % of the **request** value, not the limit

---

<a name="project-4"></a>
## Project 4: Stateful Database Cluster with Persistent Storage

### Real-World Scenario
You're deploying PostgreSQL for a production application. The database must maintain stable hostnames, and each replica must keep its own data even if a pod is killed or rescheduled. This demonstrates why StatefulSets exist and why regular Deployments cannot be used for databases.

### What you prove
- StatefulSet ordered pod creation (db-0, db-1, db-2)
- Stable network identity (headless service)
- PVC per pod — data survives pod deletion
- Why `volumeClaimTemplates` is critical for stateful apps

---

```bash
# ============================================================
# STEP 1: Create namespace and secret
# ============================================================
kubectl create namespace stateful-demo

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: pg-secret
  namespace: stateful-demo
type: Opaque
stringData:
  POSTGRES_PASSWORD: "pgpass123"
  POSTGRES_USER: "pgadmin"
  POSTGRES_DB: "mydb"
EOF


# ============================================================
# STEP 2: Create the Headless Service
# A headless service (clusterIP: None) gives each pod its own DNS entry
# db-0.pg-svc.stateful-demo.svc.cluster.local
# db-1.pg-svc.stateful-demo.svc.cluster.local
# This is what makes StatefulSet pods addressable individually
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: pg-svc
  namespace: stateful-demo
spec:
  clusterIP: None              # HEADLESS — no virtual IP, just DNS per pod
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
EOF


# ============================================================
# STEP 3: Deploy the StatefulSet
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  namespace: stateful-demo
spec:
  serviceName: "pg-svc"       # Must match the headless service name
  replicas: 2                  # 2 replicas: db-0 and db-1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        envFrom:
        - secretRef:
            name: pg-secret
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: pg-data
          mountPath: /var/lib/postgresql/data
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "pgadmin", "-d", "mydb"]
          initialDelaySeconds: 15
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: pg-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

# Watch pods come up — they start IN ORDER: db-0 first, then db-1
kubectl get pods -n stateful-demo -w
# You MUST see db-0 Running before db-1 starts — that's StatefulSet guarantee

# Check the PVCs auto-created (one per pod)
kubectl get pvc -n stateful-demo
# pg-data-db-0 and pg-data-db-1 — each pod gets its own storage


# ============================================================
# STEP 4: Verify stable DNS identity
# ============================================================
kubectl run -n stateful-demo dns-test --image=busybox:1.28 --restart=Never -it --rm -- sh -c "
  echo '=== Pod db-0 individual DNS ==='
  nslookup db-0.pg-svc.stateful-demo.svc.cluster.local
  echo '=== Pod db-1 individual DNS ==='
  nslookup db-1.pg-svc.stateful-demo.svc.cluster.local
  echo '=== Service DNS (returns all pod IPs) ==='
  nslookup pg-svc.stateful-demo.svc.cluster.local
"
# KEY INSIGHT: Each pod has its own DNS. This is how replicas know about each other.
# In real Postgres HA, db-0 is the primary and db-1 connects to db-0 by name.


# ============================================================
# STEP 5: Write data to db-0, then delete the pod
# This proves data persistence across pod restarts
# ============================================================
# Connect to db-0 and create a table with data
kubectl exec -n stateful-demo db-0 -- psql -U pgadmin -d mydb -c "
  CREATE TABLE IF NOT EXISTS survival_test (
    id SERIAL PRIMARY KEY,
    message TEXT,
    created_at TIMESTAMP DEFAULT NOW()
  );
  INSERT INTO survival_test (message) VALUES ('This data must survive pod deletion!');
  SELECT * FROM survival_test;
"

# Now DELETE the pod (simulates node failure, OOMKill, etc.)
kubectl delete pod db-0 -n stateful-demo

# Watch db-0 get recreated automatically (StatefulSet guarantees this)
kubectl get pods -n stateful-demo -w
# db-0 will go Terminating → Pending → ContainerCreating → Running


# ============================================================
# STEP 6: Verify data survived
# ============================================================
# Wait for db-0 to be ready
kubectl wait --for=condition=Ready pod/db-0 -n stateful-demo --timeout=120s

# Query the data — it must still be there
kubectl exec -n stateful-demo db-0 -- psql -U pgadmin -d mydb -c "SELECT * FROM survival_test;"
# Expected: the row with "This data must survive pod deletion!" is still there ✅

# Verify the SAME PVC is reattached (not a new empty one)
kubectl get pvc -n stateful-demo
# pg-data-db-0 should show: Bound — same PVC, same data


# ============================================================
# STEP 7: Demonstrate ORDERED scale-down
# ============================================================
kubectl scale statefulset db -n stateful-demo --replicas=1
kubectl get pods -n stateful-demo -w
# db-1 is deleted FIRST (reverse order) — StatefulSet always scales down from highest index


# ============================================================
# CLEANUP
# ============================================================
kubectl delete namespace stateful-demo
# Note: PVCs are deleted with the namespace, but in production:
# kubectl delete pvc --all -n stateful-demo would need explicit cleanup
```

**Key Learnings**:
- StatefulSet creates pods in order (0→1→2) and deletes in reverse (2→1→0)
- Each pod gets its OWN PVC via `volumeClaimTemplates` — not shared storage
- The headless service (`clusterIP: None`) gives each pod a stable DNS entry
- Data survives pod deletion because the PVC persists — the new pod reattaches the same volume

---

<a name="project-5"></a>
## Project 5: RBAC — Multi-Team Cluster Security

### Real-World Scenario
Your company has two teams: **dev-team** (can deploy apps to the `dev` namespace) and **ops-team** (has read access across the cluster for monitoring). A contractor (`contractor-alice`) should only be able to view pods in the `staging` namespace — nothing else. You need to enforce this with Kubernetes RBAC.

### What you prove
- ServiceAccount-based RBAC (practical for Killercoda)
- Role (namespaced) vs ClusterRole (cluster-wide)
- RoleBinding and ClusterRoleBinding
- `kubectl auth can-i` to verify permissions
- Least-privilege security model

---

```bash
# ============================================================
# STEP 1: Create namespaces
# ============================================================
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod


# ============================================================
# STEP 2: Create ServiceAccounts (represents teams/users)
# In a real cluster you'd use X.509 user certificates or OIDC
# ServiceAccounts work the same for permission testing
# ============================================================
kubectl create serviceaccount dev-deployer -n dev
kubectl create serviceaccount ops-monitor -n kube-system
kubectl create serviceaccount contractor-alice -n staging


# ============================================================
# STEP 3: Create Role for dev-team
# Can deploy and manage apps in the dev namespace ONLY
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-deployer-role
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# NOTE: No access to secrets or cluster-level resources
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-deployer-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: dev-deployer
  namespace: dev
roleRef:
  kind: Role
  name: dev-deployer-role
  apiGroup: rbac.authorization.k8s.io
EOF


# ============================================================
# STEP 4: Create ClusterRole for ops-team
# Read-only access across the ENTIRE cluster (all namespaces)
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ops-monitor-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods", "nodes", "services", "namespaces", "events"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "daemonsets", "statefulsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ops-monitor-binding
subjects:
- kind: ServiceAccount
  name: ops-monitor
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: ops-monitor-clusterrole
  apiGroup: rbac.authorization.k8s.io
EOF


# ============================================================
# STEP 5: Create Role for contractor-alice
# Read-only access to PODS only, in staging namespace ONLY
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: contractor-role
  namespace: staging
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
# No create, delete, update — read-only
# No access to secrets, configmaps, services — minimal access
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: contractor-binding
  namespace: staging
subjects:
- kind: ServiceAccount
  name: contractor-alice
  namespace: staging
roleRef:
  kind: Role
  name: contractor-role
  apiGroup: rbac.authorization.k8s.io
EOF


# ============================================================
# STEP 6: Verify permissions — the audit phase
# ============================================================

echo "=== dev-deployer permissions ==="
# Can deploy to dev namespace?
kubectl auth can-i create deployments --as=system:serviceaccount:dev:dev-deployer -n dev
# yes
kubectl auth can-i delete pods --as=system:serviceaccount:dev:dev-deployer -n dev
# yes

# Can access other namespaces?
kubectl auth can-i create deployments --as=system:serviceaccount:dev:dev-deployer -n staging
# no
kubectl auth can-i create deployments --as=system:serviceaccount:dev:dev-deployer -n prod
# no

# Can access secrets?
kubectl auth can-i get secrets --as=system:serviceaccount:dev:dev-deployer -n dev
# no — secrets not in the Role

echo ""
echo "=== ops-monitor permissions ==="
# Can view across all namespaces?
kubectl auth can-i list pods --as=system:serviceaccount:kube-system:ops-monitor -n dev
# yes
kubectl auth can-i list pods --as=system:serviceaccount:kube-system:ops-monitor -n prod
# yes
kubectl auth can-i list nodes --as=system:serviceaccount:kube-system:ops-monitor
# yes

# Can create or delete anything?
kubectl auth can-i delete pods --as=system:serviceaccount:kube-system:ops-monitor -n dev
# no — read-only

echo ""
echo "=== contractor-alice permissions ==="
# Can view pods in staging?
kubectl auth can-i list pods --as=system:serviceaccount:staging:contractor-alice -n staging
# yes

# Can view pods in dev?
kubectl auth can-i list pods --as=system:serviceaccount:staging:contractor-alice -n dev
# no

# Can view services in staging?
kubectl auth can-i list services --as=system:serviceaccount:staging:contractor-alice -n staging
# no — only pods are allowed

# Can delete pods in staging?
kubectl auth can-i delete pods --as=system:serviceaccount:staging:contractor-alice -n staging
# no — only get/list/watch


# ============================================================
# STEP 7: List ALL permissions for a subject
# ============================================================
kubectl auth can-i --list --as=system:serviceaccount:dev:dev-deployer -n dev
# Shows every action this ServiceAccount can do in dev namespace


# ============================================================
# STEP 8: Real-world test — try to deploy as dev-deployer
# ============================================================
# Create a token for dev-deployer
DEV_TOKEN=$(kubectl create token dev-deployer -n dev)

# Use the token to make an API call (demonstrates real auth)
kubectl --token=$DEV_TOKEN get pods -n dev
# Works ✅

kubectl --token=$DEV_TOKEN get pods -n staging
# Error: Forbidden ✅


# ============================================================
# CLEANUP
# ============================================================
kubectl delete namespace dev staging prod
kubectl delete clusterrole ops-monitor-clusterrole
kubectl delete clusterrolebinding ops-monitor-binding
kubectl delete serviceaccount ops-monitor -n kube-system
```

**Key Learnings**:
- **Role** is namespaced; **ClusterRole** is cluster-wide
- **RoleBinding** grants access in one namespace; **ClusterRoleBinding** grants across all
- Always use `kubectl auth can-i` to verify before and after applying RBAC
- Principle of least privilege: only grant exactly what's needed — no more
- ServiceAccounts in pods automatically get a token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`

---

<a name="project-6"></a>
## Project 6: Network Policy — Zero-Trust Microservices

### Real-World Scenario
You're running a microservices app: **frontend** (public), **backend-api** (internal), and **database** (restricted). By default, Kubernetes allows all pods to talk to each other — that's a security risk. A compromised frontend pod should not be able to directly reach the database. You'll implement a zero-trust network model.

### What you prove
- Default open network (all pods can reach all pods)
- Deny-all policy (lock down)
- Selective allow policies
- Testing connectivity with `wget`/`nc`

**Requires**: Calico or Cilium CNI (Killercoda has Calico ✅)

---

```bash
# ============================================================
# STEP 1: Create namespaces with labels (for namespaceSelector)
# ============================================================
kubectl create namespace netpol-demo

# Label the namespace — NetworkPolicy namespaceSelector uses labels
kubectl label namespace netpol-demo env=demo


# ============================================================
# STEP 2: Deploy the three tiers
# ============================================================
cat <<'EOF' | kubectl apply -f -
# Database pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: netpol-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
      tier: db
  template:
    metadata:
      labels:
        app: database
        tier: db
    spec:
      containers:
      - name: db
        image: nginx:1.28          # Simulating a DB listener
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: database-svc
  namespace: netpol-demo
spec:
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432               # nginx won't actually respond on 5432 but we test network layer
---
# Backend API pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: netpol-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend-api
      tier: api
  template:
    metadata:
      labels:
        app: backend-api
        tier: api
    spec:
      containers:
      - name: api
        image: nginx:1.28
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-api-svc
  namespace: netpol-demo
spec:
  selector:
    app: backend-api
  ports:
  - port: 80
    targetPort: 80
---
# Frontend pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: netpol-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
      tier: web
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: web
        image: nginx:1.28
        ports:
        - containerPort: 80
EOF

kubectl wait --for=condition=Ready pods --all -n netpol-demo --timeout=90s
kubectl get pods -n netpol-demo -o wide


# ============================================================
# STEP 3: Prove default behavior — everything can reach everything
# ============================================================
# Get the backend and database pod names
BACKEND_POD=$(kubectl get pod -n netpol-demo -l app=backend-api -o jsonpath='{.items[0].metadata.name}')
FRONTEND_POD=$(kubectl get pod -n netpol-demo -l app=frontend -o jsonpath='{.items[0].metadata.name}')

echo "=== BEFORE NetworkPolicy — frontend reaching database directly ==="
# Frontend → database: THIS SHOULD FAIL in a secure setup but currently works!
kubectl exec -n netpol-demo $FRONTEND_POD -- wget -qO- --timeout=3 http://database-svc:5432 2>&1 | head -3
# You'll get a TCP connection (nginx responds) — security gap!

echo "=== Frontend reaching backend ==="
kubectl exec -n netpol-demo $FRONTEND_POD -- wget -qO- --timeout=3 http://backend-api-svc 2>&1 | head -3
# Also works


# ============================================================
# STEP 4: Apply DENY-ALL policy to the database
# Lock down the database so NO pod can reach it unless explicitly allowed
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-deny-all
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      tier: db              # This policy applies to pods with label tier: db
  policyTypes:
  - Ingress                 # Block ALL incoming connections to DB
  - Egress                  # Block ALL outgoing connections from DB
  ingress: []               # Empty = deny all ingress
  egress: []                # Empty = deny all egress
EOF

echo ""
echo "=== AFTER deny-all — frontend trying to reach database ==="
kubectl exec -n netpol-demo $FRONTEND_POD -- wget -qO- --timeout=5 http://database-svc:5432 2>&1 | head -3
# Expected: wget: can't connect — connection refused/timeout ✅


# ============================================================
# STEP 5: Allow ONLY backend-api to reach the database
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-backend
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      tier: db              # This policy adds to the db pod's rules
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api         # ONLY pods labeled tier: api can connect
    ports:
    - protocol: TCP
      port: 5432            # ONLY on port 5432
EOF

echo ""
echo "=== backend-api reaching database (should work now) ==="
kubectl exec -n netpol-demo $BACKEND_POD -- wget -qO- --timeout=5 http://database-svc:5432 2>&1 | head -3
# Works — backend-api has label tier: api ✅

echo "=== frontend still CANNOT reach database ==="
kubectl exec -n netpol-demo $FRONTEND_POD -- wget -qO- --timeout=5 http://database-svc:5432 2>&1 | head -3
# Still blocked — frontend has label tier: web, not tier: api ✅


# ============================================================
# STEP 6: Apply DENY-ALL to backend, then allow only frontend
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-deny-all
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  ingress: []
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-frontend
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: web         # Only pods labeled tier: web (frontend)
    ports:
    - protocol: TCP
      port: 80
EOF

echo ""
echo "=== Final test — all policies in place ==="
echo "Frontend → Backend (should WORK):"
kubectl exec -n netpol-demo $FRONTEND_POD -- wget -qO- --timeout=5 http://backend-api-svc 2>&1 | head -3

echo "Frontend → Database (should FAIL):"
kubectl exec -n netpol-demo $FRONTEND_POD -- wget -qO- --timeout=5 http://database-svc:5432 2>&1 | head -3

echo "Backend → Database (should WORK):"
kubectl exec -n netpol-demo $BACKEND_POD -- wget -qO- --timeout=5 http://database-svc:5432 2>&1 | head -3

# Show all policies
kubectl get networkpolicies -n netpol-demo


# ============================================================
# CLEANUP
# ============================================================
kubectl delete namespace netpol-demo
```

**Key Learnings**:
- Without NetworkPolicy, ALL pods can reach ALL pods — dangerous in shared clusters
- A deny-all policy + explicit allow is the zero-trust pattern
- NetworkPolicy is additive: multiple policies on the same pod combine with OR logic
- Labels are the selector mechanism — label your pods thoughtfully with security in mind
- NetworkPolicy only works if your CNI supports it (Calico ✅, Flannel ❌)

---

<a name="project-7"></a>
## Project 7: Node Drain & Pod Rescheduling (Maintenance Operations)

### Real-World Scenario
You need to perform kernel upgrades on `node01`. You must safely evacuate all pods from the node first, then after maintenance, bring it back. One of your pods must NOT be evicted (it's a DaemonSet). This is a core production operations task done weekly in real clusters.

### What you prove
- `kubectl cordon` and `kubectl uncordon`
- `kubectl drain` (graceful pod eviction)
- Node taints and their effects
- DaemonSet pods are NOT evicted by drain
- Scheduling behavior before/after maintenance

---

```bash
# ============================================================
# STEP 1: Deploy a workload that spreads across both nodes
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: spread-app
  template:
    metadata:
      labels:
        app: spread-app
    spec:
      containers:
      - name: app
        image: nginx:1.28
      # No node affinity — pods schedule anywhere
EOF

# Also create a DaemonSet (one pod per node — should survive drain)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      containers:
      - name: monitor
        image: busybox:1.28
        command: ["sh", "-c", "while true; do echo Monitoring node; sleep 30; done"]
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
EOF

kubectl wait --for=condition=Ready pods --all --timeout=90s

# See pods distributed across nodes
kubectl get pods -o wide
# spread-app pods should be on BOTH controlplane and node01
# node-monitor pods: one on each node


# ============================================================
# STEP 2: CORDON node01 (prevent new scheduling, keep existing pods)
# ============================================================
kubectl cordon node01

# Check node status
kubectl get nodes
# node01 should show: Ready,SchedulingDisabled

# Try to create a new pod — it should land on controlplane only
kubectl run test-sched --image=busybox:1.28 --restart=Never -- sleep 60
kubectl get pod test-sched -o wide
# Should be on controlplane, NOT node01

# Existing pods on node01 are still running (cordon doesn't evict)
kubectl get pods -o wide | grep node01
# spread-app pods on node01 still Running


# ============================================================
# STEP 3: DRAIN node01 (evict all evictable pods)
# ============================================================
kubectl drain node01 \
  --ignore-daemonsets \      # Don't evict DaemonSet pods (they can't run elsewhere)
  --delete-emptydir-data \   # Allow deletion of pods using emptyDir
  --force \                  # Force deletion of pods not managed by a controller
  --grace-period=30          # Give pods 30 seconds to gracefully shutdown

# Watch pods move to controlplane
kubectl get pods -o wide
# All spread-app pods should now be on controlplane
# node-monitor pod on node01 is STILL THERE (DaemonSet, exempt from drain)

# Check node status
kubectl get nodes
# node01: Ready,SchedulingDisabled  — cordoned + drained


# ============================================================
# STEP 4: Simulate maintenance (wait a moment)
# ============================================================
echo "Performing node maintenance..."
echo "In real world: this is where you'd run:"
echo "  apt-get upgrade -y && reboot"
sleep 5
echo "Maintenance complete. Bringing node back online."


# ============================================================
# STEP 5: UNCORDON node01 (return to service)
# ============================================================
kubectl uncordon node01

# Check nodes
kubectl get nodes
# node01: Ready — back to normal

# Scale up to see new pods land on node01 again
kubectl scale deployment spread-app --replicas=6
kubectl get pods -o wide
# New pods should spread to node01


# ============================================================
# STEP 6: Explore taints (what drain actually does)
# ============================================================
# Drain adds a taint to prevent scheduling. Let's see what taints look like:
kubectl describe node node01 | grep -A5 Taints

# Manually add a custom taint
kubectl taint node node01 maintenance=true:NoSchedule
kubectl get pods -o wide     # New pods avoid node01

# Remove the taint
kubectl taint node node01 maintenance=true:NoSchedule-
kubectl get pods -o wide     # Scheduling resumes


# ============================================================
# CLEANUP
# ============================================================
kubectl delete deployment spread-app
kubectl delete daemonset node-monitor
kubectl delete pod test-sched
```

**Key Learnings**:
- `cordon` = stop NEW pods from scheduling on the node (existing pods stay)
- `drain` = cordon + evict all evictable pods gracefully
- DaemonSet pods are automatically skipped by drain (`--ignore-daemonsets`)
- After maintenance, `uncordon` re-enables the node for scheduling
- Taints are the mechanism behind cordon — `node.kubernetes.io/unschedulable:NoSchedule` is added

---

<a name="project-8"></a>
## Project 8: Self-Healing App — Probes & Automatic Recovery

### Real-World Scenario
Your application has a bug where it stops serving HTTP requests after 60 seconds (a common memory leak pattern). You need Kubernetes to automatically detect this and restart the container. Also, you need a readiness probe to ensure traffic is NEVER sent to a pod that's starting up or in a bad state.

### What you prove
- Liveness probe detecting failure → container restart
- Readiness probe controlling traffic routing
- Startup probe for slow-starting applications
- Reading pod events to understand restarts
- CrashLoopBackOff and how to diagnose it

---

```bash
# ============================================================
# STEP 1: Deploy an app that "breaks" after 30 seconds
# We use a busybox trick: create a file, sleep, then delete it.
# The liveness probe checks for this file — when it's gone, pod restarts.
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: self-healing-demo
spec:
  containers:
  - name: app
    image: busybox:1.28
    command:
    - sh
    - -c
    - |
      # Simulate app startup
      echo "App starting..."
      touch /tmp/healthy
      echo "App started. healthy file created."
      # "Run" for 30 seconds then simulate a crash/hang
      sleep 30
      echo "Simulating app failure — removing health file"
      rm /tmp/healthy
      # Stay alive so we can observe, but health check fails
      sleep 600
    livenessProbe:
      exec:
        command:
        - test
        - -f
        - /tmp/healthy
      initialDelaySeconds: 5    # Wait 5s after start before first check
      periodSeconds: 5          # Check every 5 seconds
      failureThreshold: 3       # Fail 3 times (15s) before restarting
    readinessProbe:
      exec:
        command:
        - test
        - -f
        - /tmp/healthy
      initialDelaySeconds: 3
      periodSeconds: 5
EOF


# ============================================================
# STEP 2: Watch it in real-time
# ============================================================
# In a SECOND terminal, watch the pod status
# watch kubectl get pod self-healing-demo

# In MAIN terminal, watch events
kubectl get pod self-healing-demo -w
# You'll see: Running → READY becomes 0/1 → then container restarts
# After restart: Running → 1/1 → then fails again → repeats

# Timeline:
# 0-5s:   startup
# 5s:     liveness probe starts (file exists → pass)
# 30s:    file deleted (liveness probe starts failing)
# 45s:    3rd failure → Kubernetes kills container
# 46s:    container restarts → file recreated → healthy again


# ============================================================
# STEP 3: Inspect what happened
# ============================================================
kubectl describe pod self-healing-demo
# Look for EVENTS section:
#   Warning  Unhealthy  xx  Liveness probe failed: command...exited with 1
#   Normal   Killing    xx  Container app failed liveness probe, will be restarted
#   Normal   Pulled     xx  Container image already present on machine

# Check restart count
kubectl get pod self-healing-demo
# RESTARTS column will be increasing: 0 → 1 → 2 → 3...

# Read logs from current container
kubectl logs self-healing-demo

# Read logs from the PREVIOUS (crashed) container
kubectl logs self-healing-demo --previous
# KEY: This is how you diagnose what caused the last crash


# ============================================================
# STEP 4: Deploy an app with a readiness probe
# Shows how Kubernetes stops sending traffic to a bad pod
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ready-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ready-demo
  template:
    metadata:
      labels:
        app: ready-demo
    spec:
      containers:
      - name: web
        image: nginx:1.28
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 2    # Remove from endpoints after 2 failures (10s)
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
          failureThreshold: 3    # Restart after 3 failures (30s)
---
apiVersion: v1
kind: Service
metadata:
  name: ready-demo-svc
spec:
  selector:
    app: ready-demo
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl rollout status deployment/ready-demo


# ============================================================
# STEP 5: Simulate readiness failure on one pod
# ============================================================
# Get a pod name
POD=$(kubectl get pod -l app=ready-demo -o jsonpath='{.items[0].metadata.name}')

# Break nginx by deleting the index.html (causes 403 on readiness check)
kubectl exec $POD -- rm /usr/share/nginx/html/index.html

# Watch the pod become NOT ready
kubectl get pods -l app=ready-demo -w
# The pod with missing index.html will show 0/1 READY
# Other 2 pods remain 1/1 READY — they still get traffic

# Check endpoints (traffic routing)
kubectl get endpoints ready-demo-svc
# The broken pod's IP should be REMOVED from endpoints
# Only 2 IPs listed (the 2 healthy pods)

# Liveness probe will restart the container after 30s
# After restart: index.html is back, readiness probe passes again
# Wait for it to recover
kubectl get pods -l app=ready-demo -w


# ============================================================
# STEP 6: Trigger CrashLoopBackOff deliberately
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crashloop-demo
spec:
  containers:
  - name: app
    image: busybox:1.28
    command: ["sh", "-c", "echo Starting && exit 1"]   # Always exits with error
  restartPolicy: Always
EOF

# Watch CrashLoopBackOff
kubectl get pod crashloop-demo -w
# 0: Running → Completed/Error → CrashLoopBackOff (5s delay)
# 1: Running → Error → CrashLoopBackOff (10s delay)
# 2: Running → Error → CrashLoopBackOff (20s delay)  ← exponential backoff

kubectl describe pod crashloop-demo | grep -A20 Events:
# Shows the repeated restart attempts with increasing backoff times


# ============================================================
# CLEANUP
# ============================================================
kubectl delete pod self-healing-demo crashloop-demo
kubectl delete deployment ready-demo
kubectl delete service ready-demo-svc
```

**Key Learnings**:
- **Liveness probe** = "is the container alive?" → fail → restart container
- **Readiness probe** = "is the container ready for traffic?" → fail → remove from Service endpoints (no traffic, no restart)
- `kubectl logs --previous` is how you read the crash log of a restarted container
- CrashLoopBackOff has exponential backoff: Kubernetes waits longer between each restart attempt
- Startup probes protect slow-starting apps from being killed by liveness probes during startup

---

<a name="project-9"></a>
## Project 9: Persistent Storage — Data Survival Test

### Real-World Scenario
You're deploying a web server that serves custom content. The content files are stored on a persistent volume. When pods crash, restart, or are rescheduled, the content must survive. You'll also demonstrate what happens with `emptyDir` to understand WHY persistent storage is necessary.

### What you prove
- PersistentVolume (PV) manual provisioning
- PersistentVolumeClaim (PVC) binding
- Data survival across pod deletion
- `emptyDir` data loss (contrast with PV)
- StorageClass and dynamic provisioning concept

---

```bash
# ============================================================
# STEP 1: Create a PersistentVolume (admin-provisioned storage)
# hostPath = uses a directory on the node (works on Killercoda)
# In production: this would be AWS EBS, GCP PD, NFS, etc.
# ============================================================
# First, create the directory on the node
ssh node01 "mkdir -p /mnt/pv-data && echo 'PV directory created'"
# If SSH not available in your Killercoda lab:
kubectl debug node/node01 -it --image=busybox:1.28 -- sh -c "mkdir -p /mnt/pv-data && echo created"

# Create the PersistentVolume
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: demo-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce               # One pod on one node can read/write
  persistentVolumeReclaimPolicy: Retain   # Keep data even if PVC is deleted
  storageClassName: ""          # Empty = manual (no StorageClass)
  hostPath:
    path: /mnt/pv-data          # Directory on node01
    type: DirectoryOrCreate     # Create if doesn't exist
  nodeAffinity:                 # hostPath PVs MUST be pinned to a node
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node01
EOF

kubectl get pv
# STATUS: Available — ready to be claimed


# ============================================================
# STEP 2: Create a PVC (application claiming the storage)
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: ""          # Must match PV's storageClassName
  volumeName: demo-pv           # Explicitly bind to our specific PV
EOF

kubectl get pvc
# STATUS: Bound — PVC is matched to the PV
# PV STATUS becomes: Bound

kubectl get pv,pvc
# See both are Bound to each other


# ============================================================
# STEP 3: Deploy nginx using the PVC
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pv-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pv-nginx
  template:
    metadata:
      labels:
        app: pv-nginx
    spec:
      nodeSelector:
        kubernetes.io/hostname: node01  # Must run on node01 (where hostPath is)
      containers:
      - name: web
        image: nginx:1.28
        ports:
        - containerPort: 80
        volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: web-data
        persistentVolumeClaim:
          claimName: demo-pvc         # Use the PVC we created
---
apiVersion: v1
kind: Service
metadata:
  name: pv-nginx-svc
spec:
  type: NodePort
  selector:
    app: pv-nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30090
EOF

kubectl wait --for=condition=Ready pod -l app=pv-nginx --timeout=60s


# ============================================================
# STEP 4: Write custom content to the persistent volume
# ============================================================
POD=$(kubectl get pod -l app=pv-nginx -o jsonpath='{.items[0].metadata.name}')

# Write content to the volume (this goes to /mnt/pv-data on node01)
kubectl exec $POD -- sh -c "echo '<h1>Persistent content created at: '$(date)'</h1>' > /usr/share/nginx/html/index.html"

# Read it back through the service
NODE_IP=$(kubectl get node controlplane -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
curl -s http://$NODE_IP:30090
# Shows: <h1>Persistent content created at: ...</h1>


# ============================================================
# STEP 5: THE KEY TEST — delete the pod and verify data survives
# ============================================================
kubectl delete pod $POD

# Watch the deployment recreate it automatically
kubectl get pods -l app=pv-nginx -w
# New pod starts, mounts the SAME PVC, reads from SAME /mnt/pv-data

# Read the content again — it must still be there
sleep 10
curl -s http://$NODE_IP:30090
# Expected: <h1>Persistent content created at: [original timestamp]</h1> ✅
# The timestamp from BEFORE the delete is still there — data survived!


# ============================================================
# STEP 6: Contrast — emptyDir loses data on pod deletion
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: web
    image: nginx:1.28
    volumeMounts:
    - name: ephemeral
      mountPath: /usr/share/nginx/html
  volumes:
  - name: ephemeral
    emptyDir: {}              # Created fresh when pod starts, DELETED when pod ends
EOF

kubectl wait --for=condition=Ready pod/emptydir-demo --timeout=60s

# Write content
kubectl exec emptydir-demo -- sh -c "echo '<h1>emptyDir content</h1>' > /usr/share/nginx/html/index.html"
kubectl exec emptydir-demo -- cat /usr/share/nginx/html/index.html
# Shows the content

# Delete and recreate the pod
kubectl delete pod emptydir-demo
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: web
    image: nginx:1.28
    volumeMounts:
    - name: ephemeral
      mountPath: /usr/share/nginx/html
  volumes:
  - name: ephemeral
    emptyDir: {}
EOF
kubectl wait --for=condition=Ready pod/emptydir-demo --timeout=60s

# Check content after recreation
kubectl exec emptydir-demo -- cat /usr/share/nginx/html/index.html
# Error: /usr/share/nginx/html/index.html: No such file or directory ❌
# Data is GONE — emptyDir is ephemeral (tied to pod lifecycle)


# ============================================================
# STEP 7: Storage summary comparison
# ============================================================
echo "
Volume Type | Survives pod restart? | Survives pod deletion? | Shared?
------------|----------------------|----------------------|--------
emptyDir    | YES (same pod)       | NO (deleted with pod) | No
hostPath    | YES (same node)      | YES (on same node)    | No
PV hostPath | YES                  | YES (PVC persists)    | No
PV NFS      | YES                  | YES                   | YES (RWX)
"


# ============================================================
# CLEANUP
# ============================================================
kubectl delete pod emptydir-demo
kubectl delete deployment pv-nginx
kubectl delete service pv-nginx-svc
kubectl delete pvc demo-pvc
kubectl delete pv demo-pv
# Data in /mnt/pv-data on node01 still exists (Retain policy)
# In production: manual cleanup of the data directory
```

**Key Learnings**:
- `emptyDir` = pod-lifetime storage — data is gone when pod is deleted
- `hostPath` PV = node-pinned — pod MUST run on the same node to access data
- PVC-to-PV binding is the abstraction layer — your app only knows about the PVC
- `Retain` reclaim policy keeps data after PVC deletion — requires manual cleanup
- In production, use StorageClasses for dynamic provisioning (no manual PV creation)

---

<a name="project-10"></a>
## Project 10: Ingress Controller — API Gateway Routing

### Real-World Scenario
You're running multiple microservices (v1 and v2 of an API, plus a documentation site). You want all of them behind a single external IP on port 80, with routing based on the URL path. This is how every production microservices platform works — a single entry point routes to multiple internal services.

### What you prove
- Nginx Ingress Controller installation
- Path-based routing (`/api/v1` → service-v1, `/api/v2` → service-v2)
- Host-based routing (bonus)
- Ingress rules and annotations
- How Ingress replaces multiple NodePort services

---

```bash
# ============================================================
# STEP 1: Install the Nginx Ingress Controller
# On Killercoda (bare-metal): use the baremetal deploy manifest
# ============================================================
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml

# Wait for the ingress controller pod to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s

# Check what port the ingress controller is listening on
kubectl get svc -n ingress-nginx
# You'll see a NodePort service — note the port (e.g., 80:3XXXX/TCP)
INGRESS_HTTP_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
echo "Ingress NodePort: $INGRESS_HTTP_PORT"

NODE_IP=$(kubectl get node controlplane -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')


# ============================================================
# STEP 2: Deploy two API services
# ============================================================
cat <<'EOF' | kubectl apply -f -
# API v1
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-v1-html
data:
  index.html: |
    <h1>API v1 Response</h1>
    <p>You reached: /api/v1/</p>
    <p>Service: api-v1</p>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-v1
  template:
    metadata:
      labels:
        app: api-v1
    spec:
      containers:
      - name: web
        image: nginx:1.27
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: api-v1-html
---
apiVersion: v1
kind: Service
metadata:
  name: api-v1-svc
spec:
  selector:
    app: api-v1
  ports:
  - port: 80
    targetPort: 80
---
# API v2
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-v2-html
data:
  index.html: |
    <h1>API v2 Response</h1>
    <p>You reached: /api/v2/</p>
    <p>Service: api-v2 (latest)</p>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-v2
  template:
    metadata:
      labels:
        app: api-v2
    spec:
      containers:
      - name: web
        image: nginx:1.28
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: api-v2-html
---
apiVersion: v1
kind: Service
metadata:
  name: api-v2-svc
spec:
  selector:
    app: api-v2
  ports:
  - port: 80
    targetPort: 80
---
# Docs site
apiVersion: v1
kind: ConfigMap
metadata:
  name: docs-html
data:
  index.html: |
    <h1>API Documentation</h1>
    <p>You reached: /docs/</p>
    <p>Service: docs-site</p>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docs-site
spec:
  replicas: 1
  selector:
    matchLabels:
      app: docs-site
  template:
    metadata:
      labels:
        app: docs-site
    spec:
      containers:
      - name: web
        image: nginx:1.28
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: docs-html
---
apiVersion: v1
kind: Service
metadata:
  name: docs-svc
spec:
  selector:
    app: docs-site
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl rollout status deployment/api-v1 api-v2 docs-site


# ============================================================
# STEP 3: Create the Ingress resource (path-based routing)
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /   # Rewrite path to / before forwarding
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-svc
            port:
              number: 80
      - path: /api/v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-svc
            port:
              number: 80
      - path: /docs
        pathType: Prefix
        backend:
          service:
            name: docs-svc
            port:
              number: 80
EOF

# Check the Ingress
kubectl get ingress api-gateway
kubectl describe ingress api-gateway


# ============================================================
# STEP 4: Test path-based routing
# ============================================================
echo "=== Testing /api/v1 routing ==="
curl -s http://$NODE_IP:$INGRESS_HTTP_PORT/api/v1
# Expected: <h1>API v1 Response</h1>

echo ""
echo "=== Testing /api/v2 routing ==="
curl -s http://$NODE_IP:$INGRESS_HTTP_PORT/api/v2
# Expected: <h1>API v2 Response</h1>

echo ""
echo "=== Testing /docs routing ==="
curl -s http://$NODE_IP:$INGRESS_HTTP_PORT/docs
# Expected: <h1>API Documentation</h1>

echo ""
echo "=== Testing undefined path (404) ==="
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://$NODE_IP:$INGRESS_HTTP_PORT/undefined
# Expected: HTTP 404


# ============================================================
# STEP 5: Bonus — host-based routing
# Route traffic based on the Host header (virtual hosting)
# ============================================================
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-routing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: api.myapp.local         # Requests with Host: api.myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-v2-svc
            port:
              number: 80
  - host: docs.myapp.local        # Requests with Host: docs.myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: docs-svc
            port:
              number: 80
EOF

# Test with Host header (simulates DNS resolution)
curl -s -H "Host: api.myapp.local" http://$NODE_IP:$INGRESS_HTTP_PORT/
# Expected: <h1>API v2 Response</h1>

curl -s -H "Host: docs.myapp.local" http://$NODE_IP:$INGRESS_HTTP_PORT/
# Expected: <h1>API Documentation</h1>


# ============================================================
# STEP 6: Show Ingress vs NodePort — the advantage
# ============================================================
echo "
WITHOUT Ingress (NodePort approach):
  api-v1-svc → NodePort :30091
  api-v2-svc → NodePort :30092
  docs-svc   → NodePort :30093
  = 3 different ports to manage, no unified routing

WITH Ingress:
  All traffic → NodePort :$INGRESS_HTTP_PORT
  /api/v1  → api-v1-svc
  /api/v2  → api-v2-svc
  /docs    → docs-svc
  = One entry point, clean URL routing
"

kubectl get ingress


# ============================================================
# CLEANUP
# ============================================================
kubectl delete ingress api-gateway host-based-routing
kubectl delete deployment api-v1 api-v2 docs-site
kubectl delete service api-v1-svc api-v2-svc docs-svc
kubectl delete configmap api-v1-html api-v2-html docs-html
# Keep ingress controller for future use, or delete:
# kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml
```

**Key Learnings**:
- Ingress is NOT a Service type — it's a routing layer that sits in front of Services
- The Ingress resource is just config — you need an **Ingress Controller** (nginx, traefik) to process it
- `rewrite-target: /` strips the path prefix before forwarding to the backend
- Path-based routing: one IP, multiple services based on URL path
- Host-based routing: one IP, multiple virtual domains based on HTTP Host header
- In production: combine Ingress with cert-manager for automatic TLS certificates

---

## Quick Reference — All Projects

| Project | Namespace | Key Command to Verify | Cleanup |
|---------|-----------|----------------------|---------|
| 1 - Multi-tier | `webapp` | `curl http://NODE:30080` | `kubectl delete ns webapp` |
| 2 - Rollback | `default` | `kubectl rollout history deployment/web-app` | `kubectl delete deploy,svc web-app web-app-svc` |
| 3 - HPA | `default` | `kubectl get hpa hpa-web` | `kubectl delete hpa,deploy,svc hpa-web hpa-web-svc` |
| 4 - StatefulSet | `stateful-demo` | `kubectl get pvc -n stateful-demo` | `kubectl delete ns stateful-demo` |
| 5 - RBAC | `dev/staging/prod` | `kubectl auth can-i --list --as=...` | `kubectl delete ns dev staging prod` |
| 6 - NetworkPolicy | `netpol-demo` | `kubectl get networkpolicies -n netpol-demo` | `kubectl delete ns netpol-demo` |
| 7 - Node Drain | `default` | `kubectl get nodes` | `kubectl delete deploy daemonset` |
| 8 - Probes | `default` | `kubectl describe pod self-healing-demo` | `kubectl delete pod,deploy,svc` |
| 9 - Storage | `default` | `kubectl get pv,pvc` | `kubectl delete deploy,svc,pvc,pv` |
| 10 - Ingress | `default` | `kubectl get ingress` | `kubectl delete ingress,deploy,svc` |

## Skills Coverage Map

```
CKA Domain                    Covered By
─────────────────────────────────────────────────────────────────
Cluster Architecture          Project 7 (node ops), Project 8 (probes)
Workloads & Scheduling        Project 1, 2, 3, 4, 7
Services & Networking         Project 1, 6, 10
Storage                       Project 4, 9
Security                      Project 5, 6
Troubleshooting               Project 2 (rollback), Project 8 (CrashLoopBackOff)
```
