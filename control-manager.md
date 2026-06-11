# Kubernetes Controller Manager — Ultra Deep Dive

This note explains the Kubernetes Controller Manager in depth: what it does, how it works, the controllers it runs, its architecture, leader election, failure behavior, configuration, and how it fits into the control plane.

---

## What is the Controller Manager?

The Kubernetes Controller Manager is a control plane component that runs a set of controllers. These controllers are control loops that watch the API server for changes and take actions to make the actual cluster state match the desired state declared by users.

The Controller Manager is not a single controller itself; it is a process that hosts multiple controller processes.

Key responsibilities:
- Run controllers that reconcile objects such as ReplicaSets, Deployments, Services, Nodes, Endpoints, and PersistentVolumeClaims.
- Manage replications, endpoints, nodes, service accounts, namespace lifecycle, and more.
- Ensure the cluster converges toward declared desired state.

---

## Controller Manager architecture

The Controller Manager runs as a single binary called `kube-controller-manager`.

It typically runs on the control plane node and communicates with:
- `kube-apiserver`: reads/writes Kubernetes API objects
- `etcd`: indirectly through the API server
- other control plane components like the scheduler and cloud provider APIs

Important elements:
- **Controllers**: independent control loops
- **Leader election**: when multiple instances exist, one leader is active
- **Client to API server**: watches object changes and updates status
- **Controller queue**: each controller queues events and reconciles them

### Control loop pattern

Each controller follows the same basic algorithm:

1. Watch API objects (via watch/list on the API server)
2. Enqueue events or keys for reconciliation
3. Process the queue one item at a time
4. Read the current object state from the API server
5. Compute expected state from desired object spec
6. Perform actions to achieve the desired state
7. Repeat continuously

This pattern is often described as:

```
Desired state -> API objects -> Controller loop -> Actual state
```

---

## Built-in controllers hosted by kube-controller-manager

The Controller Manager includes many built-in controllers. Some of the most important are:

### Node controller
- Monitors node health and readiness
- Detects node failures and marks nodes as `NotReady`
- Creates `NodeNotReady` conditions and evicts pods after a timeout
- Manages node lease renewals when `Lease` API is enabled

### Replication controller
- Ensures a specified number of Pod replicas exist for a `ReplicationController` resource
- Creates or deletes Pods to match the desired replica count

### Deployment controller
- Manages `Deployment` objects by creating and scaling ReplicaSets
- Controls rolling updates, rollbacks, and deployment history
- Reconciles the replica count and template when the manifest changes

### ReplicaSet controller
- Ensures the desired number of replicas for `ReplicaSet` objects
- Tracks Pods matching the ReplicaSet selector and manages creation/deletion

### StatefulSet controller
- Creates and manages `StatefulSet` Pods
- Ensures stable network IDs and persistent storage for each replica
- Controls ordered pod creation and termination

### DaemonSet controller
- Ensures a Pod runs on every eligible node or selected nodes
- Creates daemon Pods when nodes join the cluster
- Deletes daemon Pods when nodes are removed or tainted

### Job controller
- Creates Pods to complete jobs defined by `Job` objects
- Tracks completions and retries failed Pods
- Handles parallelism and completion counts

### CronJob controller
- Creates Jobs on schedule for `CronJob` objects
- Maintains job history and cleanup of old jobs

### Service controller
- Keeps `Service` endpoints in sync with matching Pods
- Creates and updates `Endpoints` or `EndpointSlices`
- When cloud provider integration is enabled, it may create external load balancers

### Endpoint controller
- Populates `Endpoints` objects for Services using available Pods
- Ensures Service selectors map to the correct backend pods

### PersistentVolume controller
- Binds `PersistentVolumeClaims` to available `PersistentVolumes`
- Checks storage class and access mode compatibility
- Reclaims volumes after release if configured

### Namespace controller
- Cleans up resources in a namespace when deletion is requested
- Removes finalizers and ensures namespace terminates cleanly

### Service Account controller
- Creates default service account secrets for new namespaces
- Manages token secrets and service account lifecycle

### TTL controller (deprecated/disabled in newer versions)
- Used to clean up finished resources after a TTL period
- Mostly replaced by `CronJob` and owner references

---

## Controller-manager configuration and flags

The Controller Manager is configured using command-line flags and its config file. Common flags include:

