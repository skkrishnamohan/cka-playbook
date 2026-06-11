# Kubernetes Troubleshooting — CKA Domain 5 (30%)

> Highest weight domain. Every sub-topic below maps to a real CKA exam scenario.
> **Golden rule**: always start with `kubectl describe` → look at `Events:` section at the bottom.

---

## Quick triage checklist (use for any problem)

```bash
# 1. What's wrong?
kubectl get nodes                         # Node status
kubectl get pods -A                       # All pods, all namespaces
kubectl get events -A --sort-by=.metadata.creationTimestamp  # Recent events

# 2. Narrow down
kubectl describe pod <pod-name> -n <ns>   # Events + container state
kubectl describe node <node-name>         # Node conditions, capacity, pods

# 3. Dig into logs
kubectl logs <pod-name>                   # Current log
kubectl logs <pod-name> --previous        # Previous crash log
kubectl logs <pod-name> -c <container>    # Multi-container pod

# 4. Get inside
kubectl exec -it <pod-name> -- /bin/sh    # Shell into container
```

---

## 1. Troubleshoot Nodes — `NotReady` and beyond

### Step 1 — Identify the problem

```bash
kubectl get nodes
# NAME     STATUS     ROLES           AGE   VERSION
# node1    NotReady   <none>          2d    v1.34.0
# cp       Ready      control-plane   2d    v1.34.0

kubectl describe node node1
# Look for "Conditions:" section
# Type                 Status   Reason
# ----                 ------   ------
# Ready                False    KubeletNotReady
# MemoryPressure       False    KubeletHasSufficientMemory
# DiskPressure         False    KubeletHasNoDiskPressure
# PIDPressure          False    KubeletHasSufficientPID
```

### Step 2 — Check kubelet on the node

SSH into the node, then:
```bash
# Is kubelet running?
systemctl status kubelet
# Look for: Active: active (running) or failed

# If failed, get the error:
journalctl -u kubelet -n 50 --no-pager
# Common errors:
# - "failed to run Kubelet: unable to load client CA file"  → certificate issue
# - "connection refused" to API server                       → network issue
# - "node not found"                                         → node was deleted from API

# Restart kubelet
systemctl restart kubelet

# Enable kubelet to start on boot (if it's not set)
systemctl enable kubelet
```

### Step 3 — Check kubelet config
```bash
# Kubelet config file
cat /var/lib/kubelet/config.yaml

# Kubelet args/flags
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# After editing, reload
systemctl daemon-reload
systemctl restart kubelet
```

### Node conditions — what each means

| Condition | Status False = Problem | Cause |
|-----------|----------------------|-------|
| `Ready` | Node is not ready | kubelet not running, network issue |
| `MemoryPressure` | Status True = low memory | Too many pods consuming RAM |
| `DiskPressure` | Status True = low disk | Container images / logs filling disk |
| `PIDPressure` | Status True = too many processes | Process limit hit |
| `NetworkUnavailable` | Status True = no network | CNI plugin not working |

### Cordon and drain — safe node maintenance

```bash
# Cordon — mark node as unschedulable (no new pods scheduled)
kubectl cordon node1
# Node node1 cordoned

# Drain — evict all pods from node (for maintenance)
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data
# --ignore-daemonsets: DaemonSet pods can't be evicted (they'll restart anyway)
# --delete-emptydir-data: allows draining pods using emptyDir volumes

# After maintenance — uncordon to allow scheduling again
kubectl uncordon node1
```

> **Exam tip**: If drain hangs, it's usually because a pod has no ReplicaSet controller. Add `--force` to forcibly delete it.

---

## 2. Troubleshoot Cluster Components — static pods

Control plane components run as **static pods** managed by kubelet. Their YAML manifests live in:
```
/etc/kubernetes/manifests/
├── kube-apiserver.yaml
├── kube-controller-manager.yaml
├── kube-scheduler.yaml
└── etcd.yaml
```

Kubelet watches this directory. If you **edit** a manifest → kubelet automatically restarts that component within ~30 seconds.

### Diagnose a broken component

```bash
# Are control plane pods running?
kubectl get pods -n kube-system
# kube-apiserver-cp            1/1     Running   ← good
# kube-scheduler-cp            0/1     CrashLoop ← broken
# kube-controller-manager-cp   1/1     Running

# Get details
kubectl describe pod kube-scheduler-cp -n kube-system
# Read the Events: section

# Get logs from static pod
kubectl logs kube-scheduler-cp -n kube-system
```

### If API server is down (kubectl won't work)

Use crictl directly on the control plane node:
```bash
# List all running containers
crictl ps

# Get logs from static pod container directly
crictl logs <container-id>

# If kube-apiserver pod isn't even starting, check the manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml
# Look for typos in flags, wrong cert paths, wrong etcd endpoints
```

