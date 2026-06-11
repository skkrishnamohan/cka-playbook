# Kubernetes `/etc/kubernetes` Deep Dive Guide
## Beginner to Expert Level Reference

---

# Table of Contents

1. Introduction
2. `/etc/kubernetes` Overview
3. Configuration Files
4. Static Pod Manifests
5. PKI Infrastructure
6. ETCD Certificates
7. Authentication Flow
8. TLS Communication Flow
9. Control Plane Startup Sequence
10. Important Troubleshooting Commands
11. Security Best Practices
12. CKA/CKS Important Points
13. Real-World Production Notes
14. Architecture Summary

---

# 1. Introduction

The directory:

```bash
/etc/kubernetes
```

is one of the most important directories in a Kubernetes control plane node.

This directory contains:

- Kubernetes cluster configuration files
- Static pod manifests
- TLS certificates
- Private keys
- Authentication material
- kubeconfig files
- Control plane definitions

This folder is mainly created by:

```bash
kubeadm init
```

If this directory is damaged, deleted, or misconfigured:
- control plane may fail
- API server may not start
- etcd may become inaccessible
- cluster authentication may fail

This directory is essentially the "brain + identity + startup blueprint" of the cluster.

---

# 2. `/etc/kubernetes` Overview

Directory structure:

```text
/etc/kubernetes
├── admin.conf
├── controller-manager.conf
├── kubelet.conf
├── manifests/
├── pki/
├── scheduler.conf
└── super-admin.conf
```

---

# 3. Configuration Files

---

# 3.1 admin.conf

## Purpose

Primary kubeconfig file for cluster administration.

Used by:
- cluster admins
- kubectl
- automation tools
- troubleshooting

Contains:
- cluster endpoint
- CA certificate
- client certificate
- authentication details

---

## Usage

```bash
kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes
```

Usually copied to user home:

```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
```

---

## Important Fields

Contains:
- cluster
- user
- context

Example:

```yaml
clusters:
users:
contexts:
```

---

## Security Importance

This file gives:
- cluster-admin access

Anyone possessing this file can fully control cluster.

Protect carefully.

---

# 3.2 controller-manager.conf

## Purpose

Authentication configuration for:

```text
kube-controller-manager
```

Used internally by:
- control plane
- Kubernetes controllers

---

## Controllers Using This

Examples:
- node controller
- replication controller
- service account controller
- deployment controller
- job controller

---

## Why Important

Without this:
- controllers cannot communicate with API server
- deployments stop reconciling
- nodes may become unhealthy

---

# 3.3 kubelet.conf

## Purpose

Authentication file used by kubelet.

Allows kubelet to:
- register node
- communicate with API server
- report node status
- manage pods

---

## Used By

```text
systemd kubelet service
```

---

## Common Failure

Expired kubelet certificate causes:
- node NotReady
- kubelet authentication failure

---

## Verify

```bash
systemctl status kubelet
```

---

# 3.4 scheduler.conf

## Purpose

Authentication config for:

```text
kube-scheduler
```

---

## Scheduler Responsibilities

Scheduler decides:
- which node runs which pod

It evaluates:
- CPU
- memory
- taints
- affinity
- anti-affinity
- topology
- resource availability

---

## If Scheduler Fails

Pods remain:

```text
Pending
```

---

# 3.5 super-admin.conf

## Purpose

Special high-privilege kubeconfig.

Not always present in all setups.

Usually:
- emergency admin
- recovery access
- break-glass access

---

## Security Risk

Must be highly protected.

Equivalent to:
- root access
- full cluster ownership

---

# 4. manifests/

Directory:

```bash
/etc/kubernetes/manifests
```

Contains static pod definitions.

---

# What Are Static Pods?

Static pods are:
- directly managed by kubelet
- not created via API server

Kubelet continuously watches this directory.

If YAML changes:
- kubelet automatically recreates pod

---

## Important Concept

Control plane components are usually static pods.

Why?

Because:
- API server may not exist yet
- cluster must bootstrap itself

---

# Static Pod Files

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

---

# 4.1 etcd.yaml

## Purpose

Defines ETCD database pod.

ETCD stores:
- cluster state
- secrets
- configmaps
- deployments
- nodes
- RBAC
- everything

---

