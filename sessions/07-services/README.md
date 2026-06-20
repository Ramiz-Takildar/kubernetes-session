# Session 07 — Services

## What You Will Learn

Services provide a stable network endpoint for a set of Pods. They handle load balancing across pod replicas and keep traffic flowing even as Pods are created or destroyed.

---

## Core Concepts

- Services decouple the network address from individual Pods
- The `selector` is the glue that connects a Service to its Pods
- `NodePort` exposes the Service on a static port on every node's IP
- Pods must have labels matching the Service selector for endpoints to be created
- `ClusterIP` is internal only; `NodePort` and `LoadBalancer` are for external access

---

## YAML Walkthrough

### pod-service.yaml

Backend Pod for the Service:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-service-pod
  labels:
    app: nginx-service-demo
spec:
  containers:
  - name: nginx
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
```

### service.yaml

The Service resource:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-service-demo
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: NodePort
```

| Field | Meaning |
|-------|---------|
| `spec.selector` | Routes traffic to Pods with matching labels |
| `spec.ports[].port` | The port the Service listens on (cluster-internal) |
| `spec.ports[].targetPort` | The container port to forward traffic to |
| `spec.type: NodePort` | Exposes Service on each node's IP at a static port |

| Service Type | Use Case |
|--------------|----------|
| `ClusterIP` | Internal cluster communication only |
| `NodePort` | Exposes Service on a static port on each node |
| `LoadBalancer` | Exposes Service externally (cloud providers) |
| `ExternalName` | Maps Service to a DNS name |

---

## Step-by-Step Implementation

```bash
# 1. Apply the pod and service
cd sessions/07-services
kubectl apply -f .

# 2. Verify the Service
kubectl get service nginx-service
# Expected: TYPE = NodePort, CLUSTER-IP = <internal IP>

# 3. Check which pods are endpoints
kubectl get endpoints nginx-service
# Expected: IP of nginx-service-pod

# 4. Test via port-forward (works on any cluster)
kubectl port-forward svc/nginx-service 8080:80
# In another terminal:
curl http://localhost:8080
# Expected: Welcome to nginx!

# 5. Find the NodePort
kubectl get svc nginx-service -o jsonpath='{.spec.ports[0].nodePort}'
# Example output: 30001

# 6. (Optional) Test via NodePort on minikube/kind
# On minikube:
# minikube service nginx-service --url
# On kind:
# NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
# curl http://$NODE_IP:<nodePort>
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get service nginx-service` | Service exists with correct type and port |
| `kubectl get endpoints nginx-service` | Endpoints list contains the Pod IP |
| `kubectl port-forward svc/nginx-service 8080:80` | The nginx web server is accessible via the Service |

---

## Try This

After completing Session 06 (Deployment), scale the Deployment to 3 replicas. Then edit the Service selector to `app: nginx` and watch the endpoints update:

```bash
kubectl get endpoints nginx-service -w
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| No endpoints | Pod labels don't match the Service selector exactly | Check `kubectl get pods --show-labels` and compare |
| Connection refused | Pod may still be starting | Wait and retry |

---

## Key Takeaways

1. Services decouple frontend access from backend Pod lifecycle
2. The `selector` is the critical link between Services and Pods
3. `NodePort` is the easiest way to test external access on local clusters
4. Endpoints object shows which Pods are currently receiving traffic

---

## Cleanup

```bash
kubectl delete -f .
```