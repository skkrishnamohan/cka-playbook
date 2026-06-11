# Kubernetes Workloads & Containers — Beginner-Friendly Notes

This note explains container, image, Pod, ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, and CronJob in simple terms — why each abstraction exists, when we move from one to the next, who controls them, and the benefits they provide.

---

## Quick overview
- Image: a packaged filesystem + metadata for an app (read-only). Think of it like a boxed app.
- Container: a running instance of an image. The process + filesystem running on a node.
- Pod: the smallest deployable unit in Kubernetes. One or more containers that share network and storage.
- ReplicaSet (RS): ensures a specified number of identical Pods are running.
- Deployment: manages ReplicaSets to provide declarative updates (rolling updates, history, rollback).
- StatefulSet: manages stateful applications that need stable network IDs and persistent storage.
- DaemonSet: ensures a Pod runs on every node (or selected nodes) in the cluster.
- Job: runs one or more Pods to completion (for batch/one-time tasks).
- CronJob: runs a Job on a schedule (like cron in Linux).

---

## Image vs Container (why both?)
- Image: immutable artifact built by you (e.g., `nginx:1.28`). It contains the app, libraries, and metadata.
- Container: a running image. Containers are ephemeral — they can start, stop, crash, and be replaced.

Why we use images and containers:
- Build once (image), run many times (containers).
- Images make deployments reproducible; containers are the runtime instances that Kubernetes schedules.

Example:
```bash
# build an image locally
docker build -t myapp:1.0 .

# run it as a container (locally)
docker run --name myapp myapp:1.0
```

---

## Pod — grouping containers for scheduling
- What: A Pod is one or more containers that share the same network namespace (same IP) and can share volumes.
- Why pods: Containers that must work together (sidecars, logging, proxy) are bundled in a Pod so they can communicate on localhost and share storage.
- Controlled by: usually managed indirectly by higher-level controllers (Deployment, ReplicaSet, StatefulSet). You can also create Pods directly.
- Benefits: co-located containers, shared networking and volumes, simple unit for scheduling.

When to use a Pod directly: for quick tests or learning. In production, use controllers (Deployment/StatefulSet) so Kubernetes can manage lifecycle and scaling.

Minimal Pod YAML example:
```yaml
apiVersion: v1
kind: Pod
metadata:
	name: simple-pod
spec:
	containers:
	- name: myapp
		image: nginx:1.28
```

---

## ReplicaSet — keep N copies of a Pod
- What: A ReplicaSet ensures that a specified number of replicas of a Pod are running at any time.
- Controlled by: ReplicaSet controller (or indirectly by a Deployment).
- Benefits: self-healing (recreates failed pods), scaling (you can increase/decrease replicas).

Why move from Pod → ReplicaSet:
- A single Pod is ephemeral. ReplicaSet adds availability by ensuring multiple identical Pods are running.

Example use:
```bash
kubectl create -f replicaset.yaml
kubectl scale rs my-rs --replicas=5
```

---

## Deployment — declarative updates and rollout management
- What: A Deployment manages ReplicaSets and provides declarative updates (rolling updates), history, and rollback.
- Controlled by: Deployment controller.
- Benefits: zero-downtime updates, version history, easy rollbacks, predictable updates (maxSurge/maxUnavailable).

Why move from ReplicaSet → Deployment:
- ReplicaSet only keeps replicas; Deployment adds safe update strategies and lifecycle management for evolving applications.

Simple Deployment YAML snippet:
```yaml
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

---

## StatefulSet — for stateful apps (databases, queues)
- What: A controller for stateful applications that need stable network IDs and persistent storage per replica.
- Controlled by: StatefulSet controller.
- Features/Benefits:
	- Stable, unique pod names (pod-0, pod-1, ...)
	- Stable persistent volume claims (PVCs) per pod
	- Ordered, graceful scaling and rolling updates (can control startup/termination order)

Why use StatefulSet instead of Deployment:
- If your app requires stable identity or stable storage (example: databases like MongoDB, Cassandra), StatefulSet provides guarantees that Deployments do not.

Minimal StatefulSet idea (conceptual):
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
	name: db
spec:
	serviceName: "db"
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
				image: postgres:13
	volumeClaimTemplates:
	- metadata:
			name: db-data
		spec:
			accessModes: ["ReadWriteOnce"]
			resources:
				requests:
					storage: 1Gi
```

