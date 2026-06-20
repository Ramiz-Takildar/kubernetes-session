# Session 06 — Deployments

## What You Will Learn

Deployments provide declarative updates for Pods. They handle scaling, rolling updates, and self-healing (restarting failed Pods automatically).

---

## Core Concepts

- Deployments manage **ReplicaSets** which manage **Pods**
- Scaling is instant and declarative: change `replicas` and apply
- If a Pod dies, the Deployment automatically creates a replacement
- `selector.matchLabels` must match the pod template labels
- `RollingUpdate` strategy replaces pods gradually to avoid downtime

---

## YAML Walkthrough

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:stable-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

| Field | Meaning |
|-------|---------|
| `spec.replicas` | Desired number of pods |
| `spec.selector.matchLabels` | Which pods this Deployment manages (must match template) |
| `spec.template.metadata.labels` | Labels applied to every pod created by this Deployment |
| `spec.strategy.type: RollingUpdate` | Replace pods gradually, not all at once |
| `spec.strategy.rollingUpdate.maxUnavailable` | Max pods that can be unavailable during update |
| `spec.strategy.rollingUpdate.maxSurge` | Max extra pods that can exist during update |
| `spec.template.spec.containers[].resources.requests` | Guaranteed minimum resources |
| `spec.template.spec.containers[].resources.limits` | Maximum allowed resources |

---

## Step-by-Step Implementation

```bash
# 1. Apply the Deployment
cd sessions/06-deployments
kubectl apply -f deployment.yaml

# 2. Verify the Deployment
kubectl get deployment nginx-deployment
# Expected: READY = 3/3

# 3. Verify the ReplicaSet was created
kubectl get replicasets -l app=nginx

# 4. List all pods managed by the Deployment
kubectl get pods -l app=nginx
# Expected: 3 pods, all Running

# 5. Watch the rollout
kubectl rollout status deployment/nginx-deployment
# Expected: deployment "nginx-deployment" successfully rolled out

# 6. Scale up to 5 replicas
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods -l app=nginx
# Expected: 5 pods

# 7. Scale down to 2 replicas
kubectl scale deployment nginx-deployment --replicas=2
kubectl get pods -l app=nginx
# Expected: 2 pods (Kubernetes deletes extras gracefully)

# 8. Verify self-healing: delete a pod manually
kubectl delete pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl get pods -l app=nginx
# Expected: A new pod is automatically created to maintain 2 replicas

# 9. View Deployment history
kubectl rollout history deployment/nginx-deployment
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get deployment nginx-deployment` | Deployment exists with correct replica count |
| `kubectl get replicasets -l app=nginx` | ReplicaSet was created by the Deployment |
| `kubectl get pods -l app=nginx` | Pods are Running with the correct label |
| `kubectl rollout status deployment/nginx-deployment` | Latest rollout completed successfully |
| `kubectl scale deployment nginx-deployment --replicas=5` | Scaling command works — 5 pods appear |
| `kubectl delete pod <name>` + `kubectl get pods -l app=nginx` | Self-healing: replacement pod is created |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Pods stuck in `Pending` | Node has insufficient resources | Check `kubectl describe nodes` |
| `ImagePullBackOff` | Wrong image name or tag | Verify the image is accessible |

---

## Key Takeaways

1. Deployments manage ReplicaSets which manage Pods
2. Scaling is instant: `kubectl scale deployment <name> --replicas=N`
3. If a Pod dies, the Deployment creates a replacement automatically
4. Resource limits prevent a single pod from consuming all node resources
5. `RollingUpdate` keeps the application available during updates

---

## Cleanup

```bash
kubectl delete -f deployment.yaml
```