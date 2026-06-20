# Session 11 — Gateway API

## What You Will Learn

Gateway API is the next-generation Kubernetes ingress and service networking model, replacing Ingress with more expressive, role-oriented resources. In this session, you will install the Gateway API CRDs, create a Gateway, GatewayClass, and HTTPRoute, and observe how they declaratively route HTTP traffic to a backend Service.

---

## Prerequisites

Before applying the manifests, install the Gateway API CRDs:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
```

> **Note:** Installing the CRDs registers the `Gateway`, `GatewayClass`, and `HTTPRoute` types in your cluster. This lets you create the resources, but the actual traffic routing requires a Gateway controller (e.g., NGINX Gateway Fabric or Envoy Gateway) to be running.
>
> Without a controller installed, the `Gateway` resource will exist but its status will show `Accepted=False`. This is **expected** in a training cluster and is intentional — it demonstrates the API model without requiring a full controller setup.
>
> If you want a fully working data plane, install a controller after this session. For now, the resource model and YAML structure are what matters.

---

## Core Concepts

- **GatewayClass** is a cluster-wide template that defines a class of load balancers, similar to a StorageClass for storage provisioners; it lets infrastructure providers offer different gateway implementations without changing application manifests.
- **Gateway** is a namespaced resource that represents a running traffic entry point, binding to a GatewayClass and declaring which ports, protocols, and TLS settings it accepts traffic on.
- **HTTPRoute** attaches to a Gateway via `parentRefs` and declares routing rules such as host matching, path prefix matching, and header-based filtering, decoupling traffic routing logic from the infrastructure that terminates it.
- Gateway API introduces **role-oriented resource design**: infrastructure teams manage GatewayClasses and Gateways, while application developers manage HTTPRoutes, eliminating the permission and coordination conflicts common with the monolithic Ingress resource.
- Unlike Ingress, which conflates listener configuration and routing rules into a single object, Gateway API separates concerns into distinct resources, making it possible to share a Gateway across multiple teams while keeping each team's routes independent and secure.
- The **`Accepted=False` status** on a Gateway is expected when no Gateway controller is installed, because the CRDs only register the resource schema; an actual controller must reconcile the Gateway and implement the data plane.
- Gateway API supports **multiple protocol types** beyond HTTP, including HTTPS with TLS termination, TCP, UDP, and gRPC, making it a unified evolution path for all ingress and service mesh traffic management.

---

## YAML Walkthrough

### backend.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gateway-demo
---
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
  namespace: gateway-demo
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
kind: Service
metadata:
  name: backend-svc
  namespace: gateway-demo
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
```

| Field | Meaning |
|-------|---------|
| `kind: Namespace` | Isolates all demo resources in `gateway-demo` |
| `metadata.labels.app: backend` | Selector label used by the Service |
| `spec.selector.app: backend` | Service routes traffic to pods with this label |
| `spec.ports[].port: 80` | Service port |
| `spec.ports[].targetPort: 80` | Pod container port the Service forwards to |

### gateway.yaml

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gateway
  namespace: gateway-demo
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

| Field | Meaning |
|-------|---------|
| `apiVersion: gateway.networking.k8s.io/v1` | Gateway API version |
| `spec.gatewayClassName: nginx` | References the GatewayClass named `nginx` |
| `spec.listeners[].name: http` | Named listener for reference in HTTPRoutes |
| `spec.listeners[].protocol: HTTP` | Protocol this listener accepts |
| `spec.listeners[].port: 80` | Port the Gateway binds to |

### httproute.yaml

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
  namespace: gateway-demo
spec:
  parentRefs:
  - name: demo-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: backend-svc
      port: 80
```

| Field | Meaning |
|-------|---------|
| `spec.parentRefs[].name` | Attaches this HTTPRoute to the named Gateway |
| `spec.rules[].matches[].path.type: PathPrefix` | Match type: prefix-based path matching |
| `spec.rules[].matches[].path.value: /` | Match all paths |
| `spec.rules[].backendRefs[].name` | Service to forward matched traffic to |
| `spec.rules[].backendRefs[].port` | Port on the backend Service |

---

## Step-by-Step Implementation

```bash
# 1. Install the Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml

# 2. Apply the backend Namespace, Pod, and Service
kubectl apply -f backend.yaml

# 3. Verify the backend pod is Running
kubectl get pods -n gateway-demo
# Expected: backend-pod Running

# 4. Apply the Gateway resource
kubectl apply -f gateway.yaml

# 5. Describe the Gateway to observe its status
kubectl describe gateway demo-gateway -n gateway-demo
# Expected: Conditions show Accepted=False if no controller is installed
# This is expected in a training cluster without a Gateway controller

# 6. Apply the HTTPRoute
kubectl apply -f httproute.yaml

# 7. Verify the HTTPRoute was created
kubectl get httproute -n gateway-demo
# Expected: NAME        AGE / RESOURCES
#           demo-route  ...

# 8. Show detailed HTTPRoute status
kubectl describe httproute demo-route -n gateway-demo
# Expected: ParentRefs shows reference to demo-gateway

# 9. List all resources in the gateway-demo namespace
kubectl get all -n gateway-demo
# Expected: pod/backend-pod, service/backend-svc
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get pods -n gateway-demo` | Backend pod is Running with label `app=backend` |
| `kubectl get gateway -n gateway-demo` | Gateway `demo-gateway` was created |
| `kubectl describe gateway demo-gateway -n gateway-demo` | Gateway status visible; Accepted=False without a controller |
| `kubectl get httproute -n gateway-demo` | HTTPRoute `demo-route` was created |
| `kubectl describe httproute demo-route -n gateway-demo` | HTTPRoute attached to demo-gateway via parentRefs |
| `kubectl get all -n gateway-demo` | Namespace contains all demo resources |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Gateway shows `Accepted=False` | No Gateway controller installed | This is expected without a controller; the resource model still works |
| `error: unable to recognize "gateway.yaml"`: no matches for kind "Gateway" | Gateway API CRDs not installed | Run `kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml` |
| Backend pod stuck in `Pending` | Node has insufficient resources | Check `kubectl describe nodes` |
| `ImagePullBackOff` | Image not accessible | Verify `nginx:stable-alpine` is available |

---

## Key Takeaways

1. Gateway API is the successor to Ingress, offering a more expressive and role-oriented model
2. `GatewayClass` is a cluster-scoped template that a controller implements
3. `Gateway` declares where traffic enters; `HTTPRoute` declares how it gets routed
4. `parentRefs` binds an HTTPRoute to a specific Gateway listener
5. `Accepted=False` is expected without a Gateway controller — the API model is still demonstrable
6. Gateway API separates infrastructure concerns from developer routing rules

---

## Cleanup

```bash
kubectl delete -f httproute.yaml
kubectl delete -f gateway.yaml
kubectl delete -f backend.yaml
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
```