---

## DaemonSet — one Pod per node
- What: A DaemonSet ensures that a copy of a Pod runs on every node (or on nodes matching a selector) in the cluster.
- Controlled by: DaemonSet controller.
- Features/Benefits:
	- Automatic scaling: new nodes automatically get the Pod
	- Auto-removal: when a node is removed, its Pod is removed
	- No replicas field: you don't specify how many; it's determined by cluster size
	- Runs on all nodes or selected nodes (via nodeSelector or affinity)

Why use DaemonSet instead of Deployment:
- Deployment runs a fixed number of replicas spread across nodes. DaemonSet runs on EVERY node, making it ideal for infrastructure/monitoring tasks.

Common use cases:
- Log collection (Fluentd, Logstash running on every node to collect container logs)
- Monitoring agents (Prometheus Node Exporter, Datadog agent on every node)
- Network plugins (Calico, Weave on every node for networking)
- Storage daemons (Ceph on every node)
- Node management (kubelet, Docker daemon technically run as DaemonSets on nodes)

Minimal DaemonSet YAML example:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
	name: log-collector
spec:
	selector:
		matchLabels:
			app: log-collector
	template:
		metadata:
			labels:
				app: log-collector
		spec:
			containers:
			- name: collector
				image: fluentd:latest
```

Example with nodeSelector (run only on specific nodes):
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
	name: monitoring-agent
spec:
	selector:
		matchLabels:
			app: monitor
	template:
		metadata:
			labels:
				app: monitor
		spec:
			nodeSelector:
				node-role.kubernetes.io/worker: ""  # Only on worker nodes
			containers:
			- name: agent
				image: prometheus-node-exporter:latest
```

Example commands:
```bash
# Create DaemonSet
kubectl apply -f daemonset.yaml

# Check DaemonSets
kubectl get daemonsets
kubectl describe daemonset log-collector

# Pods created by DaemonSet (one per node)
kubectl get pods -l app=log-collector
```

---

## Job — run Pods to completion
- What: A Job runs one or more Pods to completion and tracks successful completion.
- Controlled by: Job controller.
- Features/Benefits:
	- Runs until completion (not continuously like Deployment)
	- Can retry failed Pods (configurable)
	- Parallel execution (run multiple Pods in parallel for batch processing)
	- Can specify completion count and parallelism

Why use Job instead of Deployment:
- Deployment runs continuously. Job is for finite tasks (data processing, backups, reports) that should finish and stop.

Use cases:
- Batch processing (ETL, image processing)
- Database migrations
- Backups
- Report generation
- One-time setup tasks

Minimal Job YAML example:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
	name: data-processor
spec:
	completions: 1  # Pod must complete successfully 1 time
	parallelism: 1  # Run 1 Pod at a time
	backoffLimit: 3  # Retry up to 3 times
	template:
		spec:
			containers:
			- name: processor
				image: myapp:1.0
				command: ["python", "process.py"]
			restartPolicy: Never  # Don't restart on success
```

Example with parallelism (process data in parallel):
```yaml
apiVersion: batch/v1
kind: Job
metadata:
	name: parallel-job
spec:
	completions: 10  # Need 10 successful Pod completions
	parallelism: 3   # Run 3 Pods in parallel
	template:
		spec:
			containers:
			- name: worker
				image: worker:latest
			restartPolicy: Never
```

Example commands:
```bash
# Create Job
kubectl apply -f job.yaml

# Check Jobs
kubectl get jobs
kubectl describe job data-processor

