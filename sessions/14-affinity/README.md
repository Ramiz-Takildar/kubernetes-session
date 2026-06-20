# Session 14 — Affinity & Anti-affinity Scheduling

## What You Will Learn

Node affinity and anti-affinity give you fine-grained control over Pod placement by matching Pods to nodes with specific labels. You can require certain node characteristics for compliance or performance, or simply prefer them to optimize resource utilization. This session shows how to use labels and affinity rules to place workloads exactly where they should run.

---

## Core Concepts

- **Node affinity** tells the Kubernetes scheduler which nodes are acceptable for a Pod by evaluating node labels, enabling workload placement based on hardware, zone, or custom organizational tags
- `requiredDuringSchedulingIgnoredDuringExecution` creates a **hard requirement** that must be satisfied; if no matching node exists, the Pod remains in Pending state until one becomes available
- `preferredDuringSchedulingIgnoredDuringExecution` expresses a **soft preference** that the scheduler tries to honor but will ignore if no suitable node exists, providing flexibility without blocking deployment
- Pods without any affinity rules can be scheduled on **any available node**, which is efficient for stateless workloads but risky for services that need hardware proximity or regulatory isolation
- **Anti-affinity** spreads Pods across nodes or availability zones, preventing multiple replicas from landing on the same node and improving fault tolerance during node failures
- Affinity is often combined with **taints and tolerations** to reserve specialized nodes (like GPU or SSD-equipped nodes) for specific Pods, creating dedicated node pools within a cluster
- Labeling nodes is the prerequisite for all affinity-based scheduling; without consistent and accurate node labels, affinity rules have no data to evaluate and are effectively ignored

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

1. Node affinity rules control which nodes a Pod can be scheduled on by matching Pod requirements against node labels
2. `requiredDuringSchedulingIgnoredDuringExecution` is a hard requirement that blocks scheduling if no matching node exists
3. `preferredDuringSchedulingIgnoredDuringExecution` is a soft preference that the scheduler honors when possible
4. Without affinity rules, Pods can land on any available node, which may violate performance or compliance needs
5. Labeling nodes is the prerequisite for all affinity-based scheduling, and removing required labels causes new Pods to go Pending
6. Combining affinity rules with taints and tolerations creates dedicated node pools for specialized workloads like GPU or high-memory applications

---

## Cleanup

```bash
kubectl delete -f pod-node-affinity.yaml
kubectl delete -f pod-no-affinity.yaml
kubectl label node <NODE_NAME> training-node-
```