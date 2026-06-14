# Kubernetes Practical Training Sessions

A set of progressive, hands-on Kubernetes sessions designed for instructor-led delivery. Each session covers one core concept with working YAML manifests, detailed walkthroughs, and verification commands.

## Prerequisites

- A running Kubernetes cluster (minikube, kind, Docker Desktop, or kubeadm)
- `kubectl` configured and connected to your cluster
- Basic familiarity with YAML syntax
- For **Session 09 (Jenkins)**, `dev` namespace must exist (created in Session 01)

> **Tip for Instructors:** If using **kind** on Apple Silicon (M1/M2/M3), the `mysql:8.0` image included in Sessions 3, 4, and 8 supports ARM64 natively.

---

## Session Overview

| Session | Concept | Files | Description |
|---------|---------|-------|-------------|
| 01 | **Namespace** | `namespace.yaml` | Create and verify a namespace |
| 02 | **Pods** | `pod.yaml`, `pod-namespace.yaml` | Run pods in default and custom namespaces |
| 03 | **ConfigMaps** | `configmap.yaml`, `pod-configmap.yaml` | Inject non-sensitive config into pods |
| 04 | **Secrets** | `secret.yaml`, `pod-secret.yaml` | Inject sensitive data securely into pods |
| 05 | **Persistent Storage** | `pv.yaml`, `pvc.yaml`, `pod-pvc.yaml` | Use hostPath PV and PVC for persistence |
| 06 | **Deployments** | `deployment.yaml` | Declarative pod management with rolling updates |
| 07 | **Services** | `pod-service.yaml`, `service.yaml` | Expose pods via NodePort Service |
| 08 | **StatefulSets** | `statefulset-service.yaml`, `statefulset.yaml` | Workloads with stable network identity |
| 09 | **Jenkins Demo** | `jenkins.yaml` | Real-world combo: Pod + Service in a namespace |

---

## How to Use This Guide

Each session follows the same structure:

1. **Concept** — What are we learning and why?
2. **YAML Walkthrough** — Key fields and what they do
3. **Implementation Steps** — Apply and verify
4. **Verification Commands** — Prove it works
5. **Troubleshooting** — Common issues and fixes
6. **Cleanup** — Remove resources before the next session

**Quick workflow for any session:**

```bash
cd sessions/XX-<name>

# Apply all manifests
kubectl apply -f .

# Follow the verification steps in this guide
# ...

# Clean up when done
kubectl delete -f .
```

---

## Session 01 — Namespace

### What You Will Learn
Namespaces provide logical isolation within a single cluster. They let you divide cluster resources between multiple users, teams, or environments (dev, staging, prod).

### YAML Walkthrough

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

| Field | Meaning |
|-------|---------|
| `apiVersion: v1` | Core Kubernetes API group |
| `kind: Namespace` | Resource type |
| `metadata.name` | The namespace identifier |

### Step-by-Step Implementation

```bash
# 1. Navigate to the session folder
cd sessions/01-namespace

# 2. Apply the manifest
kubectl apply -f namespace.yaml
# Expected output: namespace/dev created

# 3. List all namespaces
kubectl get namespaces
# You should see 'dev' in the list

# 4. Describe the namespace to see its details
kubectl describe namespace dev
# Look for Labels, Annotations, and Resource Quotas

# 5. Switch your current context to use the dev namespace
kubectl config set-context --current --namespace=dev
# This saves you from typing -n dev on every command

# 6. Verify the context switch
kubectl config view --minify | grep namespace
```

### Troubleshooting
- **Namespace already exists:** `kubectl delete namespace dev` and reapply, or use `kubectl apply` which is idempotent.

### Cleanup

```bash
# Reset context back to default namespace
kubectl config set-context --current --namespace=default

# Delete the namespace
kubectl delete -f namespace.yaml
```

---

## Session 02 — Pods

### What You Will Learn
A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share networking and storage. In practice, you rarely create Pods directly—you use Deployments, Jobs, etc. But understanding Pods is essential.

### YAML Walkthrough

**pod.yaml** — Basic pod in the `default` namespace:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: basic-nginx-pod
  labels:
    app: nginx          # Labels are key/value pairs used by selectors
spec:
  containers:
  - name: nginx
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
```

**pod-namespace.yaml** — Pod placed in the `dev` namespace:

```yaml
metadata:
  name: namespaced-nginx-pod
  namespace: dev        # Explicitly places the pod in the dev namespace
