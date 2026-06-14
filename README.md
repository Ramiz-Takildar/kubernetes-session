# Kubernetes Practical Training Sessions

A set of progressive, hands-on Kubernetes sessions designed for instructor-led delivery. Each session covers one core concept with working YAML manifests and verification commands.

## Prerequisites

- A running Kubernetes cluster (minikube, kind, Docker Desktop, or kubeadm)
- `kubectl` configured and connected to your cluster
- For **Session 09 (Jenkins)**, `dev` namespace must exist (created in Session 01)

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

## How to Use

Each session is self-contained. Apply, verify, and clean up before moving to the next session.

**Quick workflow for any session:**

```bash
cd sessions/XX-<name>

# Apply all manifests
kubectl apply -f .

# Verify (see session details below)
# ...

# Clean up when done
kubectl delete -f .
```

---

## Session 01 — Namespace

**Learning Objective:** Understand namespace isolation and how to organize resources.

```bash
cd sessions/01-namespace
kubectl apply -f namespace.yaml
kubectl get namespace dev
kubectl describe namespace dev
```

**Cleanup:**
```bash
kubectl delete -f namespace.yaml
```

---

## Session 02 — Pods

**Learning Objective:** Create basic pods and place them in specific namespaces.

```bash
cd sessions/02-pods
kubectl apply -f .
```

**Verify:**
```bash
# Pod in default namespace
kubectl get pod basic-nginx-pod
kubectl describe pod basic-nginx-pod

# Pod in dev namespace
kubectl get pod namespaced-nginx-pod -n dev
kubectl describe pod namespaced-nginx-pod -n dev
```

**Test the pod:**
```bash
kubectl port-forward basic-nginx-pod 8080:80
curl http://localhost:8080
```

**Cleanup:**
```bash
kubectl delete -f .
```

---

## Session 03 — ConfigMaps

**Learning Objective:** Decouple configuration from container images using ConfigMaps.

```bash
cd sessions/03-configmaps
kubectl apply -f .
```

**Verify:**
```bash
kubectl get configmap mysql-config
kubectl describe configmap mysql-config

# Check pod is running
kubectl get pod mysql-configmap-pod

# Verify environment variables inside the pod
kubectl exec mysql-configmap-pod -- env | grep MYSQL
```

**Cleanup:**
```bash
kubectl delete -f .
```

---

## Session 04 — Secrets

**Learning Objective:** Securely inject sensitive data into pods using Secrets.

```bash
cd sessions/04-secrets
kubectl apply -f .
```

**Verify:**
```bash
kubectl get secret mysql-secret
kubectl describe secret mysql-secret

# Check pod is running
kubectl get pod mysql-secret-pod

# Verify the secret is exposed as an environment variable
kubectl exec mysql-secret-pod -- env | grep MYSQL_ROOT_PASSWORD
```

**Cleanup:**
```bash
kubectl delete -f .
```

---

## Session 05 — Persistent Storage

**Learning Objective:** Use PersistentVolumes (PV) and PersistentVolumeClaims (PVC) for data persistence.

> **Note:** This session uses `hostPath` which works on single-node clusters (minikube, kind, Docker Desktop). For multi-node clusters, use a proper storage class.

```bash
cd sessions/05-storage
kubectl apply -f .
```

**Verify:**
```bash
kubectl get pv my-pv
kubectl get pvc my-pvc
kubectl get pod nginx-pvc-pod

# Check volume is mounted and file exists
kubectl exec nginx-pvc-pod -- cat /usr/share/nginx/html/index.html

# Port-forward and test
kubectl port-forward nginx-pvc-pod 8080:80
curl http://localhost:8080
```

**Cleanup:**
```bash
kubectl delete -f .
```

---

## Session 06 — Deployments

**Learning Objective:** Manage scalable, self-healing workloads with Deployments.

```bash
cd sessions/06-deployments
kubectl apply -f deployment.yaml
```

**Verify:**
```bash
# Check replicas
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx

# Watch rollout
kubectl rollout status deployment/nginx-deployment

# Scale up
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods -l app=nginx

# Scale down
kubectl scale deployment nginx-deployment --replicas=2
kubectl get pods -l app=nginx
```

**Cleanup:**
```bash
kubectl delete -f deployment.yaml
```

---

## Session 07 — Services

**Learning Objective:** Expose pods to network traffic using Services.

```bash
cd sessions/07-services
kubectl apply -f .
```

**Verify:**
```bash
kubectl get service nginx-service
kubectl describe service nginx-service

# Check endpoints
kubectl get endpoints nginx-service

# Get NodePort
kubectl get svc nginx-service -o jsonpath='{.spec.ports[0].nodePort}'

# Test via port-forward (works on any cluster)
kubectl port-forward svc/nginx-service 8080:80
curl http://localhost:8080
```

> **Try This:** In Session 06, the Deployment pods also have label `app: nginx`. Change the Service selector to `app: nginx` and watch the endpoints switch to the Deployment pods.

**Cleanup:**
```bash
kubectl delete -f .
```

---

## Session 08 — StatefulSets

**Learning Objective:** Understand workloads with stable hostname and network identity.

```bash
cd sessions/08-statefulsets
kubectl apply -f .
```

**Verify:**
```bash
# StatefulSet
kubectl get statefulset mysql-sts
kubectl get pods -l app=mysql

# Headless service provides DNS
kubectl get svc mysql-service
kubectl describe svc mysql-service

# Verify stable network identity
kubectl exec mysql-sts-0 -- hostname
# Expected: mysql-sts-0

# DNS resolution inside cluster
kubectl run -it --rm debug --image=busybox:1.36 --restart=Never -- nslookup mysql-sts-0.mysql-service
```

**Cleanup:**
```bash
kubectl delete -f .
```

---

## Session 09 — Jenkins Demo

**Learning Objective:** Combine Namespace + Pod + Service for a real-world application.

> **Prerequisite:** `dev` namespace must exist (Session 01).

```bash
cd sessions/09-jenkins
kubectl apply -f .
```

**Verify:**
```bash
kubectl get pod jenkins -n dev
kubectl get svc jenkins-service -n dev

# Port-forward to access Jenkins UI
kubectl port-forward pod/jenkins 8080:8080 -n dev
# Open http://localhost:8080 in browser

# Or get Jenkins initial admin password
kubectl exec jenkins -n dev -- cat /var/jenkins_home/secrets/initialAdminPassword
```

**Cleanup:**
```bash
kubectl delete -f .
```

---

## Full Reset

To clean up everything across all sessions:

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

## Quick Validation (Instructor Checklist)

Before delivering sessions, run this to verify all manifests are structurally valid:

```bash
for f in $(find sessions -name '*.yaml'); do
  echo "Checking $f..."
  kubectl apply --dry-run=client -f "$f" || echo "FAILED: $f"
done
```

All files should report `created (dry run)` or `configured (dry run)`.
