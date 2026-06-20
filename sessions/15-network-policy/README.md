# Session 15 — Network Policies

## What You Will Learn

Network Policies are the Kubernetes-native way to enforce firewall rules between Pods, controlling which services can communicate at the pod level. Unlike external firewalls, they operate inside the cluster and follow the same declarative model as other Kubernetes resources. This session teaches you how to isolate workloads, enforce zero-trust networking, and verify that policies are actually blocking or allowing traffic.

---

## Core Concepts

- **Network Policies** are namespaced resources that define ingress and egress rules for Pods, acting as a micro-firewall that operates at the pod level rather than the host level
- They use **pod selectors** (`podSelector`) and namespace selectors to target specific groups of Pods, enabling fine-grained segmentation without hard-coding IP addresses
- Policy behavior is **additive**: if any policy in a namespace allows traffic, it is permitted, which means you must explicitly define all allowed sources to achieve true isolation
- By default, Kubernetes clusters allow **all ingress and egress traffic** unless a NetworkPolicy is created; this permissive default makes clusters easy to use but insecure without explicit rules
- A NetworkPolicy that selects Pods but contains no ingress rules effectively **blocks all incoming traffic** to those Pods, providing a deny-by-default posture for sensitive services
- **Not all CNI plugins enforce NetworkPolicy** rules; the policy object may exist in the API without any effect if the underlying network plugin (like the default minikube bridge CNI) does not implement them
- Network Policies secure east-west traffic between Pods in addition to north-south traffic, making them essential for zero-trust architectures where every internal connection is explicitly authorized

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

1. Network Policies control traffic flow to and from Pods at the Kubernetes API level, but only work if the underlying CNI plugin enforces them
2. Policies use `podSelector` and namespace selectors to target specific groups of Pods without hard-coding IP addresses
3. By default, all ingress and egress traffic is allowed unless a NetworkPolicy restricts it
4. A policy that selects Pods but defines no ingress rules effectively blocks all incoming traffic to those Pods
5. Only Pods matching the `from.podSelector` can send traffic to protected Pods, creating explicit allow-lists
6. Network Policy behavior is additive: if any policy allows traffic, it is permitted, so complete isolation requires comprehensive rule coverage
7. Network Policies secure both east-west traffic between Pods and north-south traffic, forming the foundation of zero-trust cluster networking

---

## Cleanup

```bash
kubectl delete -f networkpolicy.yaml
kubectl delete -f pods.yaml
```