# Check Job Pods
kubectl get pods -l job-name=data-processor

# View Job logs
kubectl logs <pod-name>

# Delete Job (and its Pods)
kubectl delete job data-processor
```

---

## CronJob — scheduled Jobs
- What: A CronJob runs a Job on a repeating schedule (like cron in Linux).
- Controlled by: CronJob controller.
- Features/Benefits:
	- Cron schedule syntax (minute hour day month weekday)
	- Creates a new Job on each schedule run
	- Automatic cleanup of old Jobs based on history limits

Why use CronJob instead of Job:
- Job runs once. CronJob repeats on a schedule (e.g., daily backups, hourly reports).

Use cases:
- Database backups (every night at 2 AM)
- Log rotation (every day)
- Periodic reports (every Monday)
- Cleanup tasks (every week)
- Health checks (every 5 minutes)

Minimal CronJob YAML example:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
	name: backup
spec:
	schedule: "0 2 * * *"  # Run at 2 AM every day (cron syntax)
	jobTemplate:
		spec:
			template:
				spec:
					containers:
					- name: backup-tool
						image: backup:latest
						command: ["backup.sh"]
					restartPolicy: OnFailure  # Retry on failure
```

Common schedule examples:
```
"0 0 * * *"      → Every day at midnight
"0 0 * * 0"      → Every Sunday at midnight
"0 */4 * * *"    → Every 4 hours
"*/15 * * * *"   → Every 15 minutes
"0 2 1 * *"      → First day of every month at 2 AM
```

Example commands:
```bash
# Create CronJob
kubectl apply -f cronjob.yaml

# Check CronJobs
kubectl get cronjobs
kubectl describe cronjob backup

# Check Jobs created by CronJob
kubectl get jobs

# View a Job's Pods and logs
kubectl get pods -l job-name=backup-xxxxx
```

---

## Namespaces — logical cluster isolation
- What: A Namespace is a way to partition Kubernetes cluster resources among multiple users/teams (logical isolation, not network isolation).
- Why Namespaces: Allows multiple teams/projects to share a cluster without interfering with each other. Enables resource quotas and access control per namespace.
- Controlled by: Admin creates namespaces; resources can be deployed into specific namespaces.
- Benefits: Resource isolation, separate RBAC policies, resource quotas per namespace, cleaner resource organization.

Default namespaces in Kubernetes:
- `default`: where resources go if no namespace is specified
- `kube-system`: Kubernetes system components (DNS, controller-manager, scheduler)
- `kube-public`: publicly readable resources
- `kube-node-lease`: node heartbeat information

Creating and using namespaces:
```yaml
# Create a namespace via YAML
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

Common namespace commands:
```bash
# Create namespace
kubectl create namespace production
kubectl create ns dev

# List namespaces
kubectl get ns

# Use a specific namespace
kubectl apply -f deployment.yaml -n production

# Set default namespace for context
kubectl config set-context --current --namespace=production

# View resources in a namespace
kubectl get pods -n production
kubectl get all -n production

# Delete namespace (deletes all resources in it)
kubectl delete namespace production
```

Example: deploying app to a specific namespace
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production  # Specify namespace
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
        image: nginx
```

---

## Readiness & Liveness Probes — health checks for Pods
- What: Probes are mechanisms to check if a container is healthy and ready to serve traffic.
- Two types:
  - **Liveness Probe**: checks if a container is alive. If it fails, Kubernetes restarts the container.
  - **Readiness Probe**: checks if a container is ready to serve traffic. If it fails, traffic is not sent to this Pod.
- Why: Helps Kubernetes manage unhealthy containers automatically (restart dead ones, stop sending traffic to unready ones).

Three probe mechanisms:
1. **httpGet**: HTTP request to a path (most common for web apps)
2. **exec**: execute a command inside container
3. **tcpSocket**: TCP connection to a port

Readiness Probe example (HTTP check):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: http-probe-pod
  namespace: default
