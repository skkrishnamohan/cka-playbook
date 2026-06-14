# CKA Exam Practice — Day 1: Warmup 🟢

> **Difficulty**: Easy | **Questions**: 5 | **Target Time**: ~45 minutes
> **Goal**: Build confidence with quick-win questions. Master these first — they're free points on exam day.

---

## Question 5 — Create Default StorageClass

| | |
|---|---|
| **Weightage** | 8% |
| **Difficulty** | 🟢 Easy |
| **CKA Domain** | Storage (10%) |
| **Estimated Exam Time** | ⏱️ 3–4 minutes |
| **Topics** | StorageClass, volumeBindingMode, default annotation |
| **Related Notes** | [k8s-storage-pv-pvc.md](k8s-storage-pv-pvc.md) |

### 📋 Question

First, create a new StorageClass named local-path for an existing provisioner named rancher.io/local-path. Set the volume binding mode to WaitForFirstConsumer.

Note - Not setting the volume binding mode or setting it to anything other than WaitForFirstConsumer may result in reduced score. Next, configure the StorageClass local-path as the default StorageClass.

Note - Do not modify any existing Deployments or PersistentVolumeClaims. Failure to do so may result in a reduced score.

---

### ⚙️ LAB SETUP (Skip in Exam)

No lab setup needed — this question creates everything from scratch.

---

### ✅ Full Solution with Explanations

**Step 1 — Find the StorageClass YAML in docs**

Search keyword in docs: **"StorageClass"**

```bash
# Check if any existing StorageClass is set as default
kubectl get storageclass
```

> **WHY?** — You need to know if another StorageClass is currently the default. If yes, you must remove its default annotation first, otherwise you'll have two defaults.

**Step 2 — Create the StorageClass YAML**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
```

> **WHY each field:**
> - `provisioner: rancher.io/local-path` — The provisioner that handles volume creation. Given in the question.
> - `volumeBindingMode: WaitForFirstConsumer` — Delays PV binding until a Pod actually needs it. This ensures the PV is created on the same node as the Pod. **Critical for scoring.**
> - `storageclass.kubernetes.io/is-default-class: "true"` — Makes this the default StorageClass. Any PVC without an explicit `storageClassName` will use this.

**Step 3 — Apply and verify**

```bash
kubectl apply -f storageclass.yaml

# Verify — look for "(default)" next to the name
kubectl get storageclass
```

Expected output:
```
NAME                   PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path  Delete          WaitForFirstConsumer   false                  5s
```

**Step 4 — If another StorageClass was already default, remove its annotation**

```bash
kubectl patch storageclass <old-default-name> -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'
```

> **WHY?** — Kubernetes allows multiple StorageClasses to be marked as default, but it causes confusion. The question says "configure as default" — ensure only one has the annotation.

---

### ⚡ Exam Speed Strategy (Target: 2 minutes)

```bash
# One-liner: search docs for "StorageClass", copy the YAML template, modify 3 fields
# The YAML is only 6 lines — just type it directly into vi

cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
EOF

# Verify
kubectl get sc
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| StorageClass docs | https://kubernetes.io/docs/concepts/storage/storage-classes/ |
| Default StorageClass | https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/ |
| **Search keyword** | `StorageClass` or `default storage class` |

---

### 💡 Tips & Memory Aids

- 🧠 **Remember**: `volumeBindingMode` has only 2 values: `Immediate` (default) and `WaitForFirstConsumer`
- ⚠️ **Common mistake**: Forgetting the annotation to make it default — the question explicitly asks for this
- ⚠️ **Common mistake**: Setting `volumeBindingMode: Immediate` or omitting it — the question warns this reduces score
- 🔑 **Shortcut**: `kubectl get sc` is the short form for `kubectl get storageclass`
- 📝 **No `kubectl create storageclass` command exists** — you must write the YAML

---

### ✅ Verification Checklist

- [ ] `kubectl get sc` shows `local-path (default)` with `WaitForFirstConsumer`
- [ ] No other StorageClass is marked as default
- [ ] No existing Deployments or PVCs were modified

---
---

## Question 6 — Helm Template ArgoCD

