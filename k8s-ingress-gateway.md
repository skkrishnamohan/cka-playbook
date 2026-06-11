# Kubernetes Ingress, Gateway API & MetalLB — Session Notes

---

## The Problem: Exposing Apps on Bare-Metal

- `LoadBalancer` type Services work automatically on cloud providers (AWS, GCP, Azure).
- On **bare-metal / on-prem clusters**, `LoadBalancer` services stay in `<Pending>` forever — there is no cloud to assign an IP.
- Solution: **MetalLB** gives your cluster a real load balancer for bare-metal environments.
- Once you have external IPs, you still need **Ingress** or **Gateway API** to route HTTP/HTTPS traffic to the right service.

Traffic path:
```
External Client
      ↓
MetalLB (assigns external IP)
      ↓
Ingress Controller / Gateway Controller
      ↓
Ingress / HTTPRoute rules
      ↓
Service (ClusterIP)
      ↓
Pod(s)
```

---

## MetalLB — Load Balancer for Bare-Metal

- What: Software load balancer that watches for `type: LoadBalancer` Services and assigns external IPs from a configured pool.
- Why: Cloud-managed K8s clusters (EKS, AKS, GKE) get external IPs automatically. Bare-metal clusters need MetalLB.
- Modes: **Layer 2 (L2)** — simplest, uses ARP; **BGP** — for production with routers.

### Install MetalLB
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml

# Verify
kubectl get all -n metallb-system
```

### Configure IP Address Pool
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: my-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.56.100-192.168.56.110   # Range of IPs MetalLB can assign
```

### Enable L2 Advertisement (Layer 2 mode)
```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - my-ip-pool
```

After applying these, any `type: LoadBalancer` Service will get an IP from the pool automatically.

---

## Ingress — HTTP/HTTPS Routing

- What: Ingress is a Kubernetes resource that defines routing rules for HTTP/HTTPS traffic into the cluster.
- Why: Instead of creating a separate LoadBalancer Service for every app, one Ingress controller handles all HTTP routing.
- Requires: An **Ingress Controller** (NGINX, Traefik, HAProxy, etc.) — Kubernetes itself does NOT implement Ingress routing.

### Install NGINX Ingress Controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Basic Ingress — path-based routing
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress app-ingress
```

### Ingress with TLS (HTTPS)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: default
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: my-tls-secret      # TLS cert stored as a Secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Common Ingress Annotations (NGINX)
```yaml
metadata:
  annotations:
    # SSL passthrough (send TLS direct to backend, no termination)
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rpm: "60"
    nginx.ingress.kubernetes.io/limit-rps: "5"

    # Basic authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret

    # Rewrite target path
    nginx.ingress.kubernetes.io/rewrite-target: /
```

### Ingress commands
```bash
# List all ingresses
kubectl get ingress
kubectl get ing                        # Short form

# Describe ingress (shows routes and backend status)
kubectl describe ingress app-ingress

# Check ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

---

## Gateway API — Next-Generation Ingress

- What: The Gateway API is the official successor to Ingress — more expressive, role-oriented, and extensible.
- Why over Ingress:
  - Ingress has limited features, relies heavily on annotations (NGINX-specific).
  - Gateway API is standardized, supports TCP/UDP/gRPC, multi-team, multi-namespace routing.
- Status: Stable in Kubernetes v1.24+ (GA in 1.28).
- Key difference: Gateway API separates **infrastructure concerns** (Gateway) from **app routing** (HTTPRoute).

### Gateway API vs Ingress
| Feature | Ingress | Gateway API |
|---------|---------|-------------|
| Protocol support | HTTP/HTTPS only | HTTP, HTTPS, TCP, UDP, gRPC |
| Role separation | None | GatewayClass (infra) / HTTPRoute (app) |
| Multi-namespace routing | Limited | Native |
| Extensibility | Annotations (vendor-specific) | Typed resources |
| Standard | Basic | Rich, typed API |

### Install Gateway API CRDs
```bash
# Step 1: Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml

# Step 2: Install NGINX Gateway Fabric CRDs
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.6.2/deploy/crds.yaml

# Step 3: Install NGINX Gateway Controller
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.6.2/deploy/default/deploy.yaml
```

### Gateway API Components

```
GatewayClass  →  defines which controller handles it (like IngressClass)
     ↓
Gateway       →  defines port, protocol, TLS listeners
     ↓
HTTPRoute     →  defines host, path, and backend service rules
     ↓
Service/Pod
```

### GatewayClass — which controller handles this
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
```

### Gateway — listener configuration
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate              # Terminate TLS at the gateway
      certificateRefs:
      - kind: Secret
        name: my-tls-secret
```

### HTTPRoute — routing rules (equivalent to Ingress rules)
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-httproute
  namespace: default
spec:
  parentRefs:
  - name: my-gateway              # Attach to the Gateway
  hostnames:
  - "myapp.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /app1
    backendRefs:
    - name: app1-service
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /app2
    backendRefs:
    - name: app2-service
      port: 80
```

### Gateway API commands
```bash
# Check Gateway API CRDs installed
kubectl get crd | grep gateway

# List Gateways
kubectl get gateway

# List HTTPRoutes
kubectl get httproute

# Describe Gateway (shows listener status)
kubectl describe gateway my-gateway

# Describe HTTPRoute
kubectl describe httproute my-httproute
```

---

## How it all connects (session mental model)

```
Bare-Metal Cluster
      │
      ├── MetalLB
      │     └── Assigns external IP to LoadBalancer Services
      │
      ├── Ingress (NGINX) — current standard
      │     ├── IngressClass: nginx
      │     ├── Ingress rules (host + path → service)
      │     └── TLS termination via Secrets
      │
      └── Gateway API — future standard
            ├── GatewayClass (controller selector)
            ├── Gateway (port/protocol/TLS config)
            └── HTTPRoute (host/path/backend rules)
```

---

## Quick reference — when to use what

| Scenario | Solution |
|----------|----------|
| Bare-metal cluster, need external IPs | MetalLB |
| Simple HTTP/HTTPS routing, single team | Ingress (NGINX) |
| Multi-team routing, gRPC, TCP, advanced | Gateway API |
| TLS termination | Both support it (via Secret) |
| Rate limiting / auth on Ingress | NGINX annotations |

---

## One-line takeaways
- MetalLB gives bare-metal clusters the external IP assignment that cloud providers give for free.
- Ingress routes HTTP traffic to services — but needs an Ingress Controller to work.
- Gateway API replaces Ingress with a cleaner, more powerful, standardized API.
- GatewayClass = which controller; Gateway = listeners (port/TLS); HTTPRoute = routing rules.
- Both Ingress and Gateway API terminate TLS using Kubernetes Secrets.
