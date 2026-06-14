# CKA Exam Practice — Day 5: Cluster Operations 🔴

> **Difficulty**: Hard | **Questions**: 2 + Mock Exam Review | **Target Time**: ~60 minutes
> **Goal**: Master the highest-stakes operations — cluster upgrades and troubleshooting. These are multi-step, SSH-heavy tasks worth significant points. Finish with a full mock review.

---

## Question 19 — kubeadm Upgrade Master Node

| | |
|---|---|
| **Weightage** | ~10% (estimated) |
| **Difficulty** | 🔴 Hard |
| **CKA Domain** | Cluster Architecture, Installation & Configuration (25%) |
| **Estimated Exam Time** | ⏱️ 10–12 minutes |
| **Topics** | kubeadm upgrade, drain, uncordon, kubelet |
| **Related Notes** | [Create-Cluster.md](Create-Cluster.md) · [k8s-architecture.md](k8s-architecture.md) |

### 📋 Question

kubeadm upgrade version 1.30.1 on master node only.

---

### ⚙️ LAB SETUP (Skip in Exam)

The cluster already exists in the exam. For practice, use a kubeadm cluster on Killercoda.

---

### ✅ Full Solution with Explanations

> **IMPORTANT**: In the exam, the target version will be given. Follow the exact version they specify. This example uses `1.30.1`.

**Step 1 — Switch context and verify current version**

```bash
kubectl config use-context <context-name>
kubectl get nodes
```

> **WHY?** — Confirm you're on the right cluster and note the current Kubernetes version.

**Step 2 — SSH into the master node**

```bash
ssh controlplane
# If not root:
sudo -i
```

> **WHY sudo -i?** — kubeadm and package operations require root access.

**Step 3 — Drain the control plane node**

```bash
kubectl drain controlplane --ignore-daemonsets --delete-emptydir-data
```

> **WHY drain?**
> - Safely evicts all non-system pods from the node
> - Marks the node as `SchedulingDisabled` (cordoned)
> - `--ignore-daemonsets` — DaemonSet pods can't be evicted, so skip them
> - `--delete-emptydir-data` — Allow eviction of pods using emptyDir volumes (data will be lost)
>
> **WHY drain before upgrade?** — Ensures no workload disruption during the upgrade process. The control plane components restart during upgrade.

**Step 4 — Update the package repository and install the new kubeadm**

```bash
# Check available versions
apt-cache madison kubeadm | grep 1.30.1

# Update kubeadm to the target version
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.30.1-*
apt-mark hold kubeadm
```

> **WHY `apt-mark unhold/hold`?**
> - `unhold` — Allows the package to be updated (it was previously held to prevent accidental updates)
> - `hold` — Locks the package again after upgrade to prevent unintended version changes

> **Note**: The exam may have a specific package version format (e.g., `1.30.1-1.1`). Use `apt-cache madison kubeadm` to find the exact version string.

**Step 5 — Verify the new kubeadm version**

```bash
kubeadm version
```

**Step 6 — Check the upgrade plan**

```bash
kubeadm upgrade plan
```

> **WHY?** — Shows what version you can upgrade to and checks for compatibility issues. This is informational but good practice.

**Step 7 — Apply the upgrade**

```bash
kubeadm upgrade apply v1.30.1
```

> **WHY `apply` (not `node`)?** — On the FIRST control plane node, use `kubeadm upgrade apply`. On additional control plane nodes (in HA setups), use `kubeadm upgrade node`.

Wait for: `[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.30.1". Enjoy!`

**Step 8 — Upgrade kubelet and kubectl**

```bash
apt-mark unhold kubelet kubectl
apt-get update && apt-get install -y kubelet=1.30.1-* kubectl=1.30.1-*
apt-mark hold kubelet kubectl
```

> **WHY separately?** — kubeadm upgrades the control plane components (API server, controller-manager, scheduler, etcd). But kubelet and kubectl must be upgraded separately via package manager.

**Step 9 — Restart kubelet**

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

> **WHY daemon-reload?** — After upgrading kubelet, the systemd service file may have changed. `daemon-reload` tells systemd to re-read service files.

**Step 10 — Exit the node and uncordon**

```bash
exit    # Exit from the master node back to bastion

kubectl uncordon controlplane
```

> **WHY uncordon?** — The node was cordoned during drain. Uncordoning makes it schedulable again so new pods can be placed on it.

**Step 11 — Verify the upgrade**

```bash
kubectl get nodes
```

Expected:
```
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   1h    v1.30.1
```

---

### ⚡ Exam Speed Strategy (Target: 8 minutes)