Also check Docker/containerd directly:
```bash
# With containerd
crictl pods   # Show pods managed by kubelet

# Check kubelet's own log for static pod errors
journalctl -u kubelet -n 100 --no-pager | grep -i error
```

### Fix a broken static pod — example: wrong flag

Scenario: `kube-scheduler` keeps crashing because of a typo in a flag.
```bash
# SSH to control plane node
ssh cp-node

# Edit the manifest
vi /etc/kubernetes/manifests/kube-scheduler.yaml
# Fix the typo in the command args

# Wait ~30 seconds — kubelet restarts the pod automatically
kubectl get pods -n kube-system -w
```

### etcd troubleshooting

```bash
# Check etcd pod
kubectl get pod etcd-cp -n kube-system
kubectl describe pod etcd-cp -n kube-system

# Exec into etcd and check health (from control plane node)
kubectl exec -it etcd-cp -n kube-system -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# List cluster members
kubectl exec -it etcd-cp -n kube-system -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

### Component locations summary

| Component | Type | Config location | Log command |
|-----------|------|----------------|-------------|
| kube-apiserver | Static pod | `/etc/kubernetes/manifests/kube-apiserver.yaml` | `kubectl logs kube-apiserver-cp -n kube-system` |
| kube-scheduler | Static pod | `/etc/kubernetes/manifests/kube-scheduler.yaml` | `kubectl logs kube-scheduler-cp -n kube-system` |
| kube-controller-manager | Static pod | `/etc/kubernetes/manifests/kube-controller-manager.yaml` | `kubectl logs kube-controller-manager-cp -n kube-system` |
| etcd | Static pod | `/etc/kubernetes/manifests/etcd.yaml` | `kubectl logs etcd-cp -n kube-system` |
| kubelet | systemd service | `/var/lib/kubelet/config.yaml` | `journalctl -u kubelet` |
| kube-proxy | DaemonSet | ConfigMap `kube-proxy` in `kube-system` | `kubectl logs -l k8s-app=kube-proxy -n kube-system` |
| CoreDNS | Deployment | ConfigMap `coredns` in `kube-system` | `kubectl logs -l k8s-app=kube-dns -n kube-system` |

---

## 3. Monitor Resource Usage — `kubectl top`

`kubectl top` requires **Metrics Server** to be installed in the cluster.

```bash
# CPU and memory usage per node
kubectl top nodes
# NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# cp       213m         10%    1804Mi          47%
# node1    56m          2%     942Mi           24%

# CPU and memory usage per pod (default namespace)
kubectl top pods
kubectl top pods -n kube-system

# Sort by CPU or memory
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory

# Show containers inside each pod
kubectl top pods --containers

# Watch top (repeat every 2 seconds — not built in, use watch)
watch kubectl top pods
```

### If `kubectl top` returns "error: Metrics API not available"

Metrics Server is not installed:
```bash
# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Killercoda (kubeadm): patch to allow insecure TLS — required or metrics-server stays Pending
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Wait for it to be ready
kubectl -n kube-system rollout status deployment metrics-server

# Test it
kubectl top nodes
```

### Resource pressure — how to find memory hogs

```bash
# Which pods are consuming the most memory?
kubectl top pods -A --sort-by=memory | head -10

# Which node is under pressure?
kubectl top nodes

# Describe node to see what's consuming resources
kubectl describe node node1 | grep -A 20 "Allocated resources"
```

### Events — the most useful troubleshooting tool

```bash
# All events across all namespaces (sorted by time)
kubectl get events -A --sort-by=.metadata.creationTimestamp

# Events in a specific namespace
kubectl get events -n production

# Watch events in real time
kubectl get events -n default -w

# Filter for warning events only
kubectl get events -A --field-selector type=Warning
```

---

## 4. Container Output Streams — logs management

### Kubernetes log fundamentals

Every container writes to **stdout** and **stderr**. Kubernetes collects these via the container runtime and makes them accessible via `kubectl logs`. Logs are stored on the node at:
```
/var/log/pods/<namespace>_<pod-name>_<uid>/<container-name>/0.log
```

### kubectl logs — complete reference

```bash
# Current logs
kubectl logs <pod-name>
kubectl logs <pod-name> -n <namespace>

# Previous container's logs (after a crash — most useful for CrashLoopBackOff)
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -p    # shorthand

# Specific container in a multi-container pod
kubectl logs <pod-name> -c <container-name>

# Stream logs in real time (like tail -f)
kubectl logs <pod-name> -f
kubectl logs <pod-name> --follow

# Show only last N lines
kubectl logs <pod-name> --tail=50