## ETCD Is Kubernetes Database

Without ETCD:
- cluster loses state
- API server cannot function

---

## Common Settings

Contains:
- cert paths
- peer communication
- data dir
- listen ports

---

## Important Ports

```text
2379 -> client traffic
2380 -> peer traffic
```

---

# 4.2 kube-apiserver.yaml

## Purpose

Defines Kubernetes API server.

This is the:
- front door
- central hub
- brain interface

All components talk through API server.

---

## Responsibilities

- authentication
- authorization
- admission control
- API handling
- REST interface
- cluster orchestration

---

## Important Ports

```text
6443
```

---

## Critical Flags

Examples:

```text
--etcd-servers
--client-ca-file
--service-account-key-file
--tls-cert-file
```

---

## If API Server Fails

Entire cluster effectively becomes unusable.

---

# 4.3 kube-controller-manager.yaml

## Purpose

Runs Kubernetes controllers.

Controllers maintain desired state.

---

## Example

Desired:
```text
3 replicas
```

Actual:
```text
2 replicas
```

Controller creates missing pod.

This reconciliation loop is core Kubernetes behavior.

---

# 4.4 kube-scheduler.yaml

## Purpose

Schedules pods onto nodes.

---

## Scheduler Logic

Evaluates:
- node resources
- constraints
- policies
- affinities
- taints/tolerations

---

## Important Concept

Scheduler only decides placement.

Kubelet actually runs container.

---

# 5. PKI Directory

Directory:

```bash
/etc/kubernetes/pki
```

PKI = Public Key Infrastructure

Contains:
- certificates
- keys
- certificate authorities

These secure Kubernetes communication.

---

# Why PKI Matters

Kubernetes heavily depends on:
- TLS
- mutual authentication
- encrypted communication

Without PKI:
- components cannot trust each other

---

# PKI Files Overview

```text
apiserver.crt
apiserver.key
ca.crt
ca.key
front-proxy-ca.crt
front-proxy-ca.key
sa.key
sa.pub
```

---

# 5.1 ca.crt + ca.key

## Purpose

Main Kubernetes Certificate Authority.

Signs:
- API server certs
- kubelet certs
- client certs

---

## Extremely Critical

This is the root of trust.

If compromised:
- cluster trust collapses

---

## Best Practice

Protect:
```text
ca.key
```

Never expose publicly.

---

# 5.2 apiserver.crt + apiserver.key

## Purpose

TLS certificate for API server.

Used when clients connect:

```text
kubectl -> API server
```

---

## Provides

- encryption
- server identity

---

# 5.3 apiserver-etcd-client.crt/key

## Purpose

Allows API server to securely communicate with ETCD.

Flow:

```text
API Server -> ETCD
```

---

# Why Important

ETCD contains all cluster data.

Unauthorized access is catastrophic.

---

# 5.4 apiserver-kubelet-client.crt/key

## Purpose

Used by API server to talk securely with kubelets.

Examples:
- logs
- exec
- port-forward

---

# Flow

```text
kubectl logs
kubectl exec
```

internally requires:
```text
API Server -> kubelet
```

---

# 5.5 front-proxy-ca.crt/key

## Purpose

Used for front proxy authentication.

Mainly related to:
- extension APIs
- aggregation layer

---

# Aggregation Layer

Allows Kubernetes to extend APIs.

Examples:
- metrics-server
- custom metrics
- API extensions

---

# 5.6 front-proxy-client.crt/key

## Purpose

Client identity used with aggregation layer.

---

# 5.7 sa.key + sa.pub

## Purpose

Used for Service Account token signing.

---

## Extremely Important

Every pod using service accounts depends on these keys.

---

## Flow

```text
Pod
 ↓
Service Account Token
 ↓
Signed by sa.key
 ↓
Verified using sa.pub
```

---

# 6. ETCD PKI

Directory:

```bash
/etc/kubernetes/pki/etcd
```

---

# ETCD Security

ETCD communication is encrypted separately.

Because ETCD is highly sensitive.

---

# Files

```text
ca.crt
ca.key
healthcheck-client.crt
healthcheck-client.key
peer.crt
peer.key
server.crt
server.key
```

---

# 6.1 etcd/ca.crt + ca.key

## Purpose