```bash
# The upgrade docs have the EXACT commands — just follow them step by step
# Search keyword in docs: "kubeadm upgrade"

# 1. Drain
kubectl drain controlplane --ignore-daemonsets --delete-emptydir-data

# 2. SSH + become root
ssh controlplane
sudo -i

# 3. Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.30.1-*
apt-mark hold kubeadm

# 4. Apply upgrade
kubeadm upgrade apply v1.30.1

# 5. Upgrade kubelet + kubectl
apt-mark unhold kubelet kubectl
apt-get update && apt-get install -y kubelet=1.30.1-* kubectl=1.30.1-*
apt-mark hold kubelet kubectl

# 6. Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# 7. Exit and uncordon
exit
kubectl uncordon controlplane
kubectl get nodes
```

> **Memory trick**: **D-U-A-U-R-U** = Drain → Upgrade kubeadm → Apply → Upgrade kubelet → Restart → Uncordon

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| **kubeadm upgrade** ⭐ | https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/ |
| Version skew policy | https://kubernetes.io/docs/setup/release/version-skew-policy/ |
| **Search keyword** | `kubeadm upgrade` → follow the steps EXACTLY as documented |

> **Exam tip**: This is one of the few questions where you should have the docs page open and follow it step by step. The commands are all listed in order.

---

### 💡 Tips & Memory Aids

- 🧠 **Upgrade order matters**: kubeadm first → control plane → kubelet/kubectl → uncordon
- 🧠 **Master vs Worker**: Master uses `kubeadm upgrade apply`, Workers use `kubeadm upgrade node`
- ⚠️ **CRITICAL: Don't forget to drain BEFORE and uncordon AFTER**
- ⚠️ **CRITICAL: Don't forget `systemctl daemon-reload` before restarting kubelet**
- ⚠️ **Common mistake**: Upgrading kubelet before kubeadm → can cause version skew issues
- ⚠️ **Common mistake**: Staying on the master node after upgrade → exit back to bastion!
- 🔑 **Package version format**: `1.30.1-*` (the `*` matches any build number)
- 🔑 **If `apt-get` hangs**: The exam environment may use a mirror. Check `/etc/apt/sources.list.d/` for K8s repo config
- 📝 **Question says "master node only"** — do NOT upgrade worker nodes

---

### ✅ Verification Checklist

- [ ] `kubectl get nodes` shows master at the new version
- [ ] Master node status is `Ready` (not `SchedulingDisabled`)
- [ ] `kubeadm version` shows the upgraded version
- [ ] `kubelet --version` shows the upgraded version
- [ ] You have exited back to the bastion/student machine

---
---

## Question 22 — Troubleshoot Broken Cluster

| | |
|---|---|
| **Weightage** | ~10% (estimated) |
| **Difficulty** | 🔴 Hard |
| **CKA Domain** | Troubleshooting (30%) |
| **Estimated Exam Time** | ⏱️ 8–12 minutes |
| **Topics** | Static pods, etcd, kubelet, API server, cluster troubleshooting |
| **Related Notes** | [k8s-troubleshooting.md](k8s-troubleshooting.md) · [k8s-etc-kubernetes-deep-dive.md](k8s-etc-kubernetes-deep-dive.md) · [k8s-architecture.md](k8s-architecture.md) |

### 📋 Question

Kubernetes Cluster is not working. Some components are down after a cluster migration.
Trouble shoot the cluster and fix the cluster.

---

### ⚙️ LAB SETUP (Skip in Exam)

No setup — the broken cluster is provided in the exam.

---

### ✅ Full Solution with Explanations

> **KEY INSIGHT**: When a cluster is broken after migration, the issue is almost always in:
> 1. **etcd** — wrong data directory path, wrong certificates
> 2. **kube-apiserver** — wrong etcd endpoint, wrong cert paths
> 3. **kubelet** — stopped or misconfigured
>
> The control plane components run as **static pods** managed by kubelet from `/etc/kubernetes/manifests/`.

#### Systematic Troubleshooting Flowchart

```
Cluster Broken → Can't run kubectl?
│
├── YES → kubelet or API server issue
│   ├── Check kubelet: systemctl status kubelet
│   │   ├── Not running → systemctl start kubelet
│   │   └── Running → Check API server static pod
│   │       └── Check /etc/kubernetes/manifests/kube-apiserver.yaml
│   │           ├── Wrong etcd endpoint?
│   │           ├── Wrong cert paths?
│   │           └── Wrong image?
│   └── Check etcd: /etc/kubernetes/manifests/etcd.yaml
│       ├── Wrong data-dir path?
│       ├── Wrong cert paths?
│       └── Wrong listen URLs?
│
└── NO (kubectl works but partial failure)
    ├── kubectl get nodes → NotReady nodes?
    ├── kubectl get pods -n kube-system → Which components are down?
    └── kubectl logs <failing-pod> -n kube-system → Error details
```

---

