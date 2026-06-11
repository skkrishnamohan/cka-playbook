All nodes must execute the following tasks with root user privileges.

---

**TASK 1: Disable and Deactivate SWAP**

Remove any swap entries from the `/etc/fstab` file by executing:

```bash
sed -i '/swap/d' /etc/fstab
```

Then, disable swap immediately with:

```bash
swapoff -a
```

---

**TASK 2: Stop and Disable the Firewall**

To prevent interference with Kubernetes networking, disable and stop the firewall service using:

```bash
systemctl disable --now ufw
```

---

**TASK 3: Enable and Load Kernel Modules**

Create a configuration file to ensure the necessary kernel modules load at boot:

```bash
cat >> /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```

Load these modules immediately with:

```bash
modprobe overlay
modprobe br_netfilter
```

---

**TASK 4: Configure Kernel Parameters**

Append required kernel settings for Kubernetes networking to a sysctl configuration file:

```bash
cat >> /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF
```

Apply the new kernel parameters system-wide:

```bash
sysctl --system
```

---

**TASK 5: Add Kubernetes Repository and Install Prerequisites**

Update the package index and install essential packages for HTTPS transport and certificate management:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

---

**TASK 6: Install kubeadm, kubectl, and kubelet**

Create the directory for storing the Kubernetes apt keyrings:

```bash
sudo install -p -m 0755 -d /etc/apt/keyrings
```

Download and add the Kubernetes signing key:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod a+r /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the Kubernetes apt repository:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update the package index again and install the Kubernetes components:

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

Prevent these packages from being automatically updated:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Enable and start the kubelet service:

```bash
sudo systemctl enable --now kubelet
```

---

**TASK 7: Install containerd**

Remove Docker if installed, as containerd will be used instead:

```bash
sudo apt remove docker.io
```

Update the package index:

```bash
sudo apt-get update
```

Create the containerd configuration directory and install containerd:

```bash
sudo mkdir -p /etc/containerd/
sudo apt-get install -y containerd
```

Generate the default containerd configuration file:

```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

Modify the configuration to enable systemd cgroup driver:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Enable and restart containerd service:

```bash
sudo systemctl enable containerd
sudo systemctl restart containerd
```

---

**TASK 8: Install Network Tools**

Install essential network utilities, such as `ifconfig`:

```bash
apt install -qq -y net-tools
```

---

**TASK 9: Enable SSH Password Authentication**

Modify SSH configuration to allow password authentication and permit root login:

```bash
sed -i 's/^PasswordAuthentication .*/PasswordAuthentication yes/' /etc/ssh/sshd_config
echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
systemctl reload sshd
```

---

**TASK 10: Update Host Entries on All Nodes**

Append the master and worker node IP addresses and hostnames to the `/etc/hosts` file on all nodes:

```bash
cat >> /etc/hosts <<EOF
10.0.2.15   master.example.com     master  k8sm1
<worker 1 node ip>  worker1.example.com    worker1 k8sw1
<worker 2 node ip>  worker2.example.com    worker2 k8sw2
EOF
```

On the master node, ensure the master entry is present:

```bash
cat >> /etc/hosts <<EOF
10.0.2.15   master.example.com     master  k8sm1
EOF
```

---

### Master Node Configuration

Pull the required Kubernetes images:

```bash
kubeadm config images pull
```

Example output indicates pulling images such as `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kube-proxy`, `coredns`, `pause`, and `etcd`.

Initialize the Kubernetes control plane, specifying the API server advertise address and pod network CIDR:

```bash
kubeadm init --apiserver-advertise-address=10.0.2.15 --pod-network-cidr=192.168.0.0/16
```

*Note: The specified pod network CIDR may cause network issues in certain environments, such as Azure.*

Join worker nodes to the cluster using the token and discovery token CA certificate hash provided by the `kubeadm init` output:

```bash
kubeadm join 10.0.2.15:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

*It is essential to securely copy and save the join token output for use on worker nodes.*

Set up the Kubernetes configuration for the current user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify the status of system pods:

```bash
kubectl get po -n kube-system
```

Enable command-line completion and create convenient aliases:

```bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
alias k=kubectl
complete -o default -F __start_kubectl k
```

---

### Install Calico Network Plugin

Deploy the Calico network plugin to enable pod networking:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Confirm the Calico pods are running:

