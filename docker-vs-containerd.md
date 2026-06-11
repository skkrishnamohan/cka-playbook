# Docker vs containerd — What’s the difference?

This note compares Docker and containerd, explains why containerd is used by Kubernetes, and shows equivalent commands for common container tasks.

---

## What is Docker?

- Docker is a container platform that includes several components:
  - `dockerd`: the Docker daemon
  - Docker CLI (`docker`)
  - image builder, image registry support, container runtime, networking, volume management, and a higher-level user experience
- Docker is often used for local development and building/running containers on a laptop or server.
- It started as a complete container solution with a simple command-line interface.

## What is containerd?

- containerd is a lightweight container runtime that focuses on running containers.
- It handles:
  - pulling images
  - storing images
  - creating and running containers
  - managing container lifecycle
- containerd is a lower-level runtime used by Kubernetes and other orchestrators.
- It does not provide the full Docker CLI experience.

## What is CRI-O?

- CRI-O is a Kubernetes container runtime implementation built specifically for the Container Runtime Interface (CRI).
- It runs containers directly from OCI images and is designed to be a lightweight, Kubernetes-native runtime.
- It does not include Docker build or Docker CLI, and it is not intended as a general developer tool.
- CRI-O is often used as an alternative to containerd in Kubernetes clusters.
- CRI-O uses `crictl` for CLI debugging, rather than a Docker-like command set.

## Why Kubernetes prefers containerd and CRI-O

- Kubernetes needs a runtime that supports CRI, not the full Docker platform.
- Both containerd and CRI-O are lightweight, modular, and focus on running containers.
- containerd and CRI-O can both be used without `dockershim`.
- Kubernetes removed `dockershim` support so Docker Engine is no longer a direct runtime option.

## High-level comparison

| Feature | Docker | containerd | CRI-O |
|--------|--------|------------|-------|
| Primary role | Full container platform | Container runtime only | Kubernetes-focused container runtime |
| CLI included | Yes (`docker`) | No built-in Docker CLI | No built-in CLI (use `crictl`) |
| Kubernetes support | Via dockershim (deprecated) | Native CRI support | Native CRI support |
| Image management | Yes | Yes | Yes |
| Container lifecycle | Yes | Yes | Yes |
| Build support | Yes (`docker build`) | No native build support | No native build support |
| Complexity | Higher | Lower | Lower |
| Recommended for Kubernetes | No (deprecated) | Yes | Yes |

## Typical command comparisons

### Pull an image

Docker:
```bash
docker pull nginx:1.28
```

containerd (ctr):
```bash
ctr image pull docker.io/library/nginx:1.28
```

### List images

Docker:
```bash
docker images
```

containerd:
```bash
ctr images list
```

### Run a container

Docker:
```bash
docker run --rm --name test-nginx -d nginx:1.28
```

containerd:
```bash
ctr run -d --rm docker.io/library/nginx:1.28 test-nginx nginx
```

### List running containers

Docker:
```bash
docker ps
```

containerd:
```bash
ctr containers list
```

### Stop a container

Docker:
```bash
docker stop test-nginx
```

containerd:
```bash
ctr tasks kill test-nginx
```

### Remove a container

Docker:
```bash
docker rm test-nginx
```

containerd:
```bash
ctr containers delete test-nginx
```

### CRI-O / crictl command examples

CRI-O usually uses `crictl` to interact with containers and images.

Pull an image:
```bash
crictl pull docker.io/library/nginx:1.28
```

List images:
```bash
crictl images
```

List running containers:
```bash
crictl ps
```

Stop a container:
```bash
crictl stop <container-id>
```

Remove a container:
```bash
crictl rm <container-id>
```

> Note: `crictl` works with any CRI runtime, including CRI-O and containerd.

## Notes on command differences

