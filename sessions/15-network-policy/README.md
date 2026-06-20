# Session 15 — Network Policies

## What You Will Learn

Network Policies control traffic flow to and from Pods. They are the Kubernetes way to implement firewall rules at the pod level, allowing you to specify which pods can communicate with each other.

---

## Core Concepts

- Network Policies are namespaced resources
- They use pod selectors to target specific pods
- Policies are additive: if any policy allows traffic, it is allowed
- By default, all ingress/egress traffic is allowed if no policy exists
- A NetworkPolicy without ingress rules blocks all incoming traffic to selected pods
- **Not all CNI plugins support NetworkPolicy** (e.g., minikube with default driver). Check that your cluster CNI enforces network policies.

---

## Prerequisites

- A Kubernetes cluster with a CNI plugin that supports NetworkPolicy enforcement (Calico, Cilium, Weave Net, etc.)
- kubectl configured with access to the cluster
- **Note:** Minikube with the default `bridge` CNI does NOT enforce NetworkPolicy rules. Use `--cni=calico` or `--cni=cilium` flag when starting minikube if you need policy enforcement.

---

## YAML Walkthrough

### pods.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: netpol-demo
---
apiVersion: v1
kind: Pod
metadata:
  name: backend
  namespace: netpol-demo
  labels:
    app: backend
spec:
  containers:
  - name: nginx
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: netpol-demo
  labels:
    app: client
spec:
  containers:
  - name: busybox
    image: busybox:1.36
    command: ['sh', '-c', 'sleep 3600']
```

### networkpolicy.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
```

| Field | Meaning |
|-------|---------|
| `metadata.name` | Name of the NetworkPolicy |
| `spec.podSelector.matchLabels` | Which pods this policy applies to (backend pod) |
| `spec.policyTypes` | Types of traffic this policy controls (Ingress) |
| `spec.ingress[].from[].podSelector` | Only allow traffic from pods with label `app: client` |
| `spec.ingress[].ports` | Optional: specify allowed ports (empty = all ports) |

---

## Step-by-Step Implementation

```bash
# 1. Create the namespace and pods
cd sessions/15-network-policy
kubectl apply -f pods.yaml

# 2. Verify both pods are running
kubectl get pods -n netpol-demo
# Expected: backend Running, client Running

# 3. Get the backend pod IP
kubectl get pod backend -n netpol-demo -o wide
# Note the IP address (e.g., 10.244.1.5)

# 4. Test connectivity FROM client TO backend WITHOUT policy
# First, exec into the client pod
kubectl exec -n netpol-demo client -it -- sh

# Inside the client pod, run:
wget -qO- http://<backend-ip>:80
# Expected: HTML response from nginx welcome page

# Exit the client pod
exit

# 5. Apply the NetworkPolicy
kubectl apply -f networkpolicy.yaml

# 6. Verify the NetworkPolicy was created
kubectl get networkpolicy -n netpol-demo
# Expected: allow-client displayed

# 7. Describe the policy to see its details
kubectl describe networkpolicy allow-client -n netpol-demo

# 8. Test connectivity again FROM client TO backend WITH policy
kubectl exec -n netpol-demo client -it -- sh

# Inside the client pod, run:
wget -qO- http://<backend-ip>:80
# Expected: HTML response (client has app:client label which is allowed)

# Exit the client pod
exit

# 9. Test connectivity FROM an UNLABELED pod to backend
# Create a temporary pod without the client label
kubectl run test-pod -n netpol-demo --image=busybox:1.36 --restart=Never -- /bin/sh -c "sleep 3600"

# Wait for it to be running
kubectl get pod test-pod -n netpol-demo

# Exec into the test pod
kubectl exec -n netpol-demo test-pod -it -- sh

# Inside the test pod, run:
wget -qO- http://<backend-ip>:80
# Expected: TIMEOUT or connection refused (test-pod does NOT have app:client label)

# Exit the test pod
exit

# 10. Cleanup the test pod
kubectl delete pod test-pod -n netpol-demo
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get pods -n netpol-demo` | Both pods are Running |
| `kubectl get networkpolicy -n netpol-demo` | NetworkPolicy exists |
| `kubectl describe networkpolicy allow-client -n netpol-demo` | Policy is correctly configured |
| `kubectl exec client -n netpol-demo -- wget -qO- http://<ip>:80` | Connectivity from client pod works |
| `kubectl exec test-pod -n netpol-demo -- wget -qO- http://<ip>:80` | Unlabeled pod is blocked (if CNI supports policy) |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `wget` hangs or times out | NetworkPolicy blocks traffic (expected behavior if CNI supports it) | Verify CNI supports NetworkPolicy |
| `No resources found` for networkpolicy | CNI does not enforce NetworkPolicy | Use a supported CNI (Calico, Cilium) |
| Policy does not block traffic | Cluster CNI does not support NetworkPolicy enforcement | Minikube: `minikube delete && minikube start --cni=calico` |
| `client pod` cannot reach `backend` after policy | Client pod label does not match policy selector | Verify client has `app: client` label |

---

## Key Takeaways

1. Network Policies control traffic flow to/from Pods at the Kubernetes level
2. Policies use pod selectors (`podSelector`) to target specific pods
3. By default, no traffic is blocked — you must create a policy to restrict access
4. A policy with `Ingress` and no `from` rules blocks ALL ingress traffic
5. Only pods matching the `from.podSelector` can send traffic to protected pods
6. **Not all CNI plugins enforce NetworkPolicy** — verify your CNI supports it
7. Network Policies are additive — if any policy allows traffic, it is permitted

---

## Cleanup

```bash
kubectl delete -f networkpolicy.yaml
kubectl delete -f pods.yaml
```