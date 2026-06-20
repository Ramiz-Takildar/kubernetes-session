# Session 04 — Secrets

## What You Will Learn

Secrets are the standard Kubernetes resource for managing sensitive data like passwords, API tokens, and TLS certificates. In this session you will create a Secret, understand the difference between base64 encoding and true encryption, and inject credentials securely into a running container.

> **Instructor Note:** The value `cGFzc3dvcmQxMjM=` is base64 for `password123`. You can encode/decode:
> ```bash
> echo -n 'password123' | base64          # encode
> echo 'cGFzc3dvcmQxMjM=' | base64 -d     # decode
> ```

---

## Core Concepts

- **Secrets** store sensitive data like passwords, tokens, and SSH keys, and are base64-encoded by default for transport and storage.
- Base64 encoding is **not encryption** — anyone with cluster access can decode the values, so additional protection is required for production workloads.
- For production clusters, enable **encryption at rest** via an EncryptionConfiguration so etcd stores Secret data encrypted.
- **`valueFrom.secretKeyRef`** is the secure way to inject credentials into containers as environment variables without exposing them in Pod specs or logs.
- Secrets must reside in the **same Namespace** as the Pod that references them, and they can also be mounted as files inside a volume for applications that read credentials from disk.
- Kubernetes has built-in Secret types such as **Opaque** for generic data, **tls** for certificates, and **docker-registry** for image pull credentials.
- Hardcoding passwords in YAML is a common anti-pattern; Secrets exist precisely to separate credentials from application configuration and version control.

---

## YAML Walkthrough

### secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: cGFzc3dvcmQxMjM=
```

### pod-secret.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-secret-pod
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-secret
          key: MYSQL_ROOT_PASSWORD
    ports:
    - containerPort: 3306
```

| Field | Meaning |
|-------|---------|
| `type: Opaque` | Generic secret type (default) |
| `data` | Base64-encoded key/value pairs |
| `env[].valueFrom.secretKeyRef.name` | Name of the Secret to read from |
| `env[].valueFrom.secretKeyRef.key` | Specific key within the Secret |

---

## Step-by-Step Implementation

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

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get secret mysql-secret` | Secret was created |
| `kubectl describe secret mysql-secret` | Secret exists but values are hidden from display |
| `kubectl get secret mysql-secret -o yaml` | Shows base64-encoded values |
| `kubectl exec mysql-secret-pod -- env \| grep MYSQL_ROOT_PASSWORD` | Env var is injected and decoded |
| `kubectl exec mysql-secret-pod -- mysql -uroot -ppassword123 -e "SELECT 1;"` | MySQL accepts the decoded password |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Secret value is wrong | Incorrect base64 encoding | Use `echo -n` to avoid encoding a trailing newline |
| Pod fails to start | Same as Session 03 — wait for MySQL initialization | `kubectl describe pod mysql-secret-pod` for details |

---

## Key Takeaways

1. Secrets are base64-encoded, **not encrypted** by default — do not rely on this for real security
2. For production, enable encryption at rest: `EncryptionConfiguration`
3. `valueFrom.secretKeyRef` is the secure way to inject credentials
4. The contrast between Session 03 (hardcoded password) and Session 04 (Secret) is the main teaching point
5. Secrets must live in the same Namespace as the consuming Pod
6. Mounting Secrets as volumes is safer when applications read credentials from files instead of env vars

---

## Cleanup

```bash
kubectl delete -f .
```