spec:
  containers:
  - name: http-container
    image: nginx
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
      timeoutSeconds: 1
      failureThreshold: 3
```

Liveness Probe example (exec command):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "touch /tmp/healthy && sleep 30 && rm /tmp/healthy && sleep 600"]
    livenessProbe:
      exec:
        command:
        - test
        - -e
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

Combined Readiness + Liveness Probes:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
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
        image: nginx
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 2
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 10
          failureThreshold: 3
```

Probe configuration parameters:
- `initialDelaySeconds`: Wait before first probe check (allows app startup time)
- `periodSeconds`: How often to check
- `timeoutSeconds`: Max time for probe to complete
- `failureThreshold`: How many failures before restarting/removing from service
- `successThreshold`: How many successes to mark as healthy

---

## Lifecycle Hooks — actions on Pod events
- What: Pre-start (postStart) and pre-stop (preStop) hooks allow you to run code when a container starts or is about to stop.
- Why: Useful for cleanup, initialization, graceful shutdown, health checks before termination.
- Hook types:
  - **postStart**: runs immediately after container starts (runs in parallel with main process)
  - **preStop**: runs before container is terminated (allows graceful cleanup)

postStart hook example:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-hooks-example
spec:
  containers:
  - name: nginx
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Started > /var/log/nginx.log"]
```

preStop hook example (graceful shutdown):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-shutdown
spec:
  containers:
  - name: app
    image: myapp:latest
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 15"]  # Wait 15 sec for existing connections
    terminationGracePeriodSeconds: 30  # Total time allowed for graceful shutdown
```

Combined example with both hooks:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-complete
spec:
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080
    lifecycle:
      postStart:
        httpGet:
          path: /health
          port: 8080
      preStop:
        exec:
          command: ["/bin/sh", "-c", "echo Stopping >> /var/log/app.log && sleep 10"]
  terminationGracePeriodSeconds: 30
```

---

## Services — exposing Pods for communication
- What: A Service is an abstraction that defines a logical set of Pods and a policy for accessing them.
- Why Services: Pods are ephemeral (can be created/destroyed). Services provide a stable IP and DNS name for accessing Pods.
- Controlled by: Service controller assigns cluster IP and manages endpoint updates.
- Benefits: load balancing, DNS discovery, abstraction from Pod IP changes.

Four types of Services:

### 1. ClusterIP (default)
- Exposes service on a cluster-internal IP
- Only accessible from within the cluster
- Most commonly used for internal communication between services

ClusterIP example:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
```

Usage: `curl web-service` (DNS resolution within cluster)

### 2. NodePort
- Exposes service on each node's IP at a static port
- Accessible from outside the cluster via `<NodeIP>:<NodePort>`
- Port range: 30000-32767

NodePort example:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

Usage: `curl <NodeIP>:30080`

### 3. LoadBalancer
- Exposes service externally using a cloud provider's load balancer
- Automatically assigns an external IP
- Combines NodePort + cloud LB

LoadBalancer example:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

Usage: `curl <LoadBalancerIP>`

### 4. Headless Service (ClusterIP with clusterIP: None)
- No cluster IP assigned
- Used for stateful apps that need direct Pod-to-Pod communication
- DNS returns Pod IPs directly

Headless Service example:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  clusterIP: None  # Headless
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
```

Service with multiple ports:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
```

Common Service commands:
```bash
# Create Service
kubectl apply -f service.yaml

# List Services
kubectl get svc
kubectl get service

# Describe Service
kubectl describe svc web-service

# Get Service endpoints (actual Pod IPs)
kubectl get endpoints web-service

# Port forward to Service
kubectl port-forward svc/web-service 8080:80
```

---

## Horizontal Pod Autoscaler (HPA) — auto-scaling replicas
- What: HPA automatically scales the number of Pods in a Deployment based on CPU/memory usage or custom metrics.
- Why HPA: Manually scaling is error-prone. HPA responds to load automatically.
- Controlled by: HPA controller monitors metrics and adjusts replica count.
- Benefits: cost efficiency (scale down when load is low), high availability (scale up when load is high).

