# Fullstack Chart — Helm + ArgoCD GitOps on Kubernetes

![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white)
![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=flat&logo=argo&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![MongoDB Atlas](https://img.shields.io/badge/MongoDB_Atlas-47A248?style=flat&logo=mongodb&logoColor=white)

A production-grade Helm chart for a fullstack application (Express.js frontend + Flask backend) deployed on Kubernetes using ArgoCD GitOps — with automated sync, self-healing, and declarative infrastructure management.

---

## 📌 What It Does

A fullstack web application where:
- **Express.js frontend** serves the UI and handles user interactions
- **Flask backend** processes requests via Gunicorn WSGI server
- **MongoDB Atlas** stores all data in the cloud
- **Helm** packages all Kubernetes manifests with templated, environment-specific values
- **ArgoCD** continuously reconciles the cluster state against this Git repo

---

## 🏗️ Architecture

```
User → Express Frontend (NodePort :30001)
              ↓
       Flask Backend (ClusterIP :5000)
              ↓
        MongoDB Atlas (Cloud)

All managed by:
ArgoCD → watches this Git repo → syncs Kubernetes cluster
```

---

## 🔄 GitOps Flow

```
Git push (values.yaml change)
         ↓
ArgoCD detects drift (every 3 min or webhook)
         ↓
ArgoCD syncs cluster to match Git state
         ↓
Status: Healthy + Synced
```

**Self-healing:** Any manual `kubectl` changes are automatically reverted to match Git state within seconds.

---

## 🔐 Secrets Design — Deliberately Kept Outside GitOps

This chart does **not** manage the MongoDB credential through Helm or ArgoCD. The `MONGO_URI` Secret is created **once, manually, directly against the cluster** — it is never stored in Git, in `values.yaml`, or in any ArgoCD `Application` manifest.

**Why:** a value embedded in a Helm `--set` flag, an ArgoCD `Application`'s `helm.parameters` block, or a committed `values.yaml` is not treated as sensitive by Kubernetes or Git — it is stored and displayed in plaintext everywhere it travels (`kubectl get application -o yaml`, `helm get values`, shell history, permanent Git history). Provisioning it out-of-band avoids all of that, at the cost of one manual step per environment.

For genuinely production-scale secret management, tools like **Sealed Secrets** (encrypt the value, safely commit the ciphertext) or **External Secrets Operator** (pull live from AWS Secrets Manager / Vault at runtime) solve this more completely — a legitimate next step beyond this project's current scope.

---

## 📦 Helm Chart Structure

```
fullstack-chart/
├── Chart.yaml                    # Chart metadata
├── values.yaml                   # Default configuration values (no secrets)
├── argocd-app.yaml               # ArgoCD Application manifest — no embedded credentials
└── templates/
    ├── namespace.yaml
    ├── configmap.yaml
    ├── backend-deployment.yaml
    ├── backend-service.yaml
    ├── frontend-deployment.yaml
    └── frontend-service.yaml
```

Note: there is deliberately no `secret.yaml` template — see **Secrets Design** above.

---

## ⚙️ Configuration (values.yaml)

```yaml
namespace: fullstack-app

backend:
  image:
    repository: aakash0908/flask-backend
    tag: v1
  replicaCount: 2
  port: 5000

frontend:
  image:
    repository: aakash0908/express-frontend
    tag: v1
  replicaCount: 1
  port: 3000
  nodePort: 30001

config:
  mongoDb: "users_db"
  mongoCollection: "submissions"
  flaskBackend: "http://flask-backend-service:5000"
```

---

## 🔑 Prerequisite: Create the MongoDB Secret Manually

Run this once, before deploying via either method below:

```bash
kubectl create namespace fullstack-app
kubectl create secret generic app-secrets \
  --from-literal=MONGO_URI="<your-mongodb-atlas-uri>" \
  -n fullstack-app
```

Verify it exists (without exposing the value):
```bash
kubectl get secret app-secrets -n fullstack-app
```

---

## 🚀 Deploy with Helm

```bash
# Install
helm install fullstack-release . -n fullstack-app --create-namespace

# Upgrade
helm upgrade fullstack-release . -n fullstack-app

# Rollback to previous revision
helm rollback fullstack-release 1 -n fullstack-app

# View release history
helm history fullstack-release -n fullstack-app
```

---

## 🔁 Deploy with ArgoCD

```bash
# Apply the ArgoCD Application manifest (lives in this same repo)
kubectl apply -f argocd-app.yaml

# ArgoCD will automatically sync and deploy
# Monitor at: https://<node-ip>:<argocd-nodeport>
```

The `Application` manifest points at this repo's root and syncs everything in `templates/` — since the Secret is provisioned separately (see above), it is never touched, overwritten, or deleted by ArgoCD's automated `prune`/`selfHeal` behavior.

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Express.js |
| Backend | Python, Flask, Gunicorn |
| Database | MongoDB Atlas |
| Packaging | Helm |
| GitOps | ArgoCD |
| Orchestration | Kubernetes |
