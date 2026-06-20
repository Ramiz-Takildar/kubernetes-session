# Session 07 — Services

## What You Will Learn

Services provide a stable network endpoint for a set of Pods, handling load balancing across replicas and keeping traffic flowing even as Pods are created or destroyed. In this session, you will create a Service, understand how selectors bind it to Pods, and explore the differences between ClusterIP, NodePort, and LoadBalancer types.

---

## Core Concepts

- Services solve the **Pod IP volatility problem**: Pods are ephemeral and their IPs change on every restart, so a Service creates a stable virtual IP and DNS name that clients can rely on regardless of which backend Pods are running.
- The **`selector`** is the critical link between a Service and its Pods: the Service controller continuously watches for Pods whose labels match the selector and adds their IPs to the Endpoints object automatically.
- **ClusterIP** is the default Service type and provides an internal-only IP address that is reachable only from within the cluster, making it ideal for microservice-to-microservice communication.
- **NodePort** exposes the Service on a static port across every node in the cluster, allowing external traffic to reach the Service directly via `<NodeIP>:<NodePort>` without requiring a cloud load balancer.
- **LoadBalancer** integrates with cloud-provider APIs to provision an external load balancer with a public IP, abstracting NodePort management and providing a single entry point for production traffic.
- The **Endpoints** object tracks which Pod IPs are currently backing a Service, and it is updated dynamically as Pods scale, restart, or fail, ensuring traffic is never routed to dead backends.
- **Label matching** must be exact: if a Pod's labels do not match the Service selector, the Pod will not receive traffic, which is a common source of "connection refused" errors in practice.

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

1. Services decouple frontend access from backend Pod lifecycle by providing a stable virtual IP
2. The `selector` is the critical link between Services and Pods; labels must match exactly
3. `ClusterIP` is for internal cluster communication only
4. `NodePort` is the easiest way to test external access on local clusters
5. The Endpoints object shows which Pods are currently receiving traffic and updates automatically
6. Pod labels that do not match the Service selector will cause the Service to have zero endpoints

---

## Cleanup

```bash
kubectl delete -f .
```