HPA requires:
- Metrics Server installed in cluster (for CPU/memory metrics)
- Resource requests defined on container (for percentage-based scaling)

Basic HPA example (scale based on CPU):
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-nginx
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-nginx
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

This HPA configuration:
- Targets the `hpa-nginx` Deployment
- Maintains 1-5 replicas
- Scales up if average CPU > 50%
- Scales down if average CPU < 50%

HPA with both CPU and memory:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

Deployment with resource requests (required for HPA):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-nginx
  template:
    metadata:
      labels:
        app: hpa-nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: "100m"        # 100 milliCPU (0.1 CPU)
            memory: "128Mi"    # 128 Megabytes
          limits:
            cpu: "500m"        # Max 500 milliCPU
            memory: "512Mi"    # Max 512 Megabytes
```

HPA commands:
```bash
# Create HPA
kubectl apply -f hpa.yaml

# List HPAs
kubectl get hpa

# Describe HPA
kubectl describe hpa hpa-nginx

# Watch HPA scaling in action
kubectl get hpa hpa-nginx --watch

# View HPA metrics
kubectl get hpa hpa-nginx -o wide
```

---

## Vertical Pod Autoscaler (VPA) — auto-scaling resource requests
- What: VPA automatically adjusts CPU/memory requests and limits based on actual usage.
- Why VPA: Manual resource tuning is difficult. VPA learns from actual consumption and optimizes.
- Difference from HPA: HPA scales number of Pods; VPA scales individual Pod resources.
- Controlled by: VPA admission controller recommends and applies new resource values.
- Benefits: right-sizing resources, cost optimization, preventing OOM kills.

Basic VPA example:
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"  # Options: Off, Initial, Recreate, Auto
```

VPA update modes:
- `Off`: Only provide recommendations, don't apply them
- `Initial`: Only apply on Pod creation
- `Recreate`: Apply by recreating Pods when recommendations change
- `Auto`: Use best strategy (usually Recreate)

VPA with resource guidelines:
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-with-limits
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: "50m"
        memory: "64Mi"
      maxAllowed:
        cpu: "2"
        memory: "2Gi"
      controlledResources: ["cpu", "memory"]
```

HPA + VPA together (for optimal scaling):
```bash
# HPA scales replica count based on metrics
# VPA scales individual Pod resources based on actual usage
# Together they provide both horizontal and vertical scaling
```

VPA commands:
```bash
# Create VPA
kubectl apply -f vpa.yaml

# List VPAs
kubectl get vpa

# Describe VPA
kubectl describe vpa my-app-vpa

# View VPA recommendations
kubectl describe vpa my-app-vpa | grep -A 5 "Recommendation"
```

---

## Resource Quota — limiting resources per namespace
- What: ResourceQuota restricts aggregate resource consumption in a namespace.
- Why ResourceQuota: Prevents one team/app from consuming all cluster resources.
- Controlled by: Admin creates quotas; Kubernetes enforces them.
- Benefits: fair resource sharing, cost control, prevents resource exhaustion.

Quota enforces limits on:
- Compute resources (CPU, memory)
- Storage resources
- Object counts (Pods, Services, Deployments, etc.)

Basic ResourceQuota example:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"           # Total requested CPU
    requests.memory: "20Gi"      # Total requested memory
    limits.cpu: "20"             # Total CPU limits
    limits.memory: "40Gi"        # Total memory limits
    pods: "100"                  # Max Pods in namespace
```

ResourceQuota with object limits:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota
  namespace: dev
spec:
  hard:
    pods: "50"
    services: "10"
    deployments.apps: "20"
    replicasets.apps: "20"
    jobs.batch: "10"
    persistentvolumeclaims: "5"
