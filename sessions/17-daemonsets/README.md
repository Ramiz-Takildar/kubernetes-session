# Session 17 — DaemonSets

## What You Will Learn

DaemonSets ensure that a specific Pod runs on every node (or a subset of nodes) in your cluster, automatically scaling as nodes join or leave. They are the standard pattern for cluster-wide infrastructure services like log collection and monitoring. This session demonstrates how DaemonSets differ from Deployments and when to choose one over the other.

---

## Core Concepts

- A **DaemonSet** deploys exactly one Pod per matching node, automatically creating Pods when nodes join the cluster and garbage-collecting them when nodes are removed or drained
- Unlike Deployments, DaemonSets have **no `replicas` field** because their scale is derived directly from the node count, making them self-adjusting as the cluster grows or shrinks
- DaemonSets are the canonical pattern for **node-level infrastructure services** such as log shippers (Fluentd, Filebeat), monitoring agents (Prometheus Node Exporter), and security scanners that need host-level visibility
- CNI plugins like **Calico and Cilium** run as DaemonSets because every node must have the networking stack installed before regular Pods can connect to the network
- You can restrict a DaemonSet to specific nodes using `nodeSelector`, **nodeAffinity**, or **tolerations**, which is useful when only GPU nodes need drivers or tainted control-plane nodes need specialized agents
- When a node becomes unavailable, the DaemonSet does not reschedule its Pod elsewhere because the workload is inherently node-bound; this is a key semantic difference from ReplicaSets
- DaemonSet Pods can access the host filesystem and network namespace through `hostPath` volumes and `hostNetwork`, enabling deep system integration that regular application Pods typically avoid

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

1. DaemonSets guarantee exactly one Pod per matching node, not a fixed number of replicas
2. No `replicas` field is needed because scheduling is automatic and scales with the node count
3. DaemonSets are the standard pattern for cluster-wide infrastructure services like log collectors, monitoring agents, and CNI plugins
4. You can use `nodeSelector`, `nodeAffinity`, or `tolerations` to restrict DaemonSets to specific node types
5. DaemonSets automatically handle node addition and removal, creating and garbage-collecting Pods accordingly
6. DaemonSets are inherently node-bound: if a node fails, its DaemonSet Pod is not rescheduled to another node, unlike ReplicaSet-managed Pods

---

## Cleanup

```bash
kubectl delete -f daemonset.yaml
```