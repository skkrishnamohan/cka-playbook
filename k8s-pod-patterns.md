# Kubernetes Pod Patterns — Session Notes

---

## Multi-container Pod overview

- A Pod can have multiple containers that share the same network namespace and storage.
- All containers in a pod share the same IP address, localhost, and can mount the same volumes.
- Common patterns: **Init Container**, **Sidecar**, **Ambassador**, **Adapter**.

---

## Init Containers

- Run BEFORE the main application containers start.
- Must complete successfully before main containers begin.
- Run sequentially (one after another) if there are multiple init containers.
- Use cases: setup, config download, wait for dependencies, database migrations.

### Key rules
| Rule | Description |
|------|-------------|
| Run first | Always run before app containers |
| Must succeed | If init container fails, pod restarts |
| Run once | Don't run again after success (unless pod restarts) |
| Sequential | Multiple init containers run one at a time in order |
| No probes | Don't support readiness/liveness probes |

---

## Session YAML — full-demo pod (init + app + sidecar)

This pod from the session demonstrates ALL three patterns in one:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: full-demo
spec:
  initContainers:
  - name: init-setup
    image: busybox
    command: ["sh", "-c", "echo 'Init started'; sleep 2; echo 'Init done' > /work/data.txt"]
    volumeMounts:
    - name: workdir
      mountPath: /work
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /work/data.txt && sleep 3600"]
    volumeMounts:
    - name: workdir
      mountPath: /work
  - name: sidecar
    image: busybox
    command: ["sh", "-c", "while true; do echo 'Sidecar is watching...' && sleep 30; done"]
  volumes:
  - name: workdir
    emptyDir: {}
```

### What happens step by step:
```
1. Pod starts
2. init-setup runs:
   - Prints "Init started"
   - Sleeps 2 seconds
   - Writes "Init done" to /work/data.txt
   - Exits with success (exit code 0)
3. Only AFTER init completes, main containers start:
   - app: reads /work/data.txt (sees "Init done") → sleeps
   - sidecar: prints "Sidecar is watching..." every 30s
4. All containers run together until pod is deleted
```

### Session commands to observe
```bash
# Apply the pod
kubectl apply -f full-demo.yaml

# Watch pod status — you'll see "Init:0/1" then "Running"
kubectl get pod full-demo -w

# Check init container logs
kubectl logs full-demo -c init-setup

# Check app container logs
kubectl logs full-demo -c app
# Output: "Init done"

# Check sidecar logs
kubectl logs full-demo -c sidecar
# Output: "Sidecar is watching..."

# Exec into app container
kubectl exec -it full-demo -c app -- /bin/sh
cat /work/data.txt    # "Init done"
```

---

## Container States — what each phase means

Every container in a Pod goes through states. You can see them with `kubectl describe pod <pod-name>`.

| State | Meaning | Common Cause |
|-------|---------|-------------|
| `ContainerCreating` | Image is being pulled, volumes being mounted | Normal startup, or slow image pull |
| `Running` | Container is live and the process is running | Healthy state |
| `Waiting` | Container is not yet running — held up by something | Pending image pull, CrashLoopBackOff |
| `Terminated` | Container finished (either success or failure) | Completed Job, or app exited/crashed |

### CrashLoopBackOff — the most common error state

`CrashLoopBackOff` means the container keeps crashing and Kubernetes keeps restarting it with an increasing back-off delay (10s, 20s, 40s, 80s…).

Common causes:
- Application bug (panics, unhandled exception)
- Missing environment variable or config
- Wrong command / entrypoint
- Out of memory (OOMKilled)
- Image starts but fails immediately (wrong binary)

How to diagnose:
```bash
# See the state and restart count
kubectl get pods

# See exact error reason
kubectl describe pod <pod-name>
# Look for: "Last State", "Reason: OOMKilled / Error / Completed"

# Get logs from the CURRENT (crashing) container
kubectl logs <pod-name>

# Get logs from the PREVIOUS crash (most useful — shows why it crashed)
kubectl logs <pod-name> --previous

