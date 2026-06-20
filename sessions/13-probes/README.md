# Session 13 — Health Probes & Init Containers

## What You Will Learn

Health probes keep your applications healthy by automatically detecting failures and taking corrective action. Init containers run before the main container starts, enabling setup tasks like configuration, data pre-loading, or waiting for dependencies. Together, these mechanisms ensure that Pods start correctly and remain available throughout their lifecycle.

---

## Core Concepts

- **Liveness probes** detect when a container has entered a broken state (like a deadlock or memory leak) and trigger an automatic restart, reducing the need for manual intervention during failures
- **Readiness probes** determine whether a container is prepared to handle requests; when they fail, the Pod is temporarily removed from Service endpoints so traffic routes only to healthy instances
- **Startup probes** protect slow-booting applications by delaying liveness and readiness checks until initialization is complete, preventing premature restarts on long-starting services
- **Init containers** execute setup logic—such as waiting for a database, generating configs, or pre-loading data—before any main container begins, ensuring dependencies are satisfied first
- Init containers run **sequentially** in the order defined, and each must complete successfully before the next one starts, creating a predictable initialization pipeline
- If an init container fails, Kubernetes restarts the entire Pod and re-runs all init containers from the beginning, enforcing a clean and consistent startup state
- By default, init containers and regular containers do not share a filesystem, but mounting a shared **emptyDir volume** allows data produced during initialization to be consumed by the main application

---

## YAML Walkthrough

### pod-probes.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: nginx
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 2
      periodSeconds: 5
    startupProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 6
```

| Field | Meaning |
|-------|---------|
| `livenessProbe` | Restarts container if HTTP check fails |
| `livenessProbe.initialDelaySeconds` | Wait time before first check |
| `livenessProbe.periodSeconds` | How often to check (10s here) |
| `readinessProbe` | Removes pod from Service endpoints if check fails |
| `startupProbe` | Delays liveness/readiness checks until app is ready |
| `startupProbe.failureThreshold` | Number of failures before giving up (6 x 5s = 30s max startup) |

### pod-init.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: init-wait
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Initialization complete" > /workdir/init.txt']
    volumeMounts:
    - name: workdir
      mountPath: /workdir
  containers:
  - name: nginx
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /workdir
  volumes:
  - name: workdir
    emptyDir: {}
```

| Field | Meaning |
|-------|---------|
| `initContainers[]` | List of containers that run before main containers |
| `initContainers[].name` | Unique name among init containers in this pod |
| `initContainers[].command` | Command to execute during initialization |
| `volumes[]` | Shared storage accessible to init and regular containers |
| `volumeMounts` | Where to mount the shared volume in each container |
| `emptyDir: {}` | Empty volume type; lives as long as the Pod |

---

## Step-by-Step Implementation

```bash
# 1. Apply the Pod with health probes
cd sessions/13-probes
kubectl apply -f pod-probes.yaml

# 2. Verify the probes are configured
kubectl describe pod probe-demo | grep -A 5 "Liveness"
# Expected: Liveness probe configured, initial delay 5s

# 3. Apply the Pod with init container
kubectl apply -f pod-init.yaml

# 4. Check init container status (Pod will be Pending until init completes)
kubectl get pod init-demo
# Expected: STATUS shows "Init:0/1" initially, then "Running"

# 5. Check init container logs
kubectl logs init-demo -c init-wait
# Expected: (no output - command just writes to file)

# 6. Exec into the main container and verify init work exists
kubectl exec init-demo -c nginx -- cat /workdir/init.txt
# Expected: Initialization complete

# 7. Clean up when done
kubectl delete pod probe-demo init-demo
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl describe pod probe-demo \| grep -A 5 "Liveness"` | Liveness probe is configured |
| `kubectl describe pod probe-demo \| grep -A 5 "Readiness"` | Readiness probe is configured |
| `kubectl describe pod probe-demo \| grep -A 5 "Startup"` | Startup probe is configured |
| `kubectl get pod init-demo` | Pod is Running (init completed) |
| `kubectl logs init-demo -c init-wait` | Init container executed successfully |
| `kubectl exec init-demo -c nginx -- cat /workdir/init.txt` | Init container output is visible in main container |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Pod stuck in `Init:0/1` | Init container is failing | Check logs: `kubectl logs <pod> -c <init-container>` |
| Container keeps restarting | Liveness probe failing | Check if app serves HTTP on expected path/port |
| Pod not receiving traffic | Readiness probe failing | Verify the readiness endpoint returns 200 |
| Startup probe timeout | App takes too long to start | Increase `failureThreshold` or `periodSeconds` |

---

## Key Takeaways

1. Liveness probes restart unhealthy containers automatically, while readiness probes remove Pods from load balancing until they recover
2. Startup probes prevent premature restarts on slow-booting applications by delaying all other probes until initialization is complete
3. Init containers run sequentially before main containers and must all succeed before the application starts
4. Init containers and regular containers can share data by mounting the same **emptyDir volume**
5. Setting `initialDelaySeconds` correctly prevents probes from failing too early and causing unnecessary restarts

---

## Cleanup

```bash
kubectl delete -f pod-probes.yaml -f pod-init.yaml
```