```

### Step-by-Step Implementation

```bash
# 1. Apply both pods
cd sessions/02-pods
kubectl apply -f .

# 2. Verify pod in default namespace
kubectl get pod basic-nginx-pod
# Expected: STATUS = Running, READY = 1/1

# 3. Inspect the pod
kubectl describe pod basic-nginx-pod
# Look for: Node, Start Time, Container ID, Events

# 4. Verify pod in dev namespace
kubectl get pod namespaced-nginx-pod -n dev
# Expected: STATUS = Running

# 5. Test the pod with port-forward
kubectl port-forward basic-nginx-pod 8080:80
# Keep this running, then in another terminal:
curl http://localhost:8080
# Expected: "Welcome to nginx!"

# 6. View pod logs
kubectl logs basic-nginx-pod
# You should see nginx startup messages

# 7. Execute a command inside the pod
kubectl exec basic-nginx-pod -- ps aux
# Shows running processes inside the container
```

### Troubleshooting
- **Pod stuck in ContainerCreating:** Run `kubectl describe pod <name>` and check Events for image pull errors or volume mount issues.
- **Port-forward already in use:** Kill the previous port-forward process or use a different local port (e.g., `8081:80`).

### Key Takeaways
- No `namespace` field = `default` namespace
- Labels are critical for selectors (used by Deployments and Services later)
- `kubectl exec` lets you debug inside running containers

### Cleanup

```bash
kubectl delete -f .
```

---

## Session 03 — ConfigMaps

### What You Will Learn
ConfigMaps decouple configuration from container images. Use them for non-sensitive data like database names, feature flags, or file contents.

### YAML Walkthrough

**configmap.yaml:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  MYSQL_DATABASE: mydb        # Key-value pairs
  MYSQL_USER: admin
```

**pod-configmap.yaml:**

```yaml
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    envFrom:
    - configMapRef:
        name: mysql-config     # Injects all keys as environment variables
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: rootpass123       # Inline env var (NOT for production!)
```

| Approach | Use Case |
|----------|----------|
| `envFrom.configMapRef` | Inject all ConfigMap keys as env vars |
| `env.valueFrom.configMapKeyRef` | Inject a single key |
| Volume mount | Mount ConfigMap data as files |

### Step-by-Step Implementation

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

### Troubleshooting
- **Pod stuck in Pending/ImagePullBackOff:** `kubectl describe pod mysql-configmap-pod | grep Events -A 10`
- **MySQL connection fails:** The pod may still be initializing. Wait 20-30 seconds and retry.

### Key Takeaways
- ConfigMaps are for **non-sensitive** data only
- `envFrom` is the quickest way to inject all key-value pairs
- Never hardcode passwords in YAML (shown here for demo contrast with Secrets in Session 04)

### Cleanup

```bash
kubectl delete -f .
```

---

## Session 04 — Secrets

### What You Will Learn
Secrets store sensitive data like passwords, tokens, and SSH keys. Kubernetes encodes them as base64 at rest and can inject them into pods as environment variables or mounted files.

### YAML Walkthrough

**secret.yaml:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque                # Generic secret type
data:
  MYSQL_ROOT_PASSWORD: cGFzc3dvcmQxMjM=   # base64 encoded value
```

> **Instructor Note:** The value `cGFzc3dvcmQxMjM=` is base64 for `password123`. You can encode/decode:
> ```bash
> echo -n 'password123' | base64          # encode
> echo 'cGFzc3dvcmQxMjM=' | base64 -d     # decode
> ```

**pod-secret.yaml:**

```yaml
spec:
  containers:
  - name: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-secret    # Name of the Secret
          key: MYSQL_ROOT_PASSWORD
```

### Step-by-Step Implementation

```bash
# 1. Apply Secret and Pod
cd sessions/04-secrets
kubectl apply -f .

# 2. Verify Secret exists
kubectl get secret mysql-secret
kubectl describe secret mysql-secret
# Notice: Data shows 1 item, but values are hidden

# 3. View the raw Secret (shows base64 encoded data)
kubectl get secret mysql-secret -o yaml

# 4. Verify pod is running
kubectl get pod mysql-secret-pod

# 5. Verify the env var inside the pod
kubectl exec mysql-secret-pod -- env | grep MYSQL_ROOT_PASSWORD
# Expected: MYSQL_ROOT_PASSWORD=password123