# Get logs for a specific container in a multi-container pod
kubectl logs <pod-name> -c <container-name>

# Copy files out of a running container for inspection
kubectl cp <pod-name>:/var/log/app.log ./local-app.log
```

---

## Pod Restart Policies — what happens when a container exits

`restartPolicy` is set at the **Pod level** (not container level) and controls what Kubernetes does when any container in the pod exits.

```yaml
spec:
  restartPolicy: Always    # Always | OnFailure | Never
```

| Policy | Behavior | Use case |
|--------|----------|---------|
| `Always` | Restart on any exit (success or failure) | Long-running apps (Deployments, DaemonSets) |
| `OnFailure` | Restart only on non-zero exit code (failure) | Jobs, batch tasks that should retry |
| `Never` | Never restart — pod stays Terminated | One-shot scripts, debug containers |

Important rules:
- **Deployments** force `restartPolicy: Always` (you can't change it)
- **Jobs** use `OnFailure` or `Never` (not `Always`)
- **CronJobs** inherit the Job's restartPolicy

Example — One-shot script that must not restart:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: migration-script
spec:
  restartPolicy: Never        # Run once. If it fails, do NOT retry automatically
  containers:
  - name: migrate
    image: busybox:1.28
    command: ["sh", "-c", "echo Running migration...; sleep 3; echo Migration done"]
```

Example — Batch job that retries on failure:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  restartPolicy: OnFailure    # Retry if exit code != 0, but stop if it succeeds
  containers:
  - name: worker
    image: busybox:1.28
    command: ["sh", "-c", "echo Processing...; sleep 2; exit 0"]
```

---

## Pod Termination Process — graceful shutdown sequence

When you delete a pod (or Kubernetes decides to terminate it), this exact sequence happens:

```
kubectl delete pod my-pod
         ↓
1. Pod status → Terminating
2. Kubernetes removes pod from Service endpoints
   (no new traffic routed to this pod)
3. preStop lifecycle hook runs (if defined)
4. SIGTERM sent to main container process
   (app should start graceful shutdown)
5. terminationGracePeriodSeconds countdown begins (default: 30s)
   (app has 30s to finish in-flight requests and clean up)
6. If still running after grace period → SIGKILL
   (forced kill, no more waiting)
7. Pod is removed from the cluster
```

### terminationGracePeriodSeconds

Default is 30 seconds. Increase it for slow-shutdown apps (e.g., databases flushing to disk):

```yaml
spec:
  terminationGracePeriodSeconds: 60   # Give 60s for graceful shutdown
  containers:
  - name: db
    image: postgres:15
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "pg_ctl stop -m fast"]
```

### Force-delete a pod (skip grace period)
Use only when a pod is stuck in `Terminating` state:
```bash
# Graceful delete (waits for grace period)
kubectl delete pod my-pod

# Force delete — skips grace period, immediate removal
kubectl delete pod my-pod --force --grace-period=0
```

> **Warning**: Force-delete a StatefulSet pod only if you are sure the pod is truly dead on the node. Otherwise you can have two pods with the same identity running simultaneously — which breaks StatefulSet guarantees.

### Pod termination — quick reference table

| Step | What happens | Who does it |
|------|-------------|-------------|
| `Terminating` state set | Pod shown as terminating | API server |
| Removed from endpoints | Service stops sending traffic | Endpoints controller |
| `preStop` hook | Custom cleanup code runs | kubelet |
| `SIGTERM` sent | Container gets graceful shutdown signal | kubelet |
| Grace period | App has N seconds to finish | Timer |
| `SIGKILL` | Container killed if still running | kubelet |
| Pod deleted | Removed from etcd | API server |

---

## Init Container — detailed example

### Wait for a service before starting
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ["sh", "-c", "until nslookup db-service; do echo 'Waiting for DB...'; sleep 2; done"]
  containers:
  - name: app
    image: nginx
```