- containerd commands are more explicit and lower-level.
- Docker commands are designed for user convenience and include defaults.
- In Kubernetes, you rarely run `ctr` directly; kubelet interacts with containerd through CRI.
- If you need to debug a containerd-based node, learning `ctr` helps.

## When to use each

- Use Docker for local development, image building, and simple container tasks.
- Use containerd when running Kubernetes or any system that needs a lightweight runtime.
- For production Kubernetes clusters, containerd is the safer modern choice.

## Quick takeaway

- Docker = complete developer-facing platform.
- containerd = runtime engine for Kubernetes and lightweight deployments.
- Command shapes are similar in purpose, but Docker is higher-level and easier for humans, while `ctr` is lower-level and closer to the runtime.

---

## Architecture note: Docker Engine vs containerd vs CRI-O

### Docker Engine
- Includes the Docker daemon (`dockerd`), the Docker CLI, image build tools, and the container runtime.
- It is a full platform for developers: build, push, pull, run, and manage containers.
- For Kubernetes, Docker Engine used to be supported through `dockershim`, but that path is deprecated and removed.
- Best for local development, not as Kubernetes runtime in modern clusters.

### containerd
- Focused runtime that does the core container work: pulling images, creating containers, and managing lifecycle.
- Supports the Kubernetes Container Runtime Interface (CRI) directly.
- Used by kubelet to run pods in Kubernetes clusters.
- It is the modern default runtime for many Kubernetes installations.

### CRI-O
- Also a lightweight runtime, but built specifically for Kubernetes and CRI.
- It is intended to be minimal and Kubernetes-native.
- Uses `crictl` for debugging and CLI actions.
- A strong alternative to containerd when you want a runtime designed only for Kubernetes.

---

## Additional tools and concepts

### nerdctl — Docker-compatible CLI for containerd
- `nerdctl` is a Docker-compatible CLI for containerd (drop-in replacement for `docker` commands).
- Supports `nerdctl build`, `nerdctl run`, `nerdctl compose` — familiar Docker workflow without Docker daemon.
- Useful when you want Docker-like UX on a containerd-based system.
```bash
# Same as docker run
nerdctl run -d --name web nginx:1.28

# Same as docker build
nerdctl build -t myapp:1.0 .

# Same as docker compose up
nerdctl compose up -d
```

### crictl vs ctr — when to use which
- `ctr` = containerd's native low-level CLI. Useful for debugging containerd directly.
- `crictl` = CRI-compatible debugging tool. Works with ANY CRI runtime (containerd, CRI-O).
- **For Kubernetes troubleshooting, always prefer `crictl`** — it speaks the same language as kubelet.
```bash
# crictl (CRI-level, recommended for K8s debugging)
crictl pods              # List pods as kubelet sees them
crictl ps                # List containers
crictl logs <id>         # Container logs
crictl inspect <id>      # Container details

# ctr (containerd-native, lower-level)
ctr namespaces list      # containerd uses namespaces internally
ctr -n k8s.io containers list  # K8s containers are in k8s.io namespace
```

### containerd namespaces (important concept)
- containerd organizes containers into namespaces (different from Kubernetes namespaces).
- Kubernetes containers live in the `k8s.io` namespace inside containerd.
- Default `ctr` commands operate in the `default` namespace — add `-n k8s.io` to see K8s workloads:
```bash
ctr -n k8s.io images list
ctr -n k8s.io containers list
```

### How kubelet talks to the runtime (CRI flow)
```
kubelet → CRI (gRPC) → containerd/CRI-O → OCI runtime (runc) → container process
```
- kubelet never runs containers directly.
- CRI = Container Runtime Interface (a gRPC API standard).
- The actual low-level container creation is done by an OCI runtime like `runc`.

### Checking which runtime your cluster uses
```bash
# Check kubelet config for container runtime endpoint
kubectl get nodes -o wide   # Shows CONTAINER-RUNTIME column

# On a node, check kubelet args
ps aux | grep kubelet | grep container-runtime

# Check containerd socket
ls /run/containerd/containerd.sock

# Check CRI-O socket
ls /var/run/crio/crio.sock
```