Certificate authority specifically for ETCD.

---

# 6.2 server.crt + server.key

## Purpose

ETCD server identity certificate.

Used when clients connect to ETCD.

---

# 6.3 peer.crt + peer.key

## Purpose

Used for ETCD peer-to-peer communication.

Only relevant in HA multi-node ETCD clusters.

---

# Peer Communication

```text
ETCD Node 1 <-> ETCD Node 2
```

---

# 6.4 healthcheck-client.crt/key

## Purpose

Used for ETCD health checks.

Example:

```bash
etcdctl endpoint health
```

---

# 7. Kubernetes Authentication Flow

---

# Kubectl Flow

```text
kubectl
 ↓
admin.conf
 ↓
API Server
 ↓
Certificate Validation
 ↓
Authorization
 ↓
API Action
```

---

# Kubelet Flow

```text
kubelet
 ↓
kubelet.conf
 ↓
API Server
 ↓
Authenticated Node
```

---

# Service Account Flow

```text
Pod
 ↓
Mounted Token
 ↓
API Server
 ↓
JWT Validation
 ↓
Authorized Access
```

---

# 8. TLS Communication Flow

---

# API Server to ETCD

```text
API Server
 ↓
apiserver-etcd-client.crt
 ↓
ETCD
```

---

# API Server to Kubelet

```text
API Server
 ↓
apiserver-kubelet-client.crt
 ↓
Kubelet
```

---

# Kubectl to API Server

```text
kubectl
 ↓
admin.conf
 ↓
apiserver.crt
```

---

# 9. Control Plane Startup Sequence

When node boots:

---

## Step 1

systemd starts:
```text
kubelet
```

---

## Step 2

kubelet watches:

```text
/etc/kubernetes/manifests
```

---

## Step 3

kubelet creates static pods:
- etcd
- kube-apiserver
- controller-manager
- scheduler

---

## Step 4

API server becomes available.

---

## Step 5

Other components authenticate using kubeconfig files.

---

# 10. Important Troubleshooting Commands

---

# Check Static Pods

```bash
kubectl get pods -n kube-system
```

---

# Check kubelet

```bash
systemctl status kubelet
```

---

# View Static Pod YAML

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

---

# Check Certificates

```bash
openssl x509 -in apiserver.crt -text -noout
```

---

# Check Expiration

```bash
kubeadm certs check-expiration
```

---

# ETCD Health

```bash
etcdctl endpoint health
```

---

# View API Server Logs

```bash
kubectl logs -n kube-system kube-apiserver-<node-name>
```

---

# 11. Security Best Practices

---

# Never Expose

Critical private keys:
- ca.key
- sa.key
- server.key

---

# File Permissions

Protect:

```bash
chmod 600 *.key
```

---

# Rotate Certificates

Use:

```bash
kubeadm certs renew all
```

---

# Backup ETCD

Most critical backup in cluster.

---

# 12. CKA / CKS Important Points

---

# CKA Focus Areas

Must know:
- static pods
- manifests directory
- kubelet watching manifests
- kubeconfig files
- certificate locations

---

# Common Exam Tasks

- fix expired certificates
- edit static pods
- recover control plane
- inspect manifests
- backup ETCD

---

# CKS Focus Areas

Security:
- certificate management
- RBAC
- service account security
- TLS communication
- protecting private keys

---

# 13. Real-World Production Notes

---

# API Server Is Central

Everything depends on API server.

If API server dies:
- kubectl fails
- controllers stop
- scheduling stops

---

# ETCD Is Most Critical Data

Treat ETCD like:
- production database
- crown jewel

---

# Never Randomly Edit PKI

Wrong certificate changes can:
- break trust chain
- bring cluster down

---

# Static Pod Restart Trick

Edit file in:
```text
/etc/kubernetes/manifests
```

kubelet auto-restarts pod.

---

# 14. Architecture Summary

```text
                 kubectl
                    ↓
             admin.conf
                    ↓
            kube-apiserver
               /   |    \
              /    |     \
             ↓     ↓      ↓
          ETCD  Scheduler Controller
             ↑
             |
        Certificates
             |
         /etc/kubernetes/pki
```

---

# Final Summary

The `/etc/kubernetes` directory contains:

