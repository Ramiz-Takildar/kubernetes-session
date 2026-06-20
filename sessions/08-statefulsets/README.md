# Session 08 — StatefulSets

## What You Will Learn

StatefulSets are designed for applications that require stable identity, stable storage, and ordered deployment, such as databases and message queues. In this session, you will create a StatefulSet, observe its predictable naming and sequential scaling behavior, and understand when to choose a StatefulSet over a Deployment.

---

## Core Concepts

- StatefulSets are the right abstraction for **stateful applications** like databases, caches, and message brokers because they guarantee stable network identity and ordered lifecycle management that Deployments cannot provide.
- Each StatefulSet pod receives a **predictable name** following the pattern `<statefulset-name>-<ordinal-index>` (e.g., `mysql-sts-0`), enabling application code and operators to know exactly which instance they are addressing without querying the API server.
- A **headless Service** (`clusterIP: None`) is required for StatefulSets because it causes DNS to return the individual Pod IPs directly rather than a single virtual IP, enabling direct peer-to-peer communication between replicas.
- StatefulSets deploy Pods **sequentially** (0, 1, 2, ...) and delete them in **reverse order**, ensuring that distributed stateful systems can bootstrap clusters correctly and shut down gracefully without data corruption.
- The `serviceName` field in the StatefulSet spec must match the name of the headless Service, and this pairing is what enables stable DNS entries such as `<pod-name>.<service-name>.<namespace>.svc.cluster.local`.
- Unlike Deployments, which treat all replicas as identical and interchangeable, StatefulSets assign each pod a **unique ordinal identity** and can optionally pair it with a dedicated PersistentVolumeClaim, ensuring data stays with its pod even after rescheduling.
- StatefulSets are not necessary for stateless web applications or APIs: if your workload does not care about pod identity, ordering, or persistent per-pod state, a Deployment is simpler and more flexible.

---

## YAML Walkthrough

### statefulset-service.yaml

Headless Service (required for StatefulSet DNS):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
```

### statefulset.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-sts
spec:
  serviceName: mysql-service
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password123
```

| Field | Meaning |
|-------|---------|
| `spec.serviceName` | Must match the name of the headless Service |
| `spec.clusterIP: None` | Makes this a headless Service — DNS returns Pod IPs directly |
| StatefulSet `metadata.name` | Forms the prefix of predictable Pod names: `<name>-0`, `<name>-1`, ... |

---

## Step-by-Step Implementation

```bash
# 1. Apply the StatefulSet and its headless Service
cd sessions/08-statefulsets
kubectl apply -f .

# 2. Verify the StatefulSet
kubectl get statefulset mysql-sts
# Expected: READY = 1/1

# 3. Verify the pod name is predictable
kubectl get pods -l app=mysql
# Expected: mysql-sts-0  (not a random hash!)

# 4. Verify the headless service
kubectl get svc mysql-service
# Expected: CLUSTER-IP = None

# 5. Verify stable network identity
kubectl exec mysql-sts-0 -- sh -c 'echo $HOSTNAME'
# Expected: mysql-sts-0

# 6. Test DNS resolution inside the cluster
kubectl run -it --rm debug --image=busybox:1.36 --restart=Never -- nslookup mysql-sts-0.mysql-service.default.svc.cluster.local
# Expected: IP address of mysql-sts-0

# 7. Verify MySQL is running inside the StatefulSet pod
kubectl exec mysql-sts-0 -- mysql -uroot -ppassword123 -e "SHOW STATUS LIKE 'Uptime';"
# Expected: Uptime value in seconds
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get statefulset mysql-sts` | StatefulSet exists with READY 1/1 |
| `kubectl get pods -l app=mysql` | Pod name is `mysql-sts-0` (predictable, not random) |
| `kubectl get svc mysql-service` | CLUSTER-IP is `None` (headless) |
| `kubectl exec mysql-sts-0 -- sh -c 'echo $HOSTNAME'` | Hostname matches the predictable Pod name |
| `kubectl run debug --image=busybox:1.36 -- nslookup mysql-sts-0.mysql-service...` | DNS resolves to the Pod IP |
| `kubectl exec mysql-sts-0 -- mysql -uroot -ppassword123 -e "SHOW STATUS LIKE 'Uptime';"` | MySQL is running with correct credentials |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Pod stuck in `Pending` | If using `volumeClaimTemplates`, ensure a StorageClass exists | This demo uses no PVC to avoid this issue |
| DNS lookup fails | CoreDNS may not have updated yet | Wait 10 seconds and retry |

---

## Key Takeaways

1. StatefulSet pods get predictable names like `<statefulset-name>-<index>`, unlike Deployment pods
2. A headless Service (`clusterIP: None`) is required for direct Pod-to-Pod DNS resolution
3. StatefulSets deploy pods sequentially and delete them in reverse order
4. The `serviceName` field must match the headless Service name for stable DNS
5. StatefulSets are for stateful apps (databases); Deployments are for stateless apps
6. Persistent per-pod storage can be attached via `volumeClaimTemplates` for data durability

---

## Cleanup

```bash
kubectl delete -f .
```