**Step 1 — Try kubectl to assess the situation**

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

If kubectl fails with connection errors, the API server is down. Proceed to Step 2.

**Step 2 — Check kubelet status**

```bash
sudo systemctl status kubelet
```

> **WHY kubelet first?** — kubelet manages ALL control plane static pods. If kubelet is down, nothing works.

If kubelet is not running:
```bash
sudo systemctl start kubelet
sudo systemctl enable kubelet
```

If kubelet is running but API server is still down, check kubelet logs:
```bash
journalctl -u kubelet -f --no-pager | tail -50
```

> **Common kubelet errors**:
> - Wrong `--config` path → check `/var/lib/kubelet/config.yaml`
> - Wrong `--kubeconfig` path → check `/etc/kubernetes/kubelet.conf`
> - Certificate issues → check file permissions

**Step 3 — Check static pod manifests**

```bash
ls -la /etc/kubernetes/manifests/
```

Expected files:
```
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

> **WHY?** — Control plane components are static pods. Their manifests are in `/etc/kubernetes/manifests/`. kubelet watches this directory and automatically creates/restarts pods for any YAML files here.

**Step 4 — Check etcd.yaml for common issues**

```bash
cat /etc/kubernetes/manifests/etcd.yaml
```

Common issues after migration:

```yaml
# Check these fields:
spec:
  containers:
  - command:
    - etcd
    - --data-dir=/var/lib/etcd          # ← Is this path correct?
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt    # ← Do these files exist?
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --listen-client-urls=https://127.0.0.1:2379    # ← Correct IP?
    volumeMounts:
    - mountPath: /var/lib/etcd          # ← Must match --data-dir
      name: etcd-data
  volumes:
  - hostPath:
      path: /var/lib/etcd              # ← Must exist on the host
    name: etcd-data
```

> **Common etcd fix**: The `--data-dir` in the command doesn't match the `mountPath` or the `hostPath`. Ensure all three are consistent.

Verify cert files exist:
```bash
ls -la /etc/kubernetes/pki/etcd/
```

**Step 5 — Check kube-apiserver.yaml**

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

Common issues:
```yaml
    - --etcd-servers=https://127.0.0.1:2379     # ← Must match etcd's listen URL
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
```

> **Common API server fix**: Wrong `--etcd-servers` URL, or wrong certificate file paths.

**Step 6 — Fix the issue and restart kubelet**

After fixing the YAML file:
```bash
# kubelet auto-detects changes to static pod manifests
# But restarting kubelet ensures immediate effect:
sudo systemctl restart kubelet
```

**Step 7 — Verify the cluster is working**

```bash
# Wait 30-60 seconds for components to come up
kubectl get nodes
kubectl get pods -n kube-system
kubectl get componentstatuses    # deprecated but still works in some versions
```

All nodes should be `Ready` and all kube-system pods should be `Running`.

---

### ⚡ Exam Speed Strategy (Target: 5 minutes)

```bash
# Quick diagnosis flow:
kubectl get nodes 2>/dev/null || echo "API SERVER DOWN"

# If API server is down:
sudo systemctl status kubelet
sudo journalctl -u kubelet --no-pager | tail -20

# Check static pods
sudo cat /etc/kubernetes/manifests/etcd.yaml | grep -E "(data-dir|cert-file|listen)"
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -E "(etcd-servers|etcd-ca|etcd-cert)"

# Verify paths exist
ls /etc/kubernetes/pki/etcd/
ls /var/lib/etcd/

# Fix the YAML file (common: wrong data-dir or cert path)
sudo vi /etc/kubernetes/manifests/etcd.yaml

# Restart
sudo systemctl restart kubelet

# Wait and verify
sleep 30
kubectl get nodes
kubectl get pods -n kube-system
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| Troubleshoot Clusters | https://kubernetes.io/docs/tasks/debug/debug-cluster/ |
| Static Pods | https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/ |
| etcd configuration | https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/ |
| **Search keyword** | `troubleshoot cluster` or `static pods` |

---

### 💡 Tips & Memory Aids

- 🧠 **Top 3 things to check**: kubelet → etcd.yaml → kube-apiserver.yaml (in that order)
- 🧠 **Static pod path**: Always `/etc/kubernetes/manifests/` on kubeadm clusters
- 🧠 **etcd data path**: Usually `/var/lib/etcd` — check if it matches the YAML
- ⚠️ **Common exam scenario**: etcd `--data-dir` is changed to a wrong path during "migration"
- ⚠️ **Common exam scenario**: API server `--etcd-servers` pointing to wrong IP/port
- ⚠️ **Common exam scenario**: Cert file path typo (e.g., wrong directory name)
- 🔑 **Quick check**: `crictl ps` shows which containers are actually running (even if kubectl is down)
- 🔑 **Quick check**: `journalctl -u kubelet -f` shows real-time kubelet errors
- 📝 **kubelet auto-restarts static pods** when YAML files change — but restarting kubelet is faster