### Image garbage collection
- kubelet automatically garbage collects unused images when disk pressure is detected.
- Thresholds: `--image-gc-high-threshold` (default 85%) and `--image-gc-low-threshold` (default 80%).
- You can also manually clean images:
```bash
crictl rmi --prune    # Remove unused images via CRI
ctr -n k8s.io images rm <image>  # Remove specific image in containerd
```

### How they fit together
- Docker Engine = full toolset for developers, includes an embedded runtime.
- containerd = runtime only, can run containers for Kubernetes or other orchestrators.
- CRI-O = runtime only, built specifically for Kubernetes CRI.

### Simple architecture comparison
- `docker` = developer command interface + runtime + build system.
- `containerd` = runtime engine that Kubernetes can use directly.
- `CRI-O` = Kubernetes-focused runtime engine that speaks CRI.

### Practical takeaway
- If you are learning Kubernetes architecture, think of Docker Engine as the full developer product, and containerd/CRI-O as the lightweight runtime engines used inside the cluster.
- Kubernetes clusters usually choose either containerd or CRI-O, not Docker Engine.
- The CLI and tooling differ, but the underlying goal is the same: run containers reliably.

---

## CNI — Container Network Interface

### What is CNI?

CNI (Container Network Interface) is a specification that defines how networking is set up for containers (pods) in Kubernetes.

Think of it like a plug-in standard:
- Kubernetes defines the interface ("a CNI plugin must do X").
- Third-party plugins (Calico, Flannel, Cilium, etc.) implement that interface.
- kubelet calls the CNI plugin when a pod is created/deleted to set up or tear down networking.

Without a CNI plugin:
- Pods get created but stay in `ContainerCreating` state forever.
- No IP address is assigned to the pod.
- Pods cannot communicate with each other.

```bash
# Symptom when no CNI is installed
kubectl get pods -A
# NAME                                 STATUS              
# coredns-...                          ContainerCreating   ← stuck, no IP
```

### Why CNI exists

Kubernetes itself only handles:
- Scheduling pods to nodes
- Telling kubelet to start containers

Kubernetes does NOT handle:
- Assigning IP addresses to pods
- Setting up routes between nodes so pods can talk to each other

CNI fills that gap. When kubelet creates a pod, it calls the CNI plugin to:
1. Assign an IP address to the pod
2. Set up routing rules so the pod can send/receive traffic
3. Clean up when the pod is deleted

---

## CNI Plugins — comparison

| Plugin | Key Feature | Best For |
|--------|------------|---------|
| **Flannel** | Simple overlay network (VXLAN) | Learning, small clusters |
| **Calico** | Network policies + routing (BGP) | Security, production |
| **Cilium** | eBPF-based, high performance | High-performance clusters, observability |
| **Multus** | Multiple network interfaces per pod | Multi-network setups, telco |
| **Weave** | Simple mesh networking | Small clusters |

**Calico** is the most common choice for production CKA-relevant setups — supports both networking and NetworkPolicy enforcement.

Install Calico example:
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Verify Calico pods are running
kubectl get pods -n calico-system
```

---

## Pod CIDR and IP Assignment

When you initialize a cluster with kubeadm, you specify the pod network CIDR:
```bash
kubeadm init \
  --apiserver-advertise-address=172.16.0.110 \
  --pod-network-cidr=192.168.0.0/16
```

### What is CIDR?

CIDR (Classless Inter-Domain Routing) is a way to define a range of IP addresses.

| CIDR | Total IPs | Typical Use |
|------|-----------|-------------|
| `/16` | 65,536 | Cluster-wide pod network pool |
| `/24` | 256 | Per-node pod subnet |
| `/22` | 1,024 | Larger per-node subnet |

### How pod IPs are assigned

The cluster's pod CIDR (`192.168.0.0/16`) is split into smaller subnets — one per node:

```
Cluster pod CIDR: 10.244.0.0/16