### Download config before app starts
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  initContainers:
  - name: download-config
    image: busybox
    command: ["sh", "-c", "wget -O /config/app.conf http://config-server/app.conf"]
    volumeMounts:
    - name: config-vol
      mountPath: /config
  containers:
  - name: app
    image: nginx:1.28
    volumeMounts:
    - name: config-vol
      mountPath: /etc/app
  volumes:
  - name: config-vol
    emptyDir: {}
```

### Multiple init containers (run in order)
```yaml
spec:
  initContainers:
  - name: step-1
    image: busybox
    command: ["sh", "-c", "echo Step 1 done"]
  - name: step-2
    image: busybox
    command: ["sh", "-c", "echo Step 2 done"]
  - name: step-3
    image: busybox
    command: ["sh", "-c", "echo Step 3 done"]
  containers:
  - name: app
    image: nginx
```
Order: step-1 → step-2 → step-3 → app starts.

---

## Sidecar Container pattern

- Runs alongside the main container for the entire pod lifetime.
- Provides supporting functionality: logging, monitoring, proxying, syncing.
- Shares the same network (localhost) and can share volumes.

### Sidecar from session
In `full-demo`, the sidecar runs a loop printing "Sidecar is watching..." — demonstrating a monitoring/logging sidecar pattern.

### Log shipping sidecar
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-log-sidecar
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do echo $(date) 'App log entry' >> /var/log/app.log; sleep 5; done"]
    volumeMounts:
    - name: log-vol
      mountPath: /var/log
  - name: log-shipper
    image: busybox
    command: ["sh", "-c", "tail -f /var/log/app.log"]
    volumeMounts:
    - name: log-vol
      mountPath: /var/log
  volumes:
  - name: log-vol
    emptyDir: {}
```

### Proxy sidecar (Envoy/Istio style)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-proxy
spec:
  containers:
  - name: app
    image: nginx:1.28
    ports:
    - containerPort: 80
  - name: proxy
    image: envoyproxy/envoy:v1.28
    ports:
    - containerPort: 9901
```

---

## Ambassador pattern

- A container that proxies network connections FROM the main container to the outside world.
- Main app talks to localhost → ambassador handles external communication.
- Use case: local connection to a remote database via proxy.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-ambassador
spec:
  containers:
  - name: app
    image: nginx:1.28
    env:
    - name: DB_HOST
      value: "localhost"        # App thinks DB is local
    - name: DB_PORT
      value: "5432"
  - name: db-proxy
    image: haproxy:2.8  # Routes localhost:5432 → actual DB
    ports:
    - containerPort: 5432
```

---

## Adapter pattern

- A container that transforms/processes the output of the main container.
- Use case: convert log format, aggregate metrics, normalize data.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-adapter
spec:
  containers:
  - name: app
    image: nginx:1.28
    volumeMounts:
    - name: logs
      mountPath: /var/log
  - name: log-adapter
    image: busybox:1.28    # In production: use Logstash or a log converter
    command: ["sh", "-c", "while true; do cat /var/log/nginx/access.log 2>/dev/null; sleep 10; done"]
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    emptyDir: {}
```

---

## Pattern comparison

| Pattern | Location | Purpose | Example |
|---------|----------|---------|---------|
| **Init Container** | Runs before main | Setup/prerequisites | Wait for DB, download config |
| **Sidecar** | Runs alongside main | Supporting service | Log shipping, monitoring |
| **Ambassador** | Runs alongside main | Outbound proxy | DB proxy, API gateway |
| **Adapter** | Runs alongside main | Transform output | Log formatting, metrics |

---

## CKA exam tips

1. Know the difference between init containers and sidecar containers.
2. Init containers go in `spec.initContainers[]`, sidecars go in `spec.containers[]`.
3. If asked "how to ensure container X runs before container Y" → init container.
4. If asked "how to add logging/monitoring without changing app" → sidecar.
5. `kubectl logs <pod> -c <container-name>` — specify container in multi-container pods.
6. Pod status shows `Init:0/1` during init phase, then transitions to `Running`.