# 6. Verify MySQL accepts connections using the secret password
kubectl exec mysql-secret-pod -- mysql -uroot -ppassword123 -e "SELECT 1;"
# Expected: 1 (confirms MySQL is running with the correct password)
```

### Troubleshooting
- **Secret value is wrong:** Double-check your base64 encoding. `echo -n` is required to avoid encoding a trailing newline.
- **Pod fails to start:** Same as Session 03 — wait for MySQL initialization.

### Key Takeaways
- Secrets are base64-encoded, **not encrypted** by default
- For production, enable encryption at rest: `EncryptionConfiguration`
- `valueFrom.secretKeyRef` is the secure way to inject credentials
- Compare Session 03 (hardcoded password) vs Session 04 (Secret) — this contrast is the main teaching point

### Cleanup

```bash
kubectl delete -f .
```

---

## Session 05 — Persistent Storage

### What You Will Learn
Containers are ephemeral — data is lost when they restart. PersistentVolumes (PV) and PersistentVolumeClaims (PVC) solve this by providing durable storage.

### YAML Walkthrough

**pv.yaml** — Physical storage provisioned by the cluster admin:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce          # Mounted by a single node at a time
  hostPath:
    path: /data              # Path on the host node
```

> **Note:** `hostPath` works only on single-node clusters (minikube, kind, Docker Desktop). Production clusters use NFS, EBS, Azure Disk, etc.

**pvc.yaml** — Request for storage by a user/app:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi          # Requests 500Mi of the 1Gi PV
```

**pod-pvc.yaml** — Pod that uses the PVC:

```yaml
spec:
  initContainers:
  - name: init-html
    image: busybox:1.36
    command: ['sh', '-c', 'echo "<h1>Hello from Persistent Storage!</h1>" > /usr/share/nginx/html/index.html']
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: nginx-storage
  containers:
  - name: nginx
    image: nginx:stable-alpine
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: nginx-storage
  volumes:
  - name: nginx-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

### Step-by-Step Implementation

```bash
# 1. Apply all storage manifests
cd sessions/05-storage
kubectl apply -f .

# 2. Verify PV status
kubectl get pv my-pv
# Expected: STATUS = Available (until claimed)

# 3. Verify PVC is bound
kubectl get pvc my-pvc
# Expected: STATUS = Bound, VOLUME = my-pv

# 4. Verify pod is running
kubectl get pod nginx-pvc-pod

# 5. Check that the initContainer wrote the file
kubectl exec nginx-pvc-pod -- cat /usr/share/nginx/html/index.html
# Expected: <h1>Hello from Persistent Storage!</h1>

# 6. Test the web server
kubectl port-forward nginx-pvc-pod 8080:80
# In another terminal:
curl http://localhost:8080
# Expected: Hello from Persistent Storage!

# 7. Prove persistence: delete the pod and recreate it
kubectl delete pod nginx-pvc-pod
kubectl apply -f pod-pvc.yaml
kubectl wait --for=condition=Ready pod/nginx-pvc-pod --timeout=60s

# 8. The file should still exist because the PVC survived
kubectl exec nginx-pvc-pod -- cat /usr/share/nginx/html/index.html
# Expected: Still shows "Hello from Persistent Storage!"
```

### Troubleshooting
- **PVC stuck in Pending:** The PV may not satisfy the claim. Check `kubectl describe pvc my-pvc` for events.
- **Pod stuck in Init:0/1:** The busybox image may still be pulling. Wait a moment and retry.

### Key Takeaways
- PV = cluster admin provisioned storage
- PVC = user request for storage
- `hostPath` is for training only — never use in production multi-node clusters
- InitContainers run before app containers and are perfect for setup tasks

### Cleanup

```bash
kubectl delete -f .
```

---

## Session 06 — Deployments

### What You Will Learn
Deployments provide declarative updates for Pods. They handle scaling, rolling updates, and self-healing (restarting failed Pods automatically).

### YAML Walkthrough

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3                          # Desired number of pods
  selector:
    matchLabels:
      app: nginx                       # Must match pod template labels
  template:
    metadata:
      labels:
        app: nginx                     # Labels on the pods
    spec:
      containers:
      - name: nginx
        image: nginx:stable-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"             # Guaranteed minimum
            cpu: "100m"
          limits:
            memory: "128Mi"            # Maximum allowed
            cpu: "200m"
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1                # Max pods down during update
      maxSurge: 1                      # Max extra pods during update
```

### Step-by-Step Implementation

```bash
# 1. Apply the Deployment
cd sessions/06-deployments
kubectl apply -f deployment.yaml

