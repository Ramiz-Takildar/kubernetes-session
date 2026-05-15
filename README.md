# 🚀 Kubernetes Hands-On Practical Session

## 📘 Topics Covered

- Kubernetes Architecture
- Pods
- Namespaces
- Deployments
- StatefulSets
- Services
- ConfigMaps
- Secrets
- Persistent Volumes (PV)
- Persistent Volume Claims (PVC)

---

# 🛠️ Prerequisites

- Docker Installed
- kubectl Installed
- Minikube Installed

---

# 🚀 Start Cluster

```bash
minikube start
```

---

# 🔍 Verify Cluster

```bash
kubectl get nodes
kubectl cluster-info
```

---

# 📦 Apply YAML Files

## Pod

```bash
kubectl apply -f 01-pod/
```

## Namespace

```bash
kubectl apply -f 02-namespace/
```

## Deployment

```bash
kubectl apply -f 03-deployment/
```

## StatefulSet

```bash
kubectl apply -f 04-statefulset/
```

## Service

```bash
kubectl apply -f 05-service/
```

## ConfigMap

```bash
kubectl apply -f 06-configmap/
```

## Secret

```bash
kubectl apply -f 07-secret/
```

## Persistent Volume

```bash
kubectl apply -f 08-pv/
```

## Persistent Volume Claim

```bash
kubectl apply -f 09-pvc/
```

---

# 🔍 Verification Commands

```bash
kubectl get pods
kubectl get deployments
kubectl get svc
kubectl get configmaps
kubectl get secrets
kubectl get pv
kubectl get pvc
```

---

# 🧹 Cleanup

```bash
kubectl delete -f .
```

---

# 🎯 Learning Outcome

Participants will learn:

✅ Kubernetes YAML Basics  
✅ Pods & Deployments  
✅ Services & Networking  
✅ ConfigMaps & Secrets  
✅ Persistent Storage  
✅ Stateful Applications  

---

# 🚀 Happy Learning Kubernetes ☸️
