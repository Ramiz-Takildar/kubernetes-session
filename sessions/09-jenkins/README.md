# Session 09 — Jenkins Demo

## What You Will Learn

This session ties together Namespace, Pod, and Service into a real-world example. Jenkins is a CI/CD server that requires multiple ports and a Service to expose its UI and agent port.

---

## Core Concepts

- Multi-port Services expose each port with a name for easy reference
- Namespace-scoped resources require `-n` flag or a context switch
- Real-world applications often combine multiple Kubernetes primitives
- The `dev` namespace must exist before applying this manifest (created in Session 01)

---

## Prerequisites

```bash
# Ensure the dev namespace exists from Session 01
kubectl get namespace dev
# If missing: kubectl apply -f sessions/01-namespace/namespace.yaml
```

---

## YAML Walkthrough

### jenkins.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: jenkins
  namespace: dev
  labels:
    app: jenkins
spec:
  containers:
  - name: jenkins
    image: jenkins/jenkins:lts
    ports:
    - containerPort: 8080
    - containerPort: 50000
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
    nodePort: 30080
  - name: agent
    port: 50000
    targetPort: 50000
    nodePort: 30500
```

| Field | Meaning |
|-------|---------|
| `metadata.namespace: dev` | Places this Pod in the `dev` namespace |
| `spec.ports[].name` | Named port for easy reference (used by Jenkins agents) |
| `spec.ports[].nodePort` | Fixed NodePort for the Jenkins web UI and agent port |
| `---` | YAML document separator — this file contains two resources |

---

## Step-by-Step Implementation

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

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get pod jenkins -n dev` | Pod is `Running` in the `dev` namespace |
| `kubectl get svc jenkins-service -n dev` | Service is `NodePort` with correct ports |
| `kubectl port-forward pod/jenkins 8080:8080 -n dev` | Jenkins web UI is accessible at localhost:8080 |
| `kubectl exec jenkins -n dev -- cat /var/jenkins_home/secrets/initialAdminPassword` | Initial admin password is available |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Pod stuck in `ContainerCreating` | Jenkins image is ~500MB, first pull may take 1-2 minutes | Wait and check progress with `kubectl describe pod jenkins -n dev` |
| Port-forward connection refused | Pod not yet `Ready` | Wait for STATUS = `Running` before starting port-forward |
| NodePort already in use | Port 30080 or 30500 is taken | Remove `nodePort` fields and let Kubernetes auto-assign |

---

## Key Takeaways

1. Multi-port Services expose each port with a **name** for identification
2. Namespace-scoped resources require the `-n` flag or a context switch
3. Real-world applications often combine **multiple Kubernetes primitives** (Pod + Service + Namespace)
4. Jenkins image is large — patience is required on first pull

---

## Cleanup

```bash
kubectl delete -f .
```