- cluster identity
- TLS trust chain
- control plane definitions
- authentication configs
- static pod manifests
- certificate authorities
- service account signing keys

Mastering this directory means understanding:
- Kubernetes internals
- control plane architecture
- authentication
- TLS
- bootstrap sequence
- disaster recovery
- troubleshooting
- cluster security

This is one of the highest-leverage areas for:
- CKA
- CKAD
- CKS
- production Kubernetes engineering
- platform engineering
- DevOps
- SRE work

---

## Additional CKA-critical topics

### etcd backup and restore (MUST KNOW for CKA exam)

etcd holds ALL cluster state. If you lose etcd, you lose your cluster.

**Backup etcd:**
```bash
# Find etcd certs (check the static pod manifest)
cat /etc/kubernetes/manifests/etcd.yaml | grep -E 'cert|key|ca'

# Take a snapshot
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-table
```

**Restore etcd:**
```bash
# Stop kube-apiserver (move the static pod manifest)
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Restore from snapshot to a NEW data directory
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# Update etcd manifest to use new data directory
# Edit /etc/kubernetes/manifests/etcd.yaml:
#   Change hostPath from /var/lib/etcd to /var/lib/etcd-restored

# Move API server manifest back
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Wait for pods to come back
crictl pods | grep etcd
kubectl get pods -n kube-system
```

### Certificate management

**Check certificate expiration:**
```bash
# Check all kubeadm-managed certificates
kubeadm certs check-expiration

# Check a specific certificate manually
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# Check who issued the cert
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -issuer -subject
```

**Renew certificates:**
```bash
# Renew all certificates (kubeadm 1.20+)
kubeadm certs renew all

# Renew specific certificate
kubeadm certs renew apiserver

# After renewal, restart control plane components
# (kubelet will auto-restart static pods when manifests are re-read)
systemctl restart kubelet
```

**kubelet certificate auto-rotation:**
- kubelet can automatically rotate its own client certificate.
- Enabled by default in modern K8s (1.19+).
- kubelet config: `rotateCertificates: true`
- The CSR is auto-approved by the `csrapproving` controller.
```bash
# Check kubelet certificate
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates

# Check pending CSRs
kubectl get csr
kubectl certificate approve <csr-name>
```

### Creating custom static pods
- Static pods are managed directly by kubelet (no API server needed).
- Place manifests in `/etc/kubernetes/manifests/` (default path).
- kubelet watches this directory and creates/deletes pods based on files present.

Create a custom static pod:
```bash
# Create manifest file
cat <<EOF > /etc/kubernetes/manifests/my-static-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-static-pod
  namespace: kube-system
spec:
  containers:
  - name: web
    image: nginx:1.28
    ports:
    - containerPort: 80
EOF

# kubelet automatically creates the pod
# A mirror pod appears in the API server (read-only)
kubectl get pods -n kube-system | grep my-static-pod
```

Delete a static pod:
```bash
# Simply remove the manifest file
rm /etc/kubernetes/manifests/my-static-pod.yaml
# kubelet automatically removes the pod
```

Find static pod manifest path:
```bash
# Check kubelet config for staticPodPath
cat /var/lib/kubelet/config.yaml | grep staticPodPath
# OR check kubelet process args
ps aux | grep kubelet | grep pod-manifest-path
```

### Quick reference: important file paths
| Path | Purpose |
|------|---------|
| `/etc/kubernetes/manifests/` | Static pod manifests (control plane) |
| `/etc/kubernetes/pki/` | All cluster certificates and keys |
| `/etc/kubernetes/pki/etcd/` | etcd-specific certs |
| `/etc/kubernetes/admin.conf` | Cluster admin kubeconfig |
| `/etc/kubernetes/kubelet.conf` | kubelet's kubeconfig |
| `/etc/kubernetes/scheduler.conf` | Scheduler's kubeconfig |
| `/etc/kubernetes/controller-manager.conf` | CM's kubeconfig |
| `/var/lib/etcd/` | etcd data directory |
| `/var/lib/kubelet/config.yaml` | kubelet runtime configuration |
| `/var/lib/kubelet/pki/` | kubelet certificates |
| `/etc/cni/net.d/` | CNI plugin configuration |
| `/opt/cni/bin/` | CNI plugin binaries |

