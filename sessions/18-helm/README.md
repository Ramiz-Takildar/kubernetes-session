# Session 18 — Helm

## What You Will Learn

Helm is the Kubernetes package manager. It bundles Kubernetes manifests into reusable **Charts**, uses **templates** to make manifests configurable, and manages the lifecycle of releases (install, upgrade, rollback, uninstall).

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

- **Chart**: A collection of files that describe Kubernetes resources (like a package)
- **Templates**: Go templates that generate Kubernetes manifests dynamically
- **Values**: Configuration values passed to templates (defaults in `values.yaml`, overrides via `--set`)
- **Releases**: A running instance of a Chart in the cluster

### Template Syntax

- `{{ .Values.replicaCount }}` — accesses values from `values.yaml`
- `{{ .Chart.Name }}` — accesses chart metadata from `Chart.yaml`

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

1. Helm is the package manager for Kubernetes — it bundles, installs, and manages Kubernetes applications
2. Charts are reusable packages containing templates and default values
3. `{{ .Values }}` syntax injects values from `values.yaml` into templates
4. `{{ .Chart.Name }}` accesses chart metadata from `Chart.yaml`
5. `helm install` creates a new release, `helm upgrade` updates it
6. `helm list` shows all releases, `helm uninstall` removes them
7. Helm tracks release history, enabling rollback with `helm rollback`

---

## Cleanup

```bash
helm uninstall my-nginx
```