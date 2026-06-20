# Session 14 — Affinity & Anti-affinity Scheduling

## What You Will Learn

Node affinity and anti-affinity allow you to control exactly which node a Pod runs on. You can prefer or require certain node characteristics using labels.

---

## Core Concepts

- **NodeAffinity** controls which nodes a Pod can be scheduled on
- `requiredDuringSchedulingIgnoredDuringExecution` = hard requirement (must satisfy)
- `preferredDuringSchedulingIgnoredDuringExecution` = soft preference (best effort)
- Pods without affinity rules can land on any node
- Anti-affinity can spread replicas across nodes/zones for high availability

---

## YAML Walkthrough

### pod-no-affinity.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-affinity-demo
spec:
  containers:
  - name: nginx
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
```

| Field | Meaning |
|-------|---------|
| `metadata.name` | Unique name for this Pod |
| `spec.containers[].name` | Container name within the Pod |
| `spec.containers[].image` | Container image to run |
| `spec.containers[].ports[].containerPort` | Port to expose (informational) |

### pod-node-affinity.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: training-node
            operator: In
            values:
            - "true"
  containers:
  - name: nginx
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
```

| Field | Meaning |
|-------|---------|
| `spec.affinity.nodeAffinity` | Top-level affinity configuration |
| `spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution` | Hard requirement that must be met |
| `spec.affinity.nodeAffinity.nodeSelectorTerms[]` | List of terms; any term can match |
| `spec.affinity.nodeAffinity.matchExpressions[]` | Label selector expressions |
| `spec.affinity.nodeAffinity.key` | The node label key to check |
| `spec.affinity.nodeAffinity.operator: In` | Match if label value is in the list |
| `spec.affinity.nodeAffinity.values[]` | Accepted values for the label |

---

## Step-by-Step Implementation

```bash
cd sessions/14-affinity

# 1. Apply the Pod without affinity rules first (schedules anywhere)
kubectl apply -f pod-no-affinity.yaml
kubectl get pods no-affinity-demo

# 2. Get your node name
kubectl get nodes

# 3. Label the node with training-node=true
kubectl label node <NODE_NAME> training-node=true
# Example: kubectl label node minikube training-node=true

# 4. Apply the Pod with node affinity requirement
kubectl apply -f pod-node-affinity.yaml
kubectl get pods affinity-demo
# Expected: affinity-demo is Running (scheduled on labeled node)

# 5. Remove the label from the node
kubectl label node <NODE_NAME> training-node-
# Example: kubectl label node minikube training-node-

# 6. Delete the pending pod and re-apply to show it stays Pending
kubectl delete pod affinity-demo
kubectl apply -f pod-node-affinity.yaml
kubectl get pods affinity-demo
# Expected: affinity-demo is Pending (no node satisfies the requirement)

# 7. Add the label back to unblock scheduling
kubectl label node <NODE_NAME> training-node=true
kubectl get pods affinity-demo -w
# Expected: affinity-demo becomes Running after label is restored
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get pods` | Pod status (Running or Pending) |
| `kubectl get nodes --show-labels` | Node labels including training-node |
| `kubectl describe pod affinity-demo` | Events showing scheduling decision |
| `kubectl label node <NODE> training-node=true` | Label applied successfully |
| `kubectl label node <NODE> training-node-` | Label removed |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Pod stuck in `Pending` | No node matches the affinity rule | Verify node has the required label |
| `No nodes available` in events | Affinity requirement cannot be satisfied | Check node labels with `kubectl get nodes --show-labels` |
| Wrong node selected | No affinity rule constrains placement | Add nodeAffinity to the Pod spec |

---

## Key Takeaways

1. Node affinity rules control which nodes a Pod can be scheduled on
2. `requiredDuringSchedulingIgnoredDuringExecution` is a hard requirement
3. Without affinity rules, the scheduler places Pods on any available node
4. Labeling nodes is the prerequisite for affinity-based scheduling
5. Removing required labels causes new Pods to go Pending

---

## Cleanup

```bash
kubectl delete -f pod-node-affinity.yaml
kubectl delete -f pod-no-affinity.yaml
kubectl label node <NODE_NAME> training-node-
```