# 2. Verify the Deployment
kubectl get deployment nginx-deployment
# Expected: READY = 3/3

# 3. Verify the ReplicaSet was created
kubectl get replicasets -l app=nginx

# 4. List all pods managed by the Deployment
kubectl get pods -l app=nginx
# Expected: 3 pods, all Running

# 5. Watch the rollout
kubectl rollout status deployment/nginx-deployment
# Expected: deployment "nginx-deployment" successfully rolled out

# 6. Scale up to 5 replicas
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods -l app=nginx
# Expected: 5 pods

# 7. Scale down to 2 replicas
kubectl scale deployment nginx-deployment --replicas=2
kubectl get pods -l app=nginx
# Expected: 2 pods (Kubernetes deletes extras gracefully)

# 8. Verify self-healing: delete a pod manually
kubectl delete pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl get pods -l app=nginx
# Expected: A new pod is automatically created to maintain 2 replicas

# 9. View Deployment history
kubectl rollout history deployment/nginx-deployment
```

### Troubleshooting
- **Pods stuck in Pending:** Check node resources with `kubectl describe nodes`.
- **ImagePullBackOff:** Verify the image name and tag.

### Key Takeaways
- Deployments manage ReplicaSets which manage Pods
- Scaling is instant: `kubectl scale deployment <name> --replicas=N`
- If a Pod dies, the Deployment creates a replacement automatically
- Resource limits prevent a single pod from consuming all node resources

### Cleanup

```bash
kubectl delete -f deployment.yaml
```

---

## Session 07 — Services

### What You Will Learn
Services provide a stable network endpoint for a set of Pods. They handle load balancing across pod replicas and keep traffic flowing even as pods are created or destroyed.

### YAML Walkthrough

**pod-service.yaml** — Backend pod for the Service:

```yaml
metadata:
  labels:
    app: nginx-service-demo      # This label is what the Service selects
spec:
  containers:
  - name: nginx
    image: nginx:stable-alpine
```

**service.yaml** — The Service resource:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-service-demo       # Routes traffic to pods with this label
  ports:
  - protocol: TCP
    port: 80                      # Service port
    targetPort: 80                # Container port
  type: NodePort                  # Exposes service on each node's IP
```

| Service Type | Use Case |
|--------------|----------|
| `ClusterIP` | Internal cluster communication only |
| `NodePort` | Exposes service on a static port on each node |
| `LoadBalancer` | Exposes service externally (cloud providers) |
| `ExternalName` | Maps service to a DNS name |

### Step-by-Step Implementation

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

### Try This

After completing Session 06 (Deployment), scale the Deployment to 3 replicas. Then edit the service selector to `app: nginx` and watch the endpoints update:

```bash
kubectl get endpoints nginx-service -w
```

### Troubleshooting
- **No endpoints:** Check that pod labels match the Service selector exactly.
- **Connection refused:** The pod may still be starting. Wait and retry.

### Key Takeaways
- Services decouple frontend access from backend pod lifecycle
- `selector` is the glue that connects Services to Pods
- NodePort is the easiest way to test external access on local clusters

### Cleanup

```bash
kubectl delete -f .
```

---

## Session 08 — StatefulSets

### What You Will Learn
StatefulSets are for applications that require stable identity, stable storage, and ordered deployment (databases, message queues, etc.). Unlike Deployments, StatefulSets assign predictable names and network identities.

### YAML Walkthrough

**statefulset-service.yaml** — Headless Service (required for StatefulSet DNS):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  clusterIP: None              # Headless = no virtual IP, DNS returns pod IPs directly
  selector:
    app: mysql
  ports:
  - protocol: TCP
    port: 3306