---

### ✅ Verification Checklist

- [ ] `kubectl get nodes` shows all nodes as `Ready`
- [ ] `kubectl get pods -n kube-system` shows all control plane pods `Running`
- [ ] API server responds to requests
- [ ] etcd is running and healthy
- [ ] kubelet is running and enabled

---
---

## 🏆 Full Mock Exam — 30-Minute Speed Run

Now that you've practiced all 22 questions over 4 days, simulate the exam environment:

### Setup
- Set a **90-minute timer** (target: finish 30 min early)
- Open **only** `kubernetes.io/docs` (no other resources)
- Use a Killercoda environment

### Recommended Question Order (By Speed)

| Order | Question | Est. Time | Notes |
|-------|----------|-----------|-------|
| 1 | Q17 — Count nodes | 1 min | Quick `echo` command |
| 2 | Q18 — Fix NotReady | 2 min | SSH + systemctl |
| 3 | Q5 — StorageClass | 2 min | Simple YAML |
| 4 | Q12 — CRD list | 2 min | grep + jsonpath |
| 5 | Q6 — Helm template | 3 min | 3 commands |
| 6 | Q9 — CNI install | 2 min | One kubectl apply |
| 7 | Q2 — NodePort service | 3 min | kubectl expose |
| 8 | Q16 — NetworkPolicy | 3 min | Template-based YAML |
| 9 | Q1 — HPA | 3 min | autoscale + edit |
| 10 | Q21 — Ingress | 3 min | Template YAML |
| 11 | Q3 — Sidecar | 4 min | kubectl edit |
| 12 | Q13 — Choose policy | 3 min | Read + apply |
| 13 | Q20 — TLS config | 3 min | Edit ConfigMap + restart |
| 14 | Q10 — Resources | 5 min | Calculate + edit |
| 15 | Q14 — CoreDNS | 4 min | Edit ConfigMap |
| 16 | Q4 — PriorityClass | 4 min | Create + patch |
| 17 | Q7 — MariaDB restore | 5 min | PVC + deployment |
| 18 | Q11 — cri-dockerd | 3 min | dpkg + systemctl |
| 19 | Q15 — crictl | 5 min | SSH + crictl |
| 20 | Q8 — Gateway API | 6 min | Gateway + HTTPRoute |
| 21 | Q19 — kubeadm upgrade | 8 min | Follow docs step by step |
| 22 | Q22 — Fix cluster | 5 min | Systematic troubleshooting |
| | **TOTAL** | **~80 min** | **40 min buffer!** |

### Speed Tips for Exam Day

1. **Aliases** (set these at the start):
   ```bash
   alias k=kubectl
   alias kn='kubectl config set-context --current --namespace'
   export do='--dry-run=client -o yaml'
   ```

2. **vi settings** (paste-friendly):
   ```bash
   echo 'set tabstop=2 shiftwidth=2 expandtab' >> ~/.vimrc
   ```

3. **Context switching** — ALWAYS first command for each question:
   ```bash
   kubectl config use-context <context-name>
   ```

4. **Namespace** — Add `-n <namespace>` to EVERY command

5. **Don't debug, just verify**:
   - Did the resource get created? → `kubectl get`
   - Is it in the right namespace? → `kubectl get -n <ns>`
   - Is it configured correctly? → `kubectl describe`

---

## 📊 Day 5 Summary

| Question | Topic | Target Time | Key Action |
|----------|-------|-------------|------------|
| Q19 | kubeadm upgrade | 8 min | Follow docs: drain → upgrade → uncordon |
| Q22 | Cluster troubleshooting | 5 min | Check kubelet → etcd → API server |
| Mock Review | All 22 questions | — | Timed speed run |
| **Total** | | **~60 min** | |

> 💪 **You've now practiced ALL 22 questions. On exam day, you'll recognize every question type and solve it from muscle memory!**

---

## 🎯 Final Exam Day Checklist

- [ ] Set vim/vi preferences first
- [ ] Set aliases (`k`, `kn`, `do`)
- [ ] Do easy questions first (Q17, Q18, Q5, Q12, Q6)
- [ ] Always switch context before each question
- [ ] Always verify after each answer
- [ ] Skip hard questions on first pass — come back later
- [ ] Watch the clock — 2 hours goes fast!
- [ ] Don't forget to `exit` after SSH-ing into nodes
- [ ] Double-check namespaces on every resource

---

**Previous**: [Day 4 — Advanced 🔴](exam-day4-advanced.md) | **Back to**: [Index](exam-practice-index.md)

---

> **Good luck on the exam! 🚀 You've got this!**
