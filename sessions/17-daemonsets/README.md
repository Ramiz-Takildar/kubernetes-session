# Session 17 — DaemonSets

## What You Will Learn

DaemonSets ensure that all (or specific) nodes run a copy of a specific Pod. As nodes are added to the cluster, Pods are automatically added. When nodes are removed, those Pods are garbage collected. Unlike Deployments which ensure N replicas regardless of node count, DaemonSets guarantee one pod per node.

---

## Core Concepts

- A DaemonSet creates **one Pod per node** automatically (not N replicas — one per node)
- Ideal for system-level workloads: log collectors, monitoring agents, log forwarding
- If a node is added to the cluster, the DaemonSet automatically deploys a pod there
- If a node is removed, the pod is garbage collected
- Unlike Deployments, there is no `replicas` field — scheduling is automatic
- Use `nodeSelector`, `tolerations`, or `nodeAffinity` to run on specific nodes

---

## YAML Walkthrough

### daemonset.yaml

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  labels:
    app: nginx-ds
spec:
  selector:
    matchLabels:
      app: nginx-ds
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: nginx
        image: nginx:stable-alpine
        ports:
        - containerPort: 80
```

| Field | Meaning |
|-------|---------|
| `spec.selector.matchLabels` | Which pods this DaemonSet manages (must match template) |
| `spec.template.metadata.labels` | Labels applied to every pod created by this DaemonSet |
| `spec.template.spec.containers[].name` | Container name inside the pod |
| `spec.template.spec.containers[].image` | Container image to run |
| `spec.template.spec.containers[].ports` | Ports to expose on the container |

Note: There is **no `spec.replicas`** field. DaemonSets automatically schedule one pod per matching node.

---

## Step-by-Step Implementation

```bash
# 1. Apply the DaemonSet
cd sessions/17-daemonsets
kubectl apply -f daemonset.yaml

# 2. Verify the DaemonSet
kubectl get daemonset nginx-ds
# Expected: DESIRED = CURRENT = READY matches your node count

# 3. Verify one pod per node
kubectl get pods -l app=nginx-ds -o wide
# Expected: One pod per node in the cluster

# 4. Describe the DaemonSet to show desired vs scheduled
kubectl describe daemonset nginx-ds
# Look for: "Desired Number of Nodes: <N>" and "Number of Nodes Scheduled: <N>"
# These should match if all nodes are healthy

# 5. Use case: Log collection example
# DaemonSets are commonly used for log collectors (e.g., fluentd, filebeat)
# They run on every node to collect logs without manual pod placement

# 6. Use case: Monitoring agents
# Prometheus node exporter, Datadog agent, CloudWatch agent all use DaemonSets
# They need to run on every node to gather node-level metrics

# 7. Use case: Networking solutions
# CNI plugins like Calico or Weave run as DaemonSets
# They need to be present on every node to manage pod networking
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get daemonset nginx-ds` | DaemonSet exists with correct desired count |
| `kubectl get pods -l app=nginx-ds -o wide` | One pod per node is running |
| `kubectl describe daemonset nginx-ds` | Shows desired nodes vs scheduled nodes |
| `kubectl get nodes` | Lists nodes to confirm pod count matches |
| `kubectl rollout history daemonset/nginx-ds` | View DaemonSet revision history |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Pods stuck in `Pending` | No node matches nodeSelector/tolerations | Check node labels and DaemonSet spec |
| Pods not on all nodes | Taints on nodes without tolerations | Add matching tolerations to pod spec |
| DaemonSet not creating pods | API server connectivity or RBAC issue | Check `kubectl describe daemonset` events |

---

## Key Takeaways

1. DaemonSets guarantee **one pod per node**, not N replicas
2. No `replicas` field needed — scheduling is automatic based on node count
3. Perfect for cluster-wide services: log collectors, monitoring, networking
4. Use `nodeSelector` or `tolerations` to target specific nodes
5. DaemonSets automatically handle node addition and removal

---

## Cleanup

```bash
kubectl delete -f daemonset.yaml
```