- `--master` or `--kubeconfig`: API server location or kubeconfig file
- `--service-account-private-key-file`: private key for signing service account tokens
- `--root-ca-file`: CA certificate for signing service account certificates
- `--cluster-signing-cert-file` / `--cluster-signing-key-file`: cert signing key for CSR controller
- `--controllers`: list of controllers to enable/disable
- `--allocate-node-cidrs`: if set, allocates CIDRs to nodes for cloud networking
- `--cluster-cidr`: CIDR for pod IP allocation if using CIDR-based network management
- `--enable-hostpath-provisioner`: enable default hostPath provisioner
- `--terminated-pod-gc-threshold`: threshold for garbage collecting completed pods
- `--leader-elect`: enable leader election between controller manager instances
- `--use-service-account-credentials`: use service account creds for controllers

### Config file example

A minimal kube-controller-manager configuration may look like:

```yaml
apiVersion: kubecontrolplane.config.k8s.io/v1beta1
kind: KubeControllerManagerConfiguration
clientConnection:
  kubeconfig: /etc/kubernetes/controller-manager.conf
leaderElection:
  leaderElect: true
controllerManager:
  controllers: "*"
```

Note: the exact config API version and fields may change between Kubernetes versions.

---

## Leader election

Although the Controller Manager can run multiple instances, only one should actively manage the cluster at a time. This prevents duplicate actions.

### How leader election works
- Instances compete for a lock stored in the API server (usually in a configmap or lease object)
- Only the leader performs reconciliation loops
- Followers stay in standby mode and continuously attempt to acquire leadership
- If the leader fails or stops renewing the lock, a new leader is elected

### Why leader election exists
- High availability: multiple controller manager instances can be deployed for failover
- No split-brain: only one active controller manager writes state
- Better reliability: control plane continues if the leader process crashes

### Leader election objects
- `Lease` objects in the `kube-system` namespace by default
- Sometimes `ConfigMap` is used for leader election in older configs

---

## How the Controller Manager works with the control plane

The Controller Manager is one of the three main control plane components:
- `kube-apiserver`: the central API front end
- `kube-scheduler`: decides which node a pod should run on
- `kube-controller-manager`: manages controllers and reconcilers

Workflow example for a Deployment update:

1. User applies a new Deployment YAML to the API server
2. API server stores the desired Deployment spec in etcd
3. Deployment controller sees the change via watch events
4. Deployment controller compares desired replica count and template to actual state
5. ReplicaSet controller creates a new ReplicaSet or scales the existing one
6. Scheduler places new Pods on suitable nodes
7. Node controller monitors node readiness and evicts failed pods if needed

This shows the Controller Manager as the orchestration layer that keeps resource state consistent.

---

## Controller kinds and object relationships

Some controllers do not manage resources directly; they manage other controllers or sub-resources:

- Deployment controller manages ReplicaSets, which in turn manage Pods
- StatefulSet controller manages Pods and persistent storage claims
- DaemonSet controller manages Pod placement across nodes
- Job/CronJob controllers manage Pods to completion
- Service controller manages Endpoints/EndpointSlices for Services

### Controller ownership model

Controllers use owner references and labels to track managed resources.
- A ReplicaSet owns its Pods
- A Deployment owns the ReplicaSet
- A Job owns its Pods
- Owner references enable cascading deletion and garbage collection

The controller manager ensures that controllers honor ownership and do not fight each other.

---

## Internal design details

### Work queues

Each controller typically has a work queue. Events are placed into the queue, then processed by worker threads.

Example queue flow:
- Add event received for object `my-deployment`
- Object key `default/my-deployment` enqueued
- Worker pops the key and reconciles the object
- If reconciliation fails, the key may be requeued with rate limiting

### Informers and caches

Controllers use shared informers to watch the API server and maintain local caches.
- Informers reduce API server load by reusing watch streams
- Local caches provide fast access to recently seen objects
- Event handlers fire when objects are added, updated, or deleted

### Reconciliation model

The desired state is expressed in the object spec. The controller reads the current state and makes changes to reach the desired state.

Reconciliation is idempotent: repeating the same reconcile operation should not change the system if the desired state is already met.

---

## Common controllers and example behavior

### ReplicaSet / Deployment
- A Deployment update changes its Pod template
- Deployment controller creates a new ReplicaSet and scales it
- ReplicaSet controller creates Pods to match desired replicas
- Scheduler assigns Pods to nodes
- Node controller updates Pod status and readiness

