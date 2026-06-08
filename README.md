# GitOps with ArgoCD - Local Setup Guide

A complete guide to running a GitOps workflow locally using ArgoCD, Docker Desktop Kubernetes, and a React application.

---

## Repository Structure

```
cloud_apps/                  ← This repo (GitOps repo)
├── k8s/
│   ├── deployment.yaml      ← Kubernetes Deployment for React app
│   └── service.yaml         ← Kubernetes NodePort Service
└── argocd/
    └── customer-a.yaml      ← ArgoCD Application manifest

demo-app/                    ← React app repo (separate repo)
├── src/
├── Dockerfile
└── .github/
    └── workflows/
        └── ci.yml           ← GitHub Actions CI pipeline
```

---

## Architecture

```
[demo-app repo]                        [cloud_apps repo]
  - React source code      →             - k8s/deployment.yaml
  - Dockerfile            GitHub          - k8s/service.yaml
  - ci.yml                Actions
                          (build image
                          + update tag)
                                               ↑
                                          ArgoCD watches
                                               ↓
                                     [Docker Desktop K8s]
                                      2 pods running nginx
                                      serving the React app
```

**Flow:**
1. Push code to `demo-app` → GitHub Actions triggers
2. GitHub Actions builds Docker image → pushes to `ghcr.io/jainnpiy/demo`
3. GitHub Actions updates image tag in `k8s/deployment.yaml` in this repo
4. ArgoCD detects the change → rolls out new pods automatically

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) with Kubernetes enabled
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/)
- GitHub account (for GHCR image registry)

### Docker Desktop Kubernetes Setting

> **Important:** In Docker Desktop → Settings → Kubernetes, select **`kubeadm`** as the engine (NOT `kind`).
> With `kubeadm`, NodePort services are accessible directly on `localhost`.
> With `kind`, the K8s node runs inside a Docker container (`172.18.x.x` network) and NodePort does NOT reach `localhost`.

---

## Setup Steps

### Step 1 — Enable Kubernetes in Docker Desktop

1. Open Docker Desktop → Settings → Kubernetes
2. Check **Enable Kubernetes**
3. Select engine: **kubeadm**
4. Click **Apply & Restart**
5. Verify:
```bash
kubectl config use-context docker-desktop
kubectl get nodes
# Expected: docker-desktop   Ready   ...
```

### Step 2 — Create Namespaces

```bash
kubectl create namespace argocd
kubectl create namespace demo
```

### Step 3 — Install ArgoCD

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=180s

# Verify
kubectl get pods -n argocd
```

### Step 4 — Access ArgoCD UI

```bash
# Port-forward ArgoCD server (keep this terminal open)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the initial admin password:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

Open browser: `https://localhost:8080`
- Username: `admin`
- Password: output from command above

### Step 5 — Register the App in ArgoCD

Apply the ArgoCD Application manifest:
```bash
kubectl apply -f argocd/customer-a.yaml
```

This tells ArgoCD to:
- Watch this repo (`jainnpiy/cloud_apps`) on branch `HEAD`
- Sync manifests from the `k8s/` folder
- Deploy into the `demo` namespace
- Auto-sync and self-heal on any changes

### Step 6 — Verify Deployment

```bash
# Check pods are running
kubectl get pods -n demo

# Check service
kubectl get svc -n demo

# Expected output:
# NAME            TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
# react-app-svc   NodePort   10.x.x.x     <none>        80:30080/TCP   Xm
```

### Step 7 — Access the React App

With **kubeadm** engine:
```
http://localhost:30080
```

With **kind** engine (fallback):
```bash
kubectl port-forward svc/react-app-svc 30080:80 -n demo
# Then open: http://localhost:30080
```

---

## Kubernetes Manifests

### deployment.yaml

Deploys 2 replicas of the React app served via nginx.
- Image: `ghcr.io/jainnpiy/demo:latest` (auto-updated by CI)
- Container port: `80`

### service.yaml

Exposes the deployment via NodePort.
- Port: `80` → NodePort: `30080`
- Access at `http://localhost:30080`

---

## CI/CD Pipeline (demo-app repo)

The GitHub Actions workflow in `demo-app/.github/workflows/ci.yml`:

1. Triggers on push to `main`
2. Builds Docker image using the `Dockerfile`
3. Pushes image to `ghcr.io/jainnpiy/demo:latest` and `ghcr.io/jainnpiy/demo:<sha>`
4. Checks out this (`cloud_apps`) repo
5. Updates the image tag in `k8s/deployment.yaml` using `sed`
6. Commits and pushes the change back to this repo
7. ArgoCD picks up the change and rolls out the update

### Required Secret

In the `demo-app` GitHub repo → Settings → Secrets:
| Secret | Value |
|--------|-------|
| `GITOPS_PAT` | GitHub Personal Access Token with `repo` scope (to push to this repo) |

---

## Troubleshooting

### Pods not starting
```bash
kubectl describe pod -l app=react-app -n demo
kubectl logs -l app=react-app -n demo
```

### Image pull error
Make sure the GHCR package is public:
- Go to `https://github.com/jainnpiy?tab=packages`
- Click `demo` package → Package Settings → Change visibility to **Public**

### ArgoCD not syncing
```bash
# Force a manual sync
argocd app sync react-demo-cust-1

# Or via UI: click the Sync button in ArgoCD dashboard
```

### NodePort not accessible on localhost
```bash
# Check which Kubernetes engine Docker Desktop is using
# Docker Desktop → Settings → Kubernetes → should be kubeadm

# Fallback: use port-forward
kubectl port-forward svc/react-app-svc 30080:80 -n demo
```

---

## Quick Reference Commands

```bash
# Check everything at once
kubectl get all -n demo
kubectl get all -n argocd

# Watch pods in real time (useful during rollout)
kubectl get pods -n demo -w

# ArgoCD app status
argocd app get react-demo-cust-1

# Restart deployment (force re-pull image)
kubectl rollout restart deployment/react-app -n demo

# Check rollout status
kubectl rollout status deployment/react-app -n demo
```
