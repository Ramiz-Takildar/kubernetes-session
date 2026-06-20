# Session 08 — StatefulSets

## What You Will Learn

StatefulSets are for applications that require stable identity, stable storage, and ordered deployment (databases, message queues, etc.). Unlike Deployments, StatefulSets assign predictable names and network identities.

---

## Core Concepts

- StatefulSet pods get **predictable names**: `<statefulset-name>-<index>` (e.g., `mysql-sts-0`)
- **Headless Service** (`clusterIP: None`) enables direct Pod-to-Pod DNS resolution
- StatefulSets deploy pods sequentially (0, 1, 2...) and delete them in reverse order
- StatefulSets are for **stateful** apps; Deployments are for **stateless** apps
- This session uses no PVC to keep it universally working across all cluster types

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

1. StatefulSet pods get **predictable names**: `<statefulset-name>-<index>`
2. Headless Service (`clusterIP: None`) enables direct Pod-to-Pod DNS resolution
3. StatefulSets deploy pods **sequentially** and delete them in **reverse order**
4. Compare to Deployment: StatefulSets for stateful apps (databases), Deployments for stateless apps

---

## Cleanup

```bash
kubectl delete -f .
```