### StatefulSet
- Ensures Pod names remain stable (`myapp-0`, `myapp-1`)
- Creates PVCs for each replica using `volumeClaimTemplates`
- Uses ordered pod startup and termination

### Job
- Creates Pods with `restartPolicy: Never` or `OnFailure`
- Tracks Pod completions and counts successful runs
- Cleans up Pods when complete, if configured

### Service / Endpoint
- Service object defines a stable DNS name and selector
- Endpoint controller creates or updates `Endpoints`/`EndpointSlices`
- Service controller may create external load balancers via cloud provider integration

---

## Controller Manager failure modes and recovery

### What happens if kube-controller-manager fails?
- If no standby instance exists, controllers stop reconciling
- The cluster continues running existing pods, but new desired-state changes may not be enforced
- Leader election allows a standby instance to take over if configured

### Common failure causes
- API server unreachable
- kubeconfig or authentication failure
- etcd unavailable (indirectly through API server)
- Misconfigured controller flags
- Resource exhaustion on control plane node

### Recovery
- Check controller manager logs for errors
- Verify leader election lock object
- Ensure the API server kubeconfig is valid and accessible
- Restart the process or pod if running as static pod or deployment

---

## Practical notes for exam and operations

- The Controller Manager is a core control plane component, not a user-facing workload controller.
- It runs controllers, but you rarely interact with it directly in day-to-day kubectl workflows.
- Knowing the difference between a controller and the controller manager is important.
- The scheduler decides node placement; the controller manager ensures resources reach the desired state.
- In HA clusters, there may be multiple controller manager instances, but only one leader at a time.

### Useful commands for troubleshooting controller manager
```bash
# Check controller-manager pod status
kubectl get pods -n kube-system | grep controller-manager

# View controller-manager logs (look for reconciliation errors)
kubectl logs -n kube-system kube-controller-manager-<node-name>

# Check leader election status
kubectl get lease -n kube-system kube-controller-manager

# View the static pod manifest
cat /etc/kubernetes/manifests/kube-controller-manager.yaml

# Check which controllers are enabled
kubectl -n kube-system describe pod kube-controller-manager-<node> | grep controllers
```

### Cloud Controller Manager (CCM) — important distinction
- In cloud environments (AWS, Azure, GCP), cloud-specific controllers are separated into `cloud-controller-manager`.
- CCM manages: node lifecycle (cloud instance health), route controller (cloud routes), service controller (cloud load balancers).
- This separation allows the core kube-controller-manager to remain cloud-agnostic.
- If you see `--cloud-provider=external` in kubelet config, CCM is in use.

### Garbage Collection Controller
- Automatically cleans up orphaned resources (Pods without owners, old ReplicaSets).
- Uses **owner references** to track parent-child relationships.
- Two deletion modes:
  - **Foreground**: parent waits until all dependents are deleted first.
  - **Background** (default): parent is deleted immediately, dependents cleaned up asynchronously.
- `--terminated-pod-gc-threshold` flag controls when terminated pods are garbage collected (default: 12500).

### Horizontal Pod Autoscaler Controller
- Runs inside the controller manager (not a separate component).
- Checks metrics every 15 seconds (configurable with `--horizontal-pod-autoscaler-sync-period`).
- Uses the formula: `desiredReplicas = ceil(currentReplicas * (currentMetricValue / desiredMetricValue))`.
- Has a cooldown period to prevent flapping:
  - Scale-up stabilization: 0 seconds (immediate).
  - Scale-down stabilization: 5 minutes (default).

### CKA exam tip — controller manager failure scenario
- If controller-manager stops working:
  - Deployments won't reconcile (no rolling updates).
  - ReplicaSets won't create missing pods.
  - Nodes won't be marked NotReady (node controller stops).
  - Namespaces in Terminating state won't finish deletion.
- Existing running pods are NOT affected — they continue to run.
- Fix: Check static pod manifest, logs, and certificate validity.

---

## Summary

The Kubernetes Controller Manager is the brain that hosts control loops for the cluster. It reads desired state from the API server, runs controllers to reconcile real state, and ensures the cluster keeps moving toward what users declared.

This deep dive covers:
- Controller Manager architecture
- Built-in controllers and their behaviors
- Leader election and HA
- Controller loops, queues, and informers
- Object relationships and ownership
- Failure modes and recovery
- Configuration and practical operational notes
