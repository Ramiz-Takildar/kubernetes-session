# Session 18 — Helm

## What You Will Learn

Helm is the Kubernetes package manager that bundles manifests into reusable Charts and manages their full lifecycle through templating and release tracking. It transforms static YAML into parameterized, versioned packages that can be shared across teams and environments. This session teaches you how to install, upgrade, and uninstall applications using Helm's templating engine and value overrides.

---

## Prerequisites

- Helm CLI must be installed
```bash
helm version
```
- A running Kubernetes cluster (e.g., Docker Desktop, minikube, k3d)
- `kubectl` configured to access the cluster

---

## Core Concepts

- A **Chart** is a packaged collection of Kubernetes manifests, default configuration values, and metadata that can be versioned, shared, and reused across clusters and teams
- **Templates** use Go template syntax to generate Kubernetes manifests dynamically, allowing a single Chart to deploy different configurations for development, staging, and production environments
- **Values** inject configuration into templates at install or upgrade time; defaults live in `values.yaml`, while overrides can be supplied via `--set` flags or custom values files for environment-specific tuning
- A **Release** is a running instance of a Chart in a cluster; Helm tracks every release's state, enabling deterministic upgrades, rollbacks to previous revisions, and clean uninstalls
- The `{{ .Values }}` object exposes user-supplied configuration inside templates, decoupling the chart author's defaults from the deployer's specific needs without editing template files directly
- The `{{ .Chart }}` object provides access to chart metadata defined in `Chart.yaml` (such as name and version), allowing templates to stay synchronized with the package they belong to
- `helm template` renders Charts locally without touching the cluster, providing a safe way to validate generated manifests during CI/CD pipelines before any resources are created

---

## YAML Walkthrough

### Chart.yaml

```yaml
apiVersion: v2
name: my-nginx
version: 1.0.0
description: A simple Helm chart for nginx
type: application
```

| Field | Meaning |
|-------|---------|
| `apiVersion` | Helm chart API version (v2 for Helm 3) |
| `name` | Chart name (used in release naming) |
| `version` | Chart version (for upgrade tracking) |
| `type` | `application` for standard charts |

### values.yaml

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: stable-alpine
  pullPolicy: IfNotPresent

service:
  type: NodePort
  port: 80
  targetPort: 80
  nodePort: 30080
```

| Field | Meaning |
|-------|---------|
| `replicaCount` | How many nginx pods to run |
| `image.repository` | Container image name |
| `image.tag` | Image tag (version) |
| `image.pullPolicy` | When to pull the image |
| `service.type` | Service type (NodePort for external access) |
| `service.port` | Cluster-internal port |
| `service.targetPort` | Port the container listens on |
| `service.nodePort` | External port on each node |

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: nginx
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.targetPort }}
```

| Field | Meaning |
|-------|---------|
| `name: {{ .Chart.Name }}` | Templated: uses chart name from Chart.yaml |
| `replicas: {{ .Values.replicaCount }}` | Templated: uses value from values.yaml |
| `selector.matchLabels.app` | Templated: matches pod labels |
| `template.metadata.labels.app` | Templated: labels applied to pods |
| `image` | Templated: uses repository and tag from values.yaml |
| `imagePullPolicy` | Templated: uses pullPolicy from values.yaml |
| `containerPort` | Templated: uses targetPort from values.yaml |

### templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Chart.Name }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
    nodePort: {{ .Values.service.nodePort }}
```

| Field | Meaning |
|-------|---------|
| `name: {{ .Chart.Name }}` | Templated: uses chart name from Chart.yaml |
| `type: {{ .Values.service.type }}` | Templated: uses service type from values.yaml |
| `selector.app` | Templated: matches deployment pod labels |
| `port` | Templated: cluster-internal port |
| `targetPort` | Templated: container port to route to |
| `nodePort` | Templated: external access port |

---

## Step-by-Step Implementation

```bash
# 1. Change to session directory
cd sessions/18-helm

# 2. Verify Helm is installed
helm version
# Expected: version.BuildInfo{Version:"v3.x.x", ...}

# 3. Render templates locally without installing
helm template my-nginx .
# Expected: outputs the rendered Kubernetes manifests

# 4. Install the chart
helm install my-nginx .
# Expected: NAME: my-nginx, NAMESPACE: default, STATUS: deployed

# 5. Verify pods and service
kubectl get pods,svc
# Expected: 2 nginx pods Running, 1 NodePort service

# 6. Access nginx via NodePort (if applicable)
curl http://localhost:30080
# Expected: Welcome to nginx! HTML page

# 7. Upgrade with more replicas
helm upgrade my-nginx . --set replicaCount=3
# Expected: NAME: my-nginx, NAMESPACE: default, STATUS: upgraded

# 8. Verify the upgrade
kubectl get pods
# Expected: 3 nginx pods Running

# 9. List installed releases
helm list
# Expected: NAME: my-nginx, NAMESPACE: default, REVISION: 2, STATUS: deployed

# 10. Uninstall the release
helm uninstall my-nginx
# Expected: release "my-nginx" uninstalled

# 11. Verify cleanup
kubectl get pods
# Expected: No resources found in default namespace
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `helm template my-nginx .` | Templates render without errors |
| `helm install my-nginx .` | Chart installs successfully |
| `kubectl get pods` | Pods are created with correct replica count |
| `kubectl get svc` | NodePort service exists |
| `helm list` | Release is tracked by Helm |
| `helm upgrade my-nginx . --set replicaCount=3` | Upgrade changes replica count |
| `kubectl get pods` (after upgrade) | New replica count is reflected |
| `helm uninstall my-nginx` | Release is removed from cluster |
| `kubectl get pods` (after uninstall) | All pods are cleaned up |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `helm install` fails with "cannot re-use a name" | Release already exists | Use `helm upgrade my-nginx .` or `helm uninstall my-nginx` first |
| Pods stuck in `ImagePullBackOff` | Wrong image name or tag | Verify `image.repository` and `image.tag` in values.yaml |
| `helm template` renders empty output | Not in chart directory | Verify you are in `sessions/18-helm/` |
| Service not accessible | NodePort not exposed | Check firewall rules or use port forwarding instead |

---

## Key Takeaways

1. Helm is the package manager for Kubernetes, bundling manifests into versioned, reusable Charts
2. Charts contain Go templates and default values, making them deployable across different environments with minimal changes
3. The `{{ .Values }}` object injects configuration from `values.yaml` or `--set` overrides into templates at runtime
4. The `{{ .Chart }}` object exposes metadata from `Chart.yaml`, allowing templates to reference their own package identity
5. `helm install` creates a new release, `helm upgrade` applies changes, and `helm uninstall` removes all associated resources
6. `helm list` tracks releases across namespaces, and `helm rollback` reverts to a previous revision using Helm's stored state
7. `helm template` renders manifests locally for safe validation in CI/CD pipelines before any cluster changes occur

---

## Cleanup

```bash
helm uninstall my-nginx
```