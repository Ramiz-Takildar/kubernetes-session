# Kubernetes Practical Training Sessions

A set of progressive, hands-on Kubernetes sessions designed for instructor-led delivery. Each session covers one core concept with working YAML manifests, detailed walkthroughs, and verification commands.

## Prerequisites

- A running Kubernetes cluster (minikube, kind, Docker Desktop, or kubeadm)
- `kubectl` configured and connected to your cluster
- Basic familiarity with YAML syntax
- For **Session 09 (Jenkins)** and **Session 12 (RBAC)**, `dev` namespace must exist (created in Session 01)
- For **Session 11 (Gateway API)**, the Gateway API CRDs must be installed (see session README for command)
- For **Session 15 (Network Policies)**, your CNI plugin must support NetworkPolicy enforcement (Calico, Cilium, or kind default)
- For **Session 18 (Helm)**, Helm CLI must be installed (`helm version`)

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
| 10 | **Taints and Tolerations** | `no-toleration-pod.yaml`, `toleration-pod.yaml` | Control Pod scheduling with node taints |
| 11 | **Gateway API** | `backend.yaml`, `gateway.yaml`, `httproute.yaml` | Modern traffic routing with Gateway and HTTPRoute |
| 12 | **RBAC & Security** | `serviceaccount.yaml`, `role.yaml`, `rolebinding.yaml` | Namespace-scoped access control |
| 13 | **Health Probes & Init Containers** | `pod-probes.yaml`, `pod-init.yaml` | Liveness, readiness, startup probes and init containers |
| 14 | **Affinity & Anti-affinity** | `pod-node-affinity.yaml`, `pod-no-affinity.yaml` | Node affinity scheduling rules |
| 15 | **Network Policies** | `pods.yaml`, `networkpolicy.yaml` | Namespace-scoped ingress traffic control |
| 16 | **Jobs & CronJobs** | `job.yaml`, `cronjob.yaml` | Batch and scheduled workloads |
| 17 | **DaemonSets** | `daemonset.yaml` | One pod per node for system services |
| 18 | **Helm** | `Chart.yaml`, `values.yaml`, `templates/*.yaml` | Package manager and templating for Kubernetes |

---

## How to Use This Guide

Each session has its own detailed `README.md` inside its folder. Start there for the full concept explanation, deep dive, and walkthrough.

**Quick workflow for any session:**

```bash
cd sessions/XX-<name>

# Read the session guide
cat README.md

# Apply all manifests
kubectl apply -f .

# Follow the verification steps in the session README
# ...

# Clean up when done
kubectl delete -f .
```

---

## Full Reset

To clean up everything across all sessions (in reverse order):

```bash
# Note: Session 18 uses Helm — use `helm uninstall my-nginx` instead of kubectl delete
# See Session 18 README for full cleanup
kubectl delete -f sessions/17-daemonsets/
kubectl delete -f sessions/16-jobs/
kubectl delete -f sessions/15-network-policy/
kubectl delete -f sessions/14-affinity/
kubectl delete -f sessions/13-probes/
kubectl delete -f sessions/12-rbac/
kubectl delete -f sessions/11-gateway-api/
kubectl delete -f sessions/10-taints-tolerations/
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
# 1. Verify all directories exist
for i in 01-namespace 02-pods 03-configmaps 04-secrets 05-storage \
         06-deployments 07-services 08-statefulsets 09-jenkins \
         10-taints-tolerations 11-gateway-api 12-rbac 13-probes \
         14-affinity 15-network-policy 16-jobs 17-daemonsets \
         18-helm; do
  test -d "sessions/$i" && echo "OK: $i" || echo "MISSING: $i"
done

# 2. Verify YAML syntax
for f in $(find sessions -name '*.yaml'); do
  echo "Checking $f..."
  kubectl apply --dry-run=client -f "$f" || echo "FAILED: $f"
done

# 3. Count total YAML files
find sessions -name '*.yaml' | wc -l
# Expected: 37

# 4. Verify per-session READMEs exist
for i in 01-namespace 02-pods 03-configmaps 04-secrets 05-storage \
         06-deployments 07-services 08-statefulsets 09-jenkins \
         10-taints-tolerations 11-gateway-api 12-rbac 13-probes \
         14-affinity 15-network-policy 16-jobs 17-daemonsets \
         18-helm; do
  test -f "sessions/$i/README.md" && echo "OK: $i/README.md" || echo "MISSING: $i/README.md"
done
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
| `kubectl rollout history deployment/<name>` | View deployment revision history |
| `kubectl config set-context --current --namespace=<ns>` | Switch default namespace |
| `kubectl wait --for=condition=Ready pod/<name> --timeout=60s` | Wait for a pod to be ready |
| `kubectl get <resource> -o yaml` | Output resource as YAML |
| `kubectl get endpoints <service>` | Show pods behind a Service |
| `kubectl run -it --rm debug --image=busybox:1.36 --restart=Never -- <cmd>` | Spin up a temporary debug pod |