```

Complete ResourceQuota example:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-quota
  namespace: staging
spec:
  hard:
    # Compute resources
    requests.cpu: "5"
    requests.memory: "10Gi"
    limits.cpu: "10"
    limits.memory: "20Gi"
    # Object counts
    pods: "30"
    services: "5"
    deployments.apps: "10"
    jobs.batch: "5"
    # Storage
    requests.storage: "100Gi"
    persistentvolumeclaims: "10"
```

Deployment within quota (must have resource requests):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-within-quota
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: myapp:latest
        resources:
          requests:
            cpu: "100m"        # Must specify for quota
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

ResourceQuota commands:
```bash
# Create ResourceQuota
kubectl apply -f resourcequota.yaml

# List ResourceQuotas
kubectl get resourcequota

# Describe quota (shows current usage)
kubectl describe resourcequota compute-quota -n production

# View usage
kubectl describe ns production

# Check if quota prevents resource creation
# (error message shows quota exceeded)
```

Example error when quota exceeded:
```
Error from server (Forbidden): error when creating "deployment.yaml": 
pods "app-deployment-xyz" is forbidden: 
exceeded quota: compute-quota, requested: requests.cpu=500m, used: requests.cpu=9500m, limited: requests.cpu=10
```

---

## How the abstractions evolve (complete rationale)
- Containers: run the app process.
- Pod: Groups containers that must run together and share networking/storage.
- Namespace: Logical partitioning of cluster for multi-tenancy and resource isolation.
- Service: Provides stable DNS and load balancing for Pods.
- ReplicaSet: Ensures availability by maintaining N copies of a Pod.
- Deployment: Manages ReplicaSets to support declarative upgrades, rollbacks, and scaling.
- StatefulSet: A sibling of Deployment for stateful workloads that require identity and stable storage.
- DaemonSet: Runs a Pod on every node (infrastructure/monitoring tasks).
- Job: Runs Pod(s) to completion (batch/one-time tasks).
- CronJob: Schedules Jobs repeatedly (periodic tasks).
- Readiness/Liveness Probes: Health checks to manage Pod lifecycle and traffic routing.
- Lifecycle Hooks: Pre-start and pre-stop actions for graceful initialization and shutdown.
- HPA: Auto-scales Deployment replicas based on metrics (horizontal scaling).
- VPA: Auto-scales Pod resource requests based on actual usage (vertical scaling).
- ResourceQuota: Limits aggregate resource consumption in a namespace.

Short mapping:
- Use a single Pod for simple/debug tasks.
- Use ReplicaSet when you need multiple identical pods (but prefer Deployment in most cases).
- Use Deployment for stateless apps where rolling updates and rollback matter (web servers, APIs).
- Use StatefulSet for stateful apps needing stable IDs and storage (databases, message queues).
- Use DaemonSet for infrastructure/monitoring that must run everywhere (logging, monitoring agents, network plugins).
- Use Job for batch/one-time tasks that must complete (data processing, migrations, backups).
- Use CronJob for periodic tasks (scheduled backups, reports, cleanup).
- Use Namespaces to isolate teams/projects and enable resource quotas.
- Use Services to expose Pods (ClusterIP for internal, NodePort/LoadBalancer for external).
- Use Readiness Probes to prevent traffic to unready Pods.
- Use Liveness Probes to restart dead containers.
- Use Lifecycle Hooks for initialization and graceful shutdown.
- Use HPA to auto-scale based on load.
- Use VPA to right-size resource requests.
- Use ResourceQuota to prevent one team from consuming all resources.

---

