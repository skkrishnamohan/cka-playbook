# Kubernetes Network Policy — Session Notes

---

## What is a NetworkPolicy?

- NetworkPolicy controls traffic flow between pods (who can talk to whom).
- By default, ALL pods can communicate with ALL other pods in the cluster (no restrictions).
- Once you apply a NetworkPolicy to a pod, it becomes **isolated** — only explicitly allowed traffic gets through.
- Requires a CNI plugin that supports NetworkPolicy (Calico, Cilium — NOT Flannel).

---

## Isolation rules (from session)

### Ingress Isolation (incoming traffic)
> By default, pods are non-isolated for ingress; all inbound connections are allowed.
> A pod becomes isolated for ingress if there's a NetworkPolicy that selects it and has "Ingress" in its policyTypes.
> Once isolated, only the connections allowed by the ingress rules are permitted.

### Egress Isolation (outgoing traffic)
> By default, pods are non-isolated for egress; all outbound connections are allowed.
> A pod becomes isolated for egress if there's a NetworkPolicy that selects it and has "Egress" in its policyTypes.
> Once isolated, only the connections allowed by the egress rules are permitted.

### Visual
```
Before NetworkPolicy:
  pod-a ←→ pod-b ←→ pod-c    (everyone talks to everyone)

After NetworkPolicy on pod-a (allow only from pod-b):
  pod-b → pod-a ✅
  pod-c → pod-a ❌ (blocked)
```

---

## Session Demo — step by step

### Step 1: Create namespace and pods (from session)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-a
  namespace: netpol-demo
  labels:
    app: a
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-b
  namespace: netpol-demo
  labels:
    app: b
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-c
  namespace: netpol-demo
  labels:
    app: c
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
```

### Step 2: Create a Service for pod-a (from session)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-a
  namespace: netpol-demo
spec:
  selector:
    app: a
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

### Step 3: Test — before NetworkPolicy (both can reach pod-a)
```bash
# Create namespace first
kubectl create namespace netpol-demo

# Apply all resources
kubectl apply -f pods.yaml
kubectl apply -f svc.yaml

# Test connectivity (from session)
kubectl exec -n netpol-demo pod-b -- wget -qO- svc-a    # ✅ Works
kubectl exec -n netpol-demo pod-c -- wget -qO- svc-a    # ✅ Works
```

### Step 4: Apply NetworkPolicy — allow only pod-b to reach pod-a (from session)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-b-to-a
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: a              # This policy applies TO pod-a
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: b          # Only allow traffic FROM pod-b
```

### Step 5: Test — after NetworkPolicy
```bash
kubectl apply -f netpol.yaml

kubectl exec -n netpol-demo pod-b -- wget -qO- --timeout=3 svc-a    # ✅ Still works
kubectl exec -n netpol-demo pod-c -- wget -qO- --timeout=3 svc-a    # ❌ Timeout (blocked!)
```

---

## NetworkPolicy anatomy explained

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-b-to-a
  namespace: netpol-demo
spec:
  podSelector:           # WHO does this policy apply to?
    matchLabels:
      app: a             # → Applies to pods with label app=a
  policyTypes:
  - Ingress              # → Controls INCOMING traffic to selected pods
  ingress:
  - from:                # WHO is allowed to send traffic?
    - podSelector:
        matchLabels:
          app: b         # → Only pods with label app=b can reach pod-a
```

---

## More NetworkPolicy examples

### Allow traffic only on specific port
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-80
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

### Deny ALL ingress (isolate completely)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: default
spec:
  podSelector: {}         # Applies to ALL pods in namespace
  policyTypes:
  - Ingress
  # No ingress rules = deny everything
```

### Allow ALL ingress (undo deny-all)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - {}                    # Empty rule = allow from everywhere
```

### Deny ALL egress
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # No egress rules = deny everything outbound
```

### Allow traffic from a different namespace
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          team: monitoring
      podSelector:
        matchLabels:
          app: prometheus
```

### Egress — allow pods to reach only specific destinations
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-db
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:                    # Also allow DNS (required!)
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

---

## Important gotchas

1. **DNS access**: If you restrict egress, pods can't resolve DNS. Always allow UDP port 53:
   ```yaml
   egress:
   - to:
     - namespaceSelector: {}
     ports:
     - protocol: UDP
       port: 53
   ```

2. **Namespace matters**: NetworkPolicy only applies within its own namespace.

3. **Multiple policies are additive**: If two policies select the same pod, allowed traffic is the UNION of both policies (never subtractive).

4. **`podSelector: {}`** = select ALL pods in the namespace.

5. **CNI requirement**: NetworkPolicy does NOTHING without a supporting CNI:
   - ✅ Calico, Cilium, Weave Net
   - ❌ Flannel (does not support NetworkPolicy)

---

## Useful commands
```bash
# List network policies
kubectl get networkpolicy -n netpol-demo
kubectl get netpol -n netpol-demo          # Short form

# Describe a policy
kubectl describe netpol allow-b-to-a -n netpol-demo

# Test connectivity
kubectl exec -n netpol-demo pod-b -- wget -qO- --timeout=3 svc-a

# Delete policy
kubectl delete netpol allow-b-to-a -n netpol-demo
```
