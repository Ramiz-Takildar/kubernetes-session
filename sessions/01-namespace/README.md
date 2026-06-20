# Session 01 — Namespace

## What You Will Learn

Namespaces provide logical isolation within a single cluster. They let you divide cluster resources between multiple users, teams, or environments (dev, staging, prod).

---

## Core Concepts

- A Namespace is a virtual cluster backed by the same physical cluster
- Resources in one Namespace are isolated from other Namespaces
- Kubernetes creates a `default` Namespace out of the box
- Use `kubectl config set-context --current --namespace=<name>` to switch contexts

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