## Quick commands cheat sheet
```bash
# Create from YAML
kubectl apply -f <file>.yaml

# Namespace operations
kubectl create namespace production
kubectl create ns dev
kubectl get ns
kubectl get pods -n production
kubectl config set-context --current --namespace=production
kubectl delete namespace production

# Pod and Deployment operations
kubectl get pods
kubectl get deployments
kubectl scale deployment web --replicas=5
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web

# Check resources
kubectl get rs
kubectl get statefulsets
kubectl get daemonsets
kubectl get jobs
kubectl get cronjobs

# Service operations
kubectl get svc
kubectl describe svc web-service
kubectl get endpoints web-service
kubectl port-forward svc/web-service 8080:80

# Probes and Health checks
kubectl describe pod <pod-name>  # Shows probe status
kubectl logs <pod-name>          # View container logs

# DaemonSet operations
kubectl get ds
kubectl describe daemonset log-collector
kubectl get pods -l app=log-collector

# Job operations
kubectl get jobs
kubectl describe job data-processor
kubectl get pods -l job-name=data-processor
kubectl logs <job-pod-name>
kubectl delete job data-processor

# CronJob operations
kubectl get cronjobs
kubectl describe cronjob backup
kubectl get jobs -l cronjob=backup

# HPA operations
kubectl get hpa
kubectl describe hpa hpa-nginx
kubectl get hpa hpa-nginx --watch
kubectl get hpa hpa-nginx -o wide

# VPA operations
kubectl get vpa
kubectl describe vpa my-app-vpa

# ResourceQuota operations
kubectl get resourcequota
kubectl describe resourcequota compute-quota -n production
kubectl describe ns production

# Delete resources
kubectl delete deployment web
kubectl delete statefulset db
kubectl delete daemonset log-collector
kubectl delete job batch-job
kubectl delete cronjob scheduled-task
kubectl delete svc web-service
kubectl delete hpa hpa-nginx
kubectl delete vpa my-app-vpa
kubectl delete resourcequota compute-quota -n production
```

---

## One-line takeaways (beginner-friendly)
- Build images, run containers. Containers need orchestration.
- Pods are the scheduling unit in Kubernetes — group containers that belong together.
- Namespaces logically partition the cluster for multi-team deployments.
- Services provide stable DNS and load balancing for discovering and communicating with Pods.
- ReplicaSets keep many copies of a Pod running.
- Deployments give you safe upgrades and rollbacks for ReplicaSets.
- StatefulSets are for stateful services that need stable identity and storage.
- DaemonSets run one Pod per node (logging, monitoring, networking).
- Jobs run Pods to completion (batch tasks, migrations, backups).
- CronJobs schedule Jobs to run periodically (daily backups, hourly reports).
- Readiness Probes prevent traffic to unready Pods; Liveness Probes restart dead containers.
- Lifecycle Hooks allow graceful initialization (postStart) and shutdown (preStop).
- HPA auto-scales replicas based on CPU/memory; VPA auto-scales resource requests.
- ResourceQuota prevents one team from consuming all cluster resources.

---

