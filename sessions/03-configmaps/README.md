# Session 03 — ConfigMaps

## What You Will Learn

ConfigMaps decouple configuration from container images. Use them for non-sensitive data like database names, feature flags, or file contents.

---

## Core Concepts

- ConfigMaps store non-sensitive configuration as key/value pairs
- They can be injected into Pods as environment variables or mounted files
- `envFrom.configMapRef` injects all keys as environment variables at once
- ConfigMaps are NOT encrypted — never use them for passwords or secrets

---

## YAML Walkthrough

### configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  MYSQL_DATABASE: mydb
  MYSQL_USER: admin
```

### pod-configmap.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-configmap-pod
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    envFrom:
    - configMapRef:
        name: mysql-config
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: rootpass123
    ports:
    - containerPort: 3306
```

| Field | Meaning |
|-------|---------|
| `data` | Key/value pairs stored in the ConfigMap |
| `envFrom[].configMapRef.name` | Injects all ConfigMap keys as environment variables |
| `env[].value` | Inline env var (hardcoded — not for production!) |
| `env[].valueFrom.secretKeyRef` | Injects a specific key from a Secret (covered in Session 04) |

| Approach | Use Case |
|----------|----------|
| `envFrom.configMapRef` | Inject all ConfigMap keys as env vars |
| `env.valueFrom.configMapKeyRef` | Inject a single key |
| Volume mount | Mount ConfigMap data as files |

---

## Step-by-Step Implementation

```bash
# 1. Apply ConfigMap and Pod
cd sessions/03-configmaps
kubectl apply -f .

# 2. Verify ConfigMap exists
kubectl get configmap mysql-config
kubectl describe configmap mysql-config
# Look for the Data section with MYSQL_DATABASE and MYSQL_USER

# 3. Verify pod is running
kubectl get pod mysql-configmap-pod
# Wait for STATUS = Running (MySQL may take 20-30 seconds to start)

# 4. Check environment variables inside the pod
kubectl exec mysql-configmap-pod -- env | grep MYSQL
# Expected output includes:
#   MYSQL_DATABASE=mydb
#   MYSQL_USER=admin
#   MYSQL_ROOT_PASSWORD=rootpass123

# 5. Verify MySQL is actually running and accepting connections
kubectl exec mysql-configmap-pod -- mysql -uroot -prootpass123 -e "SHOW DATABASES;"
# Expected: mydb should appear in the list
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get configmap mysql-config` | ConfigMap was created |
| `kubectl describe configmap mysql-config` | Shows all key/value pairs in Data |
| `kubectl get pod mysql-configmap-pod` | Pod is `Running` |
| `kubectl exec mysql-configmap-pod -- env \| grep MYSQL` | ConfigMap keys are injected as env vars |
| `kubectl exec mysql-configmap-pod -- mysql -uroot -prootpass123 -e "SHOW DATABASES;"` | MySQL is running with injected config |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Pod stuck in `Pending` / `ImagePullBackOff` | Image pull failed | `kubectl describe pod mysql-configmap-pod \| grep Events -A 10` |
| MySQL connection fails | Pod still initializing | Wait 20-30 seconds and retry |

---

## Key Takeaways

1. ConfigMaps are for **non-sensitive** data only
2. `envFrom` is the quickest way to inject all key-value pairs
3. Never hardcode passwords in YAML — Session 04 shows the correct approach using Secrets

---

## Cleanup

```bash
kubectl delete -f .
```