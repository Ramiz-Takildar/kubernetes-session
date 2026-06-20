# Session 02 — Pods

## What You Will Learn

A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share networking and storage. In practice, you rarely create Pods directly — you use Deployments, Jobs, etc. But understanding Pods is essential.

---

## Core Concepts

- A Pod represents a single running process on the cluster
- Containers inside a Pod share the same network namespace (same IP)
- Pods are mortal — if the node fails, the Pod is lost
- No `namespace` field in metadata means the Pod goes to the `default` namespace
- Labels are key/value pairs used by selectors (Deployments, Services)

---

## YAML Walkthrough

### pod.yaml

Basic pod in the `default` namespace:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: basic-nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
```

### pod-namespace.yaml

Pod placed explicitly in the `dev` namespace:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: namespaced-nginx-pod
  namespace: dev
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
```

| Field | Meaning |
|-------|---------|
| `metadata.name` | Pod name — must be unique within the namespace |
| `metadata.namespace` | Which namespace to place the Pod in (defaults to `default` if omitted) |
| `metadata.labels` | Key/value pairs for identification and selection |
| `spec.containers[].name` | Container name within the Pod |
| `spec.containers[].image` | Container image to run |
| `spec.containers[].ports.containerPort` | Port the container exposes (informational) |

---

## Step-by-Step Implementation

```bash
# 1. Apply both pods
cd sessions/02-pods
kubectl apply -f .

# 2. Verify pod in default namespace
kubectl get pod basic-nginx-pod
# Expected: STATUS = Running, READY = 1/1

# 3. Inspect the pod
kubectl describe pod basic-nginx-pod
# Look for: Node, Start Time, Container ID, Events

# 4. Verify pod in dev namespace
kubectl get pod namespaced-nginx-pod -n dev
# Expected: STATUS = Running

# 5. Test the pod with port-forward
kubectl port-forward basic-nginx-pod 8080:80
# Keep this running, then in another terminal:
curl http://localhost:8080
# Expected: "Welcome to nginx!"

# 6. View pod logs
kubectl logs basic-nginx-pod
# You should see nginx startup messages

# 7. Execute a command inside the pod
kubectl exec basic-nginx-pod -- ps aux
# Shows running processes inside the container
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get pod basic-nginx-pod` | Pod exists in `default` namespace with `Running` status |
| `kubectl get pod namespaced-nginx-pod -n dev` | Pod exists in `dev` namespace with `Running` status |
| `kubectl describe pod basic-nginx-pod` | Pod is scheduled on a node with a working container |
| `kubectl port-forward basic-nginx-pod 8080:80` | The nginx web server is accessible |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Pod stuck in `ContainerCreating` | Image pull error or volume mount issue | Run `kubectl describe pod <name>` and check Events |
| Port-forward already in use | A previous port-forward is still running | Kill it with Ctrl+C or use a different local port (e.g., `8081:80`) |

---

## Key Takeaways

1. No `namespace` field = `default` namespace
2. Labels are critical for selectors used by Deployments and Services (later sessions)
3. `kubectl exec` lets you debug inside running containers
4. Pods are rarely created directly in production — Deployments manage them for you

---

## Cleanup

```bash
kubectl delete -f .
```