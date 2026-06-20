# Session 12 — RBAC & Security

Role-Based Access Control (RBAC) restricts who can perform what actions on which resources. It follows the principle of least privilege: grant only the permissions needed, nothing more.

---

## What You Will Learn

Role-Based Access Control (RBAC) enforces the principle of least privilege by restricting who can perform which actions on specific resources. In this session, you will create a ServiceAccount, define a Role with granular permissions, bind them together with a RoleBinding, and verify the resulting access using `kubectl auth can-i`.

---

## Core Concepts

- **ServiceAccounts** provide identity for workloads running inside the cluster, allowing Pods to authenticate to the Kubernetes API server and receive scoped permissions rather than inheriting overly broad defaults.
- **Roles** define a set of permissions as a list of rules, each specifying an API group, resource type, and allowed verbs (such as `get`, `list`, `create`, `delete`), restricting access to only the operations a workload actually needs.
- **RoleBindings** grant a Role to a subject — typically a ServiceAccount, User, or Group — within a specific namespace, creating the actual authorization link between identity and permission.
- **ClusterRoles and ClusterRoleBindings** extend the same pattern cluster-wide, but Roles and RoleBindings are namespace-scoped by default, which is the safest starting point for most applications.
- The **`apiGroups` field** uses an empty string `""` to refer to the core Kubernetes API (pods, services, configmaps), while named groups like `apps` or `rbac.authorization.k8s.io` target specific extension APIs.
- **`kubectl auth can-i`** is the fastest way to verify RBAC configuration: it simulates a request as a specific subject and returns `yes` or `no`, making it invaluable for debugging permission denials before they affect running workloads.
- RBAC is additive only: there is no "deny" rule in standard Kubernetes RBAC, so the effective permissions of a subject are the union of all Roles and ClusterRoles bound to it, which is why restrictive default policies are essential.

---

## Prerequisites

- A `dev` namespace from [Session 01](/Users/ramiz/docker-ai/kubernetes-session/sessions/01-basics/README.md)

---

## YAML Walkthrough

### serviceaccount.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: demo-sa
  namespace: dev
```

| Field | Meaning |
|-------|---------|
| `metadata.name` | Name of the ServiceAccount |
| `metadata.namespace` | Namespace where the ServiceAccount lives |

---

### role.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

| Field | Meaning |
|-------|---------|
| `metadata.namespace` | Namespace where this Role applies |
| `rules[].apiGroups` | API group(s) the rule targets; `""` = core API (e.g., pods, services) |
| `rules[].resources` | Resource types the rule applies to |
| `rules[].verbs` | Allowed actions: get, list, watch, create, update, patch, delete |

---

### rolebinding.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: ServiceAccount
  name: demo-sa
  namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

| Field | Meaning |
|-------|---------|
| `subjects[]` | Who receives the permission (ServiceAccount, User, or Group) |
| `roleRef.kind` | Type of role to bind (Role or ClusterRole) |
| `roleRef.name` | Name of the Role to bind |
| `roleRef.apiGroup` | API group of the Role definition |

---

## Step-by-Step Implementation

```bash
# 1. Ensure the dev namespace exists
kubectl get namespace dev || kubectl create namespace dev

# 2. Apply the ServiceAccount, Role, and RoleBinding
cd sessions/12-rbac
kubectl apply -f serviceaccount.yaml
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml

# 3. Verify all three resources were created
kubectl get serviceaccount demo-sa -n dev
kubectl get role pod-reader -n dev
kubectl get rolebinding read-pods -n dev

# 4. (Optional) Describe the RoleBinding to confirm the subject and roleRef
kubectl describe rolebinding read-pods -n dev

# 5. From your local machine, verify the SA CAN list pods
kubectl auth can-i list pods --as=system:serviceaccount:dev:demo-sa -n dev
# Expected: yes

# 6. From your local machine, verify the SA CANNOT delete pods
kubectl auth can-i delete pods --as=system:serviceaccount:dev:demo-sa -n dev
# Expected: no

# 7. Clean up all RBAC resources
kubectl delete -f rolebinding.yaml
kubectl delete -f role.yaml
kubectl delete -f serviceaccount.yaml
kubectl delete pod permission-check -n dev
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get serviceaccount demo-sa -n dev` | ServiceAccount was created |
| `kubectl get role pod-reader -n dev` | Role was created in the dev namespace |
| `kubectl get rolebinding read-pods -n dev` | RoleBinding links the SA to the Role |
| `kubectl auth can-i list pods --as=system:serviceaccount:dev:demo-sa -n dev` | SA can list pods (yes) |
| `kubectl auth can-i delete pods --as=system:serviceaccount:dev:demo-sa -n dev` | SA cannot delete pods (no) |
| `kubectl describe rolebinding read-pods -n dev` | Shows subjects and role reference |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `403 Forbidden` when listing pods | RoleBinding not applied or wrong namespace | Verify RoleBinding exists in the correct namespace |
| SA still has old permissions | Role was modified after binding | Delete and re-create the RoleBinding |
| `kubectl auth can-i` returns unexpected result | `--as` subject format is wrong | Use `system:serviceaccount:<namespace>:<name>` |
| Pod cannot authenticate | ServiceAccount token not mounted | Check pod spec includes `serviceAccountName` |

---

## Key Takeaways

1. ServiceAccounts provide identity for workloads
2. Roles define what actions are allowed on which resources
3. RoleBindings connect ServiceAccounts to Roles
4. RBAC is namespace-scoped by default (Role/RoleBinding)
5. Use `kubectl auth can-i --as=system:serviceaccount:<ns>:<name>` to test permissions
6. The empty string `""` for apiGroups refers to the core Kubernetes API (pods, services, etc.)

---

## Cleanup

```bash
kubectl delete -f rolebinding.yaml
kubectl delete -f role.yaml
kubectl delete -f serviceaccount.yaml
kubectl delete pod permission-check -n dev
```