# Session 10 — Taints and Tolerations

## What You Will Learn

Taints and tolerations are Kubernetes scheduling mechanisms that control which Pods can run on which nodes. In this session, you will apply a taint to a node, observe how it repels unqualified Pods, and use a toleration to allow specific Pods to schedule anyway.

---

## Core Concepts

- **Taints** are applied to nodes to repel Pods that do not have a matching toleration, effectively partitioning the cluster into general-purpose and specialized node pools without physically separating them.
- **Tolerations** are declared in a Pod spec and tell the scheduler that the Pod is allowed to run on a node with a matching taint, enabling workloads to opt into specialized hardware or restricted environments.
- The **`NoSchedule` effect** prevents the scheduler from placing a Pod on a tainted node unless the Pod explicitly tolerates that taint, which is the standard way to dedicate nodes to specific workloads like GPU training or database hosting.
- A toleration must match the taint's **key, operator, value, and effect** exactly: if any of these fields do not align, the scheduler treats the Pod as incompatible and will not place it on the node.
- Taints are useful for **dedicated node pools** such as GPU nodes, memory-optimized instances, or training environments, ensuring that expensive or constrained resources are reserved for the workloads that actually need them.
- Removing a taint from a node returns it to the general scheduling pool, and any pending Pods that were previously blocked will be scheduled automatically without needing to modify the Pod manifests.
- Taints and tolerations work alongside labels and node selectors: while **node affinity** attracts Pods to nodes they prefer, taints actively repel Pods that do not belong, providing a stronger guarantee of node specialization.

---

## YAML Walkthrough

### no-toleration-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration-pod
  labels:
    app: taint-demo
spec:
  containers:
  - name: nginx
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
```

| Field | Meaning |
|-------|---------|
| `metadata.name` | The pod's name; without a matching toleration it will stay `Pending` |
| `metadata.labels.app: taint-demo` | Shared label used to query both pods |
| `spec.containers[].image: nginx:stable-alpine` | Lightweight NGINX image on Alpine Linux |

---

### toleration-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: toleration-pod
  labels:
    app: taint-demo
spec:
  containers:
  - name: nginx
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "training"
    effect: "NoSchedule"
```

| Field | Meaning |
|-------|---------|
| `metadata.name` | The pod's name; with a matching toleration it will be scheduled |
| `spec.tolerations[].key` | Must match the taint key applied to the node |
| `spec.tolerations[].operator: Equal` | The value must match the taint value exactly |
| `spec.tolerations[].value: training` | Must equal the taint value |
| `spec.tolerations[].effect: NoSchedule` | Must match the taint effect — without a match the toleration is ignored |

---

## Step-by-Step Implementation

```bash
# 1. Get the node name
NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
echo "Node: $NODE_NAME"

# 2. Apply taint to the node
kubectl taint nodes $NODE_NAME dedicated=training:NoSchedule
# Expected: node/<node-name> tainted

# 3. Apply the pod with no toleration
kubectl apply -f no-toleration-pod.yaml

# 4. Verify it stays Pending (no matching toleration)
kubectl get pod no-toleration-pod -o wide
# Expected: STATUS = Pending, node = <none>

# 5. Apply the pod with a matching toleration
kubectl apply -f toleration-pod.yaml

# 6. Verify it runs (tolerates the taint)
kubectl get pod toleration-pod -o wide
# Expected: STATUS = Running, NODE = <node-name>

# 7. Show why the no-toleration pod cannot schedule
kubectl describe pod no-toleration-pod
# Expected: Events show "node(s) had taints that the pod didn't tolerate"

# 8. Remove the taint
kubectl taint nodes $NODE_NAME dedicated=training:NoSchedule-
# Expected: node/<node-name> untainted

# 9. Verify the no-toleration pod is now scheduled
kubectl get pods -l app=taint-demo
# Expected: Both pods are Running
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get pod no-toleration-pod -o wide` | Without a toleration the pod stays `Pending` |
| `kubectl get pod toleration-pod -o wide` | With a matching toleration the pod is `Running` |
| `kubectl describe pod no-toleration-pod` | Events show the taint as the reason for rejection |
| `kubectl taint nodes $NODE_NAME dedicated=training:NoSchedule-` | Removes the taint so all pods can schedule |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Pod stuck in `Pending` after applying taint | Pod has no toleration matching the taint | Add a toleration with the same key, operator, value, and effect |
| `kubectl taint` command fails with "not found" | Wrong node name | Run `kubectl get nodes` to get the correct name |
| Pod is still `Pending` after removing taint | Cluster has no available resources | Check `kubectl describe nodes` for allocatable capacity |

---

## Key Takeaways

1. Taints repel Pods; tolerations allow Pods to be scheduled on tainted nodes
2. A toleration must match the taint's key, operator, value, and effect exactly
3. `NoSchedule` prevents scheduling unless a matching toleration exists
4. Taints are the standard way to dedicate nodes to specific workloads (e.g., GPU, training)
5. Removing a taint returns the node to the general pool and unblocks pending Pods
6. Taints complement node affinity by repelling unwanted Pods rather than attracting preferred ones

---

## Cleanup

```bash
kubectl delete -f no-toleration-pod.yaml
kubectl delete -f toleration-pod.yaml
kubectl taint nodes $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') dedicated=training:NoSchedule- 2>/dev/null || true
```