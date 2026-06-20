# Session 01 — Namespace

## What You Will Learn

Namespaces are the primary mechanism for resource isolation and multi-tenancy in Kubernetes. In this session you will create a custom namespace, understand how resources are scoped within it, and learn to switch your kubectl context so commands target the right environment automatically.

---

## Core Concepts

- A **Namespace** is a virtual cluster backed by the same physical cluster, enabling multiple teams or projects to share infrastructure without naming collisions.
- Resources in one Namespace are **isolated by name** from other Namespaces, so two different teams can each deploy a service called "web" without conflict.
- Kubernetes creates a **`default`** Namespace automatically, and most clusters also include `kube-system` and `kube-public` for system-level resources.
- Use **`kubectl config set-context --current --namespace=<name>`** to switch your default namespace and avoid typing `-n` on every command.
- **ResourceQuotas** and **LimitRanges** can be applied per namespace to control how much CPU, memory, and storage a team can consume.
- Namespaces provide **logical isolation**, not physical security — network policies and RBAC are still required for true multi-tenant safety.
- Deleting a Namespace **cascades** and removes every resource inside it, making it a powerful but dangerous cleanup tool.

---

## YAML Walkthrough

### namespace.yaml

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

---

## Step-by-Step Implementation

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

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get namespaces` | The `dev` namespace exists |
| `kubectl describe namespace dev` | Namespace has correct name and no unexpected settings |
| `kubectl config view --minify \| grep namespace` | Current context is switched to `dev` |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `Namespace already exists` | You already ran this session | `kubectl delete namespace dev` then reapply, or just re-run `kubectl apply` (idempotent) |

---

## Key Takeaways

1. Namespaces provide **logical isolation** — not physical security
2. The `dev` Namespace created here is required for Session 09 (Jenkins)
3. Switching context with `kubectl config set-context --current --namespace=dev` eliminates the need for `-n dev` on every command
4. Two teams can use identical resource names as long as they live in different Namespaces
5. Deleting a Namespace deletes every resource inside it automatically
6. Production clusters should pair Namespaces with ResourceQuotas and RBAC for real multi-tenancy

---

## Cleanup

```bash
# Reset context back to default namespace
kubectl config set-context --current --namespace=default

# Delete the namespace
kubectl delete -f namespace.yaml
```

---

## What's Next?

Session 02 introduces **Pods** — the smallest deployable unit in Kubernetes. You will run containers inside Pods and place one of them in the `dev` namespace.