## Init Containers — setup before main app starts
- What: Init containers run BEFORE the main app containers in a Pod. They must complete successfully before app containers start.
- Why: Useful for setup tasks like downloading config, waiting for a service, or database migration.
- Key behavior:
  - Run one at a time, in order.
  - If an init container fails, Kubernetes restarts it (the Pod won't start until all init containers succeed).
  - Init containers don't count toward resource limits of the Pod once done.

Init container example:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nslookup db-service; do echo waiting; sleep 2; done']
  - name: init-config
    image: busybox
    command: ['sh', '-c', 'wget -O /config/app.conf http://config-server/config']
    volumeMounts:
    - name: config-vol
      mountPath: /config
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: config-vol
      mountPath: /config
  volumes:
  - name: config-vol
    emptyDir: {}
```

---

## Startup Probe — for slow-starting containers
- What: A Startup Probe checks if the application has started. Until it succeeds, liveness/readiness probes are disabled.
- Why: Prevents liveness probe from killing a container that's just slow to start (e.g., Java apps, legacy apps).
- Once startup probe succeeds, it is never checked again — liveness and readiness take over.

Startup probe example:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-start-app
spec:
  containers:
  - name: app
    image: heavy-java-app:latest
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30    # 30 * 10s = 300s (5 min) to start
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10
```

---

## Pod Disruption Budget (PDB) — protect availability during disruptions
- What: A PDB limits how many Pods of a Deployment/ReplicaSet can be voluntarily disrupted at a time.
- Why: Ensures minimum availability during node drains, cluster upgrades, or maintenance.
- Voluntary disruptions: node drain, cluster autoscaler, rolling updates.
- Involuntary disruptions: hardware failure, kernel panic (PDB does NOT protect against these).

PDB example:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2         # At least 2 pods must stay running
  selector:
    matchLabels:
      app: web
```

Alternative — use maxUnavailable:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  maxUnavailable: 1       # At most 1 pod can be down at a time
  selector:
    matchLabels:
      app: web
```

PDB commands:
```bash
kubectl get pdb
kubectl describe pdb web-pdb
```

---

## Taints and Tolerations — control which Pods run on which Nodes
- What: Taints are applied to nodes to repel pods. Tolerations are applied to pods to allow them onto tainted nodes.
- Why: Keeps certain workloads off specific nodes (e.g., keep user pods off master nodes, reserve GPU nodes).

Taint a node:
```bash
# Taint a node (no pod will schedule unless it tolerates this taint)
kubectl taint nodes worker1 key=value:NoSchedule

# Remove taint
kubectl taint nodes worker1 key=value:NoSchedule-
```

Taint effects:
- `NoSchedule`: new pods won't schedule here (existing pods stay).
- `PreferNoSchedule`: soft version — scheduler tries to avoid this node.
- `NoExecute`: new pods won't schedule AND existing pods are evicted.

Pod with toleration:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: gpu-app
    image: gpu-app:latest
```

---

## Node Affinity — schedule Pods on preferred/required Nodes
- What: Node affinity lets you constrain which nodes a Pod can run on, based on node labels.
- Two types:
  - `requiredDuringSchedulingIgnoredDuringExecution` — hard requirement (must match or Pod stays Pending).
  - `preferredDuringSchedulingIgnoredDuringExecution` — soft preference (scheduler tries to match but can fallback).

Node affinity example:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: app
    image: nginx
```

Label a node (needed for affinity to work):
```bash
kubectl label nodes worker1 disktype=ssd
kubectl get nodes --show-labels
```

---

## Multi-container Pod Patterns (CKA relevant)

### Sidecar pattern
- A helper container runs alongside the main app (e.g., log shipper, proxy).
```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
  - name: log-shipper
    image: fluentd:latest
```

### Ambassador pattern
- A proxy container handles external communication for the main app.

### Adapter pattern
- A container transforms/standardizes output from the main app.

---

## ConfigMaps and Secrets — externalizing configuration

### ConfigMap (non-sensitive config)
```bash
# Create ConfigMap from literal
kubectl create configmap app-config --from-literal=DB_HOST=postgres --from-literal=DB_PORT=5432

# Create from file
kubectl create configmap nginx-conf --from-file=nginx.conf
```

Use in Pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    envFrom:
    - configMapRef:
        name: app-config
```

### Secret (sensitive data — base64 encoded, not encrypted by default)
```bash
# Create Secret
kubectl create secret generic db-creds --from-literal=username=admin --from-literal=password=s3cr3t
```

Use in Pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: username
```

Mount as volume:
```yaml
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-creds
```

---

## PersistentVolume (PV) and PersistentVolumeClaim (PVC) — storage basics
- **PV**: A piece of storage provisioned by an admin (or dynamically via StorageClass).
- **PVC**: A request for storage by a user/pod.
- Pod → PVC → PV → actual disk.

PVC example:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

Use PVC in Pod:
```yaml
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - mountPath: /data
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
```

Access Modes:
- `ReadWriteOnce (RWO)`: mounted as read-write by a single node.
- `ReadOnlyMany (ROX)`: mounted as read-only by many nodes.
- `ReadWriteMany (RWX)`: mounted as read-write by many nodes (requires NFS or similar).