# Show logs since a time period
kubectl logs <pod-name> --since=1h
kubectl logs <pod-name> --since=30m

# Combine: stream last 20 lines of previous crash
kubectl logs <pod-name> --previous --tail=20

# Logs from ALL pods matching a label (DaemonSet, Deployment)
kubectl logs -l app=nginx
kubectl logs -l app=nginx --all-containers
```

### View logs directly on the node

When `kubectl logs` doesn't work (e.g. API server is down):
```bash
# SSH to the node where the pod is running
# Find the pod's log directory
ls /var/log/pods/
# kube-system_kube-apiserver-cp_<uid>/
# default_my-app-xxxxx_<uid>/

# Read the log file directly
cat /var/log/pods/default_my-app-xxxxx_uid/app/0.log

# Or use crictl
crictl logs <container-id>
crictl ps                  # find container ID
```

### Application log patterns — what to look for

```bash
# CrashLoopBackOff — look for the crash reason
kubectl logs <pod-name> --previous
# Common output:
# Error: could not connect to database — missing env var / wrong service name
# OOMKilled — not in logs (check describe pod → "Last State: OOMKilled")
# Permission denied — wrong file permissions / security context
# exec: executable not found — wrong image or entrypoint

# OOMKilled — memory exceeded (won't be in logs)
kubectl describe pod <pod-name>
# Look for: "Last State: Terminated, Reason: OOMKilled, Exit Code: 137"
# Fix: increase memory limits or fix memory leak

# ImagePullBackOff — image not found or no pull credentials
kubectl describe pod <pod-name>
# Events: "Failed to pull image: Error response from daemon: manifest not found"
# Fix: check image name, tag, registry credentials (imagePullSecrets)
```

### Common exit codes

| Exit Code | Meaning | Usual cause |
|-----------|---------|------------|
| 0 | Success | Normal completion |
| 1 | General error | App crashed, unhandled exception |
| 137 | SIGKILL (128+9) | OOMKilled or force-killed |
| 143 | SIGTERM (128+15) | Graceful shutdown (then forced) |
| 126 | Command not executable | Wrong entrypoint / permission |
| 127 | Command not found | Missing binary in image |

---

## 5. Troubleshoot Services and Networking

### Service not routing traffic — systematic approach

```bash
# Step 1: Does the service exist?
kubectl get svc <service-name>
kubectl describe svc <service-name>
# Look for: selector, ports, endpoints

# Step 2: Are there endpoints?
kubectl get endpoints <service-name>
# NAME          ENDPOINTS             AGE
# web-service   10.244.1.5:80,10.244.1.6:80   ← Good, pods are matched
# web-service   <none>                          ← BAD — no pods match the selector

# Step 3: If <none> endpoints → check selector vs pod labels
kubectl get pods --show-labels
kubectl describe svc <service-name>   # "Selector: app=web"
# Common mistake: pod label is "app=nginx", service selector is "app=web" → mismatch

# Step 4: Test connectivity from inside cluster
kubectl run test-box --rm -it --image=busybox -- /bin/sh
# Inside the shell:
wget -qO- http://web-service.default.svc.cluster.local
wget -qO- http://web-service          # short DNS form
wget -qO- http://10.244.1.5:80        # direct pod IP
```

### DNS troubleshooting

```bash
# Test DNS resolution from inside the cluster
kubectl run dns-test --rm -it --image=busybox:1.28 -- nslookup kubernetes
# If this fails → CoreDNS is the problem

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -l k8s-app=kube-dns -n kube-system

# Check CoreDNS ConfigMap (the Corefile)
kubectl describe configmap coredns -n kube-system

# Expected DNS name format:
# <service>.<namespace>.svc.cluster.local
# Example: mysql.production.svc.cluster.local

# DNS resolution for pods:
# <pod-ip-dashes>.<namespace>.pod.cluster.local
# Example: 10-244-1-5.default.pod.cluster.local
```

### kube-proxy — iptables rules for service routing

kube-proxy runs as a DaemonSet and programs iptables/ipvs rules so traffic to a Service ClusterIP is forwarded to actual Pod IPs.

```bash
# Check kube-proxy is running on all nodes
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# View kube-proxy logs
kubectl logs -l k8s-app=kube-proxy -n kube-system

# On the node — check iptables rules for a service
iptables -t nat -L -n | grep <service-cluster-ip>
# If no rules exist for a service IP → kube-proxy isn't creating them

# kube-proxy config
kubectl describe configmap kube-proxy -n kube-system
```

### NetworkPolicy blocking traffic

When a NetworkPolicy exists in a namespace, it starts blocking. If traffic that worked before suddenly stops:
```bash
# Check if NetworkPolicies exist
kubectl get networkpolicy -n <namespace>