```

**statefulset.yaml:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-sts
spec:
  serviceName: mysql-service   # Links to the headless service
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

### Step-by-Step Implementation

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

### Troubleshooting
- **Pod stuck in Pending:** If using `volumeClaimTemplates`, ensure a StorageClass exists. This demo uses no PVC to keep it universally working.
- **DNS lookup fails:** The debug pod may start faster than CoreDNS updates. Wait 10 seconds and retry.

### Key Takeaways
- StatefulSet pods get predictable names: `<statefulset-name>-<index>`
- Headless Service (`clusterIP: None`) enables direct pod-to-pod DNS resolution
- StatefulSets deploy pods sequentially (0, 1, 2...) and delete them in reverse order
- Compare to Deployment: StatefulSets are for stateful apps, Deployments for stateless apps

### Cleanup

```bash
kubectl delete -f .
```

---

## Session 09 — Jenkins Demo

### What You Will Learn
This session ties together Namespace, Pod, and Service into a real-world example. Jenkins is a CI/CD server that requires multiple ports and a Service to expose its UI.

### YAML Walkthrough

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: jenkins
  namespace: dev              # Explicitly in the dev namespace
  labels:
    app: jenkins
spec:
  containers:
  - name: jenkins
    image: jenkins/jenkins:lts
    ports:
    - containerPort: 8080     # Jenkins web UI
    - containerPort: 50000    # Jenkins agent port
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: dev
spec:
  type: NodePort
  selector:
    app: jenkins
  ports:
  - name: web
    port: 8080
    targetPort: 8080
    nodePort: 30080           # Fixed NodePort for the UI
  - name: agent
    port: 50000
    targetPort: 50000
    nodePort: 30500           # Fixed NodePort for agents
```

### Prerequisites

```bash
# Ensure the dev namespace exists from Session 01
kubectl get namespace dev
# If missing: kubectl apply -f sessions/01-namespace/namespace.yaml
```

### Step-by-Step Implementation

```bash
# 1. Apply the Jenkins manifests
cd sessions/09-jenkins
kubectl apply -f .

# 2. Verify the pod in the dev namespace
kubectl get pod jenkins -n dev
# Expected: STATUS = Running (Jenkins may take 30-60 seconds)

# 3. Verify the Service
kubectl get svc jenkins-service -n dev
# Expected: TYPE = NodePort, PORTS = 8080:30080/TCP, 50000:30500/TCP

# 4. Port-forward to access Jenkins locally
kubectl port-forward pod/jenkins 8080:8080 -n dev
# Open http://localhost:8080 in your browser

# 5. Get the initial admin password
kubectl exec jenkins -n dev -- cat /var/jenkins_home/secrets/initialAdminPassword
# Copy this 32-character password into the Jenkins setup wizard

# 6. (Optional) Get Jenkins logs
kubectl logs jenkins -n dev
```

### Troubleshooting
- **Pod stuck in ContainerCreating:** Jenkins image is ~500MB. First pull may take 1-2 minutes.
- **Port-forward connection refused:** Wait until the pod is fully Ready before starting port-forward.
- **NodePort already in use:** If port 30080 or 30500 is taken, remove the `nodePort` fields and let Kubernetes auto-assign.

### Key Takeaways
- Multi-port Services expose each port with a name
- Namespace-scoped resources require `-n` flag or context switch
- Real-world applications often combine multiple Kubernetes primitives

### Cleanup

```bash
kubectl delete -f .
```

---

## Full Reset

To clean up everything across all sessions (in reverse order):

```bash
kubectl delete -f sessions/09-jenkins/
kubectl delete -f sessions/08-statefulsets/
kubectl delete -f sessions/07-services/
kubectl delete -f sessions/06-deployments/
kubectl delete -f sessions/05-storage/
kubectl delete -f sessions/04-secrets/
kubectl delete -f sessions/03-configmaps/
kubectl delete -f sessions/02-pods/
kubectl delete -f sessions/01-namespace/
```

---

## Instructor Checklist

Before delivering sessions, run this validation:

```bash
# 1. Verify all YAML syntax is valid
for f in $(find sessions -name '*.yaml'); do
  echo "Checking $f..."
  kubectl apply --dry-run=client -f "$f" || echo "FAILED: $f"
done

# 2. Verify no duplicate resource names
grep -h '^  name:' sessions/*/*.yaml | sort | uniq -d
# Expected: no output (empty = no duplicates)

# 3. Count total YAML files
find sessions -name '*.yaml' | wc -l
# Expected: 16
```

---

## Quick Reference: kubectl Commands

| Command | Purpose |
|---------|---------|
| `kubectl apply -f file.yaml` | Create or update resources |
| `kubectl get <resource>` | List resources |
| `kubectl describe <resource> <name>` | Detailed info about a resource |
| `kubectl delete -f file.yaml` | Delete resources |
| `kubectl logs <pod>` | View container logs |
| `kubectl exec <pod> -- <cmd>` | Run a command inside a pod |
| `kubectl port-forward <pod/service> <local>:<remote>` | Forward a local port to a pod |
| `kubectl scale deployment <name> --replicas=N` | Change replica count |
| `kubectl rollout status deployment/<name>` | Watch deployment progress |
| `kubectl config set-context --current --namespace=<ns>` | Switch default namespace |