```bash
kubectl get po -n kube-system
```

---

### Worker Node Configuration

Join the worker nodes to the cluster using the following command template, replacing placeholders with actual values obtained from the master node:

```bash
kubeadm join <Master node IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<cert key>
```

---
At times, a conntrack error may arise; to resolve this, please follow the installation steps for conntrack. The "conntrack not found" error appears during the execution of kubeadm join because the necessary conntrack utility is absent from the system path on either the worker or control-plane node. Both kubeadm and kube-proxy depend on this utility to monitor network connections and manage packet forwarding.

To install conntrack, execute the following commands:

```bash
sudo apt update  
sudo apt install -y conntrack
```

---

### Common kubectl Commands

- List pods: `kubectl get pods`
- List nodes: `kubectl get nodes`
- List services: `kubectl get services` or `kubectl get svc`
- List replicasets: `kubectl get rs` or `kubectl get replicaset`
- List namespaces: `kubectl get namespaces`
- Describe a pod: `kubectl describe pod <pod name>`
- Describe deployments: `kubectl describe deployments`
- Apply a resource configuration: `kubectl apply -f <resource/object yaml>`
- Retrieve API resource versions: `kubectl api-resources | grep -i <resource name>`

---

### Retrieving the Discovery Token Certificate Key

To obtain the token used for joining worker nodes, execute on the master node:

```bash
kubectl token list
```

To extract the certificate key for the discovery token, use:

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256
```

---

This comprehensive procedure outlines the necessary steps for setting up a Kubernetes cluster, including disabling swap, configuring kernel modules and parameters, installing Kubernetes components and container runtime, configuring SSH and host entries, initializing the master node, joining worker nodes, and managing the cluster with `kubectl`. The instructions are designed to ensure a stable and functional Kubernetes environment suitable for production or development purposes.

---

### Why each step matters (quick reference)

| Step | Why it's needed |
|------|----------------|
| Disable swap | kubelet refuses to start if swap is enabled (Kubernetes manages memory itself) |
| Disable firewall | Prevents port blocking between nodes (in production, open only required ports instead) |
| Load kernel modules | `overlay` = container filesystem support; `br_netfilter` = iptables sees bridged traffic |
| Kernel params (ip_forward, bridge-nf-call) | Enables packet forwarding between pods across nodes |
| Install containerd | The container runtime that actually runs pods |
| SystemdCgroup = true | Ensures containerd and kubelet use the same cgroup driver (prevents instability) |
| apt-mark hold | Prevents accidental upgrades that could break the cluster |

---

### Troubleshooting tips for cluster setup

Common issues and fixes:
```bash
# kubelet won't start — check logs
journalctl -u kubelet -f

# Nodes stuck in NotReady — usually CNI not installed yet
kubectl get nodes
kubectl get pods -n kube-system  # Check if CNI pods are running

# Join token expired (tokens expire after 24h by default)
kubeadm token create --print-join-command

# Reset a failed kubeadm init/join (start fresh on that node)
kubeadm reset
rm -rf /etc/cni/net.d
iptables -F && iptables -t nat -F && iptables -t mangle -F

# Check if all required ports are open
nc -zv <master-ip> 6443    # API server port

# CoreDNS pods stuck in Pending — install CNI plugin first
kubectl apply -f <calico/flannel manifest>

# containerd not running
systemctl status containerd
journalctl -u containerd -f
```

### Post-setup verification checklist
```bash
# 1. All nodes should be Ready
kubectl get nodes

# 2. System pods should be Running
kubectl get pods -n kube-system

# 3. CoreDNS should be Running (requires CNI)
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 4. Test pod creation
kubectl run test --image=nginx --restart=Never
kubectl get pod test
kubectl delete pod test

# 5. Test DNS resolution
kubectl run dns-test --image=busybox --restart=Never -- nslookup kubernetes
kubectl logs dns-test
kubectl delete pod dns-test
```

### Cluster upgrade process (CKA exam topic)
```bash
# On control plane node:
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.XX.X-*
sudo apt-mark hold kubeadm
kubeadm upgrade plan
kubeadm upgrade apply v1.XX.X

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.XX.X-* kubectl=1.XX.X-*
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# On worker nodes: drain first, upgrade, then uncordon
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data
# (SSH to worker, repeat apt install kubeadm/kubelet/kubectl)
kubectl uncordon <worker-node>
```