# Describe to see the rules
kubectl describe networkpolicy <name> -n <namespace>
# Check ingress/egress rules — are they allowing the port/pod you expect?

# Test connectivity with netcat
kubectl exec -it <source-pod> -- nc -zv <target-pod-ip> 80
# Connection to 10.244.1.5 80 port [tcp/http] succeeded!  ← open
# nc: connect to 10.244.1.5 port 80 failed: Connection refused  ← blocked

# Temporary: delete the NetworkPolicy to confirm it's the cause
kubectl delete networkpolicy <name> -n <namespace>
```

### NodePort service not accessible from outside

```bash
# 1. Verify NodePort is set
kubectl get svc web-svc
# TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# NodePort   10.99.xx.xx    <none>        80:30080/TCP   ← 30080 is the NodePort

# 2. Get node IP
kubectl get nodes -o wide
# INTERNAL-IP: 192.168.56.101

# 3. Test from outside
curl http://192.168.56.101:30080

# 4. If blocked — check firewall on the node
ufw status
iptables -L -n | grep 30080
```

---

## Troubleshooting recipes — common CKA exam scenarios

### Scenario: Pod stuck in `Pending`

```bash
kubectl describe pod <pod-name>
# Events: "0/2 nodes are available: 1 Insufficient cpu, 1 node(s) had untolerated taint"
```

Causes and fixes:

| Symptom in Events | Cause | Fix |
|------------------|-------|-----|
| `Insufficient cpu/memory` | No node has enough resources | Scale cluster or reduce pod requests |
| `node(s) had untolerated taint` | Node is tainted, pod has no toleration | Add toleration or remove taint |
| `node(s) didn't match node selector` | nodeSelector or affinity doesn't match | Fix labels on node or affinity in pod |
| `pod has unbound PVCs` | PVC not bound (no matching PV) | Create PV or fix StorageClass |

### Scenario: Pod in `CrashLoopBackOff`

```bash
kubectl describe pod <pod-name>  # see exit code
kubectl logs <pod-name> --previous   # see crash output
```

Typical fixes:
- **Exit code 1** → app crashed, read logs for the error message
- **Exit code 137 (OOMKilled)** → increase `resources.limits.memory`
- **"exec: not found"** → wrong image entrypoint, check `command:` in YAML
- **"can't connect to DB"** → wrong Service name, wrong port, or NetworkPolicy blocking

### Scenario: Deployment rollout stuck

```bash
kubectl rollout status deployment/my-app
# Waiting for deployment "my-app" rollout to finish: 0 of 3 updated replicas are available...

kubectl describe deployment my-app
# Look at: NewReplicaSet, OldReplicaSet, Events

kubectl get pods -l app=my-app   # See if new pods are crashing
kubectl logs <new-pod-name> --previous   # Find the crash cause
```

Common causes:
- New image doesn't exist → ImagePullBackOff
- New image crashes on start → CrashLoopBackOff
- Readiness probe failing (pod starts but health check fails → never becomes Ready)

### Scenario: Service has no endpoints

```bash
kubectl get endpoints web-svc
# ENDPOINTS: <none>

# Check: does the Service selector match pod labels?
kubectl get pods --show-labels
# my-pod: app=nginx-v2
kubectl describe svc web-svc
# Selector: app=nginx     ← mismatch! service looks for "nginx", pod has "nginx-v2"

# Fix: update the service selector
kubectl edit svc web-svc
# Change: selector.app: nginx → nginx-v2
```

### Scenario: DNS resolution failing inside a pod

```bash
# Test DNS
kubectl exec -it my-pod -- nslookup my-svc.default.svc.cluster.local
# Server: 10.96.0.10   (CoreDNS ClusterIP)
# ** server can't find my-svc.default.svc.cluster.local: NXDOMAIN

# NXDOMAIN = service doesn't exist OR wrong namespace
# Check: is the service in the same namespace?
kubectl get svc -n default   # vs the pod's namespace

# CoreDNS problem → check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

---

## One-line takeaways
- `kubectl describe` → always read the `Events:` section at the bottom — it tells you WHY something failed.
- Node `NotReady` → SSH to node → `systemctl status kubelet` → `journalctl -u kubelet`.
- Control plane broken → edit `/etc/kubernetes/manifests/` YAML → kubelet restarts it automatically.
- `kubectl logs --previous` → shows logs from a crashed container, not the current one.
- Service `<none>` endpoints → **selector mismatch** is the #1 cause — check service selector vs pod labels.
- `kubectl top` needs Metrics Server — if it's missing, install it first.
- Pod `Pending` → `describe` the pod → read Events → usually: resources, taint/toleration, or PVC issue.
