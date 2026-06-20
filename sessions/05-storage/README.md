# Session 05 — Persistent Storage

## What You Will Learn

Containers are ephemeral — data is lost when they restart. PersistentVolumes (PV) and PersistentVolumeClaims (PVC) solve this by providing durable storage.

---

## Core Concepts

- **PersistentVolume (PV)** = cluster admin provisions storage
- **PersistentVolumeClaim (PVC)** = user/app requests storage
- `hostPath` works only on single-node clusters (minikube, kind, Docker Desktop) — never use in production multi-node clusters
- `initContainers` run before app containers and are perfect for setup tasks
- A PVC survives Pod restarts — data persists as long as the PVC exists

---

## YAML Walkthrough

### pv.yaml

Physical storage provisioned by the cluster admin:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data
```

### pvc.yaml

Request for storage by a user/app:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

### pod-pvc.yaml

Pod that uses the PVC:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pvc-pod
spec:
  initContainers:
  - name: init-html
    image: busybox:1.36
    command: ['sh', '-c', 'echo "<h1>Hello from Persistent Storage!</h1>" > /usr/share/nginx/html/index.html']
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: nginx-storage
  containers:
  - name: nginx
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: nginx-storage
  volumes:
  - name: nginx-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

| Field | Meaning |
|-------|---------|
| `spec.capacity.storage` | How much storage the PV offers |
| `spec.accessModes` | How the volume can be mounted (ReadWriteOnce = single node) |
| `spec.hostPath.path` | Path on the host node (training only) |
| `spec.resources.requests.storage` | How much storage the PVC requests |
| `spec.initContainers` | Containers that run before the main container |
| `spec.volumes[].persistentVolumeClaim.claimName` | Links the Pod to the PVC |

---

## Step-by-Step Implementation

```bash
# 1. Apply all storage manifests
cd sessions/05-storage
kubectl apply -f .

# 2. Verify PV status
kubectl get pv my-pv
# Expected: STATUS = Available (until claimed)

# 3. Verify PVC is bound
kubectl get pvc my-pvc
# Expected: STATUS = Bound, VOLUME = my-pv

# 4. Verify pod is running
kubectl get pod nginx-pvc-pod

# 5. Check that the initContainer wrote the file
kubectl exec nginx-pvc-pod -- cat /usr/share/nginx/html/index.html
# Expected: <h1>Hello from Persistent Storage!</h1>

# 6. Test the web server
kubectl port-forward nginx-pvc-pod 8080:80
# In another terminal:
curl http://localhost:8080
# Expected: Hello from Persistent Storage!

# 7. Prove persistence: delete the pod and recreate it
kubectl delete pod nginx-pvc-pod
kubectl apply -f pod-pvc.yaml
kubectl wait --for=condition=Ready pod/nginx-pvc-pod --timeout=60s

# 8. The file should still exist because the PVC survived
kubectl exec nginx-pvc-pod -- cat /usr/share/nginx/html/index.html
# Expected: Still shows "Hello from Persistent Storage!"
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get pv my-pv` | PV exists |
| `kubectl get pvc my-pvc` | PVC is `Bound` to `my-pv` |
| `kubectl get pod nginx-pvc-pod` | Pod is `Running` |
| `kubectl exec nginx-pvc-pod -- cat /usr/share/nginx/html/index.html` | InitContainer wrote the file to the PV |
| Step 7 (delete and re-create pod) | File still present — data survives Pod restarts |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| PVC stuck in `Pending` | PV does not satisfy the claim | Check `kubectl describe pvc my-pvc` for events |
| Pod stuck in `Init:0/1` | busybox image may still be pulling | Wait a moment and retry |

---

## Key Takeaways

1. PV = cluster admin provisioned storage; PVC = user request for storage
2. `hostPath` is for training only — never use in production multi-node clusters
3. InitContainers run before app containers and are perfect for setup tasks
4. PVCs survive Pod restarts — data is persistent

---

## Cleanup

```bash
kubectl delete -f .
```