| | |
|---|---|
| **Weightage** | 7% |
| **Difficulty** | 🟢 Easy |
| **CKA Domain** | Cluster Architecture, Installation & Configuration (25%) |
| **Estimated Exam Time** | ⏱️ 4–5 minutes |
| **Topics** | Helm repos, helm template, CRDs, ArgoCD |
| **Related Notes** | [k8s-helm.md](k8s-helm.md) |

### 📋 Question

Install ArgoCD in the cluster:

Add the official Argo CD Helm repository with the name argo.

Generate a template of the ArgoCD Helm Chart version 7.7.3 for the argocd namespace, save to ~/argo-helm.yaml.

Configure the chart not to install CRDs.

Note — Argo CD CRDs already pre-installed.

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Install Helm if not available
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install ArgoCD CRDs (pre-installed in exam)
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Verify CRDs exist
kubectl get crd | grep argo
```

---

### ✅ Full Solution with Explanations

**Step 1 — Add the Argo Helm repository**

```bash
helm repo add argo https://argoproj.github.io/argo-helm
```

> **WHY?** — Helm needs to know where to find charts. The official ArgoCD Helm repo URL is `https://argoproj.github.io/argo-helm`. The name `argo` is specified in the question.

**Step 2 — Update the repository index**

```bash
helm repo update
```

> **WHY?** — After adding a repo, you need to fetch the latest chart index so Helm knows what versions are available.

**Step 3 — Verify the repo was added**

```bash
helm repo list
```

Expected output:
```
NAME    URL
argo    https://argoproj.github.io/argo-helm
```

**Step 4 — Generate the template with `helm template`**

```bash
helm template argocd argo/argo-cd \
  --version 7.7.3 \
  --namespace argocd \
  --set crds.install=false > ~/argo-helm.yaml
```

> **WHY each flag:**
> - `argocd` — the release name (you choose this, but use something descriptive)
> - `argo/argo-cd` — chart name in format `repo/chart`
> - `--version 7.7.3` — specific chart version as required
> - `--namespace argocd` — sets the namespace in the rendered YAML
> - `--set crds.install=false` — **CRITICAL**: disables CRD installation since they're already pre-installed
> - `> ~/argo-helm.yaml` — redirects output to the required file path

> **KEY DIFFERENCE**: `helm template` renders the chart YAML locally without installing anything to the cluster. It's like a dry-run that outputs the complete YAML.

**Step 5 — Verify the output**

```bash
# Check the file was created and has content
ls -la ~/argo-helm.yaml

# Verify namespace is set correctly
grep "namespace: argocd" ~/argo-helm.yaml | head -3

# Verify CRDs are NOT included
grep "kind: CustomResourceDefinition" ~/argo-helm.yaml
# Should return nothing — CRDs were excluded
```

---

### ⚡ Exam Speed Strategy (Target: 3 minutes)

```bash
# 3 commands, that's it:
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm template argocd argo/argo-cd --version 7.7.3 --namespace argocd --set crds.install=false > ~/argo-helm.yaml
```

> **Memory trick**: Think **"Add → Update → Template"** (A-U-T)

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| Helm template docs | https://helm.sh/docs/helm/helm_template/ |
| ArgoCD Helm chart | https://github.com/argoproj/argo-helm |
| **Search keyword** | `helm` in the K8s docs → Helm section |

---

### 💡 Tips & Memory Aids