Node1 gets subnet: 10.244.0.0/24
  pod-a → 10.244.0.2
  pod-b → 10.244.0.3

Node2 gets subnet: 10.244.1.0/24
  pod-c → 10.244.1.2
  pod-d → 10.244.1.3
```

Each pod gets a unique IP from its node's subnet. The CNI plugin sets this up automatically.

```bash
# See pod IPs
kubectl get pods -o wide

# Output
NAME    READY  STATUS   NODE    IP
pod-a   1/1    Running  node1   10.244.0.2
pod-b   1/1    Running  node1   10.244.0.3
pod-c   1/1    Running  node2   10.244.1.2
```

---

## Pod-to-Pod Communication

A core Kubernetes requirement: **every pod must be able to reach every other pod, regardless of which node they are on**.

### Same-node communication
Pod-a (10.244.0.2) → Pod-b (10.244.0.3)

Traffic stays on the node, routed via a virtual bridge (like `cni0` or `flannel.1`). No external routing needed.

### Cross-node communication
Pod-a on Node1 (10.244.0.2) → Pod-c on Node2 (10.244.1.2)

The CNI plugin sets up routing rules (or an overlay tunnel like VXLAN) so Node1 knows to send traffic for `10.244.1.0/24` to Node2.

Test pod-to-pod connectivity:
```bash
# Test from pod-a to pod-b by IP
kubectl exec -it pod-a -- ping 10.244.0.3

# Test by pod name (if DNS works)
kubectl exec -it pod-a -- ping pod-b
```

### Enabling IP forwarding (required on Linux nodes)
For pods to send traffic to other nodes, the Linux kernel must allow IP forwarding:
```bash
sudo sysctl -w net.ipv4.ip_forward=1

# Make it permanent
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

This is typically set up automatically by the CNI plugin installer.

---

## CoreDNS — Service Discovery Inside the Cluster

### What is CoreDNS?

CoreDNS is the default DNS server in Kubernetes. It runs as pods in the `kube-system` namespace and provides DNS resolution inside the cluster.

```bash
kubectl get pods -n kube-system | grep coredns
# coredns-5dd5756b68-2xzv8   1/1   Running
# coredns-5dd5756b68-kgjqn   1/1   Running
```

### What problem does CoreDNS solve?

Without CoreDNS: Pods can only talk to each other by IP address. But pod IPs change every time a pod restarts.

With CoreDNS: Pods can use stable DNS names to find each other:
```bash
# Instead of: curl 10.244.1.5
curl nginx-service                              # within same namespace
curl nginx-service.default.svc.cluster.local   # fully qualified
```

### DNS name format

Kubernetes creates a DNS name for every Service:
```
<service-name>.<namespace>.svc.<cluster-domain>
```

Example:
- Service name: `nginx-service`
- Namespace: `default`
- Full DNS: `nginx-service.default.svc.cluster.local`

| What you type | What it resolves to |
|--------------|-------------------|
| `nginx-service` | Works within same namespace |
| `nginx-service.default` | Works from any namespace |
| `nginx-service.default.svc.cluster.local` | Full FQDN — always works |

### Verify DNS is working
```bash
# Run a temporary pod and test DNS
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup nginx-service
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup nginx-service.default.svc.cluster.local
```

### What CoreDNS does NOT do
- It does NOT assign IPs to pods (that's CNI's job)
- It does NOT route traffic (that's kube-proxy's job)
- It only resolves DNS names → IP addresses

### Summary: CNI vs CoreDNS vs kube-proxy

| Component | Job |
|-----------|-----|
| **CNI plugin** | Assign pod IPs + set up routing between nodes |
| **CoreDNS** | Resolve DNS names (service names → cluster IPs) |
| **kube-proxy** | Manage iptables/IPVS rules to forward traffic to pod IPs |