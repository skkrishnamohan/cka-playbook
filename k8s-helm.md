# Helm — Kubernetes Package Manager

---

## What is Helm?

- Helm is the package manager for Kubernetes — like `apt` for Debian or `brew` for macOS.
- A **chart** is a Helm package: a collection of YAML templates for a Kubernetes application.
- Why Helm:
  - Deploy complex apps (databases, monitoring stacks) with a single command.
  - Manage upgrades and rollbacks cleanly.
  - Share and reuse application configurations across teams and environments.

```
Without Helm: kubectl apply -f deployment.yaml -f service.yaml -f ingress.yaml -f configmap.yaml ...
With Helm:    helm install myapp ./mychart
```

---

## Helm Chart Structure

```
mychart/
├── Chart.yaml         # Chart metadata (name, version, description)
├── values.yaml        # Default configuration values
├── charts/            # Dependency charts stored here
├── Chart.lock         # Locked dependency versions (like package-lock.json)
└── templates/         # Kubernetes YAML templates (use Go templating)
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── _helpers.tpl   # Template helper functions
```

### Chart.yaml — chart metadata
```yaml
apiVersion: v2
name: mychart
description: A Helm chart for my application
type: application
version: 0.1.0          # Chart version (increment on chart changes)
appVersion: "1.0.0"     # Application version (what your app is)
```

### values.yaml — configurable defaults
```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.25"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  host: myapp.example.com
```

---

## Core Helm Commands

### Create a chart
```bash
helm create mychart
# Creates the full chart directory structure under ./mychart/
```

### Install a chart
```bash
# Install from local chart
helm install myapp ./mychart

# Install with custom values (override values.yaml)
helm install myapp ./mychart --set replicaCount=3

# Install with a values file
helm install myapp ./mychart -f custom-values.yaml

# Install into a specific namespace
helm install myapp ./mychart -n production --create-namespace
```

### List and inspect releases
```bash
# List all installed Helm releases
helm list
helm list -n production       # In a specific namespace

# Get status of a release
helm status myapp

# Show rendered templates (without deploying)
helm template myapp ./mychart

# Preview what would change before upgrade
helm upgrade myapp ./mychart --dry-run
```

### Upgrade and rollback
```bash
# Upgrade a release (apply changes to chart/values)
helm upgrade myapp ./mychart

# Upgrade with a new value
helm upgrade myapp ./mychart --set replicaCount=5

# View release history
helm history myapp

# Rollback to a previous version
helm rollback myapp 1          # Roll back to revision 1
```

### Uninstall
```bash
# Remove a release and all its resources
helm uninstall myapp

# Uninstall from a specific namespace
helm uninstall myapp -n production
```

---

## Helm Repositories — finding published charts

A Helm repository is a collection of charts hosted on a web server.

```bash
# Add well-known repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Update local repo cache (like apt-get update)
helm repo update

# List added repos
helm repo list

# Search for a chart in added repos
helm search repo nginx
helm search repo bitnami/redis

# Install a chart from a repo
helm install my-redis bitnami/redis

# Install a specific version
helm install my-redis bitnami/redis --version 17.0.0

# Show available values for a repo chart
helm show values bitnami/redis
```

---

## Helm Dependencies — charts that depend on other charts

Dependencies let your chart automatically pull in charts it needs (e.g. your app chart depends on Redis).

### Define dependencies in Chart.yaml
```yaml
dependencies:
- name: redis
  version: "17.x.x"
  repository: https://charts.bitnami.com/bitnami
- name: postgresql
  version: "12.x.x"
  repository: https://charts.bitnami.com/bitnami
```

### Download dependencies
```bash
# Download all dependencies (creates charts/ directory and Chart.lock)
helm dependency update

# After running this, you'll see:
# charts/
#   └── redis-17.x.x.tgz
#   └── postgresql-12.x.x.tgz
# Chart.lock          ← locks exact versions (commit this to git!)
```

### Chart.lock — dependency version pinning
```yaml
dependencies:
- name: redis
  repository: https://charts.bitnami.com/bitnami
  version: 17.3.14            # Exact version locked
digest: sha256:abc123...
generated: "2024-01-01T00:00:00.000Z"
```

---

## Helm OCI Registry — storing charts like container images

Traditional Helm repos use HTTP web servers. OCI registries (like GitHub Container Registry, Docker Hub) can also store Helm charts as OCI artifacts.

### Login to OCI registry
```bash
helm registry login ghcr.io
# Enter: username + personal access token
```

### Package a chart into a .tgz
```bash
helm package mychart
# Creates: mychart-0.1.0.tgz  (version from Chart.yaml)
```

### Push to OCI registry
```bash
helm push mychart-0.1.0.tgz oci://ghcr.io/myorg/charts
```

### Pull from OCI registry
```bash
helm pull oci://ghcr.io/myorg/charts/mychart --version 0.1.0

# Pull and unpack
helm pull oci://ghcr.io/myorg/charts/mychart --version 0.1.0 --untar
```

### Install directly from OCI registry
```bash
helm install myapp oci://ghcr.io/myorg/charts/mychart --version 0.1.0
```

---

## Helm vs kubectl apply — when to use which

| Scenario | Use |
|----------|-----|
| Simple single-resource deploy | `kubectl apply -f` |
| Complex multi-resource app | Helm |
| Install community software (Redis, Postgres) | Helm (bitnami repo) |
| Want upgrade/rollback history | Helm |
| CI/CD with versioned releases | Helm |
| Share charts across teams/orgs | Helm + OCI registry |

---

## Common patterns from class

```bash
# Full workflow: create → install → upgrade → rollback
helm create myapp
helm install myapp ./myapp
# (make changes to templates or values.yaml)
helm upgrade myapp ./myapp
helm history myapp             # See revision history
helm rollback myapp 1          # Go back to revision 1

# Install from bitnami with custom values
helm install mydb bitnami/postgresql \
  --set auth.postgresPassword=secret \
  --set primary.persistence.size=20Gi \
  -n databases --create-namespace

# Render templates to stdout for debugging
helm template myapp ./myapp | kubectl apply --dry-run=client -f -
```

---

## One-line takeaways
- Helm is the Kubernetes package manager — charts bundle all resources for an app into one deployable unit.
- `helm install` deploys; `helm upgrade` updates; `helm rollback` reverts; `helm uninstall` removes.
- `values.yaml` holds defaults; override with `--set` or `-f custom.yaml` at install/upgrade time.
- Repositories (bitnami, hashicorp, etc.) are where community charts live — `helm repo add` + `helm repo update`.
- OCI registries let you store and distribute Helm charts like container images (GitHub, Docker Hub, ACR).
- Dependencies in `Chart.yaml` + `helm dependency update` automatically pull in sub-charts; `Chart.lock` pins versions.