- 🧠 **`helm template` vs `helm install`**: Template only renders YAML, install actually deploys to cluster. The question says "generate a template" — use `helm template`
- ⚠️ **Common mistake**: Forgetting `--set crds.install=false` — this is the key requirement
- ⚠️ **Common mistake**: Using `helm install` instead of `helm template` — read the question carefully
- 🔑 **How to find the `--set` value**: Run `helm show values argo/argo-cd | grep -i crd` to see available CRD options
- 📝 **Output path**: The `~` in `~/argo-helm.yaml` resolves to `/root/argo-helm.yaml` on exam machines (you're root)

---

### ✅ Verification Checklist

- [ ] `helm repo list` shows `argo` repo
- [ ] `~/argo-helm.yaml` exists and has content (should be ~96KB)
- [ ] `grep "namespace: argocd"` returns matches
- [ ] `grep "CustomResourceDefinition"` returns nothing (CRDs excluded)

---
---

## Question 12 — List Cert-Manager CRDs

| | |
|---|---|
| **Weightage** | 7% |
| **Difficulty** | 🟢 Easy |
| **CKA Domain** | Cluster Architecture, Installation & Configuration (25%) |
| **Estimated Exam Time** | ⏱️ 3–4 minutes |
| **Topics** | CRDs, jsonpath, cert-manager |
| **Related Notes** | [k8s-api-overview.md](k8s-api-overview.md) |

### 📋 Question

List all custom CRDs from cert manager and store it in custom-crd.txt.

Get the subject field from the cert manager and store it in cert-manager-subject.txt.

---

### ⚙️ LAB SETUP (Skip in Exam)

```bash
# Install cert-manager CRDs (pre-installed in exam)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.crds.yaml
```

---

### ✅ Full Solution with Explanations

**Step 1 — List all cert-manager CRDs and save to file**

```bash
kubectl get crd | grep cert-manager > custom-crd.txt
```

> **WHY?** — `kubectl get crd` lists ALL CustomResourceDefinitions in the cluster. We pipe through `grep cert-manager` to filter only cert-manager CRDs. The `>` redirects output to the required file.

**Step 2 — Verify the CRD list**

```bash
cat custom-crd.txt
```

Expected output:
```
certificaterequests.cert-manager.io     2025-04-17T17:46:07Z
certificates.cert-manager.io           2025-04-17T17:46:07Z
challenges.acme.cert-manager.io        2025-04-17T17:46:07Z
clusterissuers.cert-manager.io         2025-04-17T17:46:08Z
issuers.cert-manager.io                2025-04-17T17:46:08Z
orders.acme.cert-manager.io            2025-04-17T17:46:08Z
```

**Step 3 — Get the subject field from the certificates CRD**

```bash
kubectl get crd certificates.cert-manager.io \
  -o jsonpath='{.spec.versions[*].schema.openAPIV3Schema.properties.spec.properties.subject}' \
  > cert-manager-subject.txt
```

> **WHY this jsonpath?**
> - CRDs store their schema in `.spec.versions[*].schema.openAPIV3Schema`
> - We navigate through `properties.spec.properties.subject` to get the `subject` field definition
> - This outputs the JSON schema of the `subject` field

**Step 4 — Verify**

```bash
cat cert-manager-subject.txt
```

> **WHY jsonpath instead of `kubectl describe`?** — `describe` outputs human-readable text which is hard to extract specific fields from. `jsonpath` gives you the exact field in machine-readable format. For CRDs, the schema is deeply nested so jsonpath is the only practical way.

---

### ⚡ Exam Speed Strategy (Target: 2 minutes)

```bash
# Two commands:
kubectl get crd | grep cert-manager > custom-crd.txt
kubectl get crd certificates.cert-manager.io -o jsonpath='{.spec.versions[*].schema.openAPIV3Schema.properties.spec.properties.subject}' > cert-manager-subject.txt
```

> **Tip**: Copy the jsonpath from the docs or use `kubectl explain` to navigate the CRD structure.

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| CRD concepts | https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/ |
| JSONPath reference | https://kubernetes.io/docs/reference/kubectl/jsonpath/ |
| **Search keyword** | `custom resource definition` or `jsonpath` |

---

### 💡 Tips & Memory Aids

- 🧠 **JSONPath trick**: Always wrap jsonpath in single quotes: `-o jsonpath='{...}'`
- ⚠️ **Common mistake**: Using `grep cert` instead of `grep cert-manager` — `grep cert` might also match non-cert-manager CRDs
- 🔑 **Quick explore**: Use `kubectl get crd certificates.cert-manager.io -o json | head -50` to see the structure before crafting jsonpath
- 📝 **Remember the path**: `spec.versions[*].schema.openAPIV3Schema.properties.spec.properties.<field>`

---

### ✅ Verification Checklist

- [ ] `cat custom-crd.txt` shows 6 cert-manager CRDs
- [ ] `cat cert-manager-subject.txt` shows JSON output with the subject field schema
- [ ] Both files exist at the required paths

---
---

## Question 17 — Count Schedulable Nodes

| | |
|---|---|
| **Weightage** | ~4% (estimated) |
| **Difficulty** | 🟢 Easy |
| **CKA Domain** | Troubleshooting (30%) |
| **Estimated Exam Time** | ⏱️ 2–3 minutes |
| **Topics** | Taints, NoSchedule, node inspection |
| **Related Notes** | [k8s-scheduling-and-affinity.md](k8s-scheduling-and-affinity.md) · [k8s-troubleshooting.md](k8s-troubleshooting.md) |

### 📋 Question

Check the cluster, how many nodes are ready (not including nodes tainted as a NoSchedule / normal workloads ) and write the number to /opt/KUSC00402/kusc00402.txt.

---

### ⚙️ LAB SETUP (Skip in Exam)

No setup needed — just use the existing cluster.

---

### ✅ Full Solution with Explanations

**Step 1 — Get all nodes and their status**

```bash
kubectl get nodes
```

> **WHY?** — First see the overall picture: how many nodes exist, which are Ready/NotReady.

**Step 2 — Check taints on all nodes**

```bash
kubectl describe nodes | grep -A 2 "Taints:"
```

> **WHY?** — We need to exclude nodes with `NoSchedule` taint. The `grep -A 2` shows 2 lines after "Taints:" to capture the full taint info.

Alternative one-liner:
```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints[*].effect}{"\n"}{end}'
```

> **WHY this approach?** — This prints each node name with its taint effects in a clean table format, making it easy to count which ones have `NoSchedule`.

**Step 3 — Count nodes that are Ready AND don't have NoSchedule taint**

Look at the output:
- ❌ Skip nodes with `Taints: node-role.kubernetes.io/control-plane:NoSchedule`
- ❌ Skip nodes that are `NotReady`
- ✅ Count nodes that are `Ready` with no `NoSchedule` taint

**Step 4 — Write the count to the file**

```bash
# Create the directory if it doesn't exist
mkdir -p /opt/KUSC00402

# Write the count (replace 2 with your actual count)
echo "2" > /opt/KUSC00402/kusc00402.txt
```

> **WHY `echo` and not a command pipeline?** — It's faster and more reliable to manually count (usually 2–4 nodes) and write the number directly. Don't over-engineer this.

**Step 5 — Verify**

```bash
cat /opt/KUSC00402/kusc00402.txt
```

---

### ⚡ Exam Speed Strategy (Target: 1 minute)

```bash
# Quick way to see everything at once:
kubectl get nodes
kubectl describe nodes | grep -E "(Name:|Taints:)"

# Count ready nodes without NoSchedule, then:
mkdir -p /opt/KUSC00402
echo "2" > /opt/KUSC00402/kusc00402.txt
```

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| Taints and Tolerations | https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/ |
| **Search keyword** | `taint` or `NoSchedule` |

---

### 💡 Tips & Memory Aids

- 🧠 **Control plane nodes always have `NoSchedule` taint** — so typically only worker nodes count
- ⚠️ **Common mistake**: Counting all Ready nodes without checking taints
- ⚠️ **Common mistake**: Writing the output path wrong — double-check the path from the question
- 🔑 **Quick check**: `kubectl get nodes -o wide` gives you the full picture
- 📝 **In a typical 2-node cluster**: Control plane = NoSchedule, Worker = schedulable → answer is usually 1 or 2

---

### ✅ Verification Checklist

- [ ] `cat /opt/KUSC00402/kusc00402.txt` shows the correct number
- [ ] The number excludes NoSchedule nodes
- [ ] The number excludes NotReady nodes

---
---

## Question 18 — Fix NotReady Worker Node

| | |
|---|---|
| **Weightage** | ~5% (estimated) |
| **Difficulty** | 🟢 Easy |
| **CKA Domain** | Troubleshooting (30%) |
| **Estimated Exam Time** | ⏱️ 3–4 minutes |
| **Topics** | kubelet, systemd, node troubleshooting |
| **Related Notes** | [k8s-troubleshooting.md](k8s-troubleshooting.md) · [k8s-architecture.md](k8s-architecture.md) |

### 📋 Question

A kubernetes worker node "wk8s-node-0K0032005" is in state NotReady. Troubleshoot it and make it ready.

---

### ⚙️ LAB SETUP (Skip in Exam)

No specific setup — in the exam, the node will already be in NotReady state.

---

### ✅ Full Solution with Explanations

**Step 1 — Switch context and verify the node state**

```bash
# Switch to the correct context (given in exam)
kubectl config use-context <context-name>

# Check node status
kubectl get nodes
```

> **WHY?** — Always switch context first. Confirm which node is NotReady.

**Step 2 — SSH into the worker node**

```bash
ssh wk8s-node-0K0032005
```

> **WHY?** — You can't fix kubelet issues from the control plane. You must be on the affected node.

**Step 3 — Check kubelet status**

```bash
sudo systemctl status kubelet
```

> **WHY?** — The kubelet is the node agent. If a node is NotReady, kubelet is almost always the problem. Common findings:
> - `inactive (dead)` — kubelet stopped
> - `failed` — kubelet crashed
> - `active (running)` but with errors — config issue

**Step 4 — Start the kubelet**

```bash
sudo systemctl start kubelet
```

> **WHY?** — If kubelet was stopped/dead, starting it brings the node back online.

**Step 5 — Verify kubelet is running**

```bash
sudo systemctl status kubelet
```

Look for: `Active: active (running)`

**Step 6 — Enable kubelet to survive reboots**

```bash
sudo systemctl enable kubelet
```

> **WHY?** — `enable` creates a symlink so kubelet starts automatically on boot. Without this, the node would go NotReady again after a reboot. **This step is critical and often forgotten.**

**Step 7 — Exit the worker node**

```bash
exit
```

> **WHY?** — You MUST exit back to the control plane/bastion. If you stay SSH'd into the worker node, you'll be working on the wrong machine for subsequent questions.

**Step 8 — Verify the node is Ready**

```bash
kubectl get nodes
```

Look for: `wk8s-node-0K0032005   Ready`

> **Note**: It may take 30–60 seconds for the node status to update to Ready.

---

### ⚡ Exam Speed Strategy (Target: 2 minutes)

```bash
kubectl get nodes                          # Confirm NotReady
ssh wk8s-node-0K0032005                    # SSH into node
sudo systemctl start kubelet               # Start it
sudo systemctl enable kubelet              # Enable for reboot
sudo systemctl status kubelet              # Quick verify
exit                                       # IMPORTANT: exit back!
kubectl get nodes                          # Confirm Ready
```

> **Memory trick**: **S-S-E-E** = SSH → Start → Enable → Exit

---

### 📖 Documentation Quick-Find

| Resource | Link |
|----------|------|
| Troubleshoot Clusters | https://kubernetes.io/docs/tasks/debug/debug-cluster/ |
| kubelet reference | https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/ |
| **Search keyword** | `troubleshoot cluster` or `node not ready` |

---

### 💡 Tips & Memory Aids

- 🧠 **99% of NotReady = kubelet issue** — always check kubelet first
- ⚠️ **CRITICAL: Don't forget `systemctl enable`** — without it, kubelet won't survive a reboot
- ⚠️ **CRITICAL: Don't forget `exit`** — if you stay on the worker node, you'll accidentally run kubectl commands there
- 🔑 **If kubelet starts but node stays NotReady**: Check `journalctl -u kubelet -f` for errors — might be a config issue
- 📝 **Other things to check if kubelet isn't the issue**:
  - Container runtime: `systemctl status containerd`
  - Disk pressure: `df -h`
  - Memory: `free -m`

---

### ✅ Verification Checklist

- [ ] `kubectl get nodes` shows the node as `Ready`
- [ ] `systemctl status kubelet` shows `active (running)` and `enabled`
- [ ] You have exited back to the control plane / bastion machine

---
---

## 📊 Day 1 Summary

| Question | Topic | Target Time | Key Command |
|----------|-------|-------------|-------------|
| Q5 | StorageClass | 2 min | `kubectl apply -f sc.yaml` |
| Q6 | Helm Template | 3 min | `helm template ... > file.yaml` |
| Q12 | CRD + jsonpath | 2 min | `kubectl get crd \| grep cert` |
| Q17 | Count nodes | 1 min | `kubectl describe nodes \| grep Taint` |
| Q18 | Fix NotReady | 2 min | `systemctl start/enable kubelet` |
| **Total** | | **~10 min** | |

> 💪 **These 5 questions = ~25% of exam points in ~10 minutes**. Master them until they're muscle memory!

---

**Next**: [Day 2 — Core Skills 🟡](